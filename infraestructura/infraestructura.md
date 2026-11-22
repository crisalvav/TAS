# Infraestructura

## Distribucion de los nodos

Para el desarrollo del caso de nodo sur  como equipo decidimos distribuir la organizacion de la siguiente manera:

| VM (hostname)    | Nodo    | Rol                        | IP privada   | Servicios                                                     |
| ---------------- | ------- | -------------------------- | ------------ | ------------------------------------------------------------- |
| `tas05vm1`       | DNS     | DNS y balanceador de carga | 10.33.199.42 | BIND9 (53), Nginx balanceador HTTPS (80/443)                  |
| `tas05vm2`       | CMS 1   | CMS principal              | 10.33.199.43 | WordPress PHP-FPM + Nginx (8000)                              |
| `tas05vm3`       | CMS 2   | CMS, segunda réplica       | 10.33.199.44 | WordPress PHP-FPM + Nginx (8000)                              |
| `tas05vm4`       | DB      | Base de Datos              | 10.33.199.45 | MariaDB (3306), NFS `wp-content`, mysqld_exporter (9104) |
| `tas05vm5`       | Monitor | Monitorización y respaldos | 10.33.199.46 | Prometheus (9090), Grafana (3000), Alertmanager (9093), Blackbox exporter, respaldos |

Todas las VM ejecutan `node_exporter` (9100), accesible solo desde `tas05vm5`.

_**Obs**: considerar que la mayoria de los servicios estan dockerizados(desplegados en contenedores) a excepcion de la base de datos, el servidor NFS y los exporters de sistema (`node_exporter`, `mysqld_exporter`)._

## Flujo de una petición

```
1) Resolución (solo el nombre, sin tráfico web)
   Cliente --- consulta DNS 53/UDP ------------> BIND9 (vm1)
   Cliente <-- www.nodosur.tas = 10.33.199.42 -- BIND9 (vm1)

2) Petición web (conexión directa al balanceador)
   Cliente --- HTTPS 443 ---> Nginx balanceador (vm1)   (HTTP 80 -> 301)
                              |  least_conn · 8000/TCP · reintento en el otro backend
                              +--> vm2:8000 (Nginx + WordPress) --+
                              +--> vm3:8000 (Nginx + WordPress) --+--> MariaDB vm4:3306
                                                                  +--> NFS vm4:/srv/nfs/wp-content
```

- HTTP (80) redirige a HTTPS (301). Certificado firmado por una CA propia (dominio interno `nodosur.tas`).
- Si un backend falla 3 veces en 10 s se retira durante 10 s (`max_fails`/`fail_timeout`) y la petición se reintenta en el otro (`proxy_next_upstream`).

## Reglas de firewall (ufw)

| VM       | Puerto      | Permitido desde                 |
| -------- | ----------- | ------------------------------- |
| vm1      | 53 tcp/udp  | 10.33.199.0/24                  |
| vm1      | 80, 443     | cualquiera                      |
| vm2, vm3 | 8000        | vm1 (balanceador), vm5 (monitor) |
| vm4      | 3306        | vm2, vm3                        |
| vm4      | NFS         | vm2, vm3                        |
| vm4      | 9104        | vm5                             |
| vm5      | 3000, 9090  | 10.33.199.0/24                  |
| todas    | 9100        | vm5                             |
| todas    | 22 (ssh)    | habilitado                      |

## Respaldos

- Script `/usr/local/bin/respaldo_nodosur.sh` en vm5, diario a las 03:00 (cron). Se conecta por SSH a vm4 con un usuario `respaldo` de permisos mínimos.
- Respalda la base de datos (`mariadb-dump`) y `wp-content` (tar), con sumas `sha256`.
- Los respaldos se guardan en vm5 (`/srv/respaldos`), separados de vm4, con retención de 30 días.
- Objetivos: RPO ≤ 24 h, RTO ≤ 60 min.

## Puntos únicos de falla conocidos

- `tas05vm1`: DNS y balanceador en la misma VM.
- `tas05vm4`: MariaDB y NFS en la misma VM.

El detalle de cada instalación está en `bitacora/01-servidores/` y la justificación de las decisiones en `decisiones/`.
