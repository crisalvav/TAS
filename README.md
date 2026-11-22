# TAS - Nodo Sur — Plataforma digital resiliente

**Asignatura:** EIN-090B · Taller de Administración de Sistemas

**Colaboradores:**

- Cristobal Vergara
- Javier Ulloa
- Martin Villaroel

---
## 1. Descripción del Proyecto

El proyecto tiene como propósito transformar un problema empresarial en una plataforma operable. El objetivo principal no es solamente instalar un CMS y mostrar su portada, sino construir una solución robusta que pueda ser publicada, administrada, diagnosticada, mantenida y recuperada ante posibles fallas. 

Para lograr esto, el equipo, deberán actuar como un proveedor de infraestructura. En este rol es fundamental justificar cada decisión técnica y sobre la arquitectura, mediante el establecimiento de requerimientos, la ejecución de pruebas, la recolección de métricas, el análisis de logs y la creación de procedimientos que sean completamente reproducibles.  

---
## 2. Fechas

| Entrega | Fecha     | Descripción                                |
| :-------: | :-----: | :----------------------------------------- |
| 1       | 05/09     | Informe de requerimientos y arquitectura   |
| 2       | 05/09     | CMS funcional                              |
| 3       | 23/10     | Publicación resiliente y seguridad         |
| 4       | -/-       | Observabilidad y continuidad operacional   |
| 5       | -/-       | Defensa fina, auditoria e incidentes       |

---
## 3. Arquitectura

Cinco VM con dominio interno `nodosur.tas`. La mayoría de los servicios corren en contenedores Docker; MariaDB, NFS y los exporters de sistema van instalados directamente.

| VM         | Rol                        | IP           | Servicios principales                                   |
| ---------- | -------------------------- | ------------ | ------------------------------------------------------- |
| `tas05vm1` | DNS y balanceador          | 10.33.199.42 | BIND9 (53), Nginx HTTPS (80/443)                        |
| `tas05vm2` | CMS 1                      | 10.33.199.43 | WordPress PHP-FPM + Nginx (8000)                        |
| `tas05vm3` | CMS 2                      | 10.33.199.44 | WordPress PHP-FPM + Nginx (8000)                        |
| `tas05vm4` | Base de datos              | 10.33.199.45 | MariaDB (3306), NFS `wp-content`                        |
| `tas05vm5` | Monitorización y respaldos | 10.33.199.46 | Prometheus, Grafana, Alertmanager, respaldos            |

```mermaid
flowchart TB
    C(["👥 Clientes"])

    subgraph VM1["🖧 tas05vm1 · 10.33.199.42 · frontend"]
        DNS["BIND9<br/>DNS · 53"]
        LB["Nginx<br/>balanceador TLS · 443"]
    end

    subgraph VM2["tas05vm2 · .43 · CMS 1"]
        WP2["Nginx + PHP-FPM<br/>WordPress · 8000"]
    end

    subgraph VM3["tas05vm3 · .44 · CMS 2"]
        WP3["Nginx + PHP-FPM<br/>WordPress · 8000"]
    end

    subgraph VM4["🗄 tas05vm4 · 10.33.199.45 · datos"]
        DB[("MariaDB<br/>3306")]
        NFS[("NFS<br/>wp-content")]
    end

    subgraph VM5["📊 tas05vm5 · 10.33.199.46 · monitoreo y respaldos"]
        MON["Prometheus · Alertmanager<br/>Grafana · Blackbox"]
        BK["Respaldos diarios<br/>/srv/respaldos · 30 días"]
    end

    C <-. "1 · consulta DNS → IP de vm1" .-> DNS
    C == "2 · HTTPS 443" ==> LB
    LB -- "least_conn" --> WP2
    LB -- "least_conn" --> WP3
    WP2 --> DB & NFS
    WP3 --> DB & NFS
    DB & NFS <-- "SSH 22 · dump + tar" --- BK

    classDef front fill:#e3f2fd,stroke:#1565c0,color:#0d1b2a
    classDef web fill:#e8f5e9,stroke:#2e7d32,color:#0d1b2a
    classDef data fill:#f3e5f5,stroke:#8e24aa,color:#0d1b2a
    classDef ops fill:#fff3e0,stroke:#e65100,color:#0d1b2a
    class DNS,LB front
    class WP2,WP3 web
    class DB,NFS data
    class MON,BK ops
```

Detalle de puertos, firewall, respaldos y puntos únicos de falla en [`infraestructura/infraestructura.md`](infraestructura/infraestructura.md).

---
## 4. Estructura del repositorio

```
.
├── bitacora/
│   ├── 01-servidores/           # Paso a paso de cada instalación (VM, Docker, DNS, MariaDB,
│   │                            # WordPress + NFS, balanceador HTTPS, monitoreo, respaldos)
│   ├── 02-repositorio/          # Registro de trabajo sobre el repositorio
│   └── 03-situaciones_de_error/ # Incidentes y su resolución
├── decisiones/                  # Registros de decisiones técnicas (estructura, Docker,
│                                # balanceador, servidor web)
├── docs/informe/                # Informe del proyecto (PDF) y entregas
├── infraestructura/             # Distribución de nodos, flujo, firewall y respaldos
└── README.md
```

---
## 5. Declaración de uso de Inteligencia artificial

| Herramienta | Propósito | Fragmento relevante | Validación realizada | Cambios aplicados |
|---|---|---|---|---|
| Claude | Apoyo en redacción, estructura y corrección del informe, del README y de la documentación de scripts | Tablas de requerimientos con umbrales, política de datos, cálculo de capacidad N-1 y documentación de la configuración de Docker, NFS y monitorización | A completar por el equipo: revisar cada cifra y comando y contrastarlo con pruebas y documentación oficial | A completar por el equipo |

_Nota: la respuesta de una IA no constituye evidencia técnica. Todo código o comando debe poder ser explicado por cualquier integrante del equipo._

---
