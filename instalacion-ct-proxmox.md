# Instalación CT en proxmox

# conceptos previos

- **rootfs**: _root filesystem_ es el sistema de archivos raíz de Linux: todo lo que cuelga de `/` (`/etc`, `/bin`, `/home`, `/var`...) además de todos los archivos dentro (programas, librerías, configuraciones). En este caso, es el "disco" del CT. Se descarga como plantilla (arhivos de Debian **sin kernel**) que proxmox descomprime dentro del volumen rootfs del CT.

# descargar plantilla e instanciar

1. En **local (november)** -> CT Templates -> Templates
	- **Upload:** subir una plantilla desde el ordenador.
	- **Download from URL:** descargar una plantilla desde un enlace.
	- **Templates:** mostrar el catálogo oficial de Proxmox, con plantillas listas para descargar. 

![](https://imgur.com/jGQoRtO.png)

2. **Crear el contenedor: pestaña General** -> Pulsar **Create CT** (arriba a la derecha) y rellenar:
	- **Node:** Es el host físico donde se ejecutará el contenedor.
	- **CT ID:** el id que tendrá el CT.
	- **Hostname:** nombre del sistema dentro del contenedor.
	- **Unprivileged container:** marcado. El usuario root del contenedor corresponde en el host (proxmox) a un usuario sin privilegios (UID 100000), así que una fuga del contenedor no da acceso de root a Proxmox.
	- **Nesting:** marcado. Permite ejecutar contenedores anidados.
	- **Resource Pool:** vacío. Sirve para agrupar máquinas y asignar permisos a otros administradores; no es necesario en este caso.
	- **Password / Confirm password:** la contraseña de root del contenedor.
	- **SSH public key:** para iniciar sesión por SSH con una clave pública en lugar de con contraseña. 

**Paso 3. Pestaña Template**
- **Storage:** local. Es el almacenamiento de tipo directorio del host, donde Proxmox guarda las plantillas de contenedor, las ISOs y las copias de seguridad.
- **Template:** Cargar la plantilla descargada: `debian-13-standard_…_amd64.tar.zst`. 

**Paso 4. Pestaña Disks**
- **rootfs:** es el disco raíz del contenedor (`/`), donde quedará todo su sistema de archivos: `/etc`, `/var`, `/home`, etc. 
> Se pueden añadir más discos con **Add**, pero no hace falta para este caso.
- **Storage:** local-lvm. Es el almacenamiento LVM-thin del host, donde se guardan los discos de VMs y CTs. 
> Con _thin provisioning_, el volumen solo ocupa espacio real a medida que se escriben datos.
- **Disk size (GiB):** Para un sistema funcional 2GB es mínimo recomendable.
- **Resto de opciones** (mount options, ACLs, quota): por defecto. Son ajustes avanzados de montaje y control de espacio que no se necesitan aquí.

**Paso 5. Pestaña CPU**
- **Cores:** 1. Es el número de núcleos que el contenedor puede ver y usar. No se reservan en exclusiva: el kernel del host los reparte entre todas las máquinas mediante cgroups.
- **CPU limit / CPU units** (opciones avanzadas): por defecto. La primera pone un tope de uso de CPU y la segunda da prioridad a un contenedor frente a otros cuando hay competencia. 

**Paso 6. Pestaña Memory**
- **Memory (MiB):** 512. Es la RAM máxima que puede usar el contenedor. Es un límite, no una reserva: el contenedor solo consume lo que usa de verdad, que en este caso serán unas decenas de MB.
- **Swap (MiB):** 512. Es memoria de reserva en disco que se usa si se llena la RAM. Evita que el sistema mate procesos por falta de memoria, a cambio de ir más lento.

**Paso 7. Pestaña Network**
- **Name:** eth0. Es el nombre de la interfaz de red dentro del contenedor.
- **MAC address:** auto. Proxmox genera una dirección MAC única para la interfaz virtual.
- **Bridge:** Es el switch virtual al que se conecta la interfaz.
- **VLAN Tag:** vacío. Sirve para etiquetar el tráfico en una VLAN concreta del bridge.
- **Firewall:** por defecto. Activa el firewall de Proxmox en esta interfaz, pero solo filtra si el firewall también está habilitado a nivel de datacenter y del contenedor.
- **IPv4:** Configurar IP/CIDR y máscara
- **Opciones avanzadas** (Disconnect, MTU, Rate limit): por defecto. Sirven para desconectar la interfaz, cambiar el tamaño máximo de paquete o limitar el ancho de banda.

**Paso 8. Pestaña DNS**
- **DNS domain:** Es el sufijo DNS que se añade a los nombres cortos.
- **DNS servers:** Es el servidor que resuelve los nombres. 

> Si queda vacío, Proxmox usa el del host.


![](https://imgur.com/7ZDMCBM.png)

**Paso 9. Crear e iniciar el contenedor**
Proxmox abrirá una ventana de tarea. Cuando aparezca **TASK OK**, el contenedor estará creado y arrancado. Levantar el CT.



