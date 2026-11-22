# Comparación: propuesta original vs. proyecto implementado

**Propuesta original:** Informe de diseño y planificación, entrega 1 (`docs/informe/entregas/Informe _TAS_05.pdf`, v1.0, 07/09/2026).
**Estado actual:** bitácoras de `bitacora/01-servidores/`, decisiones de `decisiones/` e `infraestructura/infraestructura.md` (corte al 23/10/2026, entrega 3).

Este documento resume qué cambió entre lo que se propuso y lo que finalmente quedó funcionando, por qué cambió y qué quedó pendiente.

---

## 1. Resumen

| Área | Propuesta original (v1.0) | Proyecto implementado | ¿Cambió? |
| --- | --- | --- | --- |
| Número de servidores | 4 VM | 5 VM (se agrega `tas05vm5` de monitoreo) | Sí |
| Balanceador | HAProxy | Nginx (proxy inverso con `least_conn`) | Sí |
| Servidor web de los backends | WordPress en Docker (sin detallar) | Nginx + PHP-FPM, dos contenedores por backend | Se definió |
| Flujo DNS | El tráfico web pasaba de BIND9 a HAProxy | BIND9 solo resuelve; el cliente se conecta directo a Nginx | Sí (corrección) |
| Base de datos | MariaDB en Docker | MariaDB instalada directamente en el sistema | Sí |
| Redis | En el Servidor 4 (con el puerto 3306 por error) | No implementado | Sí |
| Persistencia compartida | Mencionada sin mecanismo definido | NFS de `wp-content` desde `tas05vm4` | Se definió |
| Certificado TLS | "Certificado válido" | CA propia para el dominio interno `nodosur.tas` | Se definió |
| Detección de backend caído | Health checks activos de HAProxy | Chequeo pasivo de Nginx + reintento + sondas Blackbox | Sí |
| Monitoreo | Prometheus + Grafana | Prometheus, Grafana, Alertmanager, Blackbox y exporters en vm5 | Se amplió |
| Logs | Logs centralizados (syslog 6514) | Logs locales en cada VM | Sí (no implementado) |
| Respaldos | Diarios, 30 días, copia separada | Script diario con `mariadb-dump` + `tar` + `sha256`, respaldos guardados en vm5 | Se cumplió |
| MFA en WordPress | Propuesto | Sin registro en bitácora | Pendiente |
| SSH solo con llave | Propuesto | Puerto 22 "habilitado"; sin registro de deshabilitar contraseñas | Pendiente |

---

## 2. Cambios en detalle

### 2.1 De 4 a 5 servidores

- **Propuesta:** Servidor 1 (BIND9 + HAProxy), Servidores 2 y 3 (WordPress), Servidor 4 (MariaDB, Redis, logs y respaldos). El monitoreo no tenía un servidor asignado.
- **Implementado:** se agregó `tas05vm5` (10.33.199.46) para Prometheus, Grafana, Alertmanager y Blackbox exporter. Además guarda la copia separada de los respaldos.
- **Por qué:** así el monitoreo no compite por recursos con la base de datos y puede detectar la caída de `tas05vm4`. Si el monitoreo viviera en vm4, una caída de ese servidor también apagaría las alertas que deben avisarla.

### 2.2 HAProxy → Nginx como balanceador

- **Propuesta:** HAProxy con health checks activos y página de estadísticas.
- **Implementado:** Nginx 1.27 en contenedor en `tas05vm1`, con `upstream` `least_conn`, `max_fails=3 fail_timeout=10s` y `proxy_next_upstream`. Publica solo por HTTPS, redirige HTTP con 301 y agrega HSTS.
- **Por qué** (`decisiones/03-balanceador.md`): HAProxy se probó entre el 05/10 y el 06/10. Su configuración de TLS, redirección y cabeceras resultó más larga que la equivalente en Nginx, que el equipo ya usaba en los backends. Con una sola tecnología, cualquier integrante puede leer, validar (`nginx -t`) y diagnosticar las tres capas web.
- **Qué se perdió y cómo se compensó:** Nginx open source no tiene health checks activos. La primera petición a un backend caído falla, pero `proxy_next_upstream` la reintenta en el otro y el usuario no ve el error. Las sondas de Blackbox a `/healthz` de cada backend generan la alerta en 1 a 2 minutos. Se validó el 13/10/2026 deteniendo cada backend con tráfico continuo: las respuestas siguieron en `200`.

### 2.3 Nginx + PHP-FPM en los backends

