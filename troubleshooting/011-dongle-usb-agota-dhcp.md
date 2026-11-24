# Pool DHCP agotado por BAD_ADDRESS: adaptador USB-C a Ethernet que responde a ARP de IPs ajenas

| Campo              | Valor                                                            |
| ------------------ | ---------------------------------------------------------------- |
| Ticket             | TS-011                                                           |
| Tipo               | Incidencia                                                       |
| Categoría          | Red / DHCP                                                       |
| Urgencia / Impacto | Alta / Alto (todos los clientes sin reserva se quedan sin red)   |
| Prioridad          | Alta                                                             |
| Estado             | Resuelto (solución definitiva pendiente: sustituir el adaptador) |
| Abierto / Resuelto | 24/11/2026 ~11:30 / 24/11/2026 13:24                             |

## Resumen

Los clientes del dominio no obtienen dirección IP y caen en APIPA (169.254.x.x). El ámbito DHCP se llena de entradas **BAD_ADDRESS** ("This address is already in use"). La causa es el adaptador USB-C a Ethernet de una Surface recién unida al dominio: responde a peticiones ARP de direcciones que no son suyas, así que cada cliente rechaza la IP recibida (DHCPDECLINE) y el servidor la marca como BAD_ADDRESS. Se desconecta el adaptador y se amplía el rango del ámbito para recuperar el servicio. Queda sin explicar un efecto secundario en el servidor DHCP (ver Causa raíz).

## Entorno

|Elemento|Detalle|
|---|---|
|Dominio|november.lab|
|DC1|Windows Server 2025. AD DS, DNS y DHCP. 10.0.0.10|
|Ámbito DHCP|LAB, 10.0.0.100–10.0.0.200 (ampliado a .250 durante la resolución), concesión de 8 días|
|Reserva|PC-02 → 10.0.0.106|
|Router|OpenWrt, 10.0.0.1|
|PC-01, PC-02|Windows 11, VMs en Parallels (Mac mini)|
|PC-03, PC-04|Windows 11, VMs en Proxmox (ThinkPad T480)|
|Surface|Surface Go 2 física, unida al dominio esa mañana. Adaptador USB-C a Ethernet "SZNX LAN 100M" (Naxiang, VID 35B5 / PID 3500), MAC ec-9a-0c-15-79-58|

## Síntoma

- PC-03 cambia de IP a cada arranque y no es accesible por RDP. Cae en APIPA, obtiene otra IP y vuelve a caer, una y otra vez.
- En la consola DHCP aparecen decenas de entradas BAD_ADDRESS consecutivas (desde la .102), una cada 13–14 segundos.
- PC-04, PC-01 y la Surface acaban también en APIPA. `ipconfig /renew` termina por timeout.
- Eventos en DC1 (registro System, origen DHCP-Server): **1342** (el rango del ámbito LAB se ha quedado sin direcciones) y **1063** (no hay direcciones disponibles para el ámbito LAB).

## Contexto previo

- Esa mañana se unió al dominio la Surface, conectada al router OpenWrt con un adaptador USB-C a Ethernet. Durante un rato funcionó con normalidad (concesión .108).

