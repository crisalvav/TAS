# Comparación: propuesta original vs. proyecto implementado

**Propuesta original:** Informe de diseño y planificación, entrega 1 (`docs/informe/entregas/Informe _TAS_05.pdf`, v1.0, 07/09/2026).
**Estado actual:** bitácoras de `bitacora/01-servidores/` y `bitacora/04-pruebas/`, decisiones de `decisiones/` e `infraestructura/infraestructura.md` (corte al 23/10/2026, entrega 3).

Este documento resume qué cambió entre lo que se propuso y lo que finalmente quedó funcionando, por qué cambió y qué quedó pendiente.

---

## 1. Resumen

| Área | Propuesta original (v1.0) | Proyecto implementado | ¿Cambió? |
| --- | --- | --- | --- |
| Número de servidores | 4 VM | 5 VM (se agrega `tas05vm5`: monitoreo y segunda base) | Sí |
| Balanceador | HAProxy | Nginx en round robin 1:1 | Sí |
| Servidor web de los backends | WordPress en Docker (sin detallar) | WordPress con Apache (`wordpress:latest`), un contenedor por backend, puerto 9000 | Se definió |
| Flujo DNS | El tráfico web pasaba de BIND9 a HAProxy | BIND9 solo resuelve; el cliente se conecta directo a Nginx | Sí (corrección) |
| Base de datos | MariaDB en Docker, un nodo | MariaDB nativa replicada entre vm4 y vm5, con IP virtual (keepalived) | Sí |
| Redis | En el Servidor 4 (con el puerto 3306 por error) | No implementado | Sí |
| Persistencia compartida | Mencionada sin mecanismo definido | NFS de `/var/www/html` completo desde `tas05vm4` | Se definió |
| Certificado TLS | "Certificado válido" | Certificado propio (`cert.pem`/`key.pem`) para el dominio interno | Se definió |
| Detección de backend caído | Health checks activos de HAProxy | Chequeo pasivo de Nginx + reintento en el otro backend | Sí |
| Monitoreo | Prometheus + Grafana | Prometheus + Grafana en vm5, node_exporter en las 5 VM | Se cumplió (sin alertas aún) |
| Logs | Logs centralizados (syslog 6514) | Consulta manual en cada VM; Loki pendiente | Pendiente |
| Respaldos | Diarios, 30 días, copia separada | Cada 6 h, 7 días, copia por rsync en vm5 | Parcial |
| MFA en WordPress | Propuesto | Sin registro en bitácora | Pendiente |
| SSH solo con llave | Propuesto | Puerto 22 "habilitado"; sin registro de deshabilitar contraseñas | Pendiente |

---

## 2. Cambios en detalle

### 2.1 De 4 a 5 servidores

- **Propuesta:** Servidor 1 (BIND9 + HAProxy), Servidores 2 y 3 (WordPress), Servidor 4 (MariaDB, Redis, logs y respaldos). El monitoreo no tenía un servidor asignado.
- **Implementado:** se agregó `tas05vm5` (10.33.199.46) para Prometheus y Grafana. Además tiene la segunda MariaDB (hoy la principal) y la copia de los respaldos.
- **Por qué:** así el monitoreo no corre en la misma VM que el NFS y puede detectar la caída de `tas05vm4`, y la base de datos deja de ser un solo nodo. La contrapartida es que vm5 concentra base principal y monitoreo.

### 2.2 HAProxy → Nginx como balanceador

- **Propuesta:** HAProxy con health checks activos y página de estadísticas.
- **Implementado:** Nginx (`nginx:alpine`) en contenedor en `tas05vm1`, con `upstream` en round robin 1:1 y `max_fails=2 fail_timeout=10s` hacia `vm2:9000` y `vm3:9000`. Publica solo por HTTPS y redirige HTTP con 301.
- **Por qué** (`decisiones/03-balanceador.md`): HAProxy se probó entre el 05/10 y el 06/10. Su configuración de TLS, redirección y cabeceras resultó más larga que la equivalente en Nginx, que el equipo ya conocía.
- **Qué se perdió y cómo se compensó:** Nginx open source no tiene health checks activos. La primera petición a un backend caído falla, pero Nginx la reintenta en el otro y el usuario recibe respuesta. Validado el 23/10/2026 (`bitacora/04-pruebas/06-balanceo_1a1.md` y `07-caida_de_un_backend.md`).

