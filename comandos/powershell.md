# powershell

## Ayuda

| Cmdlet PowerShell                     | Ejecutable / CMD | Para qué sirve                                                                                                             |
| ------------------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Get-Command** _patrón_              | —                | Busca comandos por nombre (por ejemplo, `*ADUser*`); **where** _programa_ en CMD solo localiza archivos, no es equivalente |
| **Get-Help** _cmdlet_ [**-Examples**] | _comando_ **/?** | Muestra la ayuda; **-Examples** muestra ejemplos en PowerShell                                                             |
## Equipo y sistema

| Cmdlet PowerShell                                                          | Ejecutable / CMD                                                 | Para qué sirve                                                                                                      |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
|                                                                            | **whoami** [**/groups** \| **/priv** **/all**]                   | Muestra tu usuario; con **/groups**, grupos y nivel obligatorio; con **/priv**, privilegios, **/all** todo de golpe |
|                                                                            | **hostname**                                                     | Muestra el nombre del equipo                                                                                        |
| **Get-HotFix** [**-Id** _KB..._]                                           |                                                                  | Lista actualizaciones instaladas                                                                                    |
| **Restart-Computer** [**-ComputerName** _equipo_] [**-Force**] [**-Wait**] | **shutdown /r /t** _segundos_ [**/f**] [**/m** **\\\\**_equipo_] | Reinicia el equipo local o remoto; **-Wait** solo sirve en remoto                                                   |
| **Stop-Computer** [**-ComputerName** _equipo_] [**-Force**]                | **shutdown /s /t** _segundos_ [**/f**] [**/m** **\\\\**_equipo_] | Apaga el equipo local o remoto                                                                                      |
| —                                                                          | **shutdown /a** [**/m** **\\\\**_equipo_]                        | Cancela un reinicio o apagado pendiente durante el retardo                                                          |
| —                                                                          | **quser** [**/server:**_equipo_]                                 | Lista quién tiene sesión de escritorio en el equipo                                                                 |
| —                                                                          | **logoff** _id_ [**/server:**_equipo_]                           | Cierra una sesión concreta (el _id_ sale de quser)                                                                  |
| **Start-Process** _programa_ **-Verb RunAs**                               | —                                                                | Lanza el programa elevado; UAC pide confirmación a un admin o credenciales a un usuario estándar                    |
| **Start-Process** _programa_ **-Credential** _dominio\usuario_             | **runas /user:**_dominio\usuario_ _programa_                     | Lanza el programa como otro usuario, sin elevarlo                                                                   |
## servicios y procesos

| Cmdlet PowerShell                                                                       | Ejecutable / CMD                                                               | Para qué sirve                                                                          |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| **Get-Service** [_nombre_]                                                              | **sc.exe query** [_nombre_]                                                    | Muestra el estado de los servicios                                                      |
| {**Start-Service** \| **Stop-Service** \| **Restart-Service**} _nombre_                 | {**net start** \| **net stop**} _servicio_                                     | Arranca, detiene o reinicia un servicio; **net** no tiene opción de reinicio            |
| **Set-Service** _nombre_ **-StartupType** {**Automatic** \| **Manual** \| **Disabled**} | **sc.exe config** _nombre_ **start=** {**auto** \| **demand** \| **disabled**} | Cambia el tipo de inicio de un servicio                                                 |
| **Get-Process** [_nombre_]                                                              | **tasklist**                                                                   | Muestra los procesos en ejecución                                                       |
| **Get-Process -IncludeUserName**                                                        | **tasklist /v**                                                                | Muestra los procesos con su propietario; el cmdlet requiere elevación                   |
| **Stop-Process** {**-Name** _proceso_ \| **-Id** _pid_} [**-Force**]                    | **taskkill** {**/im** _proceso_**.exe** \| **/pid** _pid_} [**/f**]            | Cierra procesos; en **taskkill**, **/f** fuerza la terminación                          |
| **Get-Process \| Sort-Object WorkingSet64 -Descending \| Select-Object -First** _10_    | **taskmgr**                                                                    | Localiza los procesos que más memoria usan; **taskmgr** abre el Administrador de tareas |
## reparación del sistema

| Cmdlet PowerShell                                                            | Ejecutable / CMD                                              | Para qué sirve                                                                                       |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| —                                                                            | **sfc /scannow**                                              | Comprueba y repara archivos del sistema                                                              |
| **Repair-WindowsImage -Online -RestoreHealth**                               | **DISM /Online /Cleanup-Image /RestoreHealth**                | Repara el almacén que usa **sfc**; después ejecuta **sfc /scannow**                                  |
| **Repair-Volume -DriveLetter** _letra_ {**-Scan** \| **-OfflineScanAndFix**} | **chkdsk** _unidad_**:** [**/f**]                             | Comprueba (**-Scan** o sin **/f**) o repara el sistema de archivos (**-OfflineScanAndFix** o **/f**) |
| **Export-WindowsDriver -Online -Destination** _ruta_                         | **DISM /Online /Export-Driver /Destination:**_ruta_           | Exporta los controladores de terceros                                                                |
| —                                                                            | **pnputil /add-driver** _ruta_\\\\*.inf **/subdirs /install** | Añade e instala los controladores exportados que correspondan al equipo                              |
## diagnóstico y hardware

