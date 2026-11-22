# Instalación de la base de datos MariaDB en `tas05vm4` (14/09/2026 - 22/09/2026)

**Servidor:** `tas05vm4` (10.33.199.45) · **Servicio:** MariaDB instalado directamente en el sistema (sin contenedor) · **Puerto:** 3306

> Se decidió **no** dockerizar la base de datos para tener acceso directo al almacenamiento, simplificar respaldos y evitar una capa extra en el servicio más crítico (ver `decisiones/02-dockerizacion.md`).

## 14/09/2026 — Instalación

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
   collation-server = utf8mb4_unicode_ci
   ```

   ```bash
   sudo systemctl restart mariadb
   sudo ss -lntp | grep 3306
   ```

## 16/09/2026 — Base de datos y usuario de WordPress

Se crea un usuario que solo puede conectarse desde los dos backends y solo sobre su base (mínimo privilegio):

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'10.33.199.43' IDENTIFIED BY '<contraseña_segura>';
CREATE USER 'wp_user'@'10.33.199.44' IDENTIFIED BY '<contraseña_segura>';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.33.199.43';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.33.199.44';
FLUSH PRIVILEGES;
```

```bash
sudo mariadb -e "SELECT user, host FROM mysql.user;"
```

_La contraseña real no se versiona en el repositorio; se guarda en el `.env` de cada backend._

## 17/09/2026 — Firewall

Solo los backends pueden llegar al 3306:

```bash
sudo ufw allow from 10.33.199.43 to any port 3306 proto tcp
sudo ufw allow from 10.33.199.44 to any port 3306 proto tcp
sudo ufw deny 3306
sudo ufw status numbered
```

_18/09 y 19/09 feriados (Fiestas Patrias), sin avance._

## 21/09/2026 — Pruebas de conexión

1. Desde `tas05vm2` (debe funcionar):

   ```bash
   sudo apt install -y mariadb-client
   mariadb -h 10.33.199.45 -u wp_user -p -e "SHOW DATABASES;"
   ```

2. Desde `tas05vm5` (debe fallar por firewall / host no permitido):

   ```bash
   mariadb -h 10.33.199.45 -u wp_user -p   # ERROR 2002 / timeout
   ```

## 22/09/2026 — Persistencia ante reinicio

```bash
sudo mariadb -e "CREATE TABLE wordpress.prueba (id INT); INSERT INTO wordpress.prueba VALUES (1);"
sudo reboot
# tras el reinicio
sudo mariadb -e "SELECT * FROM wordpress.prueba; DROP TABLE wordpress.prueba;"
```

## Resultado

- MariaDB operativa, accesible solo desde `tas05vm2` y `tas05vm3`.
- **Punto único de falla:** la base de datos sigue siendo un solo nodo; la réplica queda como mejora futura.
