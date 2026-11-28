# macOS

## Sistema

|Comando|Qué hace|
|---|---|
|**sw_vers**|Versión de macOS|
|**sudo shutdown** {**-r** \| **-h**} **now**|Reinicia (**-r**) o apaga (**-h**) el sistema|
|**top -o** {**cpu** \| **mem**}|Procesos ordenados por CPU o memoria (**q** para salir)|
|**diskutil list**|Discos y particiones|
|**man** _comando_|Manual del comando (**/** _texto_ busca, **q** sale)|

## Conectividad

|Comando|Qué hace|
|---|---|
|**ifconfig** _interfaz_|Muestra detalles de una interfaz en concreto|
|**ping** [**-c** _n_] _ip_|Prueba conectividad IP (ICMP)|
|**traceroute** _ip_|Saltos hasta el destino|
|**arp -a**|Tabla ARP|
|**nc -vz** _ip_ _puerto_|Comprueba si un puerto TCP remoto está abierto|
|**lsof -iTCP -sTCP:LISTEN -n -P**|Puertos que están escuchando en el propio Mac|
|**curl -I** _url_|Pide solo las cabeceras HTTP (para probar un servicio web)|
|**ssh** _usuario_@_ip_|Conexión SSH (por ejemplo, al Proxmox)|

## DNS

|Comando|Qué hace|
|---|---|
|{**nslookup** \| **dig**} _nombre_|Resuelve un nombre; **dig** da más detalle|
|**networksetup -getdnsservers** _"servicio"_|Muestra los DNS configurados en el servicio|
|**sudo networksetup -setdnsservers** _"servicio"_ {_ip_dns_ ... \| **empty**}|Cambia los DNS; **empty** vuelve a los de DHCP|
|**sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder**|Vacía la caché DNS|

## Rutas

|Comando|Qué hace|
|---|---|
|**netstat -rn** [**\| grep** _texto_]|Tabla de rutas activa, opcionalmente filtrada|
|**route -n get** _ip_destino_|Indica qué ruta, gateway e interfaz usaría para llegar a ese destino|
|**sudo route -n add -net** _red/prefijo_ _gateway_|Añade una ruta **temporal** (se pierde al reiniciar)|
|**sudo route -n delete -net** _red/prefijo_|Borra una ruta temporal|
|**networksetup -getadditionalroutes** _"servicio"_|Muestra las rutas **persistentes** guardadas para ese servicio|
|**sudo networksetup -setadditionalroutes** _"servicio"_ [_red_ _máscara_ _gateway_] ...|Define rutas persistentes. **Sustituye** las existentes; sin argumentos, las borra todas|