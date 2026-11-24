# Imágenes subidas en un backend no aparecen en el otro (21/10/2026)

**Servidores:** `tas05vm2` (10.33.199.43), `tas05vm3` (10.33.199.44), `tas05vm4` (10.33.199.45) · **Servicio afectado:** WordPress (`wp_app`) · **Impacto:** una imagen subida por un backend daba 404 cuando la petición la atendía el otro

## Qué se observó

El profesor advirtió que si se sube una imagen en WordPress desde la vm2, no se ve en la vm3. El contenido (entradas, usuarios, configuración) sí se compartía, porque ambos backends usan la misma MariaDB.

## Diagnóstico

1. Se confirmó que ambos WordPress apuntan a la base de la vm4 y que el compose no levanta ninguna base local:

   ```bash
   sudo docker exec wp_app env | grep -i WORDPRESS_DB      # WORDPRESS_DB_HOST=10.33.199.45:3306
   sudo docker exec wp_app bash -c "echo > /dev/tcp/10.33.199.45/3306 && echo OK"
   ```

2. El `docker-compose.yml` de la vm2 montaba un volumen local de Docker:

   ```yaml
   volumes:
     - wp_data:/var/www/html
   ```

   Todo `/var/www/html`, incluido `wp-content/uploads`, quedaba en el disco de cada VM. La vm3 tenía su propio `wp_data`.

3. Además, la vm3 no era igual a la vm2:

   | | vm2 | vm3 |
   | --- | --- | --- |
   | Imagen | `wordpress:latest` (Apache) | `wordpress:fpm-alpine` + `nginx:alpine` |
   | UID de `www-data` | 33 (Debian) | 82 (Alpine) |
   | Puerto | `9000:80` | `10.33.199.44:80:80` |

   El nginx de la vm3 montaba `wp_data` (volumen vacío) y no el NFS, así que los archivos estáticos darían 404. Y con UID distintos, lo que subiera una VM no sería escribible por la otra.

## Causa

Cada backend guardaba los archivos de WordPress en un volumen local, y los dos backends usaban imágenes distintas.

## Solución

1. **NFS en la vm4**, exportando la carpeta completa de WordPress:

   ```bash
   sudo apt install -y nfs-kernel-server
   sudo mkdir -p /srv/wp-data
   sudo chown 33:33 /srv/wp-data
   echo "/srv/wp-data 10.33.199.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -ra
   sudo systemctl enable --now nfs-kernel-server
   ```

2. **vm2:** se monta el NFS y se copia lo que ya había en el volumen local:

   ```bash
   sudo apt install -y nfs-common
   sudo mkdir -p /mnt/wp-data
   sudo mount -t nfs 10.33.199.45:/srv/wp-data /mnt/wp-data
   cd ~/wordpress-docker && sudo docker compose down
   sudo cp -a /var/lib/docker/volumes/wordpress-docker_wp_data/_data/. /mnt/wp-data/
   echo "10.33.199.45:/srv/wp-data /mnt/wp-data nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
   ```

3. **vm3:** solo se monta el NFS (sin copiar nada) y se reemplaza su compose por el mismo de la vm2, eliminando el nginx local:

   ```yaml
   services:
     wordpress:
       image: wordpress:latest
       container_name: wp_app
       restart: always
       environment:
         WORDPRESS_DB_HOST: 10.33.199.45:3306
         WORDPRESS_DB_USER: wp_user
         WORDPRESS_DB_PASSWORD: <contraseña_wp>
         WORDPRESS_DB_NAME: wordpress
       volumes:
         - /mnt/wp-data:/var/www/html
       ports:
         - 9000:80
   ```

   ```bash
   mount | grep wp-data && ls /mnt/wp-data   # debe verse wp-config.php, wp-content...
   sudo docker compose down --remove-orphans
   sudo docker compose up -d
   ```

Al compartir `/var/www/html` completo también se comparte `wp-config.php`, con las mismas claves (salts), por lo que la sesión de login es válida en ambos backends.

Verificación: ver `bitacora/04-pruebas/02-almacenamiento_compartido_nfs.md`.

## Lecciones

- La base de datos compartida no basta: WordPress guarda los archivos subidos en disco.
- Los backends balanceados deben ser idénticos (misma imagen, mismo compose, mismos puertos).
- **Pendiente:** el NFS en la vm4 es punto único de falla; hoy solo hay una copia por rsync en la vm5 (ver `bitacora/04-pruebas/04-respaldos.md`).
- **Pendiente:** la contraseña de la base está en texto plano en el compose; moverla a un `.env` fuera del repositorio.
