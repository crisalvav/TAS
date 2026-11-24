# Caída de la navegación por el certificado del balanceador (13/10/2026 - 14/10/2026)

**Servidor:** `tas05vm1` (10.33.199.42 · gestión 10.33.195.230) · **Servicio afectado:** Nginx balanceador (`wp_nginx`), puerto 443 · **Impacto:** no se podía llegar a la página por HTTPS

## Contexto

El balanceador se había empezado a configurar con HAProxy (prueba del 05/10 al 06/10) y luego se decidió usar Nginx (ver `decisiones/03-balanceador.md`). El certificado TLS se generó el 07/10 y se monta en el contenedor como `certs/cert.pem` y `certs/key.pem` (ver `bitacora/01-servidores/06-nginx_balanceador_https.md`).

## Qué se observó

Durante las pruebas del 13/10 no se lograba llegar a la página a través del balanceador.

- Inicio: _por completar (fecha y hora)_
- Cómo se detectó: _por completar_

## Causa

Un problema con el certificado TLS durante el cambio de HAProxy a Nginx.

- Detalle: _por completar (qué tenía mal el certificado o su configuración)_

## Solución

- _por completar (qué se cambió y comandos usados)_

Para validar:

```bash
sudo docker exec wp_nginx nginx -t
sudo docker logs wp_nginx --tail 20
curl -skI https://10.33.195.230                          # 200 o 301 de WordPress
openssl s_client -connect 10.33.195.230:443 -servername conosurtas05.lan </dev/null | grep -E 'Protocol|subject'
```

- Fin: _por completar (fecha y hora)_ · Duración aproximada: _por completar_

## Lecciones

- Al cambiar de tecnología se debe revisar que el certificado esté en el formato y en la ruta que espera la nueva herramienta (Nginx usa `ssl_certificate` y `ssl_certificate_key` por separado).
- Validar con `nginx -t` y con `curl`/`openssl` antes de dar por publicado el sitio.
