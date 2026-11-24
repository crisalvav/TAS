# Instalación de WordPress en `tas05vm2` y `tas05vm3` con persistencia compartida NFS (23/09/2026 - 21/10/2026)

**Servidores:** `tas05vm2` (10.33.199.43 · gestión 10.33.195.163), `tas05vm3` (10.33.199.44 · gestión 10.33.195.251) · **Servicio:** WordPress con Apache (imagen `wordpress:latest`) en un contenedor `wp_app` · **Puerto expuesto:** 9000
**Persistencia:** carpeta completa `/var/www/html` compartida por NFS desde `tas05vm4` (`/srv/wp-data`)

## 23/09/2026 - 02/10/2026 — Despliegue inicial

- `tas05vm2`: WordPress migrado al servidor (`wp-migracion.tar.gz`) y levantado con `wordpress:latest` (Apache incluido) en `~/wordpress-docker`, con un volumen local de Docker `wp_data:/var/www/html` y publicado en `9000:80`.
- `tas05vm3`: se levantó con `wordpress:fpm-alpine` más un contenedor `nginx:alpine` delante, publicado en `10.33.199.44:80`.
- La base de datos es la de `tas05vm4` (ver `04-mariadb.md`). WordPress lee la conexión de las variables `WORDPRESS_DB_*` del compose (`getenv_docker` en `wp-config.php`).

Con este esquema el contenido (entradas, usuarios) se compartía por la base, pero los archivos subidos quedaban en el disco de cada VM. Ver `bitacora/03-situaciones_de_error/02-archivos_no_compartidos_entre_backends.md`.

## 21/10/2026 — Servidor NFS en `tas05vm4`

Se comparte `/var/www/html` completo (no solo `wp-content`): así ambos backends ven los mismos uploads, plugins, temas y el mismo `wp-config.php`, con las mismas claves (salts), por lo que la sesión de login vale en los dos.

```bash
sudo apt update && sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/wp-data
sudo chown 33:33 /srv/wp-data            # uid 33 = www-data en la imagen Debian de WordPress
echo "/srv/wp-data 10.33.199.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
sudo exportfs -v
```

## 21/10/2026 — Cliente NFS en `tas05vm2` (con copia de los datos)

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/wp-data
sudo mount -t nfs 10.33.199.45:/srv/wp-data /mnt/wp-data
cd ~/wordpress-docker && sudo docker compose down
sudo cp -a /var/lib/docker/volumes/wordpress-docker_wp_data/_data/. /mnt/wp-data/
echo "10.33.199.45:/srv/wp-data /mnt/wp-data nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

## 21/10/2026 — Cliente NFS en `tas05vm3` (sin copiar)

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/wp-data
echo "10.33.199.45:/srv/wp-data /mnt/wp-data nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
ls /mnt/wp-data   # wp-content, wp-config.php, etc.
```

La vm3 se pasó a la misma imagen que la vm2. Con imágenes distintas los archivos del NFS tendrían dueños distintos (UID 33 en Debian, 82 en Alpine) y el nginx de la vm3 servía los estáticos desde un volumen vacío.

## 21/10/2026 — Docker Compose del backend

`~/wordpress-docker/docker-compose.yml` (idéntico en ambos backends):

```yaml
services:
  wordpress:
    image: wordpress:latest
    container_name: wp_app
    restart: always
    environment:
      WORDPRESS_DB_HOST: 10.33.199.41:3306   # IP virtual de la base (antes 10.33.199.45)
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: <contraseña_wp>
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/wp-data:/var/www/html
    networks:
      - wp_net
    ports:
      - 9000:80

networks:
  wp_net:
    driver: bridge
```

```bash
mount | grep wp-data          # confirmar el NFS antes de levantar; si no, WordPress se instala de nuevo encima
sudo docker compose down --remove-orphans
sudo docker compose up -d
sudo docker ps                # solo wp_app, 0.0.0.0:9000->80
curl -I http://10.33.195.163:9000
curl -I http://10.33.195.251:9000
```

## Resultado

- Dos backends idénticos sirviendo el mismo sitio, con base de datos y archivos compartidos (pruebas en `bitacora/04-pruebas/01-base_de_datos_compartida.md` y `02-almacenamiento_compartido_nfs.md`).
- **Punto único de falla:** el NFS en vm4. Hay una copia de `/srv/wp-data` en vm5 por rsync (ver `08-respaldos.md`), sin failover automático.
- **Pendiente:** mover la contraseña de la base a un `.env` fuera del repositorio y limitar el export NFS a `.43` y `.44` sin `no_root_squash`.
