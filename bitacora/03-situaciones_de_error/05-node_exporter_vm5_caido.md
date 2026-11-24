# Prometheus no podía leer las métricas de la vm5 (23/10/2026)

**Servidor:** `tas05vm5` (10.33.199.46) · **Servicio afectado:** `prometheus-node-exporter` · **Impacto:** sin métricas de sistema de la vm5 en el panel

## Qué se observó

Tras levantar Prometheus y Grafana en la vm5, uno de los 6 objetivos aparecía caído:

```bash
curl -s localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"' | sort | uniq -c
#   1 "health":"down"
#   5 "health":"up"
```

Para identificarlo:

```bash
curl -s localhost:9090/api/v1/targets | python3 -c "
import sys, json
for t in json.load(sys.stdin)['data']['activeTargets']:
    print(t['health'], t['scrapeUrl'], t.get('lastError',''))
"
```

```
down http://10.33.199.46:9100/metrics Get "http://10.33.199.46:9100/metrics": dial tcp 10.33.199.46:9100: connect: connection refused
```

## Causa

`connection refused` (y no un timeout) indica que nadie escuchaba en `10.33.199.46:9100`: node_exporter no estaba escuchando en la IP interna de la vm5. No era un problema de firewall.

## Solución

```bash
sudo systemctl status prometheus-node-exporter --no-pager | head -8
sudo ss -tlnp | grep 9100
```

Se dejó escuchando en todas las interfaces editando `/etc/default/prometheus-node-exporter`:

```ini
ARGS="--web.listen-address=:9100"
```

```bash
sudo systemctl restart prometheus-node-exporter
```

Como Prometheus corre en un contenedor y llega al host por la red de Docker, el firewall de la vm5 también debe permitir esa red:

```bash
sudo ufw allow from 172.16.0.0/12 to any port 9100 proto tcp
```

Tras un ciclo de lectura (15 s) aparecieron los 6 objetivos en `up` (ver `bitacora/04-pruebas/08-monitoreo.md`).

## Lecciones

- `connection refused` → el servicio no escucha; `timeout` → firewall o IP incorrecta.
- El monitor también se monitorea a sí mismo, y su acceso pasa por la red de Docker, no por la IP interna.
