# Prueba: balanceo 1:1 entre vm2 y vm3 (23/10/2026)

**Servidor:** `tas05vm1` (10.33.195.230 / 10.33.199.42) · **Servicio:** Nginx (`wp_nginx`, servicio `webserver` en `~/wordpress-docker`) · **Objetivo:** repartir las peticiones en proporción 1:1 entre los dos backends

## Configuración probada

`~/wordpress-docker/nginx/default.conf`:

```nginx
upstream server_backend {
    # Round robin 1:1 (peso igual por defecto)
    server 10.33.199.43:9000 max_fails=2 fail_timeout=10s;   # vm2
    server 10.33.199.44:9000 max_fails=2 fail_timeout=10s;   # vm3
    keepalive 16;
}
```

Dentro de `location /` se agregaron `proxy_http_version 1.1;` y `proxy_set_header Connection "";`. Para la prueba, en el `server` de 443 se agregó de forma temporal:

```nginx
add_header X-Backend $upstream_addr always;
```

Aplicación (el compose monta un solo archivo, así que tras editar con nano se reinicia el contenedor en vez de solo recargar):

```bash
cd ~/wordpress-docker
sudo docker restart wp_nginx
sudo docker exec wp_nginx nginx -t
sudo docker logs wp_nginx --tail 20
```

## Procedimiento

Desde un PC del equipo (shell fish):

```fish
for i in (seq 6); curl -skI https://10.33.195.230 | grep -i x-backend; end
```

En bash: `for i in 1 2 3 4 5 6; do curl -skI https://10.33.195.230 | grep -i x-backend; done`

## Resultado

```
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.44:9000
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.44:9000
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.44:9000
```

Alternancia exacta vm2/vm3 → balanceo 1:1 ✅

## Observaciones

- Al terminar las pruebas se debe quitar `add_header X-Backend` y reiniciar `wp_nginx`.
