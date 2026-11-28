# tlp - ajuste de batería
tlp es una app para gestionar la energía de un equipo: umbrales de batería, frecuencia y modo de la CPU, ahorro de energía en PCIe, USB, SATA, Wi-Fi, Bluetooth y audio, etc.

En este ejemplo lo utilizaremos para gestionar los umbrales de carga de la batería de un t480.

| Comando                                                                                                        | Para qué sirve                                                           |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **apt install tlp**                                                                                            | Instala TLP                                                              |
| **nano /etc/tlp.d/**_01-bateria.conf_                                                                          | Crea un fichero de configuración propio (no toca el `tlp.conf` original) |
| **systemctl enable --now tlp**                                                                                 | Activa TLP en el arranque y lo arranca ya, aplicando los umbrales        |
| **tlp start**                                                                                                  | Vuelve a aplicar la configuración si la cambia                           |
| **tlp-stat -b**                                                                                                | Muestra el estado de las baterías y los umbrales activos                 |
| **cat /sys/class/power_supply/**{**BAT0** \| **BAT1**}**/charge_control_**{**start** \| **end**}**_threshold** | Comprueba un valor directamente en el kernel                             |

En `/etc/tlp.d/01-bateria.conf`:

```
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
START_CHARGE_THRESH_BAT1=75
STOP_CHARGE_THRESH_BAT1=80
```

Para forzar la descarga sin desenchufar:
```
echo force-discharge > /sys/class/power_supply/BAT1/charge_behaviour
```
Luego restableces la configuración:
```
echo auto > /sys/class/power_supply/BAT1/charge_behaviour
```
Finalmente, confirmar con 
```
tlp-stat -b
```

> No puedes forzar la descarga en ambas baterías. Descargas una y luego descargas la otra. Para mantener un seguimiento: `watch -n 60 cat /sys/class/power_supply/BAT0/capacity`

# monitorización con lm-sensors
lm-sensors es una herramienta para leer los sensores de hardware del equipo: temperaturas de CPU, NVMe y chipset, velocidad de ventiladores y voltajes, según lo que exponga cada placa.

En este ejemplo lo utilizaremos para vigilar temperaturas y ventilador de un t480.

| Comando                             | Para qué sirve                                                          |
| ----------------------------------- | ----------------------------------------------------------------------- |
| **apt install lm-sensors**          | Instala lm-sensors                                                      |
| **sensors-detect** [**--auto**]     | Busca chips de sensores y propone qué módulos cargar                    |
| **sensors**                         | Muestra todas las lecturas actuales                                     |
| **watch -n** _segundos_ **sensors** | Refresca las lecturas cada _segundos_ en tiempo real                    |

En la salida de `sensors`:

- `coretemp-isa-0000`: temperatura de cada núcleo de la CPU
- `thinkpad-isa-0000`: ventilador (RPM) y sensores de la placa
- `nvme-pci-*`: temperatura del SSD
- `acpitz-acpi-0`: zona térmica ACPI

# iperf3 - medir ancho de banda
Herramienta para probar el ancho de banda entre dos equipos en la misma red.

| **Columna**    | **Qué significa**                                                                                                                                                              | **Cómo leerla en la práctica**                                                                                                                                               |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`[ ID]`**    | Identificador del flujo/socket de red (por ejemplo `[ 5]`).                                                                                                                    | Útil si lanzas varios flujos en paralelo (`-P`) para distinguir cada hilo.                                                                                                   |
| **`Interval`** | Tramo de tiempo medido (normalmente cada `1.00` segundo).                                                                                                                      | Te muestra si el rendimiento es estable o tiene subidas, caídas o microcortes.                                                                                               |
| **`Transfer`** | Cantidad total de datos transmitidos en ese intervalo (en `KBytes`, `MBytes` o `GBytes`).                                                                                      | Es el volumen efectivo enviado en ese segundo.                                                                                                                               |
| **`Bitrate`**  | Velocidad media en ese intervalo (`Mbits/sec` o `Gbits/sec`).                                                                                                                  | Ojo: `1 Byte = 8 bits`, y `iperf3` usa unidades decimales/binarias según el modo; fíjate en `Mbits/sec` para comparar con velocidades de enlace (100 Mbps, 1 Gbps, etc.).    |
| **`Retr`**     | **Retransmisiones TCP**: número de segmentos/paquetes que el emisor tuvo que reenviar en ese segundo porque no llegó confirmación (_ACK_) a tiempo o se perdieron/descartaron. | Valores bajos o puntuales son normales; **muchos `Retr` sostenidos** indican pérdida de paquetes, colas llenas (_bufferbloat_), congestión o un enlace/dispositivo saturado. |
| **`Cwnd`**     | **Congestion Window (Ventana de congestión)**: cuántos datos puede tener TCP enviados y pendientes de confirmación (_ACK_) en un momento dado sin saturar la red.              | Es el "acelerador/freno" de TCP: sube cuando la red va fluida y **se reduce cuando TCP detecta pérdidas/retransmisiones** para no ahogar el enlace.                          |

