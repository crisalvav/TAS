# `wp_user` solo podía conectarse desde la vm2 (21/10/2026 - 22/10/2026)

**Servidores:** `tas05vm4` (10.33.199.45), `tas05vm5` (10.33.199.46) · **Servicio afectado:** MariaDB · **Impacto:** la vm3 y cualquier IP distinta de la vm2 quedaban rechazadas por la base de datos; el failover hacia la IP virtual no habría funcionado

## Qué se observó

Antes de apuntar WordPress a la IP virtual de la base (10.33.199.41), se revisaron los permisos de `wp_user`:

```bash
sudo mariadb -e "SELECT user,host FROM mysql.user WHERE user='wp_user';"
```

```
+---------+--------------+
| User    | Host         |
+---------+--------------+
| wp_user | 10.33.199.43 |
+---------+--------------+
```

## Causa

El usuario se creó solo para el host `10.33.199.43` (vm2). La vm3 (`10.33.199.44`) no tenía permiso.

## Solución

En la vm4 (en ese momento la base principal) se agregó el usuario para toda la red interna:

```bash
sudo mariadb -e "CREATE USER IF NOT EXISTS 'wp_user'@'10.33.199.%' IDENTIFIED BY '<contraseña_wp>';
  GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'10.33.199.%'; FLUSH PRIVILEGES;"
sudo mariadb -e "SELECT user,host FROM mysql.user WHERE user IN ('wp_user','repl');"
```

El usuario de `.43` se dejó, ya que no interfiere. Los `GRANT` viajan por la replicación, así que en la vm5 no fue necesario repetirlos:

```bash
# vm5
sudo mariadb -e "SELECT user,host FROM mysql.user WHERE user='wp_user';"
sudo mariadb -e "SHOW SLAVE STATUS\G" | grep -E "Running|Error"
```

Prueba desde la vm3:

```bash
sudo docker exec wp_app bash -c "echo > /dev/tcp/10.33.199.45/3306 && echo OK"
sudo docker logs wp_app --tail 20   # sin "Error establishing a database connection"
```

## Lecciones

- La bitácora de MariaDB (`01-servidores/04-mariadb.md`) indicaba usuarios para `.43` y `.44`, pero en el servidor solo existía el de `.43`. Hay que comprobar los permisos en el servidor y no solo en la documentación.
- Con una IP virtual y dos bases, el usuario debe estar permitido para toda la subred interna.