### 2.3 WordPress con Apache en los backends

- **Propuesta:** "WordPress (Docker)", sin especificar el servidor web.
- **Implementado:** en vm2 y vm3 corre un solo contenedor `wp_app` con `wordpress:latest` (Apache), publicado en el puerto 9000.
- **Por qué** (`decisiones/04-servidor_web.md`): la vm3 había usado `fpm-alpine` + Nginx y la vm2 Apache. Al compartir los archivos por NFS había que igualarlas (mismo UID 33) y se tomó la imagen de la vm2, que ya tenía el sitio migrado.

### 2.4 Flujo DNS corregido

- **Propuesta:** el diagrama original mostraba clientes → BIND9 → HAProxy, como si el tráfico web atravesara el DNS. El docente observó este error.
- **Implementado:** BIND9 en vm1 resuelve el nombre hacia vm1, y el cliente se conecta por HTTPS directamente a Nginx. La bitácora de DNS documenta la zona `nodosur.tas`; Nginx atiende `conosurtas05.lan` (**pendiente** confirmar cuál zona está activa).
- **Por qué:** el DNS solo traduce nombres a IP; no transporta tráfico web.

### 2.5 MariaDB fuera de Docker y replicada

- **Propuesta:** "MariaDB (Docker)" en el Servidor 4.
- **Implementado:** MariaDB instalada con `apt` en vm4 y vm5, con replicación primario → réplica y keepalived moviendo la IP virtual `10.33.199.41` a la base principal. `promote.sh` deja en escritura a la réplica si la otra base no responde. Hoy la vm5 es la principal y la vm4 su réplica (ver `bitacora/03-situaciones_de_error/04-autopromocion_vm5_keepalived.md`).
- **Por qué** (`decisiones/02-dockerizacion.md`, `bitacora/01-servidores/04-mariadb.md`): acceso directo al almacenamiento, respaldos y replicación más simples, y keepalived necesita manejar la interfaz del host.

### 2.6 Redis no implementado

- **Propuesta:** Redis en el Servidor 4. En la versión original se le asignó por error el puerto 3306, que es de MariaDB; el docente lo observó.
- **Implementado:** Redis no aparece en ninguna bitácora, en la infraestructura ni en el firewall.
- **Por qué:** no hay una decisión documentada. Las sesiones se guardan en la base de datos y ambos backends comparten el mismo `wp-config.php` (y sus claves) por el NFS, así que el sitio funciona sin Redis. Conviene registrar esta decisión en `decisiones/`.

### 2.7 Persistencia compartida con NFS

- **Propuesta:** "persistencia compartida" sin definir el mecanismo.
- **Implementado:** vm4 exporta `/srv/wp-data` a la red interna; ambos backends lo montan en `/mnt/wp-data` como `/var/www/html` completo (código, `wp-config.php`, plugins, temas e imágenes).
- **Por qué:** con volúmenes locales una imagen subida en un backend daba 404 en el otro (ver `bitacora/03-situaciones_de_error/02-archivos_no_compartidos_entre_backends.md`).

### 2.8 Certificado TLS

- **Propuesta:** certificado TLS válido, con renovación.
- **Implementado:** certificado propio montado en Nginx como `certs/cert.pem` y `certs/key.pem`, con TLS 1.2 y 1.3.
- **Por qué:** el dominio es interno, así que Let's Encrypt no puede emitirle certificados.
- **Pendiente:** documentar cómo se generó, su vigencia y la renovación.

### 2.9 Monitoreo

- **Propuesta:** Prometheus + Grafana, alertas en 5 minutos o menos.
- **Implementado en vm5:** Prometheus (retención 15 días, solo en `127.0.0.1:9090`) y Grafana (puerto 3000, dashboard 1860). node_exporter en las 5 VM, todos los objetivos en `up` (`bitacora/04-pruebas/08-monitoreo.md`).
- **Pendiente:** alertas (Alertmanager), `mysqld_exporter` para vigilar la replicación, cAdvisor y exporter de Nginx.

### 2.10 Logs

- **Propuesta:** logs centralizados en el Servidor 4 por syslog TLS (puerto 6514).
- **Implementado:** por ahora se consultan en cada VM (`docker logs`, `journalctl`). No se usa syslog 6514.
- **Pendiente:** Loki + Promtail para verlos en Grafana.

### 2.11 Respaldos

