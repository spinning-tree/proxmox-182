# windows
## básicos

| Comando                                                                                      | Para qué sirve                                                                    |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Start-Process** _programa_ **-Verb RunAs**                                                 | Lanza el programa elevado. Si no eres admin, el UAC pide credenciales             |
| **Start-Process** _programa_ **-Credential** *dominio\usuario*                               | Lanza el programa como otro usuario, sin elevar (no se combina con -Verb RunAs)   |
| **runas /user:**_dominio_\_usuario_ _programa_                                               | Lo mismo con el clásico de CMD                                                    |
| **whoami** [**/groups** \| **/priv**]                                                        | Muestra quién eres, tus grupos y si estás elevado (nivel obligatorio alto)        |
| **hostname**                                                                                 | Nombre del equipo                                                                 |
| **winver**                                                                                   | Versión y compilación de Windows en ventana                                       |
| **systeminfo**                                                                               | Resumen del equipo: SO, dominio, arranque, parches, RAM                           |
| **msinfo32**                                                                                 | Información del sistema detallada en ventana                                      |
| **shutdown** {**/r** \| **/s**} **/t** _segundos_ [**/f**]                                   | Reinicia o apaga el equipo                                                        |
| **quser** [**/server**: *equipo*]                                                            | lista quién tiene sesión de escritorio en el equipo                               |
| **dir \\**{_dominio_ \| _servidor_ \| _ip_}**\**_recurso_                                    | Prueba el acceso a un recurso compartido por nombre de dominio, nombre corto o IP |
| **logoff** _id_ [**/server**:_equipo_]                                                       | cierra una sesión concreta                                                        |
| **Get-HotFix**                                                                               | Actualizaciones instaladas                                                        |
| **Get-Service** [_nombre_]                                                                   | Estado de los servicios                                                           |
| {**Start-Service** \| **Stop-Service** \| **Restart-Service**} _nombre_                      | Arranca, para o reinicia un servicio                                              |
| **Get-Process** [_nombre_]                                                                   | Procesos en ejecución (en CMD: **tasklist**)                                      |
| **Stop-Process -Name** _proceso_ [**-Force**]                                                | Mata un proceso (en CMD: **taskkill /im** _proceso_.exe [**/f**])                 |
| **Get-Process -IncludeUserName** *user*                                                      | qué procesos ejecuta cada cuenta                                                  |
| **Restart-Computer** [**-ComputerName** _equipo_] [**-Force**]                               | Reinicia el equipo local o uno remoto                                             |
| **sfc /scannow**                                                                             | Comprueba y repara archivos del sistema                                           |
| **DISM /Online /Cleanup-Image /RestoreHealth**                                               | Repara la imagen de Windows (lánzalo antes que sfc si sfc falla)                  |
| **chkdsk** _unidad_: [**/f**]                                                                | Comprueba y repara el sistema de archivos                                         |
| **Get-WindowsCapability -Online -Name** _patrón_                                             | Estado de las características a petición (RSAT, etc.)                             |
| **Add-WindowsCapability -Online -Name** _característica_                                     | Instala una característica a petición                                             |
| **Get-Command** _patrón_                                                                     | Busca cmdlets (por ejemplo, `*ADUser*`)                                           |
| **Get-Help** _cmdlet_ [**-Examples**]                                                        | Ayuda y ejemplos de un cmdlet                                                     |
## navaja suiza net

