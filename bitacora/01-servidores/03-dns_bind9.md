# Instalación del servicio DNS con BIND9 en `tas05vm1` (07/09/2026 - 11/09/2026)

**Servidor:** `tas05vm1` (10.33.199.42) · **Servicio:** BIND9 en contenedor · **Puerto:** 53/tcp y 53/udp

## 07/09/2026 — Preparación del servidor

1. Se actualiza el sistema y se confirma que Docker quedó operativo desde la etapa anterior:

   ```bash
   sudo apt update && sudo apt upgrade -y
   docker --version && docker compose version
   ```

2. Ubuntu trae `systemd-resolved` escuchando en el puerto 53, lo que impide levantar BIND9. Se desactiva el stub listener:

   ```bash
   sudo sed -i 's/^#\?DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf
   sudo systemctl restart systemd-resolved
   sudo ss -lntup | grep ':53'   # no debe aparecer nada
   ```

## 08/09/2026 — Estructura de archivos y zona

1. Se crea la carpeta del servicio:

   ```bash
   mkdir -p ~/servicios/bind9/{config,zones,cache}
   ```

2. Se escribe `config/named.conf` permitiendo consultas solo desde la red interna:

   ```conf
   options {
       directory "/var/cache/bind";
       listen-on { any; };
       allow-query { 10.33.199.0/24; 127.0.0.1; };
       recursion yes;
       allow-recursion { 10.33.199.0/24; 127.0.0.1; };
       forwarders { 8.8.8.8; 1.1.1.1; };
       dnssec-validation auto;
   };

   zone "nodosur.tas" {
       type master;
       file "/var/lib/bind/db.nodosur.tas";
   };
   ```

3. Se crea la zona `zones/db.nodosur.tas`. El registro `www` apunta al balanceador (vm1), no a los backends, para que el usuario siempre entre por un único punto:

   ```dns
   $TTL 3600
   @   IN SOA ns1.nodosur.tas. admin.nodosur.tas. (
           2026090801 ; serial (AAAAMMDDNN)
           3600       ; refresh
           900        ; retry
           604800     ; expire
           300 )      ; negative TTL
       IN NS  ns1.nodosur.tas.
   ns1     IN A   10.33.199.42
   @       IN A   10.33.199.42
   www     IN A   10.33.199.42
   cms1    IN A   10.33.199.43
   cms2    IN A   10.33.199.44
   db      IN A   10.33.199.45
   monitor IN A   10.33.199.46
   ```

## 09/09/2026 — Despliegue con Docker Compose

1. `~/servicios/bind9/docker-compose.yaml`:

   ```yaml
   services:
     bind9:
       image: ubuntu/bind9:9.18-22.04_beta
       container_name: bind9
       restart: unless-stopped
       ports:
         - "53:53/tcp"
         - "53:53/udp"
       environment:
         - TZ=America/Santiago
       volumes:
         - ./config/named.conf:/etc/bind/named.conf:ro
         - ./zones:/var/lib/bind
         - ./cache:/var/cache/bind
   ```

2. Se valida la sintaxis antes de levantar y luego se inicia:

   ```bash
   docker run --rm -v $PWD/config:/etc/bind ubuntu/bind9:9.18-22.04_beta named-checkconf /etc/bind/named.conf
   docker run --rm -v $PWD/zones:/z ubuntu/bind9:9.18-22.04_beta named-checkzone nodosur.tas /z/db.nodosur.tas
   docker compose up -d
   docker logs -f bind9
   ```

## 10/09/2026 — Firewall y pruebas

1. Se abre el puerto 53 solo para la red interna:

   ```bash
   sudo ufw allow from 10.33.199.0/24 to any port 53
   sudo ufw status numbered
   ```

2. Desde `tas05vm2` se prueba la resolución:

   ```bash
   dig @10.33.199.42 www.nodosur.tas +short   # 10.33.199.42
   dig @10.33.199.42 db.nodosur.tas +short    # 10.33.199.45
   dig @10.33.199.42 google.com +short        # prueba de reenvío
   ```

## 11/09/2026 — Clientes apuntando al DNS interno

1. En cada VM se agrega el DNS interno en netplan (`/etc/netplan/50-cloud-init.yaml`):

   ```yaml
   nameservers:
     addresses: [10.33.199.42]
     search: [nodosur.tas]
   ```

   ```bash
   sudo netplan apply
   resolvectl status | grep -A2 'DNS Servers'
   ping -c2 db
   ```

## Resultado

- El FQDN `www.nodosur.tas` resuelve hacia el balanceador desde todas las VM.
- **Pendiente:** cada cambio de zona requiere aumentar el `serial` y ejecutar `docker exec bind9 rndc reload`.