| Cmdlet PowerShell                            | Ejecutable / CMD                                      | Para qué sirve                                                                                                |
| -------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Get-CimInstance Win32_ComputerSystem**     | —                                                     | Muestra fabricante, modelo y RAM física utilizable (en bytes)                                                 |
| **Get-CimInstance Win32_BIOS**               | —                                                     | Muestra la versión de BIOS y el número de serie                                                               |
| **Get-CimInstance Win32_Processor**          | —                                                     | Muestra modelo, núcleos y carga de la CPU                                                                     |
| **Get-CimInstance Win32_OperatingSystem**    | **systeminfo**                                        | Muestra la RAM total y libre (el cmdlet, en KB)                                                               |
| **Get-CimInstance Win32_PhysicalMemory**     | —                                                     | Muestra los módulos de RAM, su capacidad y velocidad                                                          |
| **Get-CimInstance Win32_Battery**            | —                                                     | Muestra el porcentaje de carga de la batería                                                                  |
| —                                            | **powercfg /batteryreport** [**/output** _ruta.html_] | Genera un informe para comparar la capacidad de diseño con la de carga completa                               |
| **Get-PhysicalDisk**                         | —                                                     | Muestra el disco a nivel de hardware: tipo (SSD/HDD) y estado de salud                                        |
| **Get-Disk**                                 | **diskpart** → **list disk**                          | Muestra los discos que usa Windows: número y estado; el cmdlet indica además el estilo de partición (GPT/MBR) |
| **Get-Volume** [**-DriveLetter** _letra_]    | **fsutil volume diskfree** _unidad_**:**              | Muestra el espacio libre de los volúmenes; el cmdlet añade letra, sistema de archivos y estado                |
| **Get-PnpDevice -PresentOnly -Status Error** | **pnputil /enum-devices /problem**                    | Lista dispositivos con problemas                                                                              |

## archivos y carpetas

| Cmdlet PowerShell                                                                            | Ejecutable / CMD                              | Para qué sirve                                                                              |
| -------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Get-Location**                                                                             | **cd**                                        | Muestra la carpeta actual                                                                   |
| **Set-Location** _ruta_                                                                      | **cd /d** _ruta_                              | Cambia de carpeta (**/d** permite cambiar también de unidad)                                |
| **Get-ChildItem** [_ruta_] [**-Force**] [**-Recurse**]                                       | **dir** [_ruta_] [**/a**] [**/s**]            | Lista el contenido, con ocultos o subcarpetas si se indica                                  |
| **(Get-ChildItem** _ruta_ **-Recurse -File -Force \| Measure-Object Length -Sum).Sum / 1GB** | **dir** _ruta_ **/s /a**                      | Calcula el tamaño de una carpeta                                                            |
| **New-Item** _ruta_ **-ItemType Directory**                                                  | **mkdir** _ruta_                              | Crea una carpeta                                                                            |
| **New-Item** _archivo_ **-ItemType File**                                                    | **type nul >>** _archivo_                     | Crea un archivo vacío; si ya existe, ninguno borra su contenido (el cmdlet da error)        |
| **Set-Content** _archivo_ _"texto"_                                                          | **echo** _texto_ **>** _archivo_              | Escribe el texto y **sobrescribe** el contenido                                             |
| **Add-Content** _archivo_ _"texto"_                                                          | **echo** _texto_ **>>** _archivo_             | Añade el texto al final                                                                     |
| **Copy-Item** _origen_ _destino_ [**-Recurse**]                                              | **copy** _origen_ _destino_                   | Copia archivos; para árboles de carpetas en CMD, usa **robocopy** _origen_ _destino_ **/e** |
| **Move-Item** _origen_ _destino_                                                             | **move** _origen_ _destino_                   | Mueve archivos o carpetas                                                                   |
| **Rename-Item** _ruta_ _nombre_                                                              | **ren** _ruta_ _nombre_                       | Cambia el nombre                                                                            |
| **Remove-Item** _ruta_ [**-Recurse**]                                                        | {**del** _archivo_ \| **rmdir /s** _carpeta_} | Borra archivos o carpetas sin pasar por la papelera                                         |
| **Get-Content** _archivo_ [**-Tail** _n_] [**-Wait**]                                        | **type** _archivo_                            | Muestra un archivo; **-Tail** y **-Wait** permiten seguir sus últimas líneas                |
| —                                                                                            | **edit** _archivo_                            | Edita texto en la terminal (si está disponible)                                             |
| **Select-String -Path** _archivo_ **-Pattern** _"texto"_                                     | **findstr /i** _"texto"_ _archivo_            | Busca texto sin distinguir mayúsculas                                                       |
## optional features

| Cmdlet PowerShell                                                          | Ejecutable / CMD                                                  | Para qué sirve                                                                              |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Get-WindowsCapability -Online -Name** _patrón_                           | **DISM /Online /Get-Capabilities**                                | Muestra el estado de las características a petición (p. ej., `RSAT*`); DISM las lista todas |
| **Add-WindowsCapability -Online -Name** _característica_                   | **DISM /Online /Add-Capability /CapabilityName:**_característica_ | Instala una característica a petición                                                       |
| **Get-WindowsOptionalFeature -Online** [**-FeatureName** _nombre_]         | **DISM /Online /Get-Features**                                    | Lista las características opcionales y su estado                                            |
| **Enable-WindowsOptionalFeature -Online -FeatureName** _nombre_ [**-All**] | **DISM /Online /Enable-Feature /FeatureName:**_nombre_ [**/All**] | Activa una característica; con **All**, también sus características superiores necesarias   |
| **Disable-WindowsOptionalFeature -Online -FeatureName** _nombre_           | **DISM /Online /Disable-Feature /FeatureName:**_nombre_           | Desactiva una característica                                                                |
| —                                                                          | **OptionalFeatures**                                              | Abre la ventana «Activar o desactivar las características de Windows» (cliente)             |
## cuentas y grupos

