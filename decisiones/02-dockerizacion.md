# Decisión: servicios en contenedores Docker (salvo MariaDB, NFS y keepalived)

**Fecha de decisión:** 05/09/2026 · **Estado:** aceptada · **Servidores afectados:** `tas05vm1`, `tas05vm2`, `tas05vm3`, `tas05vm5` (en `tas05vm4` todo va instalado directamente)

## Contexto

Al comenzar la implementación se debía decidir cómo instalar los servicios de la plataforma (DNS, balanceador, WordPress, monitoreo y base de datos). En un principio se planteó instalarlos de forma **nativa** (paquetes del sistema con `apt` en cada VM).

Se buscaba:

1. Poder tomar el sistema y desplegarlo en cualquier otro equipo con la misma configuración.
2. Aislar los servicios entre sí, para que la actualización o falla de uno no afecte a los demás.
3. Que la configuración quede en archivos versionables en el repositorio.

## Comparación

| Criterio | Instalación nativa | Docker + Docker Compose | Comentario |
| --- | --- | --- | --- |
| Desplegar en otro equipo | Repetir a mano cada instalación y configuración | `docker compose up -d` con el mismo `docker-compose.yml` | **Ventaja de Docker** |
| Aislamiento entre servicios | Comparten librerías y versiones del sistema | Cada servicio con su propia imagen | **Ventaja de Docker** |
| Configuración versionable | Repartida en `/etc` | Compose + archivos montados junto al compose | Ventaja de Docker |
| Persistencia de datos | Directa en el disco | Hay que definir volúmenes | Ventaja nativa |
| Acceso directo al almacenamiento y respaldos | Directo | Pasa por volúmenes | Ventaja nativa |

## Decisión

Se descartó la instalación nativa como regla general y se usó **Docker con Docker Compose** en 4 de las 5 VM:

| VM | En contenedor | Instalado directamente |
| --- | --- | --- |
| `tas05vm1` | BIND9, Nginx balanceador (`wp_nginx`) | node_exporter |
| `tas05vm2`, `tas05vm3` | WordPress con Apache (`wp_app`) | cliente NFS, node_exporter |
| `tas05vm4` | Ninguno | MariaDB, servidor NFS, keepalived, node_exporter |
| `tas05vm5` | Prometheus, Grafana | MariaDB, keepalived, node_exporter |

**Excepción, MariaDB, NFS y keepalived:** quedan nativos para tener acceso directo al almacenamiento, simplificar respaldos y replicación, y porque keepalived debe manejar la IP virtual en la interfaz del propio host (ver `bitacora/01-servidores/04-mariadb.md`).

## Justificación

- **Portabilidad.** Cada servicio se levanta con su `docker-compose.yml` y los archivos de configuración montados, así que el sistema se puede desplegar en cualquier lado con la misma configuración.
- **Aislamiento.** Cada servicio corre en su propio contenedor, sin depender de lo instalado en el sistema.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Los datos de un contenedor se pierden al recrearlo | Se definieron volúmenes para todo lo que debe persistir (ver tabla siguiente) |
| Los datos de WordPress deben ser los mismos en ambos backends | `/var/www/html` completo vive en el NFS de vm4 (`/srv/wp-data`), montado en `/mnt/wp-data` y pasado al contenedor como bind mount |
| Una capa más para la base de datos | MariaDB se dejó fuera de Docker |
| Las imágenes usan `latest` y pueden cambiar de versión al recrear | **Pendiente:** fijar versiones (`wordpress`, `nginx`, `prom/prometheus`, `grafana/grafana`) |

Volúmenes usados para la persistencia:

| Servicio | Volumen | Tipo |
| --- | --- | --- |
| BIND9 | `./config/named.conf` (solo lectura), `./zones`, `./cache` | Bind mount |
| Nginx balanceador | `./nginx/default.conf` y `./certs` (solo lectura) | Bind mount |
| WordPress | `/mnt/wp-data:/var/www/html` (NFS de vm4) | Bind mount |
| Prometheus | `./prometheus.yml` (solo lectura), `prom_data` | Bind mount + volumen con nombre |
| Grafana | `./grafana/provisioning` (solo lectura), `graf_data` | Bind mount + volumen con nombre |

## Cuándo reconsiderar

Volver a evaluar MariaDB en contenedor si se automatiza todo el despliegue (incluidas vm4 y vm5).