| Comando                                                                                                     | Para qué sirve                                                                |
| ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **net user** [_usuario_] [**/domain**]                                                                      | Ver usuarios, o el detalle de uno: grupos, caducidad, último inicio de sesión |
| **net user** _usuario_ ***** **/domain**                                                                    | Restablecer la contraseña de un usuario del dominio                           |
| **net user** _usuario_ **/active:**{**yes** \| **no**} [**/domain**]                                        | Habilitar o deshabilitar una cuenta                                           |
| **net group** [_grupo_] **/domain**                                                                         | Ver los grupos del dominio y sus miembros                                     |
| **net localgroup** [_grupo_]                                                                                | Ver los grupos locales y sus miembros                                         |
| **net localgroup** _Administrators_ _dominio\usuario_ **/add**                                              | Añadir un usuario a un grupo local                                            |
| **net accounts** [**/domain**]                                                                              | Ver la directiva de contraseñas y de bloqueo                                  |
| **net use** [_X:_] \\_servidor_\_recurso_ [**/user:**_usuario_ *****] [**/persistent:**{**yes** \| **no**}] | Conectar o mapear un recurso de red                                           |
| **net use** [_X:_ \| \\_servidor_\_recurso_] **/delete**                                                    | Desconectar un recurso                                                        |
| **net share** [_nombre_]                                                                                    | Ver los recursos compartidos del equipo                                       |
| **net session**                                                                                             | Ver quién está conectado a este equipo                                        |
| **net view** \\_servidor_                                                                                   | Ver los recursos que comparte otro equipo                                     |
| **net start** \| **net stop** _servicio_                                                                    | Iniciar o detener un servicio                                                 |
## red

| Comando                                                                                         | Para qué                                                                                                                              |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **ipconfig /all**                                                                               | Ver IP, puerta de enlace y DNS configurados                                                                                           |
| **ping** [**-4** \| **-6**] _nombre_                                                            | Resuelve con todo el sistema de Windows (DNS, mDNS, LLMNR, NetBIOS) y prueba la conectividad; **-4** o **-6** fuerza la versión de IP |
| **Test-NetConnection** *nombre_DC* **-Port** *port_number*                                      | Comprobar si llegas a un puerto del DC                                                                                                |
| **Test-WSMan** _equipo_                                                                         | Comprueba si el servicio WinRM del equipo remoto responde.                                                                            |
| **Enter-PSSession -ComputerName** _equipo_                                                      | Abre una sesión interactiva de PowerShell en otro equipo, como si fuera SSH.                                                          |
| **Exit-PSSession**                                                                              | Cierra la sesión remota y vuelve a la consola local.                                                                                  |
| **Invoke-Command -ComputerName** _equipo_[**,**_equipo_...] **-ScriptBlock {** _comandos_ **}** | Ejecuta comandos en uno o varios equipos remotos sin abrir una sesión interactiva.                                                    |
## dns

| Comando                                                                             | Para qué                                                                                                 |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **ipconfig /flushdns**                                                              | Vaciar la caché DNS                                                                                      |
| **ipconfig /displaydns**                                                            | Muestra la cache dns del equipo (Get-DnsClientCache)                                                     |
| **ipconfig /registerdns**                                                           | Volver a registrar el equipo en DNS                                                                      |
| **nslookup** _nombre_ [_servidor_dns_]                                              | Consulta solo al DNS, al configurado o al que indiques; no usa mDNS, LLMNR ni NetBIOS                    |
| **nslookup -type=SRV _ldap._tcp.dc._msdcs.**_dominio_ [_servidor_dns_]              | Pregunta al DNS qué DC ofrecen LDAP en el dominio                                                        |
| **Resolve-DnsName** _nombre_ [**-Server** _servidor_dns_] [**-Type** _tipo_]        | Equivalente de nslookup en PowerShell                                                                    |
| **Get-DnsClientServerAddress**                                                      | Muestra los servidores dns configurados                                                                  |
| **Set-DnsClientServerAddress -InterfaceAlias** _interfaz_ **-ServerAddresses** _ip_ | Cambia el servidor DNS de una interfaz                                                                   |
| **nltest /dsgetdc:**_dominio_ [**/force**]                                          | Localiza un DC del dominio como lo haría Netlogon; **/force** ignora la caché y obliga a buscar de nuevo |
| **Restart-Service netlogon**                                                        | En el DC, vuelve a registrar sus SRV en el DNS                                                           |
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

