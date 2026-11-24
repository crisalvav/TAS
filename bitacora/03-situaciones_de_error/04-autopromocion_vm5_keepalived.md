# La vm5 se autopromovió a base principal y se cortó la replicación (22/10/2026 - 23/10/2026)

**Servidores:** `tas05vm4` (10.33.199.45), `tas05vm5` (10.33.199.46) · **Servicios afectados:** MariaDB, keepalived (IP virtual 10.33.199.41) · **Impacto:** la replicación vm4 → vm5 quedó detenida unas 19 h y WordPress escribía solo en la vm5, mientras la vm4 quedaba con datos desactualizados (y los respaldos de la vm4 respaldaban esos datos viejos)

## Contexto

Se configuró replicación primario → réplica (vm4 → vm5) y keepalived en ambas VM para mover la IP virtual `10.33.199.41` a la vm5 si cae la vm4. En la vm5, `notify_master` ejecutaba `/usr/local/bin/promote.sh`:

```bash
#!/bin/bash
mariadb -e "STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;"
```

Ambas instancias tenían `state BACKUP` y `nopreempt`, con prioridad 100 en vm4 y 90 en vm5. El procedimiento indicaba arrancar primero keepalived en la vm4.

## Qué se observó

- Inicio: 22/10/2026 06:02:39 UTC (arranque de keepalived en la vm5).
- Cómo se detectó: el 23/10, al revisar `ip a` en la vm5 para instalar el panel de monitoreo, apareció la IP virtual como dirección secundaria:

  ```
  3: ens19: ...
      inet 10.33.199.46/29 brd 10.33.199.47 scope global ens19
      inet 10.33.199.41/29 scope global secondary ens19
  ```

## Diagnóstico

1. Se descartó split-brain: la vm4 no tenía la `.41`.
2. Según `systemctl status keepalived`, la vm5 arrancó a las 06:02:39 y la vm4 a las 06:13:33, 11 minutos después.
3. En la vm5, `read_only = OFF` y sin estado de réplica: `promote.sh` se había ejecutado.
4. Se comparó el contenido de ambas bases:

   ```bash
   sudo mariadb -e "SELECT MAX(ID), MAX(post_modified) FROM wordpress.wp_posts;"
   sudo mariadb -e "CHECKSUM TABLE wordpress.wp_posts, wordpress.wp_options, wordpress.wp_users,
     wordpress.wp_usermeta, wordpress.wp_comments; SHOW MASTER STATUS;"
   ```

   | | vm4 | vm5 |
   | --- | --- | --- |
   | `MAX(ID)` / `MAX(post_modified)` | 14 / 2026-10-09 16:23:05 | 14 / 2026-10-09 16:23:05 |
   | `wp_posts`, `wp_users`, `wp_usermeta`, `wp_comments` | iguales | iguales |
   | `wp_options` | 3410349304 | 3280527262 |
   | Binlog | `mysql-bin.000002` / 342 | `mysql-bin.000003` / 195545 |

   Solo difería `wp_options` (transients y cron de WordPress) y la posición del binlog de la vm5 era mucho mayor: las escrituras estaban llegando a la vm5. Ambos backends apuntaban a la `.41`, así que no había escrituras repartidas entre dos bases.

## Causa

Keepalived en la vm5 arrancó antes que en la vm4. Al no ver a ningún otro nodo, pasó a MASTER, tomó la IP virtual y ejecutó `promote.sh`. Cuando llegó la vm4, quedó en BACKUP y, por `nopreempt`, nunca recuperó la IP.

Fallo de diseño: `promote.sh` promovía la base en **cualquier** transición a MASTER, incluido el arranque, aunque la otra base estuviera viva.

## Solución

Se dejó la vm5 como base principal (tenía los datos más recientes) y se reconstruyó la vm4 como su réplica.

1. **vm5:** dump completo con la posición del binlog y copia a la vm4:

   ```bash
   sudo mariadb-dump --all-databases --single-transaction --master-data=2 --routines --events > /tmp/full-vm5.sql
   grep -m1 "CHANGE MASTER" /tmp/full-vm5.sql   # MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=195545
   scp /tmp/full-vm5.sql tas05vm4@10.33.199.45:/tmp/
   ```

2. **vm4:** importar y apuntar la réplica a la IP real de la vm5 (no a la virtual, para que no dependa de dónde esté la `.41`):

   ```bash
   sudo mariadb < /tmp/full-vm5.sql
   ```

   ```sql
   CHANGE MASTER TO MASTER_HOST='10.33.199.46', MASTER_USER='repl', MASTER_PASSWORD='<clave_repl>',
     MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=195545;
   START SLAVE;
   SHOW SLAVE STATUS\G   -- Slave_IO_Running: Yes, Slave_SQL_Running: Yes, Seconds_Behind_Master: 0
   ```

   ```bash
   sudo mariadb -e "SET GLOBAL read_only=1;"
   # y read_only = 1 en /etc/mysql/mariadb.conf.d/50-server.cnf, para que sobreviva reinicios
   ```

3. **`promote.sh` corregido en ambas VM:** solo promueve si la otra base no responde (`PEER` es la otra VM: `10.33.199.45` en la vm5, `10.33.199.46` en la vm4):

   ```bash
   #!/bin/bash
   PEER=10.33.199.45
   if mariadb-admin -h $PEER -urepl -p<clave_repl> ping --silent 2>/dev/null; then
     exit 0   # el otro nodo está vivo: no promover
   fi
   mariadb -e "STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;"
   ```

   ```bash
   sudo chmod 755 /usr/local/bin/promote.sh
   sudo chown root:root /usr/local/bin/promote.sh
   ```

4. **vm4:** se agregó `notify_master "/usr/local/bin/promote.sh"` dentro de `vrrp_instance DB` y se reinició keepalived (la vm4 queda en BACKUP, sin la `.41`).

Verificación en la vm5:

```bash
sudo mariadb -e "SHOW VARIABLES LIKE 'read_only'; SHOW SLAVE HOSTS;"
# read_only OFF y la vm4 listada (Server_id 1, Master_id 2)
```

- Fin: 23/10/2026 02:16 UTC (reinicio de keepalived en la vm4 con la réplica en Yes/Yes) · Duración aproximada: 20 h

## Estado resultante

| VM | Rol MariaDB | `read_only` | IP virtual `.41` |
| --- | --- | --- | --- |
| `tas05vm5` | Principal | OFF | Sí |
| `tas05vm4` | Réplica de la vm5 | ON | No |

## Lecciones

- Un script de promoción no puede depender solo del estado de keepalived; debe comprobar que la otra base esté realmente caída.
- El orden de arranque de keepalived importa; con `nopreempt`, el primero en arrancar se queda con la IP virtual.
- Faltaba una alerta de replicación: con `mysqld_exporter` en vm4 y vm5 se puede alertar si `Slave_IO_Running`/`Slave_SQL_Running` dejan de estar en `Yes`.
- **Pendiente:** el script de respaldos corre en la vm4; ahora que es réplica respalda los mismos datos que la principal, pero conviene documentarlo.
- **Pendiente:** la prueba de failover completa con el `promote.sh` corregido (ver `bitacora/04-pruebas/05-failover_base_de_datos.md`).
