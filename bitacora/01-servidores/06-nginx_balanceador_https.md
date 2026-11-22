# Instalación del balanceador Nginx con HTTPS en `tas05vm1` (05/10/2026 - 14/10/2026)

**Servidor:** `tas05vm1` (10.33.199.42) · **Servicio:** Nginx como proxy inverso y balanceador en contenedor · **Puertos:** 80 y 443

> Inicialmente se planificó HAProxy (informe de la entrega 1). Entre el 05/10 y el 06/10 se hizo una prueba con HAProxy y luego se migró a Nginx. La justificación completa está en `decisiones/03-balanceador.md` y `decisiones/04-servidor_web.md`.

## 05/10/2026 — Prueba inicial con HAProxy

```bash
mkdir -p ~/pruebas/haproxy && cd ~/pruebas/haproxy
docker run -d --name haproxy-test -p 8080:80 -v $PWD/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy:2.9
```

Se logra balancear HTTP entre ambos backends, pero la configuración de TLS, redirección y cabeceras se vuelve más larga que la equivalente en Nginx, que el equipo ya usaba en los backends.

## 06/10/2026 — Decisión y limpieza

```bash
docker rm -f haproxy-test
```

Se decide usar Nginx para mantener una sola tecnología de servidor web/proxy en toda la plataforma.

## 06/10/2026 — Estructura del servicio

```bash
mkdir -p ~/servicios/nginx/{conf.d,certs,logs}
cd ~/servicios/nginx
```

## 07/10/2026 — Certificado TLS

Como `nodosur.tas` es un dominio interno, no se puede usar Let's Encrypt. Se genera una CA propia y un certificado firmado por ella:

```bash
cd certs
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 825 -out ca.crt -subj "/CN=NodoSur CA"
openssl genrsa -out nodosur.key 2048
openssl req -new -key nodosur.key -out nodosur.csr -subj "/CN=www.nodosur.tas"
printf "subjectAltName=DNS:www.nodosur.tas,DNS:nodosur.tas\n" > san.ext
openssl x509 -req -in nodosur.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out nodosur.crt -days 397 -sha256 -extfile san.ext
chmod 600 *.key
```

`ca.crt` se instala en los equipos cliente para que el navegador no muestre advertencias. Las llaves privadas no se versionan.

## 08/10/2026 — Configuración del balanceador

`conf.d/nodosur.conf`:

```nginx
upstream wordpress_backends {
    least_conn;
    server 10.33.199.43:8000 max_fails=3 fail_timeout=10s;
    server 10.33.199.44:8000 max_fails=3 fail_timeout=10s;
    keepalive 16;
}

server {
    listen 80;
    server_name www.nodosur.tas nodosur.tas;
    return 301 https://www.nodosur.tas$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name www.nodosur.tas;

    ssl_certificate     /etc/nginx/certs/nodosur.crt;
    ssl_certificate_key /etc/nginx/certs/nodosur.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;
    client_max_body_size 64M;

    location / {
        proxy_pass http://wordpress_backends;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;
    }

    location = /nginx_status {
        stub_status;
        allow 10.33.199.46;
        deny all;
    }
}
```

- `least_conn`: envía cada petición al backend con menos conexiones activas.
- `max_fails` / `fail_timeout`: si un backend falla 3 veces en 10 s, se retira durante 10 s.
- `proxy_next_upstream`: si un backend no responde, la misma petición se reintenta en el otro, sin que el usuario vea el error.

## 08/10/2026 — Docker Compose

`docker-compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx-lb
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./logs:/var/log/nginx
```

```bash
docker compose run --rm nginx nginx -t
docker compose up -d
docker logs nginx-lb
```

## 09/10/2026 — Firewall

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status numbered
```

_12/10 feriado, sin avance._

## 13/10/2026 — Pruebas de publicación

```bash
curl -I http://www.nodosur.tas                        # 301 -> https
curl --cacert certs/ca.crt -I https://www.nodosur.tas # 200
openssl s_client -connect www.nodosur.tas:443 -servername www.nodosur.tas </dev/null | grep -E 'Protocol|subject'
```

## 13/10/2026 — Prueba de caída de un backend

1. Se lanza tráfico continuo desde vm5:

   ```bash
   while true; do curl -sk -o /dev/null -w "%{http_code} %{time_total}\n" https://www.nodosur.tas; sleep 0.5; done
   ```

2. Se detiene el backend 1 (`docker compose stop` en `tas05vm2`). Se observa que las respuestas siguen en `200`; el log de Nginx registra el error de conexión hacia `10.33.199.43:8000` y el tráfico continúa por `10.33.199.44`.
3. Se vuelve a levantar el backend 1 y, pasado `fail_timeout`, vuelve a recibir tráfico.
4. Se repite la prueba con el backend 2.

## 13/10/2026 - 14/10/2026 — Caída de la navegación

Durante las pruebas se cayó la navegación del sitio a través del balanceador. El análisis está en `bitacora/03-situaciones_de_error/01-caida_por_balanceador.md`.

## 14/10/2026 — Recarga sin corte

Cualquier cambio de configuración se aplica validando primero y recargando en caliente (sin cortar conexiones):

```bash
docker exec nginx-lb nginx -t && docker exec nginx-lb nginx -s reload
```

## Resultado

- Sitio publicado solo por HTTPS en `https://www.nodosur.tas`, con redirección desde HTTP.
- Balanceo entre los dos backends y retiro automático del backend caído.
- **Punto único de falla:** el propio balanceador (un solo nodo). Mejora futura: segundo Nginx con `keepalived` (VRRP).
- **Limitación conocida:** Nginx open source no hace health checks activos; detecta la caída con tráfico real (chequeo pasivo). Se complementa con el monitoreo de vm5.
