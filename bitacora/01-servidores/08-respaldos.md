# Respaldos y prueba de restauración (21/10/2026 - 23/10/2026)

**Servidor de respaldos:** `tas05vm5` (10.33.199.46) · **Origen de los datos:** `tas05vm4` (10.33.199.45)
**Objetivos:** respaldo diario, retención ≥ 30 días, RPO ≤ 24 h, RTO ≤ 60 min

Los respaldos se ejecutan y se guardan en vm5, de modo que quedan en una VM distinta a la de la base de datos. vm5 se conecta por SSH a vm4 para generar el dump y empaquetar `wp-content`, sin abrir el puerto 3306 a vm5.

## 21/10/2026 — Acceso de vm5 a vm4

En vm4 se crea un usuario `respaldo` y un usuario de MariaDB con autenticación por socket y solo los permisos de lectura que necesita el dump:

```bash
# en vm4
sudo adduser --disabled-password --gecos "" respaldo
sudo mariadb -e "CREATE USER 'respaldo'@'localhost' IDENTIFIED VIA unix_socket;
  GRANT SELECT, LOCK TABLES, SHOW VIEW, TRIGGER, EVENT ON wordpress.* TO 'respaldo'@'localhost';"
# permitir solo el tar de wp-content como root
echo 'respaldo ALL=(root) NOPASSWD: /usr/bin/tar -czf - -C /srv/nfs wp-content' | sudo tee /etc/sudoers.d/respaldo
sudo chmod 440 /etc/sudoers.d/respaldo
```

En vm5 se genera una llave SSH dedicada y se copia a vm4:

```bash
# en vm5, como root
ssh-keygen -t ed25519 -f /root/.ssh/respaldo -N ""
ssh-copy-id -i /root/.ssh/respaldo.pub respaldo@10.33.199.45
```

## 21/10/2026 — Script de respaldo

`/usr/local/bin/respaldo_nodosur.sh` en vm5:

```bash
#!/usr/bin/env bash
set -euo pipefail

FECHA=$(date +%F_%H%M)
DESTINO=/srv/respaldos
ORIGEN="respaldo@10.33.199.45"
SSH="ssh -i /root/.ssh/respaldo $ORIGEN"
mkdir -p "$DESTINO"

# Base de datos (consistente sin bloquear tablas InnoDB)
$SSH "mariadb-dump --single-transaction --routines --triggers wordpress" | gzip > "$DESTINO/db_$FECHA.sql.gz"

# Archivos compartidos de WordPress
$SSH "sudo /usr/bin/tar -czf - -C /srv/nfs wp-content" > "$DESTINO/wp-content_$FECHA.tar.gz"

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

## 21/10/2026 — Programación diaria

```bash
# en vm5
sudo crontab -e
# respaldo diario a las 03:00
0 3 * * * /usr/local/bin/respaldo_nodosur.sh >> /var/log/respaldo_nodosur.log 2>&1
```

Las configuraciones de cada VM (`~/servicios/*`, sin `.env` ni llaves) quedan versionadas en el repositorio, por lo que no se respaldan aparte.

## 22/10/2026 — Verificación de integridad

```bash
# en vm5
cd /srv/respaldos
sha256sum -c sha256_<FECHA>.txt
gunzip -t db_<FECHA>.sql.gz && echo "dump OK"
tar -tzf wp-content_<FECHA>.tar.gz > /dev/null && echo "tar OK"
```

## 22/10/2026 — Prueba de restauración (ambiente controlado)

Desde vm5 se restaura en una base temporal de vm4 para no tocar producción y se mide el tiempo (`<admin>` es la cuenta de administración de vm4):

```bash
# en vm5
time (
  ssh <admin>@10.33.199.45 'sudo mariadb -e "CREATE DATABASE wordpress_restore;"'
  gunzip -c /srv/respaldos/db_<FECHA>.sql.gz | ssh <admin>@10.33.199.45 'sudo mariadb wordpress_restore'
  mkdir -p /tmp/restore && tar -xzf /srv/respaldos/wp-content_<FECHA>.tar.gz -C /tmp/restore
)
ssh <admin>@10.33.199.45 'sudo mariadb -e "SELECT COUNT(*) FROM wordpress_restore.wp_posts;"'
ssh <admin>@10.33.199.45 'sudo mariadb -e "DROP DATABASE wordpress_restore;"'; rm -rf /tmp/restore
```

Procedimiento de restauración real (runbook):

1. Detener ambos backends (`docker compose stop` en vm2 y vm3).
2. Restaurar la base `wordpress` en vm4 desde el último dump guardado en vm5.
3. Restaurar `/srv/nfs/wp-content` en vm4 desde el último `.tar.gz` guardado en vm5.
4. Levantar los backends y verificar `https://www.nodosur.tas`.

## 23/10/2026 — Registro de resultados

- Se registra el tiempo total de restauración y se compara con el RTO de 60 min.
- La pérdida máxima de datos corresponde al intervalo entre respaldos (24 h), cumpliendo el RPO.
- **Pendiente:** agregar alerta en Prometheus si el archivo de respaldo del día no existe (por ejemplo con el textfile collector de node_exporter).