- **Propuesta:** "WordPress (Docker)", sin especificar el servidor web.
- **Implementado:** en vm2 y vm3 corren dos contenedores, `wordpress:6.6-php8.3-fpm` y `nginx:1.27-alpine`, expuestos en el puerto 8000. Incluyen un endpoint `/healthz` que responde sin pasar por PHP ni por la base de datos.
- **Por qué** (`decisiones/04-servidor_web.md`): usa menos memoria en VM con recursos acotados (Nginx sirve los archivos estáticos sin cargar PHP), deja la configuración versionada en un solo archivo en vez de `.htaccess`, y permite usar la misma herramienta en el balanceador.

### 2.4 Flujo DNS corregido

- **Propuesta:** el diagrama original mostraba clientes → BIND9 → HAProxy, como si el tráfico web atravesara el DNS. El docente observó este error.
- **Implementado:** zona interna `nodosur.tas` en BIND9 (vm1). `www` apunta a vm1, y existen registros `cms1`, `cms2`, `db` y `monitor`. El cliente primero resuelve el nombre y después se conecta por HTTPS directamente a Nginx.
- **Por qué:** el DNS solo traduce nombres a IP; no transporta tráfico web.

### 2.5 MariaDB fuera de Docker

- **Propuesta:** "MariaDB (Docker)" en el Servidor 4.
- **Implementado:** MariaDB instalada con `apt` en vm4, escuchando en 10.33.199.45:3306. El usuario `wp_user` solo puede conectarse desde vm2 y vm3 y solo tiene permisos sobre la base `wordpress`.
- **Por qué** (`bitacora/01-servidores/04-mariadb.md`): acceso directo al almacenamiento, respaldos más simples y una capa menos en el servicio más crítico. La bitácora remite a `decisiones/02-dockerizacion.md`, pero ese archivo está vacío (ver sección 4).

### 2.6 Redis no implementado

- **Propuesta:** Redis en el Servidor 4. En la versión original se le asignó por error el puerto 3306, que es de MariaDB; el docente lo observó.
- **Implementado:** Redis no aparece en ninguna bitácora, en la infraestructura ni en el firewall.
- **Por qué:** no hay una decisión documentada. Las sesiones de WooCommerce se guardan en la base de datos y ambos backends comparten las mismas claves (`AUTH_KEY`, etc.), así que el sitio funciona sin Redis. Conviene registrar esta decisión en `decisiones/` o retomar Redis si las pruebas de carga lo requieren.

### 2.7 Persistencia compartida con NFS

- **Propuesta:** "persistencia compartida" sin definir el mecanismo.
- **Implementado:** vm4 exporta `/srv/nfs/wp-content` por NFS solo a vm2 y vm3. Ambos backends lo montan en `/mnt/wp-content`. WordPress y WooCommerce se instalaron una sola vez y los dos nodos ven los mismos plugins, temas e imágenes.
- **Por qué:** era la forma más simple de que una imagen subida en un backend sea visible desde el otro.

### 2.8 Certificado TLS con CA propia

- **Propuesta:** certificado TLS válido, con renovación.
- **Implementado:** CA propia (`NodoSur CA`) y certificado para `www.nodosur.tas` con SAN, vigente por 397 días. El `ca.crt` se instala en los clientes y en Blackbox. Hay una alerta `CertificadoPorVencer` 15 días antes del vencimiento.
- **Por qué:** `nodosur.tas` es un dominio interno, así que Let's Encrypt no puede emitirle certificados.

### 2.9 Monitoreo ampliado

- **Propuesta:** Prometheus + Grafana, alertas en 5 minutos o menos.
- **Implementado en vm5:** Prometheus (retención de 30 días), Alertmanager (correo), Blackbox exporter (sitio público y `/healthz` de cada backend), Grafana (dashboards 1860 y 7362, más un panel propio). En las 5 VM corre `node_exporter` y en vm4 `mysqld_exporter`. Hay alertas `NodoCaido`, `SitioNoResponde`, `DiscoCasiLleno`, `CPUAlta` y `CertificadoPorVencer`, la mayoría con `for: 1m`.
- **Por qué se amplió:** Blackbox era necesario para compensar la falta de health checks activos en Nginx (2.2), y Alertmanager para cumplir la detección en 5 minutos o menos.

### 2.10 Logs: centralización no implementada

- **Propuesta:** logs centralizados en el Servidor 4 por syslog TLS (puerto 6514).
- **Implementado:** cada VM conserva sus logs localmente (logs de Nginx en vm1, `docker compose logs` en vm2/vm3, `auth.log`, `ufw.log` y `journalctl`).
- **Por qué:** no está documentado. En la bitácora de monitoreo los logs quedan "ubicados para la entrega de observabilidad", así que se trata de un pendiente para la entrega 4.