| Cmdlet PowerShell                                                                                  | Ejecutable / CMD                                               | Para qué sirve                                                   |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Get-LocalUser** [_nombre_]                                                                       | **net user** [_usuario_]                                       | Ver usuarios locales o el detalle de uno                         |
| **Set-LocalUser -Name** _usuario_ **-Password (Read-Host -AsSecureString)**                        | **net user** _usuario_ **\***                                  | Cambiar la contraseña local sin mostrarla                        |
| {**Enable-LocalUser** \| **Disable-LocalUser**} **-Name** _usuario_                                | **net user** _usuario_ **/active:**{**yes** \| **no**}         | Habilitar o deshabilitar una cuenta local                        |
| **Get-LocalGroup** [_grupo_]                                                                       | **net localgroup**                                             | Ver grupos locales                                               |
| **Get-LocalGroupMember -Group** _grupo_                                                            | **net localgroup** _grupo_                                     | Ver miembros de un grupo local                                   |
| {**Add-LocalGroupMember** \| **Remove-LocalGroupMember**} **-Group** _grupo_ **-Member** _usuario_ | **net localgroup** _grupo_ _usuario_ {**/add** \| **/delete**} | Añadir o quitar miembros locales (de dominio: _DOMINIO\usuario_) |
| —                                                                                                  | **net accounts**                                               | Ver la directiva local de contraseñas y bloqueo                  |
## recursos compartidos

| Cmdlet PowerShell                                                                                            | Ejecutable / CMD                                                                                                 | Para qué sirve                                                                                   |
| ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Get-ChildItem \\\\**servidor**\\**_recurso_                                                                | **dir \\\\servidor**\\\\_recurso_                                                                                | Probar el acceso; el _servidor_ puede ir por nombre o IP (o usa el dominio en rutas como SYSVOL) |
| **New-SmbMapping -LocalPath** _X:_ **-RemotePath** \\\\**servidor**\\**recurso** [**-Persistent** **$true**] | **net use** _X:_ **\\\\servidor**\\**recurso** **[/user:** *usuario* **]** [**/persistent:**{**yes** \| **no**}] | Mapear una unidad de red                                                                         |
| **Get-SmbMapping**                                                                                           | **net use**                                                                                                      | Ver las conexiones y unidades mapeadas                                                           |
| **Remove-SmbMapping -LocalPath** _X:_                                                                        | **net use** _X:_ **/delete**                                                                                     | Desconectar una unidad                                                                           |
| —                                                                                                            | **net view \\**_servidor_                                                                                        | Ver los recursos visibles de otro equipo                                                         |
| **Get-SmbShare** [**-Name** _nombre_]                                                                        | **net share** [_nombre_]                                                                                         | Ver los recursos compartidos del equipo; en un DC deben estar SYSVOL y NETLOGON                  |
| **Get-SmbShareAccess -Name** _recurso_                                                                       | **net share** _recurso_                                                                                          | Ver los permisos del recurso compartido                                                          |
| **Get-SmbSession**                                                                                           | **net session**                                                                                                  | Ver, elevado y en el servidor, quién está conectado                                              |
| **Get-SmbOpenFile**                                                                                          | **net file**                                                                                                     | Ver, elevado y en el servidor, los archivos abiertos por SMB                                     |
## permisos ntfs

| Cmdlet PowerShell                     | Ejecutable / CMD                                                               | Para qué sirve                                                                                                           |
| ------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Get-Acl** _ruta_ \| **Format-List** | **icacls** _ruta_                                                              | Ver los permisos (el cmdlet muestra además el propietario)                                                               |
| —                                     | **icacls** _ruta_ **/grant "**_dominio\usuario:[(OI)(CI)] M\|F\|RX\|R\|W_**"** | Añadir un permiso a una carpeta, heredable por subcarpetas y archivos                                                    |
| —                                     | **icacls** _ruta_ **/grant:r "**_dominio\usuario:[(OI)(CI)] M\|F\|RX\|R\|W_**" | Sustituir los permisos explícitos concedidos a ese usuario                                                               |
| —                                     | **icacls** _ruta_ **/remove:g** _usuario_                                      | Quitar sus permisos explícitos de tipo Permitir                                                                          |
| —                                     | **icacls** _ruta_ **/inheritance:**{**e** \| **d** \| **r**}                   | Herencia: **e** habilita; **d** deshabilita copiando los heredados; **r** los elimina                                    |
| —                                     | **icacls** _ruta_**/remove**_usuario_                                          | Quita todos los permisos explícitos (Permitir y Denegar) del usuario                                                     |
| —                                     | **icacls** _ruta_ **/reset** [**/t**]                                          | Reemplazar las ACL por las heredadas; **/t** incluye todo el contenido (se pierden permisos personalizados)              |
| —                                     | **takeown /f** _ruta_                                                          | Tomar posesión cuando se ha perdido el acceso; requiere derechos suficientes y no concede por sí solo todos los permisos |

