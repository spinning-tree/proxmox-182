# windows
# básicos

| Comando                                                                                      | Para qué sirve                                                                  |
| -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Start-Process** _programa_ **-Verb RunAs**                                                 | Lanza el programa elevado. Si no eres admin, el UAC pide credenciales           |
| **Start-Process** _programa_ **-Credential** _dominio_\_usuario_                             | Lanza el programa como otro usuario, sin elevar (no se combina con -Verb RunAs) |
| **runas /user:**_dominio_\_usuario_ _programa_                                               | Lo mismo con el clásico de CMD                                                  |
| **whoami** [**/groups** \| **/priv**]                                                        | Muestra quién eres, tus grupos y si estás elevado (nivel obligatorio alto)      |
| **hostname**                                                                                 | Nombre del equipo                                                               |
| **winver**                                                                                   | Versión y compilación de Windows en ventana                                     |
| **systeminfo**                                                                               | Resumen del equipo: SO, dominio, arranque, parches, RAM                         |
| **msinfo32**                                                                                 | Información del sistema detallada en ventana                                    |
| **Get-HotFix**                                                                               | Actualizaciones instaladas                                                      |
| **gpresult** {**/r** \| **/h** _archivo_.html}                                               | Qué GPOs se aplican al usuario y al equipo                                      |
| **Get-Service** [_nombre_]                                                                   | Estado de los servicios                                                         |
| {**Start-Service** \| **Stop-Service** \| **Restart-Service**} _nombre_                      | Arranca, para o reinicia un servicio                                            |
| **Get-Process** [_nombre_]                                                                   | Procesos en ejecución (en CMD: **tasklist**)                                    |
| **Stop-Process -Name** _proceso_ [**-Force**]                                                | Mata un proceso (en CMD: **taskkill /im** _proceso_.exe [**/f**])               |
| **shutdown** {**/r** \| **/s**} **/t** _segundos_ [**/f**]                                   | Reinicia o apaga el equipo                                                      |
| **Restart-Computer** [**-ComputerName** _equipo_] [**-Force**]                               | Reinicia el equipo local o uno remoto                                           |
| **Get-WinEvent -LogName** {**System** \| **Application** \| **Security**} **-MaxEvents** _n_ | Últimos eventos de un registro                                                  |
| **sfc /scannow**                                                                             | Comprueba y repara archivos del sistema                                         |
| **DISM /Online /Cleanup-Image /RestoreHealth**                                               | Repara la imagen de Windows (lánzalo antes que sfc si sfc falla)                |
| **chkdsk** _unidad_: [**/f**]                                                                | Comprueba y repara el sistema de archivos                                       |
| **net user** _usuario_ [**/domain**]                                                         | Datos de una cuenta: bloqueo, caducidad de la contraseña, grupos                |
| **net accounts** [**/domain**]                                                               | Política de contraseñas y bloqueo vigente                                       |
| **Get-LocalGroupMember** _grupo_                                                             | Miembros de un grupo local (en CMD: **net localgroup** _grupo_)                 |
| **Get-WindowsCapability -Online -Name** _patrón_                                             | Estado de las características a petición (RSAT, etc.)                           |
| **Add-WindowsCapability -Online -Name** _característica_                                     | Instala una característica a petición                                           |
| **Get-Command** _patrón_                                                                     | Busca cmdlets (por ejemplo, `*ADUser*`)                                         |
| **Get-Help** _cmdlet_ [**-Examples**]                                                        | Ayuda y ejemplos de un cmdlet                                                   |

Ojo: el escape `\|` solo hace falta dentro de tablas markdown. En la consola, la barra que aparece en la notación no se escribe, solo indica alternativas.

## red

| Comando                                                    | Para qué                                    |
| ---------------------------------------------------------- | ------------------------------------------- |
| **Test-NetConnection** *nombre_DC* **-Port** *port_number* | Comprobar si llegas a un puerto del DC      |
| **ipconfig /all**                                          | Ver IP, puerta de enlace y DNS configurados |
## dns

| Comando                               | Para qué                                                                    |
| ------------------------------------- | --------------------------------------------------------------------------- |
| **ipconfig /flushdns**                | Vaciar la caché DNS                                                         |
| **ipconfig /displaydns**              | Muestra la cache dns del equipo (Get-DnsClientCache)                        |
| **ipconfig /registerdns**             | Volver a registrar el equipo en DNS                                         |
| **nslookup** *dominio*                | Comprobar que el DNS resuelve el dominio                                    |
| **nslookup -type=SRV** *fqdn_dominio* | Comprobar que DNS anuncia el DC (ejemplo: *ldap.tcp.dc.msdcs.november.lab)* |
| **Resolve-DnsName** *dominio*         | Igual que nslookup, en PowerShell                                           |
## ntp