- **Propuesta:** respaldos diarios de base de datos, archivos y configuración, retención de 30 días o más, copia separada, RPO ≤ 24 h y RTO ≤ 60 min.
- **Implementado** (21/10/2026): script `/usr/local/bin/backup-db.sh` en vm4, cada 6 horas. Hace `mariadb-dump` de `wordpress`, guarda 7 días en `/var/backups/mariadb` y copia por rsync los dumps y `/srv/wp-data` a vm5 (`~/respaldos`).
- **Diferencia:** RPO ≈ 6 h (mejor que lo propuesto), pero retención de 7 días (menor a 30). La copia de archivos no tiene historial, no hay `sha256` ni alerta si falla, y la restauración no se ha medido contra el RTO.

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
          round robin 1:1 · 9000/TCP     |
              +--------------------------+--+
              v                             v
 +---- tas05vm2 · .43 ----+    +---- tas05vm3 · .44 ----+
 | WordPress + Apache     |    | WordPress + Apache     |
 | wp_app :9000           |    | wp_app :9000           |
 +------------+-----------+    +------------+-----------+
              |   IP virtual .41:3306 · NFS |
              +--------------+--------------+
                             v
+--------------- tas05vm4 · 10.33.199.45 ----------------+
| MariaDB réplica (read_only)  · keepalived              |
| NFS /srv/wp-data · respaldo cada 6 h                   |
+----------------------------+---------------------------+
          ^ replicación      | rsync de respaldos
          |                  v
+--------------- tas05vm5 · 10.33.199.46 ----------------+
| MariaDB principal + IP virtual .41 · keepalived        |
| Prometheus · Grafana · ~/respaldos                     |
+--------------------------------------------------------+
  vm5 consulta 9100 en todas las VM
```

| VM | Rol | Servicios | En contenedor |
| --- | --- | --- | --- |
| `tas05vm1` | DNS y balanceador | BIND9 (53), Nginx (80/443) | Sí |
| `tas05vm2` | CMS 1 | WordPress con Apache (9000) | Sí |
| `tas05vm3` | CMS 2 | WordPress con Apache (9000) | Sí |
| `tas05vm4` | Réplica, archivos y respaldos | MariaDB réplica, NFS, keepalived, script de respaldo | No |
| `tas05vm5` | Base principal y monitoreo | MariaDB principal, keepalived, Prometheus, Grafana | Prometheus y Grafana |

**Puntos únicos de falla:**

- `tas05vm1`: DNS y balanceador. Mejora futura: segundo Nginx con `keepalived` (VRRP), como en la base.
- `tas05vm4`: NFS de WordPress. La base ya tiene failover; los archivos no.
- `tas05vm5`: si cae, la base pasa a vm4, pero se pierde el monitoreo.

---

## 4. Pendientes y brechas de documentación

| Pendiente | Dónde corresponde |
| --- | --- |
| Registrar el failover real de la base (apagar MariaDB en la principal) | `bitacora/04-pruebas/05-failover_base_de_datos.md` |
| Registrar la prueba del NFS por `curl` y la prueba de RTO | `bitacora/04-pruebas/` |
| Alertas, `mysqld_exporter` y Loki | `bitacora/01-servidores/07-monitoreo.md` |
| Confirmar la zona DNS activa (`nodosur.tas` vs `conosurtas05.lan`) | `bitacora/01-servidores/03-dns_bind9.md` |
| Sacar las contraseñas de los compose y de `promote.sh` a archivos no versionados | Seguridad |
| Limitar el export NFS (sin `no_root_squash`) y el puerto 3000 de Grafana | Seguridad · firewall |
| Fijar versiones de imagen en vez de `latest` | `decisiones/02-dockerizacion.md` |
| Activar MFA en WordPress y documentarlo | Seguridad |
| Deshabilitar el acceso SSH por contraseña y limitar el puerto 22 a la red de gestión | Seguridad · firewall |
| Documentar por qué no se implementó Redis | `decisiones/` |
| Completar `decisiones/01-estructura_de_servicios.md` (vacío) | `decisiones/` |
| Completar `bitacora/03-situaciones_de_error/01-caida_por_balanceador.md` | Incidentes |
| Alinear el informe v1.1 con lo implementado: menciona Redis, logs por syslog, restic, MariaDB en Docker, Alertmanager y Blackbox | `docs/informe/entregas/` |
