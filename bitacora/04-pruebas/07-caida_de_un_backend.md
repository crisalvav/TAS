# Prueba: caída de un backend detrás del balanceador (23/10/2026)

**Servidores:** `tas05vm1` (balanceador), `tas05vm3` (backend detenido) · **Objetivo:** que el sitio siga respondiendo si cae un backend

Se usa la misma configuración y el encabezado `X-Backend` de `06-balanceo_1a1.md`.

## Procedimiento

1. En la vm3: `sudo docker stop wp_app`.
2. Desde el PC:

   ```fish
   for i in (seq 6); curl -skI https://10.33.195.230 | grep -i x-backend; end
   ```

3. En la vm3: `sudo docker start wp_app` y repetir el ciclo; debe volver a alternar en unos 10 s.

## Resultado (vm3 detenida)

```
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.44:9000, 10.33.199.43:9000
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.44:9000, 10.33.199.43:9000
X-Backend: 10.33.199.43:9000
X-Backend: 10.33.199.43:9000
```

Tiempo total del ciclo: ~15 s.

## Interpretación

- `10.33.199.44:9000, 10.33.199.43:9000`: Nginx intentó primero con la vm3, falló y reintentó con la vm2 en la misma petición. El cliente recibió respuesta igual ✅.
- Tras `max_fails=2`, la vm3 sale del grupo durante 10 s (`fail_timeout`) y todo va a la vm2. Al vencer el plazo se vuelve a intentar, falla y el ciclo se repite mientras siga caída.
- Los ~15 s se deben a los tiempos de espera de conexión de los intentos fallidos (`proxy_connect_timeout 5s`).

## Pendiente

- Registrar la salida tras `docker start wp_app` (vuelta a la alternancia 1:1).