> **Permisos:** `F` control total, `M` modificar, `RX` lectura y ejecución, `R` lectura, `W` escritura; **Herencia:** `(OI)` se aplica a archivos y `(CI)` a subcarpetas; para un archivo, omite ambos. **No propagar: `(NP)`** limita la herencia a los hijos directos; no afecta solo a archivos.

## red

| Cmdlet PowerShell                                   | Ejecutable / CMD                          | Para qué sirve                                                                                                                 |
| --------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Get-NetNeighbor**                                 | **arp -a**                                | Ver la caché de vecinos (IP ↔ MAC); el cmdlet incluye también IPv6                                                             |
| **Remove-NetNeighbor** [**-IPAddress** _ip_]        | **arp -d** [_ip_]                         | Borrar entradas de la caché, elevado; sin IP, **arp -d** borra las de ARP y el cmdlet pide confirmación para las que coincidan |
| **Get-NetIpConfiguration**                          | **ipconfig /all**                         | Ver IP, máscara, puerta de enlace, DNS, DHCP y MAC                                                                             |
| —                                                   | **ipconfig** {**/release** \| **/renew**} | Liberar o renovar la IP obtenida por DHCP; por SSH o RDP, **/release** puede cortarte la conexión                              |
| **Get-NetAdapter**                                  | —                                         | Ver los adaptadores, su estado y velocidad de enlace                                                                           |
| **Test-Connection** _destino_                       | **ping** [**-4** \| **-6**] _destino_     | Probar respuesta por ICMP; que no responda no demuestra que esté apagado                                                       |
| **Test-NetConnection** _destino_ **-TraceRoute**    | **tracert** _destino_                     | Ver los saltos hasta el destino                                                                                                |
| **Test-NetConnection** _destino_ **-Port** _puerto_ | —                                         | Comprobar la conexión a un puerto TCP                                                                                          |
| **Get-NetTCPConnection**                            | **netstat -ano**                          | Ver conexiones y puertos con su PID (**netstat** incluye también UDP)                                                          |
| **Get-NetRoute**                                    | **route print**                           | Ver la tabla de rutas; útil, por ejemplo, para revisar las que añade una VPN                                                   |
## dns




| Cmdlet PowerShell                                                                                                    | Ejecutable / CMD                                                                        | Para qué sirve                                                                                               |
| -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Clear-DnsClientCache**                                                                                             | **ipconfig /flushdns**                                                                  | Vaciar la caché DNS                                                                                          |
| **Get-DnsClientCache**                                                                                               | **ipconfig /displaydns**                                                                | Ver la caché DNS                                                                                             |
| **Register-DnsClient**                                                                                               | **ipconfig /registerdns**                                                               | Volver a registrar el equipo en DNS (elevado)                                                                |
| **Resolve-DnsName** _nombre_ [**-Server** _servidor_dns_] [**-Type** _tipo_] [**-DnsOnly**] [**-NoHostsFile**]       | **nslookup** [**-type=**_tipo_] _nombre_ [_servidor_dns_]                               | Resolver un nombre; **-DnsOnly** evita LLMNR y NetBIOS; **nslookup** consulta directamente a un servidor DNS |
| **Resolve-DnsName \_ldap.\_tcp.dc.\_msdcs.**_dominio_ **-Type SRV**                                                  | **nslookup -type=SRV \_ldap.\_tcp.dc.\_msdcs.**_dominio_                                | Ver los DC que anuncian LDAP en DNS                                                                          |
| **Get-DnsClientServerAddress**                                                                                       | **ipconfig /all**                                                                       | Ver los servidores DNS configurados                                                                          |
| **Set-DnsClientServerAddress -InterfaceAlias** _"interfaz"_ {**-ServerAddresses** _ip_ \| **-ResetServerAddresses**} | **netsh interface ipv4 set dnsservers name=**_"interfaz"_ {**static** _ip_ \| **dhcp**} | Fijar un DNS o volver al automático (elevado)                                                                |
## hora

| Cmdlet PowerShell               | Ejecutable / CMD                                                                                                                                                        | Para qué sirve                                                                                                  |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Get-Date**                    | {**date /t** \| **time /t**}                                                                                                                                            | Ver la fecha y hora                                                                                             |
| —                               | **w32tm /query /status**                                                                                                                                                | Ver la fuente y el estado de sincronización                                                                     |
| —                               | **w32tm /query /peers**                                                                                                                                                 | Ver el estado de los servidores configurados                                                                    |
| —                               | **w32tm /stripchart /computer:**_servidor_ **/samples:**_5_ **/dataonly**                                                                                               | Medir el desfase con otro servidor (en Kerberos, el margen predeterminado es de 5 minutos)                      |
| —                               | **w32tm /resync** [**/nowait**] [**/rediscover**]                                                                                                                       | Forzar la sincronización (elevado); **/rediscover** vuelve a detectar fuentes                                   |
| —                               | **w32tm /config /manualpeerlist:"0.es.pool.ntp.org,0x8 1.es.pool.ntp.org,0x8 2.es.pool.ntp.org,0x8 3.es.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update** | **PDC Emulator:** sincronizar con servidores externos en modo cliente (**0x8**) y anunciarse como fuente fiable |
| **Get-TimeZone**                | **tzutil /g**                                                                                                                                                           | Ver la zona horaria actual                                                                                      |
| **Get-TimeZone -ListAvailable** | **tzutil /l**                                                                                                                                                           | Listar las zonas horarias y sus ID                                                                              |
| **Set-TimeZone -Id** _"id"_     | **tzutil /s** _"id"_                                                                                                                                                    | Cambiar la zona horaria                                                                                         |
## visor de eventos

