# Decisión: Nginx en lugar de HAProxy como balanceador

**Fecha de decisión:** 07/10/2026 · **Estado:** aceptada · **Servidor afectado:** `tas05vm1`

## Contexto

En el informe de la entrega 1 se propuso **HAProxy** como balanceador frente a los dos backends WordPress (`tas05vm2` y `tas05vm3`). Al implementar la entrega 3 (publicación resiliente y seguridad) se probó HAProxy entre el 05/10 y el 07/10 y se comparó con **Nginx**, que el equipo ya estaba usando como servidor web en los backends.

El balanceador debía cumplir:

1. Publicar el sitio solo por HTTPS y redirigir HTTP → HTTPS.
2. Repartir las peticiones entre los dos backends.
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
| Reintento automático en otro backend | `option redispatch` / `retries` | `proxy_next_upstream` | Empate en la práctica |
| Servir contenido estático / páginas de error propias | Limitado | Sí | Ventaja de Nginx |
| Misma tecnología que los backends | No | Sí (los backends usan Nginx + PHP-FPM) | **Ventaja de Nginx**: una sola sintaxis, mismos logs, mismos comandos (`nginx -t`, `nginx -s reload`) |
| Recarga sin cortar conexiones | Sí | Sí | Empate |
| Métricas | Página de estadísticas muy completa | `stub_status` (básico) + logs | Ventaja de HAProxy |
| Documentación y ejemplos para WordPress | Menos | Abundante | Ventaja de Nginx |
| Curva de aprendizaje para el equipo | Nueva herramienta | Ya conocida | **Ventaja de Nginx** |

## Decisión

Se eligió **Nginx** como balanceador y proxy inverso.

## Justificación

- **Una sola tecnología.** Los backends ya usan Nginx delante de PHP-FPM (ver `decisiones/04-servidor_web.md`). Usar Nginx también en el balanceador significa que cualquier integrante puede leer, validar y diagnosticar todas las capas con la misma sintaxis y el mismo formato de logs. Esto apoya el requisito del curso de que todo comando pueda ser explicado por cualquier integrante.
- **Cubre los requisitos.** Con `upstream`, `least_conn`, `max_fails`/`fail_timeout` y `proxy_next_upstream` se reparte la carga y, si un backend cae, la petición se reintenta en el otro. Esto se validó el 19/10/2026 deteniendo cada backend con tráfico continuo: las respuestas siguieron en `200`.
- **Menos configuración para TLS y cabeceras.** Redirección a HTTPS, HSTS y cabeceras `X-Forwarded-*` quedan en un solo archivo corto.
- **Escala de la plataforma.** Con dos backends y el tráfico esperado de una pyme, el mayor rendimiento de HAProxy en escenarios de alta concurrencia no es un factor decisivo.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Sin health checks activos: un backend caído se detecta con la primera petición que falla | `proxy_next_upstream` reintenta esa misma petición en el otro backend, así que el usuario no ve el error; además Blackbox exporter en vm5 prueba `/healthz` de cada backend cada 15 s y alerta en ~1 min |
| Métricas del balanceador más pobres que las de HAProxy | Se usa `stub_status` (restringido a vm5) y el `access.log`/`error.log` |
| El balanceador sigue siendo punto único de falla | Igual que con HAProxy. Mejora futura: segundo Nginx con `keepalived` (VRRP) e IP virtual |

## Cuándo reconsiderar

Volver a evaluar HAProxy si la plataforma crece a más backends, si se requiere balanceo TCP (por ejemplo para MariaDB con réplicas) o si se necesitan health checks activos sin depender del monitoreo externo.
