# Infraestructura

## Distribucion de los nodos

Para el desarrollo del caso de nodo sur  como equipo decidimos distribuir la organizacion de la siguiente manera:

| VM (hostname) | Nodo    | Rol                                   | IP interna (`ens19`) | IP gestión (`ens18`) | Servicios |
| ------------- | ------- | ------------------------------------- | -------------------- | -------------------- | --------- |
| `tas05vm1`    | DNS     | DNS y balanceador de carga            | 10.33.199.42 | 10.33.195.230 | BIND9 (53), Nginx balanceador HTTPS (80/443) |
| `tas05vm2`    | CMS 1   | CMS                                   | 10.33.199.43 | 10.33.195.163 | WordPress con Apache (9000) |
| `tas05vm3`    | CMS 2   | CMS                                   | 10.33.199.44 | 10.33.195.251 | WordPress con Apache (9000) |
| `tas05vm4`    | DB      | Réplica de la base, NFS y respaldos   | 10.33.199.45 | 10.33.195.246 | MariaDB réplica (3306), NFS `/srv/wp-data`, keepalived, script de respaldo |
| `tas05vm5`    | Monitor | Base principal, monitoreo y copias    | 10.33.199.46 | 10.33.195.247 | MariaDB principal (3306), keepalived, Prometheus (9090), Grafana (3000), copia de respaldos |

- Red interna: `10.33.199.40/29` (IP utilizables `.41` a `.46`). La `.41` es la **IP virtual de la base de datos**, que keepalived mantiene en la VM con la base principal (hoy la vm5).
- Todas las VM ejecutan `node_exporter` (9100), leído por Prometheus desde `tas05vm5`.

_**Obs**: considerar que la mayoria de los servicios estan dockerizados(desplegados en contenedores) a excepcion de MariaDB, el servidor NFS, keepalived y `node_exporter`._

## Flujo de una petición

```
1) Resolución (solo el nombre, sin tráfico web)
   Cliente --- consulta DNS 53/UDP ---> BIND9 (vm1)
   Cliente <-- IP de vm1 -------------- BIND9 (vm1)

2) Petición web (directo al balanceador)
   Cliente --- HTTPS 443 ---> Nginx (vm1)
                                |  80 -> 301 a HTTPS
                                |  round robin 1:1 · 9000/TCP
                                |  reintento en el otro nodo
                    +-----------+-----------+
                    v                       v
                vm2:9000                vm3:9000
           WordPress (Apache)      WordPress (Apache)
                    |                       |
                    +-----------+-----------+
                                |
              +-----------------+------------------+
              v                                    v
   IP virtual 10.33.199.41:3306          NFS vm4 /srv/wp-data
   (MariaDB principal: hoy vm5)          (/var/www/html completo)
              |
              +-- replicación --> MariaDB réplica (hoy vm4)
```

- HTTP (80) redirige a HTTPS (301). Certificado propio montado como `cert.pem`/`key.pem`; Nginx atiende `conosurtas05.lan` y `www.conosurtas05.lan`.
- Si un backend falla 2 veces se retira durante 10 s (`max_fails`/`fail_timeout`) y la petición se reintenta en el otro.
- Si cae la base principal, keepalived mueve la `.41` a la otra VM y `promote.sh` la deja en escritura.

## Reglas de firewall (ufw)

| VM       | Puerto         | Permitido desde                          |
| -------- | -------------- | ---------------------------------------- |
| vm1      | 53 tcp/udp     | 10.33.199.0/24 (incluye la /29)          |
| vm1      | 80, 443        | cualquiera                               |
| vm2, vm3 | 9000           | _por confirmar_                          |
| vm4, vm5 | 3306           | 10.33.199.0/24                           |
| vm4, vm5 | VRRP (224.0.0.18) | 10.33.199.40/29 por `ens19`           |
| vm4      | NFS (2049)     | _por confirmar_ (export a 10.33.199.0/24) |
| vm5      | 3000 (Grafana) | cualquiera                               |
| vm5      | 9090           | no expuesto (`127.0.0.1`)                |
| todas    | 9100           | vm5 (en vm5: red de Docker 172.16.0.0/12) |
| todas    | 22 (ssh)       | habilitado                               |

## Respaldos

- Script `/usr/local/bin/backup-db.sh` en vm4, cada 6 horas (`/etc/cron.d/backup-db`).
- `mariadb-dump` de la base `wordpress` comprimido en `/var/backups/mariadb`, con retención de 7 días.
- Copia por rsync a vm5 (`~/respaldos/db` y `~/respaldos/wp-data`, la carpeta NFS de WordPress).
- RPO ≈ 6 h. RTO objetivo ≤ 60 min (_prueba de restauración pendiente_).

Detalle en `bitacora/01-servidores/08-respaldos.md`.

## Puntos únicos de falla conocidos

- `tas05vm1`: DNS y balanceador en la misma VM, sin respaldo.
- `tas05vm4`: NFS de WordPress. La base ya tiene failover, pero los archivos no (solo una copia por rsync en vm5).
- `tas05vm5`: si cae, se pierden a la vez el monitoreo y la base principal (la base pasa a vm4).

El detalle de cada instalación está en `bitacora/01-servidores/`, las pruebas en `bitacora/04-pruebas/` y la justificación de las decisiones en `decisiones/`.
