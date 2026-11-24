# Instalación del monitoreo con Prometheus y Grafana en `tas05vm5` (23/10/2026)

**Servidor:** `tas05vm5` (10.33.199.46 · gestión 10.33.195.247) · **Servicios:** Prometheus y Grafana en contenedores (`~/monitoring`) · **Agentes:** `prometheus-node-exporter` en las 5 VM

Se eligió Prometheus + Grafana, el stack habitual en producción. Se arma en dos fases: primero las métricas de sistema de las 5 VM (esta bitácora) y después contenedores, Nginx, MariaDB y logs.

## 1. Docker y firewall en `tas05vm5`

```bash
docker --version || sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo ufw allow 3000/tcp                                            # Grafana
sudo ufw allow from 172.16.0.0/12 to any port 9100 proto tcp       # Prometheus (contenedor) -> node_exporter del host
```

## 2. Stack de monitoreo

```bash
mkdir -p ~/monitoring/grafana/provisioning/datasources && cd ~/monitoring
```

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: nodes
    static_configs:
      - targets: ['10.33.199.42:9100']
        labels: {vm: vm1-nginx-dns}
      - targets: ['10.33.199.43:9100']
        labels: {vm: vm2-wordpress}
      - targets: ['10.33.199.44:9100']
        labels: {vm: vm3-wordpress}
      - targets: ['10.33.199.45:9100']
        labels: {vm: vm4-mariadb}
      - targets: ['10.33.199.46:9100']
        labels: {vm: vm5-panel}
```

`grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

`docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    environment:
      GF_SECURITY_ADMIN_PASSWORD: <clave_grafana>
    volumes:
      - graf_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  prom_data:
  graf_data:
```

```bash
sudo docker compose up -d
```

## 3. node_exporter en las 5 VM

```bash
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
curl -s localhost:9100/metrics | head -3
sudo ufw allow from 10.33.199.46 to any port 9100 proto tcp   # solo el monitor (no hace falta en vm5)
```

En la vm5 hubo que dejarlo escuchando en todas las interfaces (`ARGS="--web.listen-address=:9100"` en `/etc/default/prometheus-node-exporter`). Ver `bitacora/03-situaciones_de_error/05-node_exporter_vm5_caido.md`.

## 4. Verificación

```bash
sudo docker ps
curl -s localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"' | sort | uniq -c   # 6 "health":"up"
```

## 5. Grafana

1. `http://10.33.195.247:3000`, usuario `admin`.
2. Dashboards → New → Import → ID `1860` (Node Exporter Full) → fuente de datos Prometheus.
3. El filtro de instancia permite cambiar entre las 5 VM.

Prueba en `bitacora/04-pruebas/08-monitoreo.md`.

## 6. Logs (consulta manual)

Mientras no esté Loki, los logs se revisan en cada VM:

```bash
# vm1
sudo docker logs wp_nginx --since 1h
# vm2 / vm3
sudo docker logs wp_app --since 1h
# vm4 / vm5
sudo journalctl -u mariadb -u keepalived --since today
# todas
sudo journalctl -u ssh --since today
sudo tail -f /var/log/auth.log /var/log/ufw.log
```

## Resultado

- Métricas de sistema de las 5 VM, con retención de 15 días, visibles en Grafana.
- Prometheus solo accesible desde la propia vm5 (`127.0.0.1:9090`).
- **Punto único de falla:** el nodo de monitoreo, que además hoy es la base principal.

## Pendiente (fase 2)

- `mysqld_exporter` en vm4 y vm5, con alerta si se corta la replicación.
- cAdvisor en vm2 y vm3 (contenedores) y exporter de Nginx en vm1 (`stub_status`).
- Loki + Promtail para ver los logs de todas las VM en Grafana.
- Alertmanager para avisar las alertas.
- Fijar versiones de imagen en vez de `latest`, pasar la clave de Grafana a un `.env` y limitar el 3000 a la red de gestión.
