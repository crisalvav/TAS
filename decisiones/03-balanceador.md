# Decisión: Nginx en lugar de HAProxy como balanceador

**Fecha de decisión:** 06/10/2026 · **Estado:** aceptada · **Servidor afectado:** `tas05vm1`

## Contexto

En el informe de la entrega 1 se propuso **HAProxy** como balanceador frente a los dos backends WordPress (`tas05vm2` y `tas05vm3`). Al implementar la entrega 3 (publicación resiliente y seguridad) se probó HAProxy entre el 05/10 y el 06/10 y se comparó con **Nginx**.

El balanceador debía cumplir:

1. Publicar el sitio solo por HTTPS y redirigir HTTP → HTTPS.
2. Repartir las peticiones entre los dos backends en proporción 1:1.
3. Dejar de enviar tráfico a un backend caído sin que el usuario lo note.
4. Enviar a WordPress las cabeceras del cliente (`X-Forwarded-For`, `X-Forwarded-Proto`).
5. Exponer métricas para el monitoreo.
6. Ser mantenible por un equipo de 3 personas con poco tiempo.

## Comparación

| Criterio | HAProxy | Nginx | Comentario |
| --- | --- | --- | --- |
| Balanceo HTTP (round robin, least_conn) | Sí | Sí | Empate |
| Terminación TLS y redirección HTTP→HTTPS | Sí | Sí | En Nginx se hace con menos líneas y con sintaxis que el equipo ya conocía |
| Health checks **activos** | Sí (nativo) | No en la versión open source (solo pasivos con `max_fails`) | **Ventaja de HAProxy** |
| Reintento automático en otro backend | `option redispatch` / `retries` | `proxy_next_upstream` (activo por defecto para `error`/`timeout`) | Empate en la práctica |
| Servir contenido estático / páginas de error propias | Limitado | Sí | Ventaja de Nginx |
| Recarga sin cortar conexiones | Sí | Sí | Empate |
| Métricas | Página de estadísticas muy completa | `stub_status` (básico) + logs | Ventaja de HAProxy |
| Documentación y ejemplos para WordPress | Menos | Abundante | Ventaja de Nginx |
| Curva de aprendizaje para el equipo | Nueva herramienta | Ya conocida | **Ventaja de Nginx** |

## Decisión

Se eligió **Nginx** como balanceador y proxy inverso.

## Justificación

- **Conocimiento del equipo.** El equipo ya conocía la sintaxis de Nginx, así que cualquier integrante puede leer, validar (`nginx -t`) y diagnosticar el balanceador. Esto apoya el requisito del curso de que todo comando pueda ser explicado por cualquier integrante.
- **Cubre los requisitos.** Con `upstream` en round robin (1:1 por defecto) y `max_fails`/`fail_timeout` se reparte la carga y, si un backend cae, la petición se reintenta en el otro. Se validó el 23/10/2026 (ver `bitacora/04-pruebas/06-balanceo_1a1.md` y `07-caida_de_un_backend.md`).
- **Menos configuración para TLS y cabeceras.** Redirección a HTTPS y cabeceras `X-Forwarded-*` quedan en un solo archivo corto (`~/wordpress-docker/nginx/default.conf`).
- **Escala de la plataforma.** Con dos backends y el tráfico esperado de una pyme, el mayor rendimiento de HAProxy en escenarios de alta concurrencia no es un factor decisivo.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Sin health checks activos: un backend caído se detecta con la primera petición que falla | Nginx reintenta esa misma petición en el otro backend, así que el usuario no ve el error (solo una respuesta más lenta). node_exporter en vm5 muestra si la VM está arriba, pero no si el contenedor responde: **pendiente** agregar una sonda HTTP |
| Métricas del balanceador más pobres que las de HAProxy | Por ahora solo `docker logs wp_nginx`. **Pendiente:** `stub_status` + exporter de Nginx en la fase 2 del monitoreo |
| El balanceador sigue siendo punto único de falla | Igual que con HAProxy. Mejora futura: segundo Nginx con `keepalived` (VRRP) e IP virtual |

## Cuándo reconsiderar

Volver a evaluar HAProxy si la plataforma crece a más backends, si se requiere balanceo TCP (por ejemplo para MariaDB con réplicas) o si se necesitan health checks activos sin depender del monitoreo externo.