![](https://imgur.com/1QHUM3R.png)

- Se crearon dos GPO de equipo (`EQ_Admins_Locales` y `EQ_Seguridad_Base`). No afectan al servidor DHCP.

## Diagnóstico paso a paso

1. **Reiniciar PC-03.** No aporta nada: pide otra IP y quema una más del pool. Lo correcto habría sido recoger antes la evidencia (`ipconfig /all`, eventos, consola DHCP).
2. **Encender PC-04 (Proxmox).** También cae en APIPA, así que el problema no es solo de PC-03.
3. **Encender PC-01 y PC-02 (Parallels).** PC-01 cae en APIPA, así que el problema no depende del hipervisor.
4. **`ipconfig /release` y `/renew` en la Surface.** En la consola DHCP aparece la .151 como BAD_ADDRESS. Esto demuestra que **las peticiones sí llegan al servidor**: el DC no es inalcanzable.
5. **Mecanismo de BAD_ADDRESS.** Tras recibir el DHCPACK, el cliente envía un **ARP Probe** (origen 0.0.0.0) preguntando por la IP recibida. Si alguien responde, el cliente envía un **DHCPDECLINE** y el servidor marca la dirección como BAD_ADDRESS (estado _Declined_). La detección la hace el cliente, no el servidor. La comprobación con ping del servidor (_Conflict detection attempts_) viene desactivada por defecto.

![](https://imgur.com/Hp9zExD.png)



6. **Tabla ARP de DC1 (`arp -a`).** Casi todas las direcciones BAD_ADDRESS están asociadas a la MAC `ec-9a-0c-15-79-58`, y algunas a `bc-24-11-47-85-e2`.
    - `bc-24-11-…` es el prefijo de las VMs de Proxmox. Es la MAC de PC-03: **víctima**, no culpable (son sus propios intentos con cada IP recibida).
    - `ec-9a-0c-15-79-58` coincide con la concesión de la Surface (.108, columna _Unique ID_). OUI de Shenzhen Naxiang Technology. Es la **culpable**.
    - En las entradas BAD_ADDRESS, el _Unique ID_ no es una MAC, sino la IP en hexadecimal invertida (`8000000a` → 10.0.0.128).


![](https://imgur.com/xLcvQgX.png)

![](https://imgur.com/MM24Jld.png)

7. **Desconectar el adaptador de la Surface.** No aparecen BAD_ADDRESS nuevas.
8. **Limpieza del pool.** Las BAD_ADDRESS no se borran: ni desde la consola (Delete), ni con `Remove-DhcpServerv4Lease -ScopeId 10.0.0.0 -BadLeases`, ni por IP. No da ningún error y la hora de concesión no cambia, así que no se están recreando. **Reconcile** indica que la base de datos es coherente.
9. **Los clientes siguen sin IP (timeout).** PC-02, con reserva, completa el DORA (`/release` + `/renew`), lo que descarta el servicio DHCP parado y el camino de red. El fallo afecta solo a los clientes que tienen que sacar su IP del pool.
10. **Contradicción en el servidor.** `Get-DhcpServerv4ScopeStatistics` indica 48 direcciones libres, pero a cada petición se registran nuevos 1342 y 1063. Sin directivas (Policies), filtros (Allow/Deny) ni failover configurados.

![](https://imgur.com/ow7n9PC.png)

![](https://imgur.com/ZCveiB8.png)

![](https://imgur.com/sD5kKNX.png)

11. **Ampliar el rango del ámbito a .250.** PC-01 obtiene la .201 al instante. Esto confirma que el servidor trataba la .152–.200 como no disponible.

## Causa raíz

- **Confirmada:** el adaptador USB-C a Ethernet de la Surface (Naxiang "SZNX LAN 100M", chip propio, sin web oficial con drivers) responde a ARP de direcciones IP ajenas. Cada cliente rechaza la IP ofrecida con un DHCPDECLINE y el servidor va marcando el pool entero como BAD_ADDRESS.
- **Sin confirmar:** por qué el adaptador responde. La hipótesis es un fallo del firmware o del driver en las funciones de descarga del adaptador (respuesta a ARP en reposo). No se comprobó en qué estado de la Surface ocurre.
- **Efecto secundario sin explicar:** el servidor DHCP no repartía la .152–.200 aunque sus estadísticas las daban por libres, y no permitía borrar las BAD_ADDRESS.

## Solución

1. Desconectar el adaptador USB-C a Ethernet de la Surface (solución temporal).
2. Ampliar el rango del ámbito LAB de .200 a .250 (antes se comprobó que no hay IPs fijas en .201–.250).
3. **Pendiente:** sustituir el adaptador por uno con chip conocido (ASIX o Realtek, compatible con los drivers integrados de Windows) antes de volver a conectar la Surface por cable.

## Verificación

- PC-01 obtiene 10.0.0.201 por DHCP y aparece en _Address Leases_ como `pc01.november.lab`.
- No aparecen BAD_ADDRESS nuevas tras los `ipconfig /renew`.
- Pendiente: confirmar PC-03 y PC-04, y comprobar el 02/12/2026 si caducan las BAD_ADDRESS (.102–.151) y si el servidor vuelve a repartir la .152–.200.

## Prevención y lecciones aprendidas

- El cliente DHCP comprueba con ARP la IP recibida antes de usarla, y si alguien responde, la rechaza (DHCPDECLINE). Una cascada de BAD_ADDRESS indica que **algo responde a ARP por esas IPs**, no que el servidor falle.
- La consola DHCP solo muestra lo que ese servidor ha repartido: que una IP no aparezca en _Leases_ no significa que nadie la use.
- `arp -a` en un equipo del mismo segmento (DC1) identifica rápido la MAC que responde. Se cruza con la columna _Unique ID_ de las concesiones.
- Recoger la evidencia antes de reiniciar. Aislar cambiando una sola variable cada vez (otro cliente, otro hipervisor, cliente con reserva frente a cliente sin reserva).
- No descartar un evento porque contradiga otro dato: el 1342 / 1063 era correcto aunque las estadísticas dijeran otra cosa.
- Antes de conectar hardware nuevo a la red, revisar el adaptador (VID/PID en _Device Manager > Details > Hardware Ids_). Usar adaptadores con chip conocido.