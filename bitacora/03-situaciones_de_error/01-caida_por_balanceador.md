# Caída de la navegación por el certificado del balanceador (13/10/2026 - 14/10/2026)

**Servidor:** `tas05vm1` (10.33.199.42) · **Servicio afectado:** Nginx balanceador (`nginx-lb`), puerto 443 · **Impacto:** no se podía llegar a la página `https://www.nodosur.tas`

## Contexto

El balanceador se había empezado a configurar con HAProxy (prueba del 05/10 al 06/10) y luego se decidió usar Nginx (ver `decisiones/03-balanceador.md`). El certificado TLS se generó el 07/10 con una CA propia (ver `bitacora/01-servidores/06-nginx_balanceador_https.md`).

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
docker exec nginx-lb nginx -t
curl --cacert certs/ca.crt -I https://www.nodosur.tas   # 200
openssl s_client -connect www.nodosur.tas:443 -servername www.nodosur.tas </dev/null | grep -E 'Protocol|subject'
```

- Fin: _por completar (fecha y hora)_ · Duración aproximada: _por completar_

## Lecciones

- Al cambiar de tecnología se debe revisar que el certificado esté en el formato y en la ruta que espera la nueva herramienta (Nginx usa `ssl_certificate` y `ssl_certificate_key` por separado).
- Validar con `nginx -t` y con `curl`/`openssl` antes de dar por publicado el sitio.
