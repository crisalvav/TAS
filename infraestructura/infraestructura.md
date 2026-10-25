# Infraestructura

## Distribucion de los nodos

Para el desarrollo del caso de nodo sur  como equipo decidimos distribuir la organizacion de la siguiente manera:

| VM (hostname)    | Nodo    | Rol                        | IP privada   | Servicios                  |
| ---------------- | ------- | -------------------------- | ------------ | -------------------------- |
| `tas05vm1`       | DNS     | DNS y balanceador de carga | 10.33.199.42 | DNS (53), Nginx (80/443)   |
| `tas05vm2`       | CMS 1   | CMS principal              | 10.33.199.43 | WordPress                  |
| `tas05vm3`       | CMS 2   | CMS, segunda réplica       | 10.33.199.44 | WordPress                  |
| `tas05vm4`       | DB      | Base de Datos              | 10.33.199.45 | MariaDB                    |
| `tas05vm5`       | Monitor | Monitorización             | 10.33.199.46 |                            |

_**Obs**: considerar que la mayoria de los servicios estan dockerizados(desplegados en contenedores) a excepcion de la base de datos._