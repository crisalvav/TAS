# Decisión: Nginx en lugar de Apache como servidor web

**Fecha de decisión:** 25/09/2026 · **Estado:** aceptada · **Servidores afectados:** `tas05vm2`, `tas05vm3` (y por consecuencia `tas05vm1`)

## Contexto

WordPress puede servirse principalmente de dos formas en contenedores:

- **Apache + mod_php**: imagen oficial `wordpress:<versión>-apache`. Es la opción "por defecto" y funciona sin configurar el servidor web.
- **Nginx + PHP-FPM**: imagen oficial `wordpress:<versión>-fpm` más un contenedor `nginx` que atiende HTTP y pasa los `.php` a PHP-FPM.

Al preparar los backends (25/09/2026) se compararon ambas.

## Comparación

| Criterio | Apache (mod_php) | Nginx + PHP-FPM | Comentario |
| --- | --- | --- | --- |
| Facilidad de puesta en marcha | Muy alta (un solo contenedor) | Media (dos contenedores y un archivo de configuración) | Ventaja de Apache |
| Uso de memoria con muchas conexiones | Mayor: cada proceso/hilo carga PHP aunque sirva una imagen | Menor: Nginx sirve estáticos sin PHP y PHP-FPM solo atiende `.php` | **Ventaja de Nginx**, relevante con VM de recursos limitados |
| Archivos estáticos (imágenes del catálogo, CSS, JS) | Correcto | Muy eficiente | Ventaja de Nginx |
| `.htaccess` | Sí (lo usan algunos plugins) | No; las reglas van en la configuración del servidor | Ventaja de Apache en compatibilidad; en Nginx las reglas de WordPress son solo `try_files` |
| Separar servidor web y PHP | No | Sí: se pueden reiniciar o escalar por separado | Ventaja de Nginx |
| Uso como proxy inverso / balanceador | Posible (`mod_proxy_balancer`) | Nativo y muy usado | **Ventaja de Nginx**: permite usar la misma herramienta en vm1 |
| Seguridad / superficie | Más módulos cargados por defecto | Configuración mínima y explícita | Ventaja de Nginx |
| Configuración centralizada | Repartida entre `.conf` y `.htaccess` | Toda en un archivo versionado | **Ventaja de Nginx**: es reproducible desde el repositorio |

## Decisión

Se eligió **Nginx + PHP-FPM** para los backends y, por consecuencia, **Nginx** también para el balanceador (ver `decisiones/03-balanceador.md`).

## Justificación

- **Recursos.** Las VM tienen recursos acotados. Nginx atiende las imágenes y archivos estáticos de WooCommerce sin cargar PHP, y PHP-FPM solo trabaja en las páginas dinámicas, lo que deja más memoria disponible para cada backend.
- **Configuración reproducible.** Toda la configuración del servidor web queda en `nginx/default.conf`, versionada en el repositorio, en lugar de depender de archivos `.htaccess` que WordPress o los plugins pueden modificar desde el panel.
- **Un solo stack en toda la plataforma.** Al usar Nginx en los backends, el equipo ya conocía su sintaxis al momento de elegir el balanceador. Así hay una sola herramienta que aprender, validar (`nginx -t`) y diagnosticar en las tres capas web.
- **Health check simple.** En la misma configuración se agregó `location = /healthz`, que usan el balanceador y el monitoreo sin pasar por PHP ni por la base de datos.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Dos contenedores por backend en vez de uno | Ambos en el mismo `docker-compose.yaml`; se levantan y detienen juntos |
| Plugins que dependen de `.htaccess` no aplican sus reglas | Revisar cada plugin antes de instalarlo; traducir sus reglas a la configuración de Nginx si es necesario |
| Ambos contenedores deben ver los mismos archivos | Volumen compartido `wp_core` y montaje NFS de `wp-content` en los dos (Nginx en solo lectura) |

## Cuándo reconsiderar

Si se necesitara un plugin crítico para el negocio que dependa fuertemente de `.htaccess` y no tenga equivalente, se puede volver a la imagen `-apache` en los backends manteniendo Nginx como balanceador.