### 2.11 Respaldos

- **Propuesta:** respaldos diarios de base de datos, archivos y configuración, retención de 30 días o más, copia separada, RPO ≤ 24 h y RTO ≤ 60 min.
- **Implementado** (21/10 al 23/10/2026): script `/usr/local/bin/respaldo_nodosur.sh` en vm5, programado con cron a las 03:00. Por SSH a vm4 hace `mariadb-dump --single-transaction` y un `tar` de `wp-content`, guarda ambos en vm5, genera sumas `sha256` y borra lo que tenga más de 30 días. La restauración se probó en una base temporal. La configuración de cada VM se respalda versionándola en Git.
- **Diferencia:** los respaldos quedan en vm5, separados de la base de datos pero dentro del mismo laboratorio; no es una copia fuera del sitio. Además falta la alerta si el respaldo del día no se generó (pendiente anotado en la bitácora).

---

## 3. Arquitectura final

```
                       Clientes
(1) DNS 53/UDP v ^                       (2) HTTPS 443
               | |  IP de vm1            |   (80 -> 301)
               v |                       v
+--------------- tas05vm1 · 10.33.199.42 ----------------+
| BIND9: solo resuelve      Nginx balanceador TLS        |
+----------------------------------------+---------------+
       least_conn · 8000/TCP · reintento |
              +--------------------------+--+
              v                             v
 +---- tas05vm2 · .43 ----+    +---- tas05vm3 · .44 ----+
 | Nginx + PHP-FPM        |    | Nginx + PHP-FPM        |
 | WordPress/Woo :8000    |    | WordPress/Woo :8000    |
 +------------+-----------+    +------------+-----------+
              |    3306 (MariaDB) · NFS     |
              +--------------+--------------+
                             v
+--------------- tas05vm4 · 10.33.199.45 ----------------+
| MariaDB (3306, sin Docker)                             |
| NFS /srv/nfs/wp-content (datos vivos de vm2/vm3)       |
+----------------------------+---------------------------+
                             ^ SSH 22 desde vm5: dump + tar
                             |
+--------------- tas05vm5 · 10.33.199.46 ----------------+
| Prometheus · Alertmanager · Grafana · Blackbox         |
| Respaldos: /srv/respaldos (30 días)                    |
+--------------------------------------------------------+
  vm5 consulta 9100 (todas), 9104 (vm4) y /healthz
```

| VM | Rol | Servicios | En contenedor |
| --- | --- | --- | --- |
| `tas05vm1` | DNS y balanceador | BIND9 (53), Nginx (80/443) | Sí |
| `tas05vm2` | CMS 1 | WordPress PHP-FPM + Nginx (8000) | Sí |
| `tas05vm3` | CMS 2 | WordPress PHP-FPM + Nginx (8000) | Sí |
| `tas05vm4` | Datos | MariaDB (3306), NFS, `mysqld_exporter` (9104) | No |
| `tas05vm5` | Monitoreo y respaldos | Prometheus (9090), Grafana (3000), Alertmanager (9093), Blackbox, respaldos | Sí |

**Puntos únicos de falla:** se mantienen los que el informe ya reconocía, ahora repartidos así:

- `tas05vm1`: DNS y balanceador. Mejora futura: segundo Nginx con `keepalived` (VRRP).
- `tas05vm4`: MariaDB y NFS. Mejora futura: réplica de base de datos.
- `tas05vm5`: si cae, el servicio sigue funcionando, pero sin alertas.

---

## 4. Pendientes y brechas de documentación

| Pendiente | Dónde corresponde |
| --- | --- |
| Centralizar logs (o justificar mantenerlos locales) | Entrega 4 · observabilidad |
| Alerta si no existe el respaldo del día (textfile collector) | `bitacora/01-servidores/08-respaldos.md` |
| Registrar la prueba de RTO con tiempos reales | `bitacora/01-servidores/08-respaldos.md` |
| Activar MFA en WordPress y documentarlo | Seguridad |
| Deshabilitar el acceso SSH por contraseña y limitar el puerto 22 a la red de gestión | Seguridad · firewall |
| Documentar por qué no se implementó Redis | `decisiones/` |
| Completar `decisiones/01-estructura_de_servicios.md` y `decisiones/02-dockerizacion.md` (están vacíos y la bitácora de MariaDB remite al segundo) | `decisiones/` |
| Completar `bitacora/03-situaciones_de_error/01-caida_por_balanceador.md` (vacío; la bitácora de Nginx remite a él) | Incidentes |
| Alinear el informe v1.1 con lo implementado: menciona Redis, logs por syslog, restic y MariaDB en Docker | `docs/informe/entregas/` |
