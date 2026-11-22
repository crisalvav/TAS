# Instalación del monitoreo con Prometheus y Grafana en `tas05vm5` (15/10/2026 - 20/10/2026)

**Servidor:** `tas05vm5` (10.33.199.46) · **Servicios:** Prometheus, Grafana, Alertmanager, Blackbox exporter (contenedores) · **Agentes:** node_exporter en todas las VM

## 15/10/2026 — node_exporter en todas las VM

Se instala en las 5 VM para obtener CPU, memoria, disco y red:

```bash
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
curl -s localhost:9100/metrics | head
sudo ufw allow from 10.33.199.46 to any port 9100 proto tcp   # solo el monitor
```

## 15/10/2026 — Exporter de MariaDB en `tas05vm4`

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY '<contraseña>' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

```bash
sudo apt install -y prometheus-mysqld-exporter
# credenciales en /etc/default/prometheus-mysqld-exporter o ~/.my.cnf del usuario prometheus
sudo systemctl enable --now prometheus-mysqld-exporter
sudo ufw allow from 10.33.199.46 to any port 9104 proto tcp
```

## 15/10/2026 — Estructura en `tas05vm5`

```bash
mkdir -p ~/servicios/monitoreo/{prometheus,alertmanager,blackbox,grafana}
cd ~/servicios/monitoreo
```

## 16/10/2026 — Configuración de Prometheus

`prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alertas.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: nodos
    static_configs:
      - targets:
          - 10.33.199.42:9100
          - 10.33.199.43:9100
          - 10.33.199.44:9100
          - 10.33.199.45:9100
          - 10.33.199.46:9100

  - job_name: mariadb
    static_configs:
      - targets: ['10.33.199.45:9104']

  - job_name: web
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://www.nodosur.tas
          - http://10.33.199.43:8000/healthz
          - http://10.33.199.44:8000/healthz
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

## 16/10/2026 — Reglas de alerta

`prometheus/alertas.yml` (alertas en menos de 5 minutos, según requerimiento):

```yaml
groups:
  - name: nodosur
    rules:
      - alert: NodoCaido
        expr: up == 0
        for: 1m
        labels: { severity: critica }
      - alert: SitioNoResponde
        expr: probe_success == 0
        for: 1m
        labels: { severity: critica }
      - alert: DiscoCasiLleno
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.15
        for: 5m
        labels: { severity: advertencia }
      - alert: CPUAlta
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels: { severity: advertencia }
      - alert: CertificadoPorVencer
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 15
        for: 1h
        labels: { severity: advertencia }
```

## 19/10/2026 — Alertmanager y Blackbox

`alertmanager/alertmanager.yml` (canal de correo; se puede cambiar por Telegram/Discord):

```yaml
route:
  receiver: equipo
  group_wait: 30s
  repeat_interval: 2h
receivers:
  - name: equipo
    email_configs:
      - to: '<correo_del_equipo>'
        from: '<correo_alertas>'
        smarthost: 'smtp.gmail.com:587'
        auth_username: '<correo_alertas>'
        auth_password: '<app_password>'
```

`blackbox/blackbox.yml`:

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      preferred_ip_protocol: ip4
      tls_config:
        ca_file: /etc/blackbox/ca.crt
```

(`ca.crt` es la CA propia generada en la etapa del balanceador.)

## 19/10/2026 — Docker Compose

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.54.1
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - prom_data:/prometheus
    ports: ["9090:9090"]

  alertmanager:
    image: prom/alertmanager:v0.27.0
    restart: unless-stopped
    volumes:
      - ./alertmanager:/etc/alertmanager:ro
    ports: ["9093:9093"]

  blackbox:
    image: prom/blackbox-exporter:v0.25.0
    restart: unless-stopped
    command: --config.file=/etc/blackbox/blackbox.yml
    volumes:
      - ./blackbox:/etc/blackbox:ro

  grafana:
    image: grafana/grafana:11.2.0
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin
    volumes:
      - grafana_data:/var/lib/grafana
    ports: ["3000:3000"]

volumes:
  prom_data:
  grafana_data:
```

```bash
docker run --rm -v $PWD/prometheus:/p --entrypoint promtool prom/prometheus:v2.54.1 check config /p/prometheus.yml
docker compose up -d
docker compose ps
```

## 19/10/2026 — Firewall del monitor

Paneles accesibles solo desde la red interna:

```bash
sudo ufw allow from 10.33.199.0/24 to any port 3000 proto tcp
sudo ufw allow from 10.33.199.0/24 to any port 9090 proto tcp
```

## 20/10/2026 — Grafana

1. Se ingresa a `http://10.33.199.46:3000`, se agrega Prometheus como fuente de datos (`http://prometheus:9090`).
2. Se importan los dashboards **Node Exporter Full** (ID 1860) y **MySQL Overview** (ID 7362).
3. Se crea un panel propio con `probe_success`, `probe_duration_seconds` y días restantes del certificado.

## 20/10/2026 — Pruebas de alertas

| Prueba | Acción | Resultado esperado |
| --- | --- | --- |
| Backend caído | `docker compose stop` en vm2 | Alerta `SitioNoResponde` para `10.33.199.43:8000/healthz` en ~1-2 min |
| Nodo caído | `sudo systemctl stop prometheus-node-exporter` en vm3 | Alerta `NodoCaido` |
| Disco lleno | `fallocate -l 10G /tmp/relleno` en vm4 | Alerta `DiscoCasiLleno` |

Se registra en cada caso la hora de la falla y la hora de llegada del correo.

## 20/10/2026 — Logs

Se revisan los logs relevantes y se dejan ubicados para la entrega de observabilidad:

```bash
# vm1
tail -f ~/servicios/nginx/logs/access.log ~/servicios/nginx/logs/error.log
# vm2 / vm3
docker compose logs --since 1h wordpress
# todas
sudo journalctl -u ssh --since today
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/ufw.log
```

## Resultado

- Métricas de las 5 VM, de MariaDB y de disponibilidad del sitio con retención de 30 días.
- Alertas por correo en menos de 5 minutos.
- **Punto único de falla:** el nodo de monitoreo.
