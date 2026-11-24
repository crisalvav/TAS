# Instalación de la base de datos MariaDB en `tas05vm4` y réplica en `tas05vm5` (14/09/2026 - 23/10/2026)

**Servidores:** `tas05vm4` (10.33.199.45), `tas05vm5` (10.33.199.46) · **Servicio:** MariaDB 10.11 instalado directamente en el sistema (sin contenedor) · **Puerto:** 3306 · **IP virtual de la base:** 10.33.199.41

> Se decidió **no** dockerizar la base de datos para tener acceso directo al almacenamiento, simplificar respaldos y evitar una capa extra en el servicio más crítico (ver `decisiones/02-dockerizacion.md`).

## 14/09/2026 — Instalación en `tas05vm4`

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
systemctl status mariadb
mariadb --version
```

## 15/09/2026 — Endurecimiento inicial

1. Se ejecuta el asistente de seguridad: contraseña de root, se eliminan usuarios anónimos, se deshabilita el login remoto de root y se borra la base `test`:

   ```bash
   sudo mariadb-secure-installation
   ```

2. Por defecto MariaDB solo escucha en `127.0.0.1`. Se cambia para que escuche en la IP privada, editando `/etc/mysql/mariadb.conf.d/50-server.cnf`:

   ```ini
   [mysqld]
   bind-address = 10.33.199.45
   character-set-server = utf8mb4
   collation-server = utf8mb4_general_ci
   ```

   ```bash
   sudo systemctl restart mariadb
   sudo ss -lntp | grep 3306
   ```

## 16/09/2026 — Base de datos y usuario de WordPress

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4;
CREATE USER 'wp_user'@'10.33.199.43' IDENTIFIED BY '<contraseña_wp>';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.33.199.43';
FLUSH PRIVILEGES;
```

El usuario quedó creado solo para la vm2. Al preparar el failover se detectó y se amplió a toda la red interna (ver 21/10 y `bitacora/03-situaciones_de_error/03-wp_user_restringido_a_una_ip.md`).

_18/09 y 19/09 feriados (Fiestas Patrias), sin avance._

## 21/09/2026 — Pruebas de conexión

Desde `tas05vm2`:

```bash
sudo apt install -y mariadb-client
mariadb -h 10.33.199.45 -u wp_user -p -e "SHOW DATABASES;"
```

## 22/09/2026 — Persistencia ante reinicio

```bash
sudo mariadb -e "CREATE TABLE wordpress.prueba (id INT); INSERT INTO wordpress.prueba VALUES (1);"
sudo reboot
# tras el reinicio
sudo mariadb -e "SELECT * FROM wordpress.prueba; DROP TABLE wordpress.prueba;"
```

## 21/10/2026 — Replicación `tas05vm4` → `tas05vm5`

Para que exista una segunda base con los mismos datos se configura replicación primario → réplica. Galera (multi-master) se descartó porque necesita al menos 3 nodos para tener quórum.

1. **vm4**, sección `[mysqld]` de `50-server.cnf`:

   ```ini
   bind-address     = 0.0.0.0
   server-id        = 1
   log_bin          = /var/log/mysql/mysql-bin.log
   binlog_format    = ROW
   expire_logs_days = 10
   ```

   `bind-address = 0.0.0.0` es necesario para que la base acepte conexiones en la IP virtual cuando la tenga. El acceso se limita con el firewall:

   ```bash
   sudo mkdir -p /var/log/mysql && sudo chown mysql:mysql /var/log/mysql
   sudo ufw allow from 10.33.199.0/24 to any port 3306
   sudo systemctl restart mariadb
   sudo mariadb -e "SHOW VARIABLES LIKE 'log_bin'; SHOW MASTER STATUS;"
   ```

2. **vm4**, usuario de replicación y dump inicial:

   ```bash
   sudo mariadb -e "CREATE USER 'repl'@'10.33.199.%' IDENTIFIED BY '<clave_repl>';
     GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.33.199.%'; FLUSH PRIVILEGES;"
   sudo mariadb-dump --all-databases --single-transaction --master-data=2 --routines --events > /tmp/full.sql
   scp /tmp/full.sql tas05vm5@10.33.195.247:/tmp/
   ```

