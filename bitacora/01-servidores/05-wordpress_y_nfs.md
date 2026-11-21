# Instalación de WordPress en `tas05vm2` y `tas05vm3` con persistencia compartida NFS (23/09/2026 - 02/10/2026)

**Servidores:** `tas05vm2` (10.33.199.43), `tas05vm3` (10.33.199.44) · **Servicio:** WordPress (PHP-FPM) + Nginx en contenedores · **Puerto expuesto:** 8000
**Persistencia:** carpeta `wp-content` compartida por NFS desde `tas05vm4`

## 23/09/2026 — Servidor NFS en `tas05vm4`

Ambos backends deben ver los mismos archivos subidos (imágenes, plugins, temas); si no, un usuario que sube una imagen por cms1 no la vería al ser atendido por cms2.

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/wp-content
sudo chown -R 33:33 /srv/nfs/wp-content     # uid 33 = www-data dentro del contenedor
echo "/srv/nfs/wp-content 10.33.199.43(rw,sync,no_subtree_check) 10.33.199.44(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra
sudo exportfs -v
sudo ufw allow from 10.33.199.43 to any port nfs
sudo ufw allow from 10.33.199.44 to any port nfs
```

## 24/09/2026 — Cliente NFS en ambos backends

Se repite en `tas05vm2` y `tas05vm3`:

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/wp-content
echo "10.33.199.45:/srv/nfs/wp-content /mnt/wp-content nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
df -h | grep wp-content
touch /mnt/wp-content/prueba_$(hostname) && ls /mnt/wp-content   # debe verse el archivo de la otra VM
```

## 25/09/2026 — Estructura del servicio

```bash
mkdir -p ~/servicios/wordpress/nginx
cd ~/servicios/wordpress
```

`.env` (no se sube al repositorio):

```env
WORDPRESS_DB_HOST=10.33.199.45:3306
WORDPRESS_DB_USER=wp_user
WORDPRESS_DB_PASSWORD=<contraseña_segura>
WORDPRESS_DB_NAME=wordpress
```

## 28/09/2026 — Docker Compose del backend

`docker-compose.yaml` (idéntico en ambos backends):

```yaml
services:
  wordpress:
    image: wordpress:6.6-php8.3-fpm
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    volumes:
      - wp_core:/var/www/html
      - /mnt/wp-content:/var/www/html/wp-content

  nginx:
    image: nginx:1.27-alpine
    container_name: wp-nginx
    restart: unless-stopped
    depends_on: [wordpress]
    ports:
      - "8000:80"
    volumes:
      - wp_core:/var/www/html:ro
      - /mnt/wp-content:/var/www/html/wp-content:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro

volumes:
  wp_core:
```

`nginx/default.conf`:

```nginx
server {
    listen 80;
    root /var/www/html;
    index index.php;
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass wordpress:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location = /healthz {
        access_log off;
        return 200 "ok\n";
    }
}
```

## 29/09/2026 — Despliegue

```bash
docker compose up -d
docker compose ps
docker compose logs -f
curl -I http://localhost:8000          # 302 hacia /wp-admin/install.php
curl http://localhost:8000/healthz     # ok
```

## 30/09/2026 — Instalación de WordPress (solo una vez)

1. Se completa el asistente web **solo desde `tas05vm2`** (`http://10.33.199.43:8000`), ya que la base de datos es compartida; `tas05vm3` toma la misma instalación automáticamente.
2. Se instala WooCommerce desde el panel; queda en `wp-content/plugins` (NFS), por lo que aparece en ambos backends.

## 01/10/2026 — Configuración detrás de proxy

Como el usuario entrará por HTTPS a través del balanceador, se indica a WordPress la URL pública y que confíe en la cabecera del proxy. Se agrega en `wp-config.php` (vía `WORDPRESS_CONFIG_EXTRA` en el compose):

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
define('WP_HOME', 'https://www.nodosur.tas');
define('WP_SITEURL', 'https://www.nodosur.tas');
```

Las claves `AUTH_KEY`, `SECURE_AUTH_KEY`, etc. se fijan con el **mismo valor en ambos backends**; si difieren, la sesión se pierde al cambiar de nodo.

## 02/10/2026 — Firewall y prueba de consistencia

```bash
sudo ufw allow from 10.33.199.42 to any port 8000 proto tcp   # solo el balanceador
sudo ufw allow from 10.33.199.46 to any port 8000 proto tcp   # monitoreo
```

- Se publica una entrada de prueba desde cms1 y se verifica en cms2 (`curl http://10.33.199.44:8000/?p=1`).
- Se sube una imagen desde cms2 y se verifica que exista en `/srv/nfs/wp-content/uploads` de vm4.

## Resultado

- Dos backends idénticos sirviendo el mismo sitio, con base de datos y archivos compartidos.
- **Punto único de falla:** el NFS en vm4.
