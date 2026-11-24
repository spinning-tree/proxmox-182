# RDP falla desde el Mac tras reiniciar: ruta estática no persistente

| Campo              | Valor                             |
| ------------------ | --------------------------------- |
| Ticket             | TS-001                            |
| Tipo               | Incidencia                        |
| Categoría          | Red > Enrutamiento                |
| Urgencia / Impacto | Media                             |
| Prioridad          | Media                             |
| Estado             | Cerrado                           |
| Abierto / Resuelto | 2016-10-7 10:10 / 2016-10-7 10:43 |
## Resumen
Tras reiniciar el mac, falla la conexión RDP con la VM de proxmox. La ruta estática configurada se borró durante el reinicio. Se solucionó instalando una ruta persistente.
## Entorno
En proxmox la interfaz vmbr0 es la predeterminada, y la que conecta con la LAN doméstica. Sin embargo, existe una segunda interfaz (bridge) vmbr1 dentro de proxmox para las vms. En proxmox se configuró una regla NAT para darle salida a las vms a través de vmbr0

| Equipo          | ip            | subred         |
| --------------- | ------------- | -------------- |
| mac-mini        | 192.168.1.100 | 192.168.1.0/24 |
| Router LAN      | 192.168.1.1   | 192.168.1.0/24 |
| Proxmox (vmbr0) | 192.168.1.99  | 192.168.1.0/24 |
| Proxmox (vmbr1) | 10.10.10.1    | 10.10.10.0/24  |
| VM 102          | 10.10.10.10   | 10.10.10.0/24  |