| Comando                                                           | Para qué                                                         |
| ----------------------------------------------------------------- | ---------------------------------------------------------------- |
| **dcdiag /v**                                                     | Detalles completo de diagnóstico del DC                          |
| **dcdiag /q**                                                     | Solo muestra errores                                             |
| **dcdiag /test:dns**                                              | Prueba DNS completa del DC                                       |
| **nltest /dsgetdc:** *dominio*                                    | Ver qué DC localiza el equipo                                    |
| **nltest /dsregdns**                                              | Volver a registrar los SRV del DC                                |
| **Get-SmbShare**                                                  | Ver que existan SYSVOL y NETLOGON                                |
| **net user** _usuario_ [**/domain**]                              | Datos de una cuenta: bloqueo, caducidad de la contraseña, grupos |
| **net accounts** [**/domain**]                                    | Política de contraseñas y bloqueo vigente                        |
| **Test-ComputerSecureChannel**                                    | Para verificar la relación de confianza DC <-> PC                |
| **Test-ComputerSecureChannel -Repair -Credential** *DOMINIO\user* | Reparar la relación de confianza                                 |
## kerberos

| Comando             | Para qué sirve                                                                                                           |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **klist**           | Lista los tickets de la sesión actual                                                                                    |
| **klist tgt**       | Muestra solo el TGT, con más detalle                                                                                     |
| **klist purge**     | Borra los tickets de la sesión y obliga a pedir otros nuevos, por ejemplo tras añadir al usuario a un grupo              |
| **klist get** _spn_ | Pide un ticket para un servicio concreto, para comprobar si Kerberos funciona contra él (p. ej. _cifs/dc1.november.lab_) |
| **klist sessions**  | Lista las sesiones de inicio abiertas en el equipo, con su LogonId                                                       |
## gpo

| Comando                                                                                       | Qué hace                                                                                                                                       |
| --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **gpupdate** [**/force**] [**/target:**{**computer** \| **user**}] [**/boot** \| **/logoff**] | Aplica las GPOs ya; /force las vuelve a aplicar todas, /target solo equipo o usuario, /boot y /logoff reinician o cierran sesión si hace falta |
| **gpresult /r** [**/scope** {**computer** \| **user**}]                                       | Resumen de las GPOs aplicadas y filtradas                                                                                                      |
| **gpresult** [**/user** _DOMINIO\usuario_] **/h** _informe.html_ [**/f**]                     | Informe completo en HTML; /user elige el usuario y /f sobrescribe el archivo                                                                   |
| **gpresult /v**                                                                               | Detalle verbose en la consola (ajustes que ganaron)                                                                                            |
| **gpresult /z**                                                                               | Detalle super-verbose en la consola (ajustes que ganaron y los que perdieron)                                                                  |
## laps

| Comando                                                                                       | Para qué sirve                                                                                         |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Update-LapsADSchema** [**-Confirm:$false**]                                                 | Añade los atributos de LAPS al esquema de AD. Requiere Schema Admins.                                  |
| **Set-LapsADComputerSelfPermission -Identity** _DN-OU_                                        | Deja que los equipos de la OU escriban su contraseña en AD.                                            |
| **Set-LapsADReadPasswordPermission -Identity** _DN-OU_ **-AllowedPrincipals** _DOMINIO\grupo_ | Da a un grupo permiso para leer el atributo de la contraseña.                                          |
| **Invoke-LapsPolicyProcessing**                                                               | Fuerza ahora la revisión de LAPS: aplica la directiva y rota la contraseña **solo** si ha caducado.    |
| **Reset-LapsPassword**                                                                        | Rota la contraseña al momento, haya caducado o no. Se ejecuta en el cliente, como administrador local. |
| **Set-LapsADPasswordExpirationTime -Identity** _equipo_                                       | Marca la contraseña como caducada en AD, igual que _Expire now_.                                       |
| **Get-LapsADPassword -Identity** _equipo_ [**-AsPlainText**]                                  | Muestra la contraseña de LAPS de un equipo.                                                            |

