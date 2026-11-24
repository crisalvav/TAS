# Prueba: Prometheus lee las métricas de las 5 VM (23/10/2026)

**Servidor:** `tas05vm5` (10.33.195.247 / 10.33.199.46) · **Servicios:** Prometheus y Grafana en contenedores (`~/monitoring`), `prometheus-node-exporter` en las 5 VM · **Objetivo:** que todos los objetivos de Prometheus estén en `up` y se vean en Grafana

## Configuración probada

`~/monitoring/prometheus.yml` (lectura cada 15 s):

| Objetivo | Etiqueta `vm` |
| --- | --- |
| `localhost:9090` | Prometheus |
| `10.33.199.42:9100` | vm1-nginx-dns |
| `10.33.199.43:9100` | vm2-wordpress |
| `10.33.199.44:9100` | vm3-wordpress |
| `10.33.199.45:9100` | vm4-mariadb |
| `10.33.199.46:9100` | vm5-panel |

Prometheus publica en `127.0.0.1:9090` (retención 15 días) y Grafana en `3000`, con Prometheus como fuente de datos por provisioning. En cada VM: `sudo ufw allow from 10.33.199.46 to any port 9100 proto tcp`.

## Procedimiento

```bash
# vm5
sudo docker ps
curl -s localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"' | sort | uniq -c
```

## Resultado

Primer intento: `5 up` y `1 down` (vm5). Causa y solución en `bitacora/03-situaciones_de_error/05-node_exporter_vm5_caido.md`.

Tras corregir:

```
6 "health":"up"
```

Todos los objetivos en `up` ✅

## Grafana

1. `http://10.33.195.247:3000`, usuario `admin`.
2. Dashboards → New → Import → ID `1860` (Node Exporter Full) → fuente de datos Prometheus.
3. Filtro de instancia para cambiar entre las 5 VM.

- Captura del dashboard: _por registrar_

## Pendiente (fase 2)

- `mysqld_exporter` en vm4 y vm5, con alerta si se corta la replicación.
- cAdvisor en vm2 y vm3, exporter de Nginx en vm1.
- Loki + Promtail para ver los logs en Grafana (hoy **no** está instalado).
- Alertmanager.