| Cmdlet PowerShell                                                                                                               | Ejecutable / CMD                                                                       | Para qué sirve                                               |
| ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| —                                                                                                                               | **eventvwr.msc**                                                                       | Abrir el Visor de eventos                                    |
| **Get-WinEvent -ListLog** _patrón_                                                                                              | **wevtutil el**                                                                        | Listar registros; el cmdlet permite buscar (p. ej., `*SMB*`) |
| **Get-WinEvent -LogName** _"registro"_ [**-MaxEvents** _n_]                                                                     | **wevtutil qe** _"registro"_ [**/c:**_n_] **/rd:true /f:text**                         | Ver eventos, del más reciente al más antiguo                 |
| ****Get-WinEvent -FilterHashtable @{LogName=**_"registro"_**; Id=**_id1,id2_**}**                                               | **wevtutil qe** _"registro"_ **/q:"[System[(EventID=** *id* **)]]" /rd:true /f:text**  | Filtrar por ID (el ejemplo clásico, por uno)                 |
| **Get-WinEvent -FilterHashtable @{LogName=**_"registro"_**; Id=** *id* **; StartTime=(Get-Date).AddMinutes(-** *minutos* **)}** | —                                                                                      | Filtrar por ID y tiempo                                      |
| **Get-WinEvent -FilterHashtable @{LogName=**_"registro"_**; Level=** *nivel* **}**                                              | **wevtutil qe** _"registro"_ **/q:"[System[(Level=** *nivel* **)]]" /rd:true /f:text** | Filtrar por nivel: 1 crítico, 2 error, 3 advertencia         |
| **Get-WinEvent** ... **\| Where-Object Message -like "***_texto_***"**                                                          | —                                                                                      | Filtrar por texto del mensaje (p. ej., una IP)               |
| **Get-WinEvent** ... **\| Format-List ***                                                                                       | —                                                                                      | Ver todos los campos                                         |
| **Get-WinEvent** ... **-ComputerName** _equipo_                                                                                 | **wevtutil qe** ... **/r:**_equipo_                                                    | Consultar otro equipo (requiere permisos y acceso remoto)    |
> Ejemplos de _registro_: `Security`, `System`, `Microsoft-Windows-SMBServer/Security` y `DFS Replication`.

## auditorías

|Cmdlet PowerShell|Ejecutable / CMD|Para qué sirve|
|---|---|---|
|—|**auditpol /get /category:***|Ver toda la directiva de auditoría|
|—|**auditpol /get** {**/category:**_"categoría"_ \| **/subcategory:**_"subcategoría"_}|Ver una categoría o subcategoría|
|—|**auditpol /list /subcategory:***|Listar categorías y subcategorías|
|—|**auditpol /set** {**/category:**_"categoría"_ \| **/subcategory:**_"subcategoría"_} [**/success:**{**enable** \| **disable**}] [**/failure:**{**enable** \| **disable**}]|Activar o desactivar la auditoría de éxitos o fallos; indica al menos una de las dos|
## soporte remoto

| Cmdlet PowerShell                                                             | Ejecutable / CMD                | Para qué sirve                                                                                                                                   |
| ----------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| —                                                                             | **mstsc /v:**_equipo_           | Conectar por Escritorio remoto                                                                                                                   |
| —                                                                             | **msra /offerra** [_equipo_]    | Ofrecer Asistencia remota para ver o controlar la sesión del usuario, con su consentimiento (requiere configuración previa, normalmente por GPO) |
| —                                                                             | **ssh** _usuario_**@**_equipo_  | Abrir una consola remota por SSH                                                                                                                 |
| **Test-WSMan** _equipo_                                                       | **winrm id -r:**_equipo_        | Comprobar que responde WinRM                                                                                                                     |
| **Enter-PSSession -ComputerName** _equipo_                                    | **winrs -r:**_equipo_ **cmd**   | Abrir una consola remota por WinRM                                                                                                               |
| **Exit-PSSession**                                                            | **exit**                        | Salir de la sesión remota                                                                                                                        |
| **Invoke-Command -ComputerName** _equipo_ **-ScriptBlock {** _comandos_ **}** | **winrs -r:**_equipo_ _comando_ | Ejecutar un comando en remoto; el cmdlet admite varios equipos separados por comas                                                               |
## firewall