## visor de eventos

| Comando                                                                                                                  | Para qué sirve                                                        |
| ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| **eventvwr.msc**                                                                                                         | Abrir el Visor de eventos                                             |
| **Get-WinEvent -ListLog** *                                                                                              | Listar todos los registros disponibles                                |
| **Get-WinEvent -ListLog** *_SMB_*                                                                                        | Buscar registros por nombre                                           |
| **Get-WinEvent -LogName** _Security_ [**-MaxEvents** _20_]                                                               | Ver los últimos eventos de un registro                                |
| **Get-WinEvent -FilterHashtable @{LogName='**_Security_**'; Id=**_4625_**}**                                             | Filtrar por ID de evento                                              |
| **Get-WinEvent -FilterHashtable @{LogName='**_Security_**'; Id=**_4624,4625_**}**                                        | Filtrar por varios ID                                                 |
| **Get-WinEvent -FilterHashtable @{LogName='**_Security_**'; Id=**_4625_**; StartTime=(Get-Date).AddMinutes(-**_30_**)}** | Filtrar por ID y por tiempo                                           |
| **Get-WinEvent -FilterHashtable @{LogName='**_System_**'; Level=**_2_**}**                                               | Filtrar por nivel (1 crítico, 2 error, 3 advertencia)                 |
| **Get-WinEvent** ... **\| Where-Object {$_.Message -like '***_10.0.0.11_***'}**                                          | Filtrar por texto del mensaje (por ejemplo, una IP de origen)         |
| **Get-WinEvent** ... **\| Format-List** *                                                                                | Ver el evento completo con todos sus campos                           |
| **Get-WinEvent** ... **-ComputerName** _DC1_                                                                             | Consultar los eventos de otro equipo                                  |
| **Get-WinEvent -LogName** '_Microsoft-Windows-SMBServer/Security_'                                                       | Consultar un registro de aplicaciones y servicios                     |
| **Get-WinEvent -LogName** '_DFS Replication_'                                                                            | Consultar la replicación de SYSVOL (DFSR)                             |
### auditorías

| Comando                                                                            | Para qué sirve                              |
| ---------------------------------------------------------------------------------- | ------------------------------------------- |
| **auditpol /get /category:***                                                      | Ver toda la directiva de auditoría          |
| **auditpol /get /category:**"_Logon/Logoff_"                                       | Ver una categoría                           |
| **auditpol /get /subcategory:**"_Credential Validation_"                           | Ver una subcategoría                        |
| **auditpol /list /subcategory:***                                                  | Listar categorías y subcategorías           |
| **auditpol /set /subcategory:**"_nombre_" **/success:**{**enable** \| **disable**} | Activar o desactivar la auditoría de éxitos |
| **auditpol /set /subcategory:**"_nombre_" **/failure:**{**enable** \| **disable**} | Activar o desactivar la auditoría de fallos |
| **auditpol /set /category:**"_nombre_" **/success:enable /failure:enable**         | Activar una categoría entera                |
| **auditpol /backup /file:**_C:\ruta\audit.csv_                                     | Guardar una copia de la directiva actual    |
| **auditpol /restore /file:**_C:\ruta\audit.csv_                                    | Restaurar la copia                          |
## replicación de ad

