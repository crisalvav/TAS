# Prueba: replicación de MariaDB vm4 → vm5 (21/10/2026)

**Servidores:** `tas05vm4` (10.33.199.45, principal en ese momento), `tas05vm5` (10.33.199.46, réplica) · **Versión:** MariaDB 10.11.14 en ambas · **Objetivo:** que todo cambio en la principal llegue a la réplica

## Configuración probada

`/etc/mysql/mariadb.conf.d/50-server.cnf`:

| Parámetro | vm4 | vm5 |
| --- | --- | --- |
| `bind-address` | 0.0.0.0 | 0.0.0.0 |
| `server-id` | 1 | 2 |
| `log_bin` | `/var/log/mysql/mysql-bin.log` | `/var/log/mysql/mysql-bin.log` |
| `relay_log` | — | `/var/log/mysql/relay-bin` |
| `binlog_format` | ROW | ROW |
| `read_only` | — | 1 |

Usuario de replicación: `repl@10.33.199.%` con `REPLICATION SLAVE`. La réplica se inicializó con un dump `--all-databases --master-data=2` (posición `mysql-bin.000001` / 789). El 3306 se abrió en ufw para la red interna.

## Procedimiento

1. Estado de la réplica en la vm5:

   ```sql
   SHOW SLAVE STATUS\G
   ```

2. Cambio en la principal y comprobación en la réplica:

   ```bash
   sudo mariadb -e "CREATE DATABASE prueba_repl;"   # vm4
   sudo mariadb -e "SHOW DATABASES;"                # vm5: debe aparecer prueba_repl
   sudo mariadb -e "DROP DATABASE prueba_repl;"     # vm4 (se replica solo)
   ```

## Resultado

```
Slave_IO_State: Waiting for master to send event
Master_Host: 10.33.199.45
Master_Log_File: mysql-bin.000001
Read_Master_Log_Pos: 789
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Last_Errno: 0
Seconds_Behind_Master: 0
```

Réplica funcionando, sin errores ni retraso ✅.

## Segunda verificación (23/10/2026, roles invertidos)

Tras el incidente de `bitacora/03-situaciones_de_error/04-autopromocion_vm5_keepalived.md`, la vm5 quedó como principal y la vm4 como réplica (`mysql-bin.000003` / 195545):

```
Master_Host: 10.33.199.46
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
Master_Server_Id: 2
```

```bash
# vm5
sudo mariadb -e "SHOW VARIABLES LIKE 'read_only'; SHOW SLAVE HOSTS;"
# read_only OFF · Server_id 1, Host 10.33.199.45, Master_id 2
```

**Pendiente:** registrar la prueba en vivo publicando una entrada en WordPress y buscándola en la vm4:

```bash
sudo mariadb -e "SELECT ID, post_title, post_modified FROM wordpress.wp_posts ORDER BY ID DESC LIMIT 2;"
```

## Observaciones

- La replicación es asíncrona: si la principal cae de golpe, las últimas transacciones pueden no haber llegado.