| Cmdlet PowerShell                                                                                                                                                                                                 | Ejecutable / CMD                                                                                                                                                                                         | Para qué sirve                                                                    |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| —                                                                                                                                                                                                                 | **wf.msc**                                                                                                                                                                                               | Abrir la consola del firewall                                                     |
| **Get-NetConnectionProfile**                                                                                                                                                                                      | —                                                                                                                                                                                                        | Ver el perfil de cada conexión activa                                             |
| **Set-NetConnectionProfile -InterfaceAlias** _"interfaz"_ **-NetworkCategory** {**Private** \| **Public**}                                                                                                        | —                                                                                                                                                                                                        | Cambiar el perfil de una red que no sea de dominio                                |
| **Get-NetFirewallProfile -PolicyStore ActiveStore \| Format-Table Name, Enabled, AllowLocalFirewallRules**                                                                                                        | **netsh advfirewall show allprofiles**                                                                                                                                                                   | Ver el estado de los perfiles; el cmdlet muestra además si admiten reglas locales |
| **Set-NetFirewallProfile -Profile** _perfil_ **-Enabled** {**True** \| **False**}                                                                                                                                 | **netsh advfirewall set** {**domainprofile** \| **privateprofile** \| **publicprofile**} **state** {**on** \| **off**}                                                                                   | Activar o desactivar el firewall en un perfil                                     |
| **Get-NetFirewallRule -PolicyStore ActiveStore -DisplayGroup** _"grupo"_                                                                                                                                          | —                                                                                                                                                                                                        | Ver las reglas de un grupo                                                        |
| **Get-NetFirewallRule -PolicyStore ActiveStore -Enabled True -Direction** {**Inbound** \| **Outbound**} **-Action** {**Allow** \| **Block**}                                                                      | —                                                                                                                                                                                                        | Filtrar reglas habilitadas por sentido y acción                                   |
| **Get-NetFirewallRule -PolicyStore ActiveStore -DisplayName** _"nombre"_                                                                                                                                          | **netsh advfirewall firewall show rule name=**_"nombre"_                                                                                                                                                 | Ver una regla; con **ActiveStore**, **EnforcementStatus** y origen                |
| **Set-NetFirewallRule -DisplayName** _"nombre"_ [**-Enabled** {**True** \| **False**}] [**-Profile** _perfil_] [**-Action** {**Allow** \| **Block**}] [**-RemoteAddress** _ip_]                                   | **netsh advfirewall firewall set rule name=**_"nombre"_ **new** [**enable=**{**yes** \| **no**}] [**profile=**_perfil_] [**action=**{**allow** \| **block**}] [**remoteip=**_ip_]                        | Modificar una regla existente (indica al menos un cambio)                         |
| {**Enable-NetFirewallRule** \| **Disable-NetFirewallRule**} **-DisplayName** _"nombre"_                                                                                                                           | **netsh advfirewall firewall set rule name=**_"nombre"_ **new enable=**{**yes** \| **no**}                                                                                                               | Habilitar o deshabilitar una regla                                                |
| **Get-NetFirewallRule -PolicyStore** {**PersistentStore** \| **RSOP**}                                                                                                                                            | —                                                                                                                                                                                                        | Ver reglas del almacén local o las que llegan por GPO                             |
| **New-NetFirewallRule -DisplayName** _"nombre"_ **-Direction** {**Inbound** \| **Outbound**} **-Protocol** {**TCP** \| **UDP**} {**-LocalPort** \| **-RemotePort**} _puerto_ **-Action** {**Allow** \| **Block**} | **netsh advfirewall firewall add rule name=**_"nombre"_ **dir=**{**in** \| **out**} **protocol=**{**TCP** \| **UDP**} {**localport** \| **remoteport**}**=**_puerto_ **action=**{**allow** \| **block**} | Crear una regla; para un servicio entrante se usa el puerto local                 |
| {**Enable-NetFirewallRule** \| **Disable-NetFirewallRule**} **-DisplayGroup** _"grupo"_                                                                                                                           | **netsh advfirewall firewall set rule group=**_"grupo"_ **new enable=**{**yes** \| **no**}                                                                                                               | Habilitar o deshabilitar un grupo de reglas                                       |
| **Remove-NetFirewallRule -DisplayName** _"nombre"_                                                                                                                                                                | **netsh advfirewall firewall delete rule name=**_"nombre"_                                                                                                                                               | Borrar las reglas con ese nombre (compruébalas antes; puede haber varias)         |
## bitlocker