3. **vm5**, instalación (misma versión 10.11) y `50-server.cnf`:

   ```bash
   sudo apt update && sudo apt install -y mariadb-server
   ```

   ```ini
   bind-address  = 0.0.0.0
   server-id     = 2
   log_bin       = /var/log/mysql/mysql-bin.log
   relay_log     = /var/log/mysql/relay-bin
   binlog_format = ROW
   read_only     = 1
   ```

   ```bash
   sudo mkdir -p /var/log/mysql && sudo chown mysql:mysql /var/log/mysql
   sudo ufw allow from 10.33.199.0/24 to any port 3306
   sudo systemctl restart mariadb
   sudo mariadb < /tmp/full.sql
   grep -m1 "CHANGE MASTER" /tmp/full.sql   # MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=789
   ```

   ```sql
   CHANGE MASTER TO MASTER_HOST='10.33.199.45', MASTER_USER='repl', MASTER_PASSWORD='<clave_repl>',
     MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=789;
   START SLAVE;
   SHOW SLAVE STATUS\G
   ```

   Resultado y prueba en `bitacora/04-pruebas/03-replicacion_mariadb.md`.

## 21/10/2026 - 22/10/2026 — `wp_user` para toda la red interna

```bash
sudo mariadb -e "CREATE USER IF NOT EXISTS 'wp_user'@'10.33.199.%' IDENTIFIED BY '<contraseña_wp>';
  GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.33.199.%'; FLUSH PRIVILEGES;"
```

## 22/10/2026 — IP virtual con keepalived

La red interna es `10.33.199.40/29`; la única IP libre era `10.33.199.41` (confirmado con `arping`, ver `bitacora/04-pruebas/05-failover_base_de_datos.md`).

1. En vm4 y vm5:

   ```bash
   sudo apt install -y keepalived
   sudo ufw allow in on ens19 from 10.33.199.40/29 to 224.0.0.18   # tráfico VRRP
   ```

2. `/etc/keepalived/keepalived.conf` (vm4 con `priority 100`, vm5 con `priority 90`):

   ```
   global_defs {
     script_user root
     enable_script_security
   }
   vrrp_script chk_db {
     script "/usr/bin/mariadb-admin ping --silent"
     interval 2
     fall 2
     rise 2
   }
   vrrp_instance DB {
     state BACKUP
     interface ens19
     virtual_router_id 51
     priority 100
     nopreempt
     advert_int 1
     authentication {
       auth_type PASS
       auth_pass <clave_vrrp>
     }
     virtual_ipaddress {
       10.33.199.41/29
     }
     track_script {
       chk_db
     }
     notify_master "/usr/local/bin/promote.sh"
   }
   ```

3. `/usr/local/bin/promote.sh` (root, `755`) promueve la réplica solo si la otra base no responde. `PEER` es la IP de la otra VM (`10.33.199.45` en vm5, `10.33.199.46` en vm4):

   ```bash
   #!/bin/bash
   PEER=10.33.199.45
   if mariadb-admin -h $PEER -urepl -p<clave_repl> ping --silent 2>/dev/null; then
     exit 0   # el otro nodo está vivo: no promover
   fi
   mariadb -e "STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;"
   ```

   La primera versión no tenía la comprobación de `PEER` y provocó la autopromoción de la vm5 (ver `bitacora/03-situaciones_de_error/04-autopromocion_vm5_keepalived.md`).

4. vm2 y vm3 apuntan a la IP virtual: `WORDPRESS_DB_HOST: 10.33.199.41:3306`.

## 23/10/2026 — Roles invertidos tras el incidente

Tras el incidente, la vm5 quedó como principal y la vm4 se reconstruyó como su réplica con un dump nuevo (`mysql-bin.000003` / 195545), apuntando a la IP real de la vm5:

```sql
CHANGE MASTER TO MASTER_HOST='10.33.199.46', MASTER_USER='repl', MASTER_PASSWORD='<clave_repl>',
  MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=195545;
START SLAVE;
```

En vm4 se agregó `read_only = 1` en `50-server.cnf`.

## Resultado

| VM | Rol | `read_only` | IP virtual `.41` |
| --- | --- | --- | --- |
| `tas05vm5` | Principal | OFF | Sí |
| `tas05vm4` | Réplica de la vm5 | ON | No |

- La base sigue disponible si cae una de las dos VM; la IP virtual pasa a la otra y `promote.sh` la deja en escritura.
- La replicación es asíncrona: si la principal cae de golpe, las últimas transacciones pueden perderse.
- La vuelta atrás es manual: la base que cayó se reconstruye como réplica de la nueva principal.
- **Pendiente:** registrar la prueba de failover completa y alertar si la replicación se corta (`mysqld_exporter`).
