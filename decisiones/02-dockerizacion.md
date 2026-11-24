# Decisión: servicios en contenedores Docker (salvo MariaDB y NFS)

**Fecha de decisión:** 05/09/2026 · **Estado:** aceptada · **Servidores afectados:** `tas05vm1`, `tas05vm2`, `tas05vm3`, `tas05vm5` (en `tas05vm4` los servicios van instalados directamente)

## Contexto

Al comenzar la implementación se debía decidir cómo instalar los servicios de la plataforma (DNS, balanceador, WordPress, monitoreo y base de datos). En un principio se planteó instalarlos de forma **nativa** (paquetes del sistema con `apt` en cada VM).

Se buscaba:

1. Poder tomar el sistema y desplegarlo en cualquier otro equipo con la misma configuración.
2. Aislar los servicios entre sí, para que la actualización o falla de uno no afecte a los demás.
3. Que la configuración quede en archivos versionables en el repositorio.

## Comparación

| Criterio | Instalación nativa | Docker + Docker Compose | Comentario |
| --- | --- | --- | --- |
| Desplegar en otro equipo | Repetir a mano cada instalación y configuración | `docker compose up -d` con el mismo `docker-compose.yaml` | **Ventaja de Docker** |
| Aislamiento entre servicios | Comparten librerías y versiones del sistema | Cada servicio con su imagen y versión fija | **Ventaja de Docker** |
| Configuración versionable | Repartida en `/etc` | Compose + archivos montados en `~/servicios/<servicio>` | Ventaja de Docker |
| Persistencia de datos | Directa en el disco | Hay que definir volúmenes | Ventaja nativa |
| Acceso directo al almacenamiento y respaldos | Directo | Pasa por volúmenes | Ventaja nativa |

## Decisión

Se descartó la instalación nativa como regla general y se usó **Docker con Docker Compose** en 4 de las 5 VM:

| VM | En contenedor |
| --- | --- |
| `tas05vm1` | BIND9, Nginx balanceador |
| `tas05vm2`, `tas05vm3` | WordPress (PHP-FPM) + Nginx |
| `tas05vm5` | Prometheus, Alertmanager, Blackbox exporter, Grafana |
| `tas05vm4` | Ninguno: MariaDB y NFS instalados directamente |

**Excepción, MariaDB y NFS:** quedan nativos en `tas05vm4` para tener acceso directo al almacenamiento, simplificar los respaldos y evitar una capa extra en el servicio más crítico (ver `bitacora/01-servidores/04-mariadb.md`).

## Justificación

- **Portabilidad.** Cada servicio se levanta con su `docker-compose.yaml` y los archivos de configuración montados, así que el sistema se puede desplegar en cualquier lado con la misma configuración.
- **Aislamiento.** Cada servicio corre en su propio contenedor con una versión de imagen fija (por ejemplo `nginx:1.27-alpine`, `wordpress:6.6-php8.3-fpm`, `grafana/grafana:11.2.0`), sin depender de lo instalado en el sistema.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Los datos de un contenedor se pierden al recrearlo | Se definieron volúmenes para todo lo que debe persistir (ver tabla siguiente) |
| Los datos de WordPress deben ser los mismos en ambos backends | `wp-content` vive en el NFS de vm4, montado en `/mnt/wp-content` y pasado al contenedor como bind mount |
| Una capa más para la base de datos | MariaDB se dejó fuera de Docker |

Volúmenes usados para la persistencia:

| Servicio | Volumen | Tipo |
| --- | --- | --- |
| BIND9 | `./config/named.conf` (solo lectura), `./zones`, `./cache` | Bind mount |
| Nginx balanceador | `./conf.d` y `./certs` (solo lectura), `./logs` | Bind mount |
| WordPress | `wp_core` (código de WordPress) | Volumen con nombre |
| WordPress | `/mnt/wp-content` (NFS de vm4) | Bind mount |
| Prometheus | `prom_data` | Volumen con nombre |
| Grafana | `grafana_data` | Volumen con nombre |

## Cuándo reconsiderar

Volver a evaluar MariaDB en contenedor si se automatiza todo el despliegue (incluida vm4) o si se agrega una réplica de la base de datos.