| Cmdlet PowerShell                                                                                                    | Ejecutable / CMD                                                             | Para qué sirve                                                                                             |
| -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Get-Tpm**                                                                                                          | **tpmtool getdeviceinformation**                                             | Ver si hay TPM y su estado; **tpmtool** muestra además versión y fabricante                                |
| **Get-BitLockerVolume** [**-MountPoint** _unidad_]                                                                   | **manage-bde -status** [_unidad_]                                            | Ver cifrado, protección y protectores                                                                      |
| **(Get-BitLockerVolume** _unidad_**).KeyProtector**                                                                  | **manage-bde -protectors -get** _unidad_                                     | Ver protectores, ID y clave de recuperación                                                                |
| **Enable-BitLocker -MountPoint** _unidad_ {**-TpmProtector** \| **-RecoveryPasswordProtector**} [**-UsedSpaceOnly**] | **manage-bde -on** _unidad_ [**-RecoveryPassword**] [**-UsedSpaceOnly**]     | Activar BitLocker; el protector TPM es para la unidad del sistema                                          |
| **Disable-BitLocker -MountPoint** _unidad_                                                                           | **manage-bde -off** _unidad_                                                 | Desactivarlo y **descifrar** la unidad                                                                     |
| **Add-BitLockerKeyProtector -MountPoint** _unidad_ {**-TpmProtector** \| **-RecoveryPasswordProtector**}             | **manage-bde -protectors -add** _unidad_ {**-tpm** \| **-RecoveryPassword**} | Añadir un protector                                                                                        |
| **Remove-BitLockerKeyProtector -MountPoint** _unidad_ **-KeyProtectorId "{_**ID**_}"**                               | **manage-bde -protectors -delete** _unidad_ **-id "{_**ID**_}"**             | Borrar un protector concreto                                                                               |
| **Backup-BitLockerKeyProtector -MountPoint** _unidad_ **-KeyProtectorId "{_**ID**_}"**                               | **manage-bde -protectors -adbackup** _unidad_ **-id "{_**ID**_}"**           | Guardar en AD el protector de contraseña de recuperación (AD debe estar preparado)                         |
| **Suspend-BitLocker -MountPoint** _unidad_ [**-RebootCount** _n_]                                                    | **manage-bde -protectors -disable** _unidad_ [**-RebootCount** _n_]          | Suspender la protección sin descifrar; en la unidad del sistema, **0** la deja suspendida hasta reanudarla |
| **Resume-BitLocker -MountPoint** _unidad_                                                                            | **manage-bde -protectors -enable** _unidad_                                  | Reanudar la protección                                                                                     |
| **Unlock-BitLocker -MountPoint** _unidad_ **-RecoveryPassword** _clave_                                              | **manage-bde -unlock** _unidad_ **-RecoveryPassword** _clave_                | Desbloquear con la clave de recuperación                                                                   |
| **Enable-BitLockerAutoUnlock -MountPoint** _unidad_                                                                  | **manage-bde -autounlock -enable** _unidad_                                  | Activar el desbloqueo automático de una unidad de datos                                                    |
| —                                                                                                                    | **manage-bde -forcerecovery** _unidad_                                       | **Con precaución:** elimina los protectores TPM y fuerza la recuperación en el siguiente arranque          |

# windows server

| **Cmdlet PowerShell**                                                                  | **Ejecutable / CMD**                    | **Para qué sirve**                                                          |
| -------------------------------------------------------------------------------------- | --------------------------------------- | --------------------------------------------------------------------------- |
| **Set-LapsADPasswordExpirationTime -Identity** _equipo_ **-ExpirationTime (Get-Date)** | —                                       | Marca la contraseña LAPS como caducada al momento en AD                     |
| **klist get** _spn_                                                                    | —                                       | Pide un ticket para un servicio concreto (p. ej., `cifs/`_dc1.dominio.lab_) |
| **repadmin /replicate** _destino_ _origen__DC=dominio,DC=lab_                          | —                                       | Fuerza la replicación de una partición entre dos DC                         |

## active directory

| Cmdlet PowerShell                                                                                                                                            | Ejecutable / CMD                                                      | Para qué sirve                                                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Get-ADComputer -Identity** _equipo_ [**-Properties** _propiedades_]                                                                                        | —                                                                     | Consulta en qué OU está el equipo, si está habilitado o su última conexión      |
| **Get-ADUser -Identity** _usuario_ [**-Properties** _propiedades_]                                                                                           | **net user** _usuario_ **/domain**                                    | Ver un usuario de AD; **LastLogonDate** es aproximado                           |
| **Get-ADUser -Filter "Name -like '***texto****'"**                                                                                                           | —                                                                     | Buscar usuarios AD por nombre                                                   |
| **Search-ADAccount** {**-LockedOut** \| **-AccountDisabled** \| **-PasswordExpired** \| **-AccountInactive** [**-TimeSpan** _90.00:00:00_]} [**-UsersOnly**] | —                                                                     | Buscar cuentas bloqueadas, deshabilitadas, caducadas por contraseña o inactivas |
| **Unlock-ADAccount -Identity** _usuario_                                                                                                                     | —                                                                     | Desbloquear una cuenta AD                                                       |
| **Set-ADAccountPassword -Identity** _usuario_ **-Reset -NewPassword (Read-Host -AsSecureString)**                                                            | **net user** _usuario_ *** /domain**                                  | Restablecer la contraseña sin mostrarla                                         |
| **Set-ADUser -Identity** _usuario_ **-ChangePasswordAtLogon $true**                                                                                          | —                                                                     | Obligar a cambiarla en el próximo inicio de sesión                              |
| {**Enable-ADAccount** \| **Disable-ADAccount**} **-Identity** _usuario_                                                                                      | **net user** _usuario_ **/active:**{**yes** \| **no**} **/domain**    | Habilitar o deshabilitar una cuenta AD                                          |
| **Get-ADGroup -Filter ***                                                                                                                                    | **net group /domain**                                                 | Listar grupos AD                                                                |
| **Get-ADPrincipalGroupMembership -Identity** _usuario_                                                                                                       | **net user** _usuario_ **/domain**                                    | Ver los grupos directos de un usuario                                           |
| **Get-ADGroupMember -Identity** _grupo_ [**-Recursive**]                                                                                                     | **net group** _grupo_ **/domain**                                     | Ver miembros; **-Recursive** recorre grupos anidados                            |
| {**Add-ADGroupMember** \| **Remove-ADGroupMember**} **-Identity** _grupo_ **-Members** _usuario_                                                             | **net group** _grupo_ _usuario_ {**/add** \| **/delete**} **/domain** | Añadir o quitar miembros AD                                                     |
| **Get-ADDefaultDomainPasswordPolicy**                                                                                                                        | **net accounts /domain**                                              | Ver la directiva predeterminada del dominio                                     |
| **Get-ADUserResultantPasswordPolicy -Identity** _usuario_                                                                                                    | —                                                                     | Ver la FGPP resultante; si no devuelve nada, se aplica la directiva del dominio |
## Dominio y relación de confianza

