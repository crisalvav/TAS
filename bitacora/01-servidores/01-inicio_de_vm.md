# Inicio de levantamientos de maquians virtuales y configuracion (10/082026 - 05/09/2026)

1. En primera instancia se comenzo con el levantamiento de las 5 maquians virtuales en las cuales se tuvieron que configuran mediante la plataforma de proxmox, ademas de indicar las caracteristicas que utilizaria la maquina virtual.

2. Posterior a esto se dio paso a la configuracion respectiva segun la imagen del SO que se eligio, el cual en nuestro caso fue ubuntu server 24.02.

3. Una vez finalizadala configuracion de los sistemas operativos se dio paso a la configuracion de las tarjetas de red, utilizando netplan el cual se tuvo que instalar y posteriormente configurar para poder definir una ip estatica para cada servidor, ademas de eso tuvimos que agregar el acceso  alas maquinas mediante ssh con el comando <sudo ufw allow ssh> y habilitar los puertos necesarios para la conexion de las maquinas virtuales.

