# Prueba: IP virtual y failover de la base de datos (21/10/2026 - 23/10/2026)

**Servidores:** `tas05vm4` (10.33.199.45), `tas05vm5` (10.33.199.46) · **Servicio:** keepalived (VRRP), IP virtual `10.33.199.41/29` · **Objetivo:** que WordPress siga funcionando si cae la base principal

## 1. IP libre para la IP virtual

La red interna es `10.33.199.40/29` (6 IP utilizables, `.41` a `.46`). Un ping sin respuesta no basta, así que se usó ARP:

```bash
for i in 41 42 43; do ping -c1 -W1 10.33.199.$i >/dev/null && echo ".$i ocupada" || echo ".$i libre?"; done
sudo apt install -y arping
sudo arping -I ens19 -c 3 10.33.199.41
```

| IP | Uso |
| --- | --- |
| .41 | libre → IP virtual de la base |
| .42 | vm1 |
| .43 | vm2 |
| .44 | vm3 |
| .45 | vm4 |
| .46 | vm5 |

Resultado de `arping`: `3 packets transmitted, 0 packets received, 100% unanswered` ✅

## 2. Configuración

- ufw en vm4 y vm5 permite VRRP: `sudo ufw allow in on ens19 from 10.33.199.40/29 to 224.0.0.18`.
- `vrrp_instance DB`: `state BACKUP`, `interface ens19`, `virtual_router_id 51`, `nopreempt`, chequeo `mariadb-admin ping` cada 2 s, prioridad 100 (vm4) y 90 (vm5).
- `notify_master "/usr/local/bin/promote.sh"` en ambas (versión corregida que no promueve si la otra base responde; ver `bitacora/03-situaciones_de_error/04-autopromocion_vm5_keepalived.md`).
- vm2 y vm3 con `WORDPRESS_DB_HOST: 10.33.199.41:3306`.

## 3. Verificación del estado (23/10/2026)

```bash
ip -4 a show ens19
sudo systemctl status keepalived --no-pager | head -12
sudo grep -n "notify_master" /etc/keepalived/keepalived.conf
```

| VM | IP virtual `.41` | `read_only` |
| --- | --- | --- |
| vm5 (principal) | Sí | OFF |
| vm4 (réplica) | No | ON |

Sin split-brain (solo una VM tiene la `.41`) ✅. Falta confirmar con `grep` que `notify_master` quedó guardado en la vm4.

## 4. Prueba de failover

1. Publicar una entrada en WordPress.
2. En la vm5: `sudo systemctl stop mariadb`.
3. En la vm4, a los pocos segundos:

   ```bash
   ip -4 a show ens19                                # debe aparecer 10.33.199.41
   sudo mariadb -e "SHOW VARIABLES LIKE 'read_only';" # debe decir OFF
   ```

4. Recargar WordPress: la entrada debe seguir y se debe poder publicar otra.
5. Si la `.41` pasa pero `read_only` sigue en ON: `sudo journalctl -u keepalived --since "5 min ago"`.

### Resultado obtenido

- Fecha/hora de la caída: _por registrar_
- Tiempo hasta que la vm4 tomó la `.41`: _por registrar_
- `read_only` en la vm4: _por registrar_
- WordPress escribe tras el failover: _por registrar_

## Observaciones

- La vuelta atrás es manual: la base que cayó queda con datos viejos y en solo lectura; hay que reconstruirla como réplica de la nueva principal con un dump nuevo.
- La IP virtual cubre la base, pero no el NFS (también en la vm4).