| Comando                                                           | Para qué sirve                                                                                           |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **nltest /dsgetdc:**_dominio_ [**/force**]                        | Localiza un DC del dominio como lo haría Netlogon; **/force** ignora la caché y obliga a buscar de nuevo |
| **Test-ComputerSecureChannel**                                    | Verifica la relación de confianza equipo ↔ DC                                                            |
| **Test-ComputerSecureChannel -Repair -Credential** _DOMINIO\user_ | Repara la relación de confianza                                                                          |
| **nltest /dsregdns**                                              | En el DC, vuelve a registrar sus SRV en el DNS                                                           |
| **Restart-Service netlogon**                                      | En el DC, vuelve a registrar sus SRV en el DNS (alternativa a nltest /dsregdns)                          |
## Kerberos

|Comando|Para qué sirve|
|---|---|
|**klist**|Lista los tickets de la sesión actual|
|**klist tgt**|Muestra solo el TGT, con más detalle|
|**klist get** _spn_|Pide un ticket para un servicio concreto, para comprobar si Kerberos funciona contra él (p. ej. _cifs/dc1.november.lab_)|
|**klist purge**|Borra los tickets de la sesión y obliga a pedir otros nuevos, por ejemplo tras añadir al usuario a un grupo|
|**klist sessions**|Lista las sesiones de inicio abiertas en el equipo, con su LogonId|
## GPO

|Comando|Para qué sirve|
|---|---|
|**gpupdate** [**/force**] [**/target:**{**computer** \| **user**}] [**/boot** \| **/logoff**]|Aplica las GPO ya; /force las vuelve a aplicar todas, /target solo equipo o usuario, /boot y /logoff reinician o cierran sesión si hace falta|
|**gpresult /r** [**/scope** {**computer** \| **user**}]|Resumen de las GPO aplicadas y filtradas|
|**gpresult** [**/user** _DOMINIO\usuario_] **/h** _informe.html_ [**/f**]|Informe completo en HTML; /user elige el usuario y /f sobrescribe el archivo|
|**gpresult /v**|Detalle verbose en la consola (ajustes que ganaron)|
|**gpresult /z**|Detalle super-verbose en la consola (ajustes que ganaron y los que perdieron)|
## LAPS

|Comando|Para qué sirve|
|---|---|
|**Update-LapsADSchema** [**-Confirm:$false**]|Añade los atributos de LAPS al esquema de AD. Requiere Schema Admins|
|**Set-LapsADComputerSelfPermission -Identity** _DN-OU_|Deja que los equipos de la OU escriban su contraseña en AD|
|**Set-LapsADReadPasswordPermission -Identity** _DN-OU_ **-AllowedPrincipals** _DOMINIO\grupo_|Da a un grupo permiso para leer el atributo de la contraseña|
|**Get-LapsADPassword -Identity** _equipo_ [**-AsPlainText**]|Muestra la contraseña de LAPS de un equipo|
|**Invoke-LapsPolicyProcessing**|Fuerza ahora la revisión de LAPS: aplica la directiva y rota la contraseña **solo** si ha caducado|
|**Reset-LapsPassword**|Rota la contraseña al momento, haya caducado o no. Se ejecuta en el cliente, como administrador local|
|**Set-LapsADPasswordExpirationTime -Identity** _equipo_|Marca la contraseña como caducada en AD, igual que _Expire now_|
## Salud del DC

|Comando|Para qué sirve|
|---|---|
|**dcdiag /v**|Diagnóstico completo del DC, con detalle|
|**dcdiag /q**|Diagnóstico del DC mostrando solo errores|
|**dcdiag /test:**{**dns** \| **replications** \| **sysvolcheck**}|Diagnosticar una parte concreta del DC (**dns** = prueba DNS completa)|
## Replicación de AD

|Comando|Para qué sirve|
|---|---|
|**netdom query fsmo**|Ver qué DC tiene cada rol FSMO|
|**Get-ADDomain**|Ver los roles de dominio (PDC, RID, Infraestructura)|
|**repadmin /replsummary**|Resumen de la replicación: deltas y fallos|
|**repadmin /showrepl** [_DC_] [**/errorsonly**]|Estado de la replicación de entrada por partición|
|**repadmin /queue**|Ver las replicaciones pendientes|
|**repadmin /syncall** [**/AdeP**]|Forzar la replicación con todos los socios|
|**repadmin /replicate** _destino_ _origen_ _DC=november,DC=lab_|Forzar la replicación de una partición entre dos DC|
|**repadmin /showobjmeta** _DC_ "_DN-del-objeto_"|Ver qué DC hizo cada cambio en un objeto y cuándo|
|**repadmin /kcc**|Recalcular la topología de replicación|
|**Get-ADReplicationPartnerMetadata -Target** _DC2_|Ver los socios y la última replicación (PowerShell)|
|**Get-ADReplicationFailure -Target** _DC2_|Ver los fallos de replicación (PowerShell)|
