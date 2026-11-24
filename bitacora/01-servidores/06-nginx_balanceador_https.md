# Instalación del balanceador Nginx con HTTPS en `tas05vm1` (05/10/2026 - 23/10/2026)

**Servidor:** `tas05vm1` (10.33.199.42 · gestión 10.33.195.230) · **Servicio:** Nginx como proxy inverso y balanceador en contenedor (`wp_nginx`, servicio `webserver`) · **Puertos:** 80 y 443

> Inicialmente se planificó HAProxy (informe de la entrega 1). Entre el 05/10 y el 06/10 se hizo una prueba con HAProxy y luego se migró a Nginx. La justificación completa está en `decisiones/03-balanceador.md`.

## 05/10/2026 — Prueba inicial con HAProxy

```bash
mkdir -p ~/pruebas/haproxy && cd ~/pruebas/haproxy
docker run -d --name haproxy-test -p 8080:80 -v $PWD/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy:2.9
```

Se logra balancear HTTP entre ambos backends, pero la configuración de TLS, redirección y cabeceras se vuelve más larga que la equivalente en Nginx.

## 06/10/2026 — Decisión y limpieza

```bash
docker rm -f haproxy-test
```

## 06/10/2026 — Estructura del servicio

```
~/wordpress-docker/
├── docker-compose.yml
├── nginx/default.conf
└── certs/            # cert.pem y key.pem (no se versionan)
```

## 07/10/2026 — Certificado TLS

Como el dominio es interno, no se puede usar Let's Encrypt. Se usa un certificado propio, montado en el contenedor como `certs/cert.pem` y `certs/key.pem`.

Para revisar con qué nombre quedó emitido:

```bash
openssl x509 -in ~/wordpress-docker/certs/cert.pem -noout -subject -issuer -dates -ext subjectAltName
```

_Por completar: comandos exactos con que se generó (CA propia o autofirmado) y el CN/SAN._

## 08/10/2026 — Docker Compose

`~/wordpress-docker/docker-compose.yml`:

```yaml
services:
  webserver:
    image: nginx:alpine
    container_name: wp_nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
```

## 08/10/2026 — Configuración inicial

En la primera versión el `upstream` tenía solo la vm2, con la vm3 comentada:

```nginx
upstream server_backend {
    server 10.33.199.43:9000;
    # server 10.33.199.44:80;   # Descomentar cuando agreguen el segundo nodo
}
```

## 09/10/2026 — Firewall

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status numbered
```

_12/10 feriado, sin avance._

## 13/10/2026 - 14/10/2026 — Caída de la navegación

Durante las pruebas se cayó la navegación del sitio a través del balanceador. El análisis está en `bitacora/03-situaciones_de_error/01-caida_por_balanceador.md`.

## 23/10/2026 — Balanceo 1:1 entre vm2 y vm3

Requisito: repartir las peticiones en proporción 1:1. Es el round robin por defecto de Nginx, así que basta con declarar ambos backends con el mismo peso. Ambos publican ahora en el puerto 9000.

`~/wordpress-docker/nginx/default.conf`:

```nginx
upstream server_backend {
    # Round robin 1:1 (peso igual por defecto)
    server 10.33.199.43:9000 max_fails=2 fail_timeout=10s;   # vm2
    server 10.33.199.44:9000 max_fails=2 fail_timeout=10s;   # vm3
    keepalive 16;
}

# 1. Redirección HTTP a HTTPS
server {
    listen 80;
    server_name conosurtas05.lan www.conosurtas05.lan 10.33.195.230 _;
    return 301 https://$host$request_uri;
}

# 2. Servidor HTTPS con terminación SSL y balanceador
server {
    listen 443 ssl;
    server_name conosurtas05.lan www.conosurtas05.lan 10.33.195.230 _;

    ssl_certificate     /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 64M;

    location / {
        proxy_pass http://server_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

- Round robin: las peticiones se alternan vm2, vm3, vm2…
- `max_fails=2` / `fail_timeout=10s`: si un backend falla 2 veces, se retira durante 10 s.
- Si la conexión a un backend falla, Nginx reintenta la misma petición en el otro (comportamiento por defecto de `proxy_next_upstream`: `error timeout`).
- `keepalive 16` + `proxy_http_version 1.1` + `Connection ""`: reutiliza conexiones hacia los backends.

Aplicación: el compose monta un solo archivo; si se edita con nano cambia el inode y el contenedor sigue viendo la versión vieja, por eso se reinicia en vez de solo recargar:

```bash
cd ~/wordpress-docker
sudo docker restart wp_nginx        # o: sudo docker compose restart webserver
sudo docker exec wp_nginx nginx -t
sudo docker logs wp_nginx --tail 20
```

Pruebas en `bitacora/04-pruebas/06-balanceo_1a1.md` y `07-caida_de_un_backend.md`.

## Resultado

- Sitio publicado solo por HTTPS, con redirección desde HTTP.
- Balanceo 1:1 entre los dos backends y retiro automático del backend caído.
- **Punto único de falla:** el propio balanceador (un solo nodo). Mejora futura: segundo Nginx con `keepalived` (VRRP), igual que en la base de datos.
- **Limitación conocida:** Nginx open source no hace health checks activos; detecta la caída con tráfico real (chequeo pasivo).
- **Pendiente:** confirmar que la zona de BIND9 resuelve el dominio que usa Nginx (`conosurtas05.lan`); la bitácora de DNS documenta `nodosur.tas`.
