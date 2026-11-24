# Prueba: respaldo de la base y copia a la vm5 (21/10/2026)

**Servidores:** `tas05vm4` (origen, ejecuta el script), `tas05vm5` (destino) · **Objetivo:** que exista un respaldo periódico de la base y de los archivos de WordPress fuera de la vm4

## Configuración probada

`/usr/local/bin/backup-db.sh` en la vm4, ejecutado por `/etc/cron.d/backup-db` cada 6 horas (`0 */6 * * * root`):

```bash
#!/bin/bash
F=/var/backups/mariadb/wordpress_$(date +%F_%H%M).sql.gz
mariadb-dump --single-transaction --routines --events --triggers --databases wordpress | gzip > "$F"
find /var/backups/mariadb -name '*.sql.gz' -mtime +7 -delete
rsync -a --delete /var/backups/mariadb/ tas05vm5@10.33.199.46:respaldos/db/
rsync -rlt --delete /srv/wp-data/ tas05vm5@10.33.199.46:respaldos/wp-data/
```

El envío usa una llave SSH ed25519 de root en la vm4, copiada a `tas05vm5@10.33.199.46`, para que el cron no pida contraseña.

## Procedimiento

```bash
# vm4
sudo /usr/local/bin/backup-db.sh
ls -lh /var/backups/mariadb
cat /etc/cron.d/backup-db
sudo ssh tas05vm5@10.33.199.46 "ls -lh ~/respaldos/db; ls ~/respaldos/wp-data"
```

## Resultado

```
# vm4: /var/backups/mariadb
-rw-r--r-- 1 root root 260K Oct 21 14:14 wordpress_2026-10-21_1414.sql.gz
0 */6 * * * root /usr/local/bin/backup-db.sh

# vm5: ~/respaldos/db
-rw-r--r-- 1 tas05vm5 tas05vm5 260K Oct 21 14:14 wordpress_2026-10-21_1414.sql.gz
-rw-r--r-- 1 tas05vm5 tas05vm5 260K Oct 21 14:32 wordpress_2026-10-21_1432.sql.gz

# vm5: ~/respaldos/wp-data
index.php  wp-admin  wp-config.php  wp-content  wp-includes  ...
```

Respaldo local, programación y copia a la vm5 funcionando ✅.

## Observaciones

- Retención de 7 días y frecuencia de 6 h (RPO ≈ 6 h). El informe propone 30 días de retención.
- `rsync -rlt` no conserva el dueño: al restaurar los archivos hay que ejecutar `chown -R 33:33`.
- **Pendiente:** prueba de restauración desde `~/respaldos` en la vm5 y medición del tiempo frente al RTO.