| Comando                                                           | Para qué sirve                                      |
| ----------------------------------------------------------------- | --------------------------------------------------- |
| **netdom query fsmo**                                             | Ver qué DC tiene cada rol                           |
| **Get-ADDomain**                                                  | Ver los roles de dominio                            |
| **repadmin /replsummary**                                         | Resumen de la replicación: deltas y fallos          |
| **repadmin /showrepl** [_DC_] [**/errorsonly**]                   | Estado de la replicación de entrada por partición   |
| **repadmin /syncall** [**/AdeP**]                                 | Forzar la replicación con todos los socios          |
| **repadmin /replicate** _destino_ _origen_ _DC=november,DC=lab_   | Forzar la replicación de una partición entre dos DC |
| **repadmin /queue**                                               | Ver las replicaciones pendientes                    |
| **repadmin /showobjmeta** _DC_ "_DN-del-objeto_"                  | Ver qué DC hizo cada cambio en un objeto y cuándo   |
| **repadmin /kcc**                                                 | Recalcular la topología de replicación              |
| **Get-ADReplicationPartnerMetadata -Target** _DC2_                | Ver los socios y la última replicación (PowerShell) |
| **Get-ADReplicationFailure -Target** _DC2_                        | Ver los fallos de replicación (PowerShell)          |
| **dcdiag /test:**{**replications** \| **dns** \| **sysvolcheck**} | Diagnosticar una parte concreta del DC              |
## bitlocker

| Comando                                                                      | Para qué sirve                                                                |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **tpmtool getdeviceinformation**                                             | Muestra si hay TPM, versión, fabricante                                       |
| **manage-bde -status** [_unidad_]                                            | Ver el estado de cifrado, protección y protectores                            |
| **manage-bde -protectors -get** _unidad_                                     | Listar los protectores con sus ID y la clave de recuperación                  |
| **manage-bde -on** _unidad_ [**-RecoveryPassword**] [**-UsedSpaceOnly**]     | Activar BitLocker en una unidad                                               |
| **manage-bde -off** _unidad_                                                 | Desactivar BitLocker y descifrar la unidad                                    |
| **manage-bde -protectors -delete** _unidad_ **-id** "_{ID}_"                 | Borrar un protector concreto (en PowerShell, el ID entre comillas)            |
| **manage-bde -protectors -add** _unidad_ {**-tpm** \| **-RecoveryPassword**} | Añadir un protector TPM o una clave de recuperación nueva                     |
| **manage-bde -protectors -disable** _unidad_ [**-RebootCount** _n_]          | Suspender la protección (_n_ = 0: sin límite; solo en el SO en marcha)        |
| **manage-bde -protectors -enable** _unidad_                                  | Reanudar la protección                                                        |
| **manage-bde -unlock** _unidad_ **-RecoveryPassword** _clave_                | Desbloquear una unidad con la clave de recuperación                           |
| **manage-bde -autounlock -enable** _unidad_                                  | Activar el desbloqueo automático de una unidad de datos                       |
| **manage-bde -forcerecovery** _unidad_                                       | Forzar el modo recuperación en el siguiente arranque (borra el protector TPM) |
| **manage-bde -protectors -adbackup** *unidad* **-id** *{ID}*                 | Añadir manualmente la clave a AD                                              |
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

| Comando                                                                                   | Qué hace                                                                                 |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **sw_vers**                                                                               | Versión de macOS                                                                         |
| **sudo shutdown** { **-r** \| **-h** } **now**                                            | Reinicia (**-r**) o apaga (**-h**) el sistema                                            |
| **top -o** { **cpu** \| **mem** }                                                         | Procesos ordenados por CPU o memoria (**q** para salir)                                  |
| **diskutil list**                                                                         | Discos y particiones                                                                     |
| **man** _comando_                                                                         | Manual del comando (**/** _texto_ busca, **q** sale)                                     |
| **ifconfig** _interfaz_                                                                   | Muestra detalles de interfaz en concreto                                                 |
| **ping** [ **-c** _n_ ] _ip_                                                              | Prueba conectividad IP (ICMP)                                                            |
| **traceroute** _ip_                                                                       | Saltos hasta el destino                                                                  |
| **arp -a**                                                                                | Tabla ARP                                                                                |
| { **nslookup** \| **dig** } _nombre_                                                      | Resuelve un nombre; **dig** da más detalle                                               |
| **nc -vz** _ip_ _puerto_                                                                  | Comprueba si un puerto TCP remoto está abierto                                           |
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



