# instalación de windows

> Previamente hay que [instalar proxmox](instalacion-proxmox.md)

## virtio-win.iso

1. Lo primero es instalar los drivers del hardware de proxmox (disco, red, etc.). Sin ellos Windows no reconoce el disco ni la tarjeta de red VirtIO.
2. En local -> ISO Images -> Download from URL, pegamos esto:
   `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso`
3. Esperamos a que salga `TASK OK`

![](https://i.imgur.com/SBypM6I.png)
## crear la vm de windows

1. En la web de windows (https://www.microsoft.com/es-es/evalcenter) descargar la .iso de windows 11 enterprise
2. Una vez descargadas, cargarlas en proxmox: Local -> ISO Images -> Upload

![](https://i.imgur.com/5ERrkf5.png)

3. Cargadas las .iso, vamos a **Create VM**. En **General** ponemos un nombre a la VM

![](https://i.imgur.com/BhS6z7R.png)

4. En la pesataña **OS**, configuramos

	- Storage: local
	- ISO Image: La ISO que descargamos
	- Type: Microsoft Windows
	- Add additional drive for VirtIO drivers: Seleccionamos la ISO de VirtIO

![](https://i.imgur.com/RbRexgh.png)

5. En **System**
	- Machine: q35 (es más moderno)
	- BIOS: OVMF (UEFI) porque SEABIOS es legacy
	- Add EFI Disk: activado.
	- EFI Storage: local-lvm. en un PC real, UEFI guarda su configuración (orden de arranque, claves de Secure Boot) en una memoria NVRAM de la placa. En la VM, esa memoria es este pequeño disco; sin él, la configuración se pierde en cada arranque. Va en `local-lvm` porque ahí se guardan los discos de las VMs, mientras que `local` es para ficheros como ISOs y backups
	- SCSI Controller: VirtIO SCSI single. Es el controlador de virtio-win que windows necesita
	- Qemu Agent: activado. crea un canal de comunicación entre Proxmox y la VM
	- Add TPM: activado. añade un TPM 2.0 virtual emulado por software (en una pc real, el tpm es un chip físico de seguridad que guarda claves de BitLocker, cifrado, etc.)
	- TPM-Storage: local-lvm
	- Version: 2.0

![](https://i.imgur.com/TMmbgll.png)

6. Pestaña Disks:
	- **Bus/Device:** _SCSI_, _0_. Usa el controlador VirtIO SCSI elegido
	- **Storage:** `local-lvm`.
	- **Disk size:** **64 GiB**, el mínimo que exige Windows 11. Con LVM-thin solo se ocupa en el disco físico lo que realmente se escribe.
	- **Cache:** _Default (No cache)_. Oopción segura frente a cortes de corriente.
	- **Discard:** marcado. Cuando Windows borra archivos, avisa al almacenamiento y Proxmox recupera ese espacio en el thin pool (TRIM).
	- **IO thread:** marcado. Da al disco su propio hilo de E/S.
	- **SSD emulation:** Windows trata el disco como SSD (no lo desfragmenta y usa TRIM).
	- **Backup:** marcado para que el disco se incluya en los backups.

![](https://i.imgur.com/0x4jalY.png)

7. Pestaña **CPU:**
	- **Sockets:** 1. Es el número de procesadores físicos que simula la VM; con uno basta.
	- **Cores:** **2**. Windows 11 exige un mínimo de 2 núcleos, así que con 1 la instalación falla. 
	- **Type:** **host**. Pasa a la VM el modelo real de la CPU, lo que da el máximo rendimiento.
	- **Lo demás** (CPU units, limit, affinity, NUMA, flags): queda como está

8. Pestaña **Memory**:
	- Minimum Memory: 4096
	- **Ballooning Device:** marcado. Así la VM tiene siempre 4 GB fijos (con 32 GB no necesitas ballooning dinámico). 
	- **Allow KSM:** marcado. KSM (_Kernel Samepage Merging_) fusiona las páginas de memoria idénticas entre VMs, lo que ahorra RAM cuando tienes varios Windows iguales. 

9. Pestaña **Network** marcamos `No network device` si la intención es crear una red privada virtual. De lo contrario si lo configuramos con los valores por defecto, la VM aparecerá en la red doméstica y recibirá IP por DHCP.

> Como la intención es crear una LAN privada virtual (con un DC con DHCP y varios clientes), lo mejor es configurarlo más adelante.


## antes de levantar la vm... crear el bridge

1. Antes de proceder a levantar la VM, necesitamos crear un switch virtual aislado (las VMS que conectemos a él se verán entre sí y verán a proxmox que tendrá una ip en esa subred, y es la ip que precisamente hará de gateway)
2. Vamos al nodo -> Network -> Create -> Linux Bridge y configuramos el bridge así:
	- IPV4/CIDR: configuramos la ip del proxmox en esa subred y definimos la máscara
	- Gateway: vacío (el host ya tiene salida por vmbr0)
	- autostart: activado
	- bridge ports: vacío. Aquí se indica qué tarjeta física conectar al bridge, pero el bridge queda aislado, que es la idea.

![](https://i.imgur.com/GAzNm2e.png)

> Importante: Recuerda que un nodo es una máquina física con proxmox instalado (en principio solo tienes uno por cada pc). Si quisieras unir luego varios nodos (varias pcs), se unen en un clúster desde Datacenter -> Cluster

3. Pulsar **Apply configuration** y verificar que el bridge aparece activo
4. En la shell del nodo editamos `nano /etc/network/interfaces` y justo después de bridge-fd 0, añadimos esto (respetando la identación):
```shell
           post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
           post-up   iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
           post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
```

> Aquí básicamente hacemos dos cosas: **`ip_forward = 1`** permite que el kernel reenvíe paquetes entre interfaces (actúa como router). Por defecto Linux no lo hace. Y con **MASQUERADE** declaramos dos condiciones: que los paquetes con origen `10.10.10.0/24` y que **salgan por `vmbr0`** (no hace falta NAT para que las vms se comuniquen entre ellas) se les cambia la IP de origen por la de `vmbr0` (`192.168.1.99`).

5. Guardamos (`ctrl + X`), cargamos la config y verificamos

```shell
root@november:~# cat /proc/sys/net/ipv4/ip_forward

0

root@november:~# iptables -t nat -S POSTROUTING

-P POSTROUTING ACCEPT

root@november:~# ifreload -a

root@november:~# cat /proc/sys/net/ipv4/ip_forward

1

root@november:~# iptables -t nat -S POSTROUTING

-P POSTROUTING ACCEPT

-A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE

root@november:~#
```

6. Añadir la tarjeta de red a la VM. En la VM -> Hardware -> Add -> Network Device

![](https://i.imgur.com/CgT7Nk0.png)

7. Configuramos así (firewall desmarcado para evitar conflictos con NAT):

![](https://i.imgur.com/R5YdNcB.png[/img)

## instalación de windows 11

1. Arrancar la vm. Siguiente -> Siguiente

![](https://i.imgur.com/Wm57D9m.png)

2. En la pestaña de discos aparecerá vacío (lo normal, hay que cargar los drivers para que windows vea los discos). `Load Driver -> Browse -> CD Drive virtio-win -> vioscsi -> w11 -> amd64`

![](https://i.imgur.com/z6nXotF.png)


3. El disco de 64 gb debería aparecer.

> Es buena idea cargar también los drivers de red `Load Driver -> Browse -> CD Drive virtio-win -> netKVM -> w11 -> amd64` para que Windows ya los tenga.

4. Instalar

![](https://i.imgur.com/YHpWX9F.png)


> En versiones recientes, si pide conexión y solo queremos usuario local: `start ms-cxh:localonly` abre directamente la creación de una cuenta local, sin reiniciar.`OOBE\BYPASSNRO` funciona en versiones anteriores a 2025

5. Cuando reinicie, poner "i don't have internet", configurar un nombre de usuario, contraseña, etc.
6. Una vez en el escritorio de windows, ejecutar el `virtio-win-guest-tools.exe` 

![](https://i.imgur.com/ZQPMnsx.png)

> Recordar: este paso permite instalar el QEMU Guest Agent, un servicio de windows que habla con proxmox por un canal serie virtual sin pasar por la red. Esto permite ver las ips de la vm, apagarla, reiniciarla limpiamente, hacer snapshots, etc.

7. Como paso final, configurar una ip estática y probar conexión con el gateway e internet.

# convertir una vm en plantilla

1. Es buena idea tomar una vm y convertirla en plantilla de otras vms (esto evita tener que repetir todos los pasos anteriores)
2. Opcional:
    - Windows Update hasta que no quede nada pendiente.
    - `virtio-win-guest-tools.exe`: instala los drivers que falten y el QEMU Guest Agent.
    - Zona horaria (Madrid) e idioma/teclado correctos.
    - Activa RDP para administrar los clones
    - Plan de energía en Alto rendimiento
    - Limpieza: `Dism /Online /Cleanup-Image /StartComponentCleanup` y vaciar temporales, para que la plantilla ocupe menos.
3. Lo primero es preparar el sistema. En CMD, ejecutar 

```
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

> Durante este proceso, BitLocker puede dar problemas, mejor desactivarlo con `manage-bde -off C:` con permisos de admin.
> En la práctica, es mejor desactivarlo completo con `PreventDeviceEncryption` = 1 (DWORD) en `HKLM\SYSTEM\CurrentControlSet\Control\BitLocker`.

![](https://i.imgur.com/PB2FqjB.png)

3. Cuando la VM se apague, click derecho en VM100 -> Convert to template

> Esto es irreversible. La vm no se puede arrancar, solo sirve para clonar.

4. Aquí conviene saber: 
	- **Full-Clone**: se hará una copia completa e independiente del disco (tarda más y ocupa más, pero puedes borrar o modificar la plantilla sin afectar al clon.
	- **Linked clone:** el clon solo guarda los **cambios** respecto al disco de la plantilla. Se crea al instante y ocupa muy poco, pero depende de ella: no se puede borrar la plantilla mientras existan clones enlazados, y si su disco se corrompe, caen todos.

5. Para clonar, click derecho en la plantilla -> Clone, le damos un nombre y elegimos linked o full

![](https://imgur.com/RDJ5rzU.png)

6. Iniciamos la VM y pasamos el asistente. Listo!

> Si quieres que tus equipos de la red de fuera vean los equipos de la red interna de proxmox (vmbr1) tienes que aplicar rutas estáticas. En mi caso, una mac, basta con añadir la ruta de la red interna hacia la ip de proxmox en la red externa `sudo route -n add -net 10.10.10.0/24 192.168.1.99` 