| **Comando**                                                                                | **Descripción**                                                                                                                                   |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **iperf3 -s**                                                                              | Inicia `iperf3` en modo servidor a la espera de conexiones entrantes.                                                                             |
| **iperf3 -c** _ip_servidor_                                                                | Ejecuta un test TCP como cliente apuntando al servidor indicado.                                                                                  |
| **iperf3 -c** _ip_servidor_ **-p** _puerto_                                                | Especifica el puerto TCP/UDP donde escucha el servidor (por defecto el `5201`).                                                                   |
| **iperf3 -c** _ip_servidor_ **-t** _segundos_                                              | Define la duración de la prueba en segundos (por defecto `10`).                                                                                   |
| **iperf3 -c** _ip_servidor_ **-P** _num_flujos_                                            | Abre varios flujos paralelos simultáneos para medir concurrencia o saturar enlaces.                                                               |
| **iperf3 -c** _ip_servidor_ **-R**                                                         | Invierte el sentido del tráfico (el servidor envía datos al cliente).                                                                             |
| **iperf3 -c** _ip_servidor_ **--bidir**                                                    | Envía y recibe tráfico de forma bidireccional y simultánea.                                                                                       |
| **iperf3 -c** _ip_servidor_ **-u** [**-b** _ancho_banda_]                                  | Cambia a modo UDP y fija opcionalmente el ancho de banda objetivo (ej. `100M`, `1G`, o `0` sin límite) para medir _jitter_ y pérdida de paquetes. |
| **iperf3 -c** _ip_servidor_ **-i** _segundos_                                              | Ajusta el intervalo de tiempo entre reportes intermedios en pantalla.                                                                             |
| **iperf3 -c** _ip_servidor_ **-J**                                                         | Muestra el resultado completo en formato JSON, útil para guardar o automatizar.                                                                   |
| **iperf3 -c** _ip_servidor_ [**-w**_tam_ventana_] [**-l** _tam_bloque_] [**-4** \| **-6**] | Ajusta opcionalmente el tamaño de ventana TCP, el tamaño de bloque/paquete o fuerza IPv4 o IPv6.                                                  |
## tailscale

**1. En el host Proxmox (habilitar el TUN)**

| Comando                                                             | Para qué sirve                              |
| ------------------------------------------------------------------- | ------------------------------------------- |
| **pct stop** _ID_                                                   | Parar el CT antes de tocar su configuración |
| **nano /etc/pve/lxc/_**ID**_.conf**                                 | Editar la configuración del CT              |
| **lxc.cgroup2.devices.allow:** c 10:200 rwm                         | Añadir: permite el TUN en el CT             |
| **lxc.mount.entry:** /dev/net/tun dev/net/tun none bind,create=file | Añadir: monta el TUN dentro del CT          |
| **pct start** _ID_                                                  | Arrancar el CT con el TUN disponible        |
> Un TUN es una interfaz de red virtual, con su ip y tabla de rutas propias (pero sin cable físico). Lo que sale por ahí se lo entrega el kernel a Tailscale, y al revés, lo que envía Tailscale por ahí, el kernel lo trata como un paquete que llega por la red.

**2. Dentro del CT (instalar y anunciar la red)**

| Comando                                                                                   | Para qué sirve                                                                 |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **apt update && apt install -y curl**                                                     | Instalar curl                                                                  |
| **curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) \| sh** | Instalar Tailscale con el script oficial                                       |
| **echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-tailscale.conf**                      | Activar el reenvío de IP de forma permanente                                   |
| **sysctl -p /etc/sysctl.d/99-tailscale.conf**                                             | Aplicar el reenvío ya                                                          |
| **tailscale up --advertise-routes=**_10.0.0.0/24_ **[--ssh]**                             | Conectar a la tailnet y anunciar la red del lab (`--ssh` activa Tailscale SSH) |

> `tailscale up` te da una URL para iniciar sesión: abrirla en el navegador y entrar con la cuenta indicada.

> Por defecto Linux se comporta como un PC: si le llega un paquete que no es para él, lo descarta. Con `net.ipv4.ip_forward = 1`  en un .conf dentro de sysctl (lo lee el sistema al arrancar) pasa a comportarse como un router y reenvía ese paquete hacia donde corresponda. 

> `--advertise-routes=10.0.0.0/24` le dice a Tailscale qué red ofrecer al resto de tu tailnet: "si quieres llegar a 10.0.0.x, mándame el tráfico a mí".

**3. Aprobar la ruta**

- En `login.tailscale.com/admin/machines`, abrir el menú del CT, entrar en _Edit route settings_ y activa `10.0.0.0/24`. En el mismo menú desactivar el vencimiento de la clave (_Disable key expiry_).
- En el iPhone, abrir Tailscale, iniciar sesión con la misma cuenta y activar la VPN.
- Para probar, desactivar el Wi-Fi y usa 5G. Conectar por SSH con una app a la IP 100.x del CT y luego a `10.0.0.10` (DC1).
