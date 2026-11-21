# Respaldos y prueba de restauración (16/11/2026 - 21/11/2026)

**Servidor:** `tas05vm4` (10.33.199.45) como origen · **Copia separada:** `tas05vm5` (10.33.199.46)
**Objetivos:** respaldo diario, retención ≥ 30 días, RPO ≤ 24 h, RTO ≤ 60 min

## 16/11/2026 — Script de respaldo

`/usr/local/bin/respaldo_nodosur.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

FECHA=$(date +%F_%H%M)
DESTINO=/srv/respaldos
mkdir -p "$DESTINO"

# Base de datos (consistente sin bloquear tablas InnoDB)
mariadb-dump --single-transaction --routines --triggers wordpress | gzip > "$DESTINO/db_$FECHA.sql.gz"

# Archivos compartidos de WordPress
tar -czf "$DESTINO/wp-content_$FECHA.tar.gz" -C /srv/nfs wp-content

# Integridad
sha256sum "$DESTINO"/*_"$FECHA".* > "$DESTINO/sha256_$FECHA.txt"

# Retención de 30 días
find "$DESTINO" -type f -mtime +30 -delete
```

```bash
sudo chmod 700 /usr/local/bin/respaldo_nodosur.sh
sudo /usr/local/bin/respaldo_nodosur.sh
ls -lh /srv/respaldos
```

## 17/11/2026 — Copia en ubicación separada

Se crea un usuario `respaldo` en vm5 con llave SSH dedicada y se sincroniza la carpeta:

```bash
# en vm4, como root
ssh-keygen -t ed25519 -f /root/.ssh/respaldo -N ""
ssh-copy-id -i /root/.ssh/respaldo.pub respaldo@10.33.199.46
rsync -az --delete -e "ssh -i /root/.ssh/respaldo" /srv/respaldos/ respaldo@10.33.199.46:/srv/respaldos_vm4/
```

Se agrega la línea de `rsync` al final del script.

## 18/11/2026 — Programación diaria

```bash
sudo crontab -e
# respaldo diario a las 03:00
0 3 * * * /usr/local/bin/respaldo_nodosur.sh >> /var/log/respaldo_nodosur.log 2>&1
```

Las configuraciones de cada VM (`~/servicios/*`, sin `.env` ni llaves) quedan versionadas en el repositorio, por lo que no se respaldan aparte.

## 19/11/2026 — Verificación de integridad

```bash
cd /srv/respaldos
sha256sum -c sha256_<FECHA>.txt
gunzip -t db_<FECHA>.sql.gz && echo "dump OK"
tar -tzf wp-content_<FECHA>.tar.gz > /dev/null && echo "tar OK"
```

## 20/11/2026 — Prueba de restauración (ambiente controlado)

Se restaura en una base temporal para no tocar producción y se mide el tiempo:

```bash
time (
  sudo mariadb -e "CREATE DATABASE wordpress_restore;"
  gunzip -c /srv/respaldos/db_<FECHA>.sql.gz | sudo mariadb wordpress_restore
  mkdir -p /tmp/restore && tar -xzf /srv/respaldos/wp-content_<FECHA>.tar.gz -C /tmp/restore
)
sudo mariadb -e "SELECT COUNT(*) FROM wordpress_restore.wp_posts;"
sudo mariadb -e "DROP DATABASE wordpress_restore;"; rm -rf /tmp/restore
```

Procedimiento de restauración real (runbook):

1. Detener ambos backends (`docker compose stop` en vm2 y vm3).
2. Restaurar la base `wordpress` desde el último dump.
3. Restaurar `/srv/nfs/wp-content` desde el último `.tar.gz`.
4. Levantar los backends y verificar `https://www.nodosur.tas`.

## 21/11/2026 — Registro de resultados

- Se registra el tiempo total de restauración y se compara con el RTO de 60 min.
- La pérdida máxima de datos corresponde al intervalo entre respaldos (24 h), cumpliendo el RPO.
- **Pendiente:** agregar alerta en Prometheus si el archivo de respaldo del día no existe (por ejemplo con el textfile collector de node_exporter).