| Comando                                                                                                                                                                     | Para qué                                                                                    |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **w32tm /config /manualpeerlist:"0.es.pool.ntp.org,0x8 1.es.pool.ntp.org,0x8 2.es.pool.ntp.org,0x8 3.es.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update**<br> | Sincroniza el dc con servidores externos. El :0x8 decide **cómo** consulta: en modo cliente |
| **w32tm /query /status**                                                                                                                                                    | Fuente y estado de la sincronizac                                                           |
| **w32tm /resync /nowait**                                                                                                                                                   | Forzar la sincroniza                                                                        |
| **w32tm /query /peers**                                                                                                                                                     | Muestra el estado de cada servidor configurado                                              |
| **w32tm /resync /rediscover**                                                                                                                                               | Volver a detectar fuentes y sincronizar (tras cambiar la configura                          |
| **tzutil /g**                                                                                                                                                               | Muestra la zona horaria actual                                                              |
| **tzutil /s** "*id_interno*"                                                                                                                                                | Ajusta la zona horaria del equipo                                                           |
| **tzutil /l**                                                                                                                                                               | Para ver correspondencia entre el nombre y el id                                            |
## diagnóstico del dc

| Comando                        | Para qué                                |
| ------------------------------ | --------------------------------------- |
| **dcdiag /v**                  | Detalles completo de diagnóstico del DC |
| **dcdiag /q**                  | Solo muestra errores                    |
| **dcdiag /test:dns**           | Prueba DNS completa del DC              |
| **nltest /dsgetdc:** *dominio* | Ver qué DC localiza el equipo           |
| **nltest /dsregdns**           | Volver a registrar los SRV del DC       |
| **Get-SmbShare**               | Ver que existan SYSVOL y NETLOGON       |
## gpo

| Comando                                                                                                                                                                               | Qué hace                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **gpupdate** [**/force**] [**/target:**{**computer** \| **user**}] [**/boot** \| **/logoff**]                                                                                         | Aplica las GPOs ya; /force las vuelve a aplicar todas, /target solo equipo o usuario, /boot y /logoff reinician o cierran sesión si hace falta |
| **gpresult /r** [**/scope** {**computer** \| **user**}]                                                                                                                               | Resumen de las GPOs aplicadas y filtradas                                                                                                      |
| **gpresult** [**/user** _DOMINIO\usuario_] **/h** _informe.html_ [**/f**]                                                                                                             | Informe completo en HTML; /user elige el usuario y /f sobrescribe el archivo                                                                   |
| **gpresult /z**                                                                                                                                                                       | Detalle máximo en la consola                                                                                                                   |

# proxmox
## nodo

| Comando                               | Qué hace                                          |
| ------------------------------------- | ------------------------------------------------- |
| **pveversion -v**                     | Versión de Proxmox y paquetes                     |
| **pvesh get /nodes**                  | Lista nodos y estado ram, cpu, online             |
| **apt update && apt full-upgrade -y** | Actualizar el sistema (nunca apt upgrade a secas) |
| **systemctl status pveproxy**         | Estado de la interfaz web                         |
| **systemctl status pvedaemon**        | Estado del daemon                                 |
| **systemctl restart pveproxy**        | Reiniciar la web (puerto 8006) si no carga        |
| **journalctl -xe**                    | Logs recientes del sistema                        |
| **pvereport**                         | Informe completo del nodo (diagnóstico)           |
| **pvesm status**                      | Almacenamientos y espacio libre                   |
| **pvesm list local**                  | Contenido de un almacenamiento                    |

## máquinas virtuales

| Comando                                            | Qué hace                                      |
| -------------------------------------------------- | --------------------------------------------- |
| **qm list**                                        | Lista VMs con estado                          |
| **qm start** *vmid*                                | Arrancar                                      |
| **qm shutdown** *vmid*                             | Apagado ordenado (ACPI)                       |
| **qm stop** *vmid*                                 | Apagado forzado (como quitar corriente)       |
| **qm reboot** *vmid*                               | Reiniciar                                     |
| **qm status** *vmid*                               | Estado de una VM                              |
| **qm config** *vmid*                               | Ver configuración (CPU, RAM, discos, red)     |
| **qm set** _111_ **--name** _win11-02_             | Cambiar nombre                                |
| **qm set** *vmid* **--memory 4096**                | Cambiar RAM (MB)                              |
| **qm set** *vmid* **--cores 2**                    | Cambiar núcleos                               |
| **qm set** *vmid* **--onboot 1**                   | Arranque automático con el nodo               |
| **qm clone** *vmid* *nuevo_id* **--name** *nombre* | Clonar VM                                     |
| **qm template** *vmid*                             | Convertir en plantilla                        |
| **qm destroy** *vmid* **--purge**                  | Borrar VM y sus discos                        |
| **qm unlock** *vmid*                               | Quitar bloqueo (tras backup/snapshot fallido) |
| **qm terminal** *vmid*                             | Consola serie (si está configurada)           |
| **cat** */etc/pve/qemu-server/vmid.conf*           | Config de cada VM                             |
## snapshots

| Comando                                   | Qué hace             |
| ----------------------------------------- | -------------------- |
| **qm snapshot** *vmid* *snapshot_name*    | Crear snapshot       |
| **qm listsnapshot** *vmid*                | Listar snapshots     |
| **qm rollback** *vmid* *snapshot_name*    | Volver a un snapshot |
| **qm delsnapshot** *vmid* *snapshot_name* | Borrar snapshot      |
## backups y restauración

| Comando                                                               | Qué hace             |
| --------------------------------------------------------------------- | -------------------- |
| **vzdump** *vmid* **--storage local --mode snapshot --compress zstd** | Backup en caliente   |
| **vzdump --all --storage local**                                      | Backup de todo       |
| **qmrestore** *archivo.vma.zst* *vmid*                                | Restaurar VM         |
# macos

## básicos

| Comando                                        | Qué hace                                                |
| ---------------------------------------------- | ------------------------------------------------------- |
| **sw_vers**                                    | Versión de macOS                                        |
| **sudo shutdown** { **-r** \| **-h** } **now** | Reinicia (**-r**) o apaga (**-h**) el sistema           |
| **top -o** { **cpu** \| **mem** }              | Procesos ordenados por CPU o memoria (**q** para salir) |
| **diskutil list**                              | Discos y particiones                                    |
| **man** _comando_                              | Manual del comando (**/** _texto_ busca, **q** sale)    |
## red

| Comando                                                                                   | Qué hace                                                                                 |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **ping** [ **-c** _n_ ] _ip_                                                              | Prueba conectividad IP (ICMP); **-c** limita el número de paquetes                       |
| **traceroute** _ip_                                                                       | Muestra los saltos hasta el destino                                                      |
| { **nslookup** \| **dig** } _nombre_                                                      | Resuelve un nombre (útil para probar el DNS del DC); **dig** da más detalle              |
| **nc -vz** _ip_ _puerto_                                                                  | Comprueba si un puerto TCP remoto está abierto                                           |
| **arp -a**                                                                                | Tabla ARP: IPs de la LAN y sus MAC                                                       |
| **lsof -iTCP -sTCP:LISTEN -n -P**                                                         | Puertos que están escuchando en el propio Mac                                            |
| **curl -I** _url_                                                                         | Pide solo las cabeceras HTTP (para probar un servicio web)                               |
| **ssh** _usuario_@_ip_                                                                    | Conexión SSH (por ejemplo, al Proxmox)                                                   |
| **netstat -rn** [ **\| grep** _texto_ ]                                                   | Tabla de rutas activa, opcionalmente filtrada                                            |
| **route -n get** _ip_destino_                                                             | Indica qué ruta, gateway e interfaz usaría para llegar a ese destino                     |
| **sudo route -n add -net** _red/prefijo_ _gateway_                                        | Añade una ruta **temporal** (se pierde al reiniciar)                                     |
| **sudo route -n delete -net** _red/prefijo_                                               | Borra una ruta temporal                                                                  |
| **networksetup -getadditionalroutes** _"servicio"_                                        | Muestra las rutas **persistentes** guardadas para ese servicio                           |
| **sudo networksetup -setadditionalroutes** _"servicio"_ [ _red_ _máscara_ _gateway_ ] ... | Define rutas persistentes. **Sustituye** las existentes; sin argumentos, las borra todas |
| **networksetup -getdnsservers** _"servicio"_                                              | Muestra los DNS configurados en el servicio                                              |
| **sudo networksetup -setdnsservers** _"servicio"_ { _ip_dns_ ... \| **empty** }           | Cambia los DNS; **empty** vuelve a los de DHCP                                           |
| **sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder**                         | Vacía la caché DNS                                                                       |


