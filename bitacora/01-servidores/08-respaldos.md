# Respaldos de la base y de los archivos de WordPress (21/10/2026)

**Origen:** `tas05vm4` (10.33.199.45, ejecuta el script) · **Destino:** `tas05vm5` (10.33.199.46)
**Configuración actual:** cada 6 horas, retención de 7 días, copia en otra VM · **RPO:** ≈ 6 h

El respaldo se genera en la vm4 y se copia a la vm5 por rsync, para que no quede solo en la VM que se respalda.

## 21/10/2026 — Script de respaldo en `tas05vm4`

```bash
sudo mkdir -p /var/backups/mariadb
sudo tee /usr/local/bin/backup-db.sh >/dev/null <<'EOF'
#!/bin/bash
F=/var/backups/mariadb/wordpress_$(date +%F_%H%M).sql.gz
mariadb-dump --single-transaction --routines --events --triggers --databases wordpress | gzip > "$F"
find /var/backups/mariadb -name '*.sql.gz' -mtime +7 -delete
EOF
sudo chmod +x /usr/local/bin/backup-db.sh
sudo /usr/local/bin/backup-db.sh && ls -lh /var/backups/mariadb
```

## 21/10/2026 — Programación

```bash
echo "0 */6 * * * root /usr/local/bin/backup-db.sh" | sudo tee /etc/cron.d/backup-db
```

## 21/10/2026 — Copia a `tas05vm5`

1. Llave SSH de root en la vm4, para que el cron no pida contraseña:

   ```bash
   sudo ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
   sudo ssh-copy-id -i /root/.ssh/id_ed25519.pub tas05vm5@10.33.199.46
   sudo ssh tas05vm5@10.33.199.46 "mkdir -p ~/respaldos/db ~/respaldos/wp-data"
   ```

2. Se agrega el envío al final del script (dumps y carpeta NFS de WordPress):

   ```bash
   sudo tee -a /usr/local/bin/backup-db.sh >/dev/null <<'EOF'
   rsync -a --delete /var/backups/mariadb/ tas05vm5@10.33.199.46:respaldos/db/
   rsync -rlt --delete /srv/wp-data/ tas05vm5@10.33.199.46:respaldos/wp-data/
   EOF
   sudo /usr/local/bin/backup-db.sh
   sudo ssh tas05vm5@10.33.199.46 "ls -lh ~/respaldos/db; ls ~/respaldos/wp-data"
   ```

Resultado en `bitacora/04-pruebas/04-respaldos.md`.

## Procedimiento de restauración (runbook)

1. Detener ambos backends (`sudo docker compose down` en `~/wordpress-docker` de vm2 y vm3).
2. Base de datos, en la base principal (hoy la vm5):

   ```bash
   gunzip -c ~/respaldos/db/wordpress_<FECHA>.sql.gz | sudo mariadb
   ```

   El dump incluye `--databases wordpress`, así que recrea la base.
3. Archivos, en la vm4:

   ```bash
   sudo rsync -rlt tas05vm5@10.33.199.46:respaldos/wp-data/ /srv/wp-data/
   sudo chown -R 33:33 /srv/wp-data     # rsync -rlt no conserva el dueño
   ```

4. Levantar los backends y verificar el sitio por el balanceador.

## Observaciones

- Desde el 23/10 la vm4 es la réplica, así que el script respalda una copia al día de la principal. Si la réplica se corta, el respaldo queda desactualizado: hay que vigilar la replicación.
- El rsync usa `--delete`: si se borra un archivo en el NFS, también desaparece de la copia en la vm5 en la siguiente ejecución. La copia de archivos no tiene historial.
- **Pendiente:** prueba de restauración medida frente al RTO, sumas `sha256` de los dumps, alerta si no hay respaldo del día y unificar con el objetivo de 30 días de retención del informe.
