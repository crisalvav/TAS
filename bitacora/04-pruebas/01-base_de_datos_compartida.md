# Prueba: ambos WordPress usan la MariaDB de la vm4 (21/10/2026)

**Servidores:** `tas05vm2`, `tas05vm3`, `tas05vm4` · **Objetivo:** confirmar que los backends no usan una base local y que comparten el contenido

## Procedimiento

1. En cada backend, a qué base apunta WordPress:

   ```bash
   sudo docker exec wp_app env | grep -i WORDPRESS_DB
   sudo docker exec wp_app grep -E "DB_HOST|DB_NAME|DB_USER" /var/www/html/wp-config.php
   sudo docker ps   # no debe haber contenedor de base de datos
   ```

2. Conexión TCP desde el contenedor hacia la vm4:

   ```bash
   sudo docker exec wp_app bash -c "echo > /dev/tcp/10.33.199.45/3306 && echo OK"
   ```

3. En la vm4, que la IP sea la suya, que MariaDB escuche en ella y que existan las tablas:

   ```bash
   ip -4 addr show | grep 10.33.199.45
   sudo ss -tlnp | grep 3306
   sudo mariadb -e "SELECT user, host FROM mysql.user;"
   sudo mariadb -e "SHOW TABLES FROM wordpress;"
   ```

4. Crear una entrada desde un backend y buscarla en la vm4:

   ```bash
   sudo mariadb -e "SELECT ID, post_title FROM wordpress.wp_posts ORDER BY ID DESC LIMIT 3;"
   ```

## Resultado

| Comprobación | Resultado |
| --- | --- |
| `WORDPRESS_DB_HOST` en vm2 | `10.33.199.45:3306` ✅ |
| `wp-config.php` | Lee las variables `WORDPRESS_DB_*` del entorno (`getenv_docker`) ✅ |
| Contenedor de base local | Ninguno (la vm4 no tiene Docker; MariaDB nativo) ✅ |
| `wp-content/uploads` en vm2 | Vacío (no se había subido nada desde la migración) |

**Conclusión:** el contenido se comparte por la base de datos, pero los archivos no (ver `bitacora/03-situaciones_de_error/02-archivos_no_compartidos_entre_backends.md`).

Después de configurar keepalived, ambos backends apuntan a la IP virtual `10.33.199.41:3306` (ver `05-failover_base_de_datos.md`).
