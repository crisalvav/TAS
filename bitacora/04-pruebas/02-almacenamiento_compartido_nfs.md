# Prueba: almacenamiento compartido por NFS entre vm2 y vm3 (21/10/2026)

**Servidores:** `tas05vm2` (10.33.195.163 / 10.33.199.43), `tas05vm3` (10.33.195.251 / 10.33.199.44), `tas05vm4` (NFS `/srv/wp-data`) · **Objetivo:** que un archivo escrito por un backend se sirva también por el otro

La prueba se hizo directo contra cada backend (puerto 9000), sin pasar por el balanceador, porque en ese momento el nginx de la vm1 estaba caído.

## Procedimiento

1. Montaje en ambos backends:

   ```bash
   mount | grep wp-data
   ls /mnt/wp-data   # wp-content, wp-config.php, etc.
   ```

2. Archivo de prueba desde la vm2:

   ```bash
   sudo touch /mnt/wp-data/prueba.txt          # vm2
   ls /mnt/wp-data/prueba.txt                  # vm3
   sudo mkdir -p /mnt/wp-data/wp-content/uploads/test
   echo hola | sudo tee /mnt/wp-data/wp-content/uploads/test/prueba.txt
   ```

3. Pedirlo por HTTP a cada backend (los archivos estáticos no redirigen):

   ```bash
   curl -I http://10.33.195.163:9000/wp-content/uploads/test/prueba.txt   # vm2
   curl -I http://10.33.195.251:9000/wp-content/uploads/test/prueba.txt   # vm3
   ```

4. Subir una imagen desde el panel por un backend y abrirla por el otro.

5. Borrar la carpeta `test`.

## Resultado esperado

`200` en ambos backends y la imagen visible sin importar qué backend responde.

## Resultado obtenido

- Fecha/hora: _por registrar_
- `curl` vm2: _por registrar_ · `curl` vm3: _por registrar_
- Imagen subida por el panel: _por registrar_

## Observaciones

- Si aparece error de permisos al subir, en la vm4: `sudo chown -R 33:33 /srv/wp-data`.
- El NFS en la vm4 es punto único de falla para los archivos.