![](https://imgur.com/27dg3aj.png)


Las interfaces quedan detalladas a continuación:

En el t480:

```
root@november:~# ip a | grep vmbr1
5: vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 10.10.10.1/24 scope global vmbr1
28: fwpr102p0@fwln102i0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master vmbr1 state UP group default qlen 1000
```

```
root@november:~# ip a | grep vmbr0
2: nic0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr0 state UP group default qlen 1000
4: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 192.168.1.99/24 scope global vmbr0
root@november:~#
```

En el mac-mini:

```
cesar@cesars-Mac-mini ~ % ifconfig en1
en1: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
options=6460<TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
ether d0:11:e5:e1:36:f9
inet6 fe80::c9c:92df:133a:3232%en1 prefixlen 64 secured scopeid 0x10 
inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255
nd6 options=201<PERFORMNUD,DAD>
media: autoselect
status: active
cesar@cesars-Mac-mini ~ %
```
## Síntoma

1. Se conecta el cable ethernet al t480

> El cable debe atravesar un pasillo a lo largo de la casa. Durante la noche se desconecta para que no estorbe.

2. Se inicia la vm y se comprueba su operatividad

```bash
root@november:~# qm list
      VMID NAME                 STATUS     MEM(MB)    BOOTDISK(GB) PID       
       100 win11-template       stopped    4096              64.00 0         
       101 deb13-01             stopped    1024              22.00 0         
       102 win-server-DC1       stopped    4096              64.00 0         
      9000 debian13-template    stopped    1024              22.00 0         
     10000 win-server-template  stopped    4096              64.00 0         
root@november:~#
root@november:~# qm start 102
swtpm_setup: Not overwriting existing state file.
root@november:~# qm status 102
status: running
root@november:~#
```

3. Al intentar conectar por RDP a través de la Windows App, la app lanza un error de conexión

![](https://imgur.com/2Yus0Ch.png)

## Contexto previo
Durante la implementación de la red, se estableció una ruta no persistente en el mac mediante el siguiente comando:

```
sudo route -n add -net 10.10.10.0/24 192.168.1.99
```

> Esto permitía al mac-mini enviar los paquetes de la red interna (vmbr1) a la ip de proxmox en la LAN doméstica (vmbr0), y este se encargaba de enrutar hacia la red interna

El día anterior la conexión se realizaba sin problemas. La última acción realizada fue una actualización del DC1, tras lo cual se apagó y se retiró el cable ethernet. 

Adicionalmente, tras un comportamiento errático del mac-mini (no respondía la barra de búsqueda Spotlight Search), se decidió reiniciar el mac-mini.

## Diagnóstico paso a paso

1. El primer paso consiste en comprobar la conectividad desde el mac mini hacia el puerto 3389 de RDP

```
cesar@cesars-Mac-mini ~ % nc -vz 10.10.10.10 3389
nc: connectx to 10.10.10.10 port 3389 (tcp) failed: Operation timed out
cesar@cesars-Mac-mini ~ %
```

2. El puerto no responde. Verificamos (desde la consola de proxmox) que la opción Remote Desktop no haya quedado deshabilitada

![](https://imgur.com/L5f51ko.png)


3. La opción Remote Desktop está activada. Comprobamos que el mac-mini tiene conectividad con la VM 102

```
cesar@cesars-Mac-mini ~ % ping 10.10.10.10
PING 10.10.10.10 (10.10.10.10): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
--- 10.10.10.10 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
```

4. El ping falla. Una primera hipótesis sugiere que el firewall de windows puede estar bloqueando las conexiones. Para comprobarlo, verificamos conectividad con el gateway de proxmox en vmbr1

```
cesar@cesars-Mac-mini ~ %
cesar@cesars-Mac-mini ~ % ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
--- 10.10.10.1 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
```

5. El segundo ping también falla (el DC1 y su actualización no son la causa). La hipótesis del firewall queda descartada. Comprobamos conectividad directa con proxmox a través de su ip en vmbr0

```
cesar@cesars-Mac-mini ~ % ping 192.168.1.99
PING 192.168.1.99 (192.168.1.99): 56 data bytes
64 bytes from 192.168.1.99: icmp_seq=0 ttl=64 time=4.028 ms
64 bytes from 192.168.1.99: icmp_seq=1 ttl=64 time=3.679 ms
64 bytes from 192.168.1.99: icmp_seq=2 ttl=64 time=8.368 ms
64 bytes from 192.168.1.99: icmp_seq=3 ttl=64 time=8.172 ms
--- 192.168.1.99 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.679/6.062/8.368/2.213 ms
cesar@cesars-Mac-mini ~ %
```

6. La evidencia demuestra que el mac-mini es capaz de llegar a proxmox (el cable y la LAN funcionan), pero no logra comunicar con la red interna. Una segunda hipótesis sugiere que se deba a un problema de enrutamiento (el mac mini no sabe a dónde enviar los paquetes de la red 10.10.10.0/24).

```
cesar@cesars-Mac-mini ~ % netstat -rn | grep 10.10.10 
cesar@cesars-Mac-mini ~ %
```

7. Efectivamente, el mac-mini ha perdido la ruta no persistente que tenía para llegar a la red 10.10.10.0/24 tras el reinicio.

## Causa raíz
El mac-mini perdió la ruta no persistente hacia la red interna de proxmox (vmbr1) a causa de un reinicio del sistema.

## Solución

1. El primer paso para aplicar una ruta persistente es determinar el nombre exacto de las interfaces

```
cesar@cesars-Mac-mini ~ % networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Ethernet
Thunderbolt Bridge
Wi-Fi
cesar@cesars-Mac-mini ~ %
```

2. El mac está conectado por wifi al router. Seleccionamos la interfaz wifi y añadimos la ruta adecuada.

```
cesar@cesars-Mac-mini ~ % sudo networksetup -setadditionalroutes "Wi-Fi" 10.10.10.0 255.255.255.0 192.168.1.99
```

> `-setadditionalroutes` sobrescribe las rutas existentes para el servicio Wi-Fi. Es buena práctica consultar (`-getadditionalroutes`) las rutas que ya existían y añadirlas también con el comando `-setadditionalroutes`
## Verificación

1. Reiniciamos el mac mini y verificamos que la ruta, efectivamente, ha quedado instalada de forma persistente y que la ruta está activa en la tabla de rutas.

```
cesar@cesars-Mac-mini ~ % sudo networksetup -getadditionalroutes "Wi-Fi"
10.10.10.0 255.255.255.0 192.168.1.99

cesar@cesars-Mac-mini ~ % netstat -rn | grep 10.10.10 
10.10.10/24        192.168.1.99       UGSc                  en1       
cesar@cesars-Mac-mini ~ %
```

2. Verificamos ping a proxmox en la red interna y a la vm 102

```
cesar@cesars-Mac-mini ~ % ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1): 56 data bytes
64 bytes from 10.10.10.1: icmp_seq=0 ttl=64 time=8.764 ms
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=3.683 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=3.866 ms
^C
--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.683/5.438/8.764/2.353 ms

cesar@cesars-Mac-mini ~ % ping 10.10.10.10
PING 10.10.10.10 (10.10.10.10): 56 data bytes
64 bytes from 10.10.10.10: icmp_seq=0 ttl=127 time=8.475 ms
64 bytes from 10.10.10.10: icmp_seq=1 ttl=127 time=3.507 ms
64 bytes from 10.10.10.10: icmp_seq=2 ttl=127 time=8.037 ms
^C
--- 10.10.10.10 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.507/6.673/8.475/2.246 ms
```

> Windows responde con TTL 128; al atravesar Proxmox, que enruta entre vmbr1 y vmbr0, el TTL baja a 127. Esto confirma un salto de capa 3 en el camino

3. Verificamos conexión con el puerto RDP

```
cesar@cesars-Mac-mini ~ % nc -vz 10.10.10.10 3389
Connection to 10.10.10.10 port 3389 [tcp/ms-wbt-server] succeeded!
cesar@cesars-Mac-mini ~ %
```

4. Finalmente, conectamos con VM 102 desde Windows App

![](https://imgur.com/0wGpcYZ.png)



## Prevención y lecciones aprendidas

1. Instalar rutas persistentes que sobrevivan a un reinicio para garantizar la operatividad del servicio
2. Comenzar el diagnóstico desde el equipo que falla. Diagnosticar de abajo arriba cada capa del modelo OSI. En el punto 2 del **diagnóstico paso a paso** comencé revisando Remote Desktop (capa 7), cuando debí comenzar por capa 3.












