# Decisión: Apache (imagen `wordpress:latest`) como servidor web de los backends

**Fecha de decisión:** 21/10/2026 · **Estado:** aceptada · **Servidores afectados:** `tas05vm2`, `tas05vm3`

## Contexto

WordPress puede servirse principalmente de dos formas en contenedores:

- **Apache + mod_php**: imagen oficial `wordpress:latest` (o `wordpress:<versión>-apache`). Un solo contenedor que atiende HTTP y PHP.
- **Nginx + PHP-FPM**: imagen oficial `wordpress:<versión>-fpm` más un contenedor `nginx` que atiende HTTP y pasa los `.php` a PHP-FPM.

Hasta el 21/10 los backends no eran iguales: `tas05vm2` usaba `wordpress:latest` (Apache) y `tas05vm3` usaba `wordpress:fpm-alpine` con un `nginx:alpine` delante. Al compartir los archivos por NFS había que dejar ambos con la misma imagen.

## Comparación

| Criterio | Apache (mod_php) | Nginx + PHP-FPM | Comentario |
| --- | --- | --- | --- |
| Facilidad de puesta en marcha | Muy alta (un solo contenedor) | Media (dos contenedores y un archivo de configuración) | **Ventaja de Apache** |
| Uso de memoria con muchas conexiones | Mayor: cada proceso carga PHP aunque sirva una imagen | Menor: Nginx sirve estáticos sin PHP | Ventaja de Nginx |
| Archivos estáticos | Correcto | Muy eficiente | Ventaja de Nginx |
| `.htaccess` | Sí (lo usan WordPress y algunos plugins) | No; las reglas van en la configuración del servidor | **Ventaja de Apache** |
| Ambos contenedores deben ver los mismos archivos | No aplica (un contenedor) | Nginx y PHP-FPM deben montar el mismo NFS | **Ventaja de Apache**: en la vm3 el nginx montaba un volumen vacío y los estáticos daban 404 |
| Usuario de los archivos en el NFS | UID 33 (imagen Debian) | UID 82 en la variante Alpine | **Ventaja de Apache**: el mismo UID que ya usaba la vm2 |
| Instalación existente | La de la vm2 ya corría con esta imagen | Había que migrar la vm2 | Ventaja de Apache |

## Decisión

Se usa **`wordpress:latest` (Apache)** en ambos backends, en un único contenedor `wp_app` publicado en `9000:80`. Nginx queda solo como balanceador en `tas05vm1`.

## Justificación

- **Backends idénticos.** Con el NFS compartido ambos backends deben escribir los archivos con el mismo usuario (UID 33). Se igualó la vm3 a la vm2 en vez de lo contrario porque la vm2 ya tenía el sitio migrado.
- **Menos piezas.** Un contenedor por backend en vez de dos; no hay un segundo Nginx que configurar ni que montar el NFS.
- **El balanceo y TLS ya están en vm1.** El Nginx de la vm3 sobraba, porque la terminación TLS y el reparto se hacen en `tas05vm1`.
- **Compatibilidad.** WordPress y sus plugins funcionan con `.htaccess` sin traducir reglas.

## Consecuencias y cómo se mitigan

| Consecuencia | Mitigación |
| --- | --- |
| Más memoria por conexión que Nginx + PHP-FPM | Con el tráfico esperado y dos backends no es un problema; se vigila con node_exporter en Grafana |
| `wordpress:latest` cambia de versión sin aviso | **Pendiente:** fijar la versión de la imagen |
| Distinta tecnología en vm1 (Nginx) y en los backends (Apache) | El diagnóstico de los backends se hace con `docker logs wp_app` |

## Cuándo reconsiderar

Si la carga crece y la memoria de los backends se vuelve un límite, evaluar Nginx + PHP-FPM con la imagen Debian (`wordpress:<versión>-fpm`, UID 33), montando el NFS en ambos contenedores.
