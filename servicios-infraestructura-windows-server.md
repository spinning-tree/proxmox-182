# servicios de infraestructura de windows server

> Para este módulo consultar las secciones 
> - [configuracion basica de windows server](configuracion-basica-windows-server) 
> - [configuracion avanzada de windows server](configuracion-avanzada-windows-server)

# glosario básico

- **rsop.msc**: es la versión gráfica y antigua de gpresult. Abre una consola con el aspecto del editor de GPO, pero solo de lectura, y muestra la configuración final que tiene aplicada ese equipo y ese usuario, junto con la GPO que ganó cada ajuste.
- **eventvwr.msc**: Es el libro de registro del sistema. Cada componente de Windows (el kernel, los servicios, AD, DNS, las GPO) apunta lo que hace en su propio libro, y el visor te deja leerlos, filtrarlos y buscar en ellos
- **LAPS** (Local Administrator Password Solution) gestiona la contraseña del **administrador local** de cada equipo. Le pone una contraseña aleatoria y distinta en cada PC, la cambia sola cada cierto tiempo (30 días por defecto) y la guarda en AD, en el objeto de equipo. Las IPs no intervienen en nada de esto.
- **Asistencia Remota de Windows (`msra.exe`)** se inicia la conexión al PC del usuario, él solo pulsa “Aceptar” y ve lo mismo que el técnico. Si le da permiso, puede tomar el control. Se configura por GPO.
- **Servidor de Impresión**: es un Windows Server que tiene instaladas las impresoras, sus drivers y sus colas. Los clientes no hablan con la impresora: mandan el trabajo al servidor (`\\servidor\impresora`), el servidor lo encola y se lo pasa a la impresora por red.
# herramientas de diagnóstico y soporte

## event Viewer

Para este ejemplo, simularemos dos escenarios: el usuario inserta mal la contraseña, y finalmente, el usuario bloquea su cuenta tras superar el límite de intentos configurado en la DDP.

Antes de proceder, tener en cuenta lo siguiente:
- **Firewall de los clientes por GPO**: habilitar el grupo **Remote Event Log Management** (NP-In, RPC, RPC-EPMAP), con Scope limitado a DC1 y a los PCs de administración.
- **Cuenta**: Al conectarse por otra pc, la MMC le exige elevación, así que se lanza con `Start-Process powershell -Verb RunAs` y desde ahí `eventvwr.msc`.
- En **Connect as another user**, poner el nombre completo del equipo (**Check Names** → PC-02.november.lab).
- Para actualizar las GPOs en clientes -> click derecho en la OU Equipos -> Group Policy Updates. Esto puede dar fallo si la GPO del Firewall no tiene habilitado **Remote Scheduled Tasks Management (RPC, RPC-EPMAP)** y **WMI (WMI-In)**.

![400](https://imgur.com/8rE1MXM.png)

1. Simulamos el intento fallido, tres veces y consultamos desde el eventvwr.msc para la máquina en concreto
	- El 4625 (Inicio de sesión fallido) se registra en la máquina donde se intentó iniciar la sesión, no en el DC.
	- Sub Status **0xC000006A**: el usuario existe y la contraseña es incorrecta.

![400](https://imgur.com/JUhZHeG.png)

2. Comprobamos el evento en DC-1. 
Solo registraba los eventos con éxito del 4776. Se activaron las auditorías para las categorías necesarias. Ver [activar auditoría en DDCP](#activar-auditoria-en-la-ddcp)

- Activadas las auditoría, el DC registra correctamente los intentos fallidos con código `0xC000006A`

![400](https://imgur.com/TIHog60.png)

2. Seguidamente, provocamos el bloqueo de cuenta por exceder el número de intentos.

- Se crea la custom view cuentas_bloqueadas y se filtra por seguridad y 4740 como event id.

![400](https://imgur.com/U0LaStY.png)

> Conclusión: Para seguir el rastro: **4740** (cuenta bloqueada) → **4776** en el DC (_Source Workstation_) → **4625** en el equipo destino (_Source Network Address_ = .100).
### activar auditoria en la DDCP
1. Abrir GPMC y edita **Default Domain Controllers Policy**.
2. Ir a **Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies**.
3. Configurar las subcategorías de la tabla.
4. En el DC, ejecuta `gpupdate /force` y comprueba con `auditpol /get /category:*`.

![400](https://imgur.com/6VjHmnf.png)

|Categoría > Subcategoría|Valor|
|---|---|
|Account Logon > Credential Validation|Success and Failure|
|Account Logon > Kerberos Authentication Service|Success and Failure|
|Account Logon > Kerberos Service Ticket Operations|Success and Failure|
|Logon/Logoff > Logon|Success and Failure|
|Logon/Logoff > Logoff|Success|
|Logon/Logoff > Account Lockout|Failure|
|Account Management > User Account Management|Success and Failure|
|Account Management > Security Group Management|Success|
|_Opcional:_ Policy Change > Audit Policy Change|Success|
|_Opcional:_ Account Management > Computer Account Management|Success|
|_Opcional:_ Logon/Logoff > Special Logon|Success|

> En cuanto defines la auditoría avanzada por GPO, todas las subcategorías que no configures se quedan en _No Auditing_, hay que configurarlas todas.
### principales event ids

#### security

|Código|Significado|
|---|---|
|4624|Inicio de sesión correcto|
|4625|Inicio de sesión fallido (en el equipo destino)|
|4634|Cierre de sesión|
|4648|Inicio de sesión con credenciales explícitas (runas, unidades con otra cuenta)|
|4672|Inicio de sesión con privilegios de administrador|
|4720 / 4726|Usuario creado / eliminado|
|4722 / 4725|Cuenta habilitada / deshabilitada|
|4723 / 4724|El usuario cambia su contraseña / un admin la resetea|
|4728 / 4732 / 4756|Miembro añadido a un grupo global / local de dominio / universal|
|4740|Cuenta bloqueada (en el PDC)|
|4767|Cuenta desbloqueada|
|4768 / 4769|Kerberos: petición de TGT / de ticket de servicio|
|4771|Fallo de preautenticación Kerberos (en el DC)|
|4776|Validación NTLM (en el DC)|
|1102|Log de Security borrado|

**Status / Sub Status (4625, 4776)**

|Código|Significado|
|---|---|
|0xC000006D|Genérico: credenciales incorrectas (mira el Sub Status)|
|0xC000006A|Contraseña incorrecta (el usuario existe)|
|0xC0000064|El usuario no existe|
|0xC0000234|Cuenta bloqueada|
|0xC0000072|Cuenta deshabilitada|
|0xC0000071|Contraseña caducada|
|0xC0000193|Cuenta caducada|
|0xC0000224|Debe cambiar la contraseña en el próximo inicio|
|0xC000006F|Fuera del horario de inicio de sesión permitido|
|0xC0000070|Equipo no permitido para esa cuenta|
|0xC000015B|Tipo de inicio de sesión no concedido (p. ej., sin permiso de RDP)|
|0xC0000133|Desfase horario con el DC|

**Failure Code del 4771 (Kerberos)**

|Código|Significado|
|---|---|
|0x18|Contraseña incorrecta|
|0x12|Cuenta deshabilitada, bloqueada o caducada|
|0x17|Contraseña caducada|
|0x25|Desfase horario (Kerberos tolera 5 minutos)|
|0x6|Usuario desconocido|

**Logon Type**

|Código|Significado|
|---|---|
|2|Interactivo, delante del teclado|
|3|Red (carpetas compartidas, RDP con NLA)|
|4 / 5|Proceso por lotes (tarea programada) / servicio|
|7|Desbloqueo de pantalla|
|10|Escritorio remoto (sesión creada)|
|11|Credenciales en caché|

#### system

|Código|Significado|
|---|---|
|41 (Kernel-Power)|Reinicio sin apagado limpio|
|6008|Apagado inesperado anterior|
|1074|Apagado o reinicio iniciado por un usuario o proceso|
|1030 / 1058 / 1129|Fallos de GPO (no obtiene la directiva / no lee gpt.ini / sin red hacia el DC)|
|5719 (Netlogon)|No puede establecer el canal seguro con un DC|
|3210 (Netlogon)|No puede autenticarse con el DC (relación de confianza rota)|
## soporte remoto

### rdp

Hablitar RDP requiere lo siguiente:
- **Crear grupo Helpdesk** (G_Helpdesk) con las cuentas de soporte 

> Es buena idea no entrar por RDP con cuenta Domain Admin porque expone credenciales

- **Crear la GPO**: En GPMC -> OU Equipos -> crear nueva GPO (GPO_RDP_Clientes)
- **Habilitar RDP.** _Computer Configuration > Policies > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Connections > Allow users to connect remotely by using Remote Desktop Services_ → **Enabled**.

![400](https://imgur.com/yNmh0IT.png)

- **Exigir NLA.** En la misma rama, dentro de _Security_: _Require user authentication for remote connections by using Network Level Authentication_ → **Enabled**.

> NLA (Network Level Authentication) permite comprobar las credenciales antes de iniciar sesión en el equipo remoto (seguridad, y ahorro de recursos)

![400](https://imgur.com/WVp8fYz.png)

- **Firewall.** _Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Inbound Rules > New Rule > Predefined: Remote Desktop_. Marcar las reglas que ofrece y elegir **Allow**. Después, en cada regla, ir a _Properties > Scope > Remote IP address_ y poner las IPs de los PCs de los técnicos.

![400](https://imgur.com/nlaLqYP.png)

- **Autorizar al grupo.** _Computer Configuration > Preferences > Control Panel Settings > Local Users and Groups > New > Local Group_:
    - Group name: **Remote Desktop Users (built-in)**
    - Action: **Update**
    - Members: **Add** → _G_Helpdesk_

![400](https://imgur.com/cEL0JoP.png)

- **Aplicar en el cliente.** En la OU Equipos -> click derecho -> Group Policy Update

> Para aplicar desde DC, hay que habilitar primero por GPO las reglas inbound del firewall para **Remote Schedule Task Management** y **Windows Management Instrumentation (WMI)**

![400](https://imgur.com/WHZBvLE.png)

- **Verificar.**
    - En el cliente, `gpresult /r` para confirmar que se aplica _GPO_RDP_Clientes_.
    - Desde tu PC, `Test-NetConnection pc-02 -Port 3389`.
    - Conéctate con `mstsc /v:pc-02.november.lab` usando la cuenta de soporte.

### laps

Generalmente, en muchas empresas es que todos los PCs tengan el mismo administrador local con la misma contraseña, lo cual es un problema grave de seguridad. LAPS se encarga de gestionar estas contraseñas y guardarlas en AD.

1. **Extender el esquema.** En DC1, desde un PowerShell elevado con una cuenta de **Schema Admins**:  
    `Update-LapsADSchema`
2. **Dejar que los equipos escriban su contraseña** en su objeto de AD:  
    `Set-LapsADComputerSelfPermission -Identity "OU=Equipos,DC=november,DC=lab"`
3. **Dar permiso de lectura a Helpdesk.** Domain Admins ya lo tiene por defecto.  
    `Set-LapsADReadPasswordPermission -Identity "OU=Equipos,DC=november,DC=lab" -AllowedPrincipals "NOVEMBER\G_Helpdesk"`

![677](https://imgur.com/Q0NVVeH.png)

4. **Crear la GPO** _GPO_LAPS_, vinculada a la OU de equipos. Los ajustes están en _Computer Configuration > Policies > Administrative Templates > System > LAPS_:
- _Configure password backup directory_ → **Enabled** → **Active Directory**
- _Password Settings_ → **Enabled** (complejidad alta, longitud 14 o más, antigüedad 30 días)

![678](https://imgur.com/Q0NVVeH.png)

- _Enable password encryption_ → **Enabled**. Requiere nivel funcional 2016 o superior, y con tu DC 2025 lo tienes.
- _Configure authorized password decryptors_ → **Enabled** → _NOVEMBER\DL_LAPS_Decrypt_

> Para el último paso, crear ung rupo DL donde pongas a los usuarios que pueden ver la contraseña (por ejemplo: G_Helpdesk y Domain Admins)

> LAPS rota la contraseña sola después de que alguien use la cuenta. Si no se configura _Post-authentication actions_, por defecto, 24 horas después de usar localadmin, **cambiar la contraseña y cerrar la sesión** de esa cuenta.

5. **Comprobar que la cuenta esté habilitada**: LAPS gestiona por defecto el **Administrator** integrado, y en Windows 11 viene **deshabilitado**. LAPS le cambia la contraseña igualmente, pero no se podrá entrar con esa cuenta. En la misma GPO: _Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Accounts: Administrator account status_ → **Enabled**.

> En caso de tener otra cuenta configurada como administrador local (Ej. `localadmin`) hay que activarlo por GPO -> _Administrative Templates > System > LAPS > Name of administrator account to manage_ → **Enabled** → _localadmin_

![400](https://imgur.com/cuphS0R.png)

6. **Aplicar en el cliente.** `gpupdate /force` y después `Invoke-LapsPolicyProcessing` para forzar la primera rotación.
7. **Verificar.**
- En el cliente: Visor de eventos > _Applications and Services Logs > Microsoft > Windows > LAPS > Operational_.
- En ADUC: _Propiedades_ de pc-02 > pestaña **LAPS**.

![400](https://imgur.com/CUZYlSC.png)

- En PowerShell: `Get-LapsADPassword -Identity pc-02 -AsPlainText`

![400](https://imgur.com/w6vBToj.png)

### winrm
Para habilitar la inyecciones de comandos remotos por PowerShell en clientes.

> La política “Allow remote server management through WinRM” crea un listener HTTP en el 5985. Y cuando se lanza `Invoke-Command` contra un equipo del dominio, Windows usa Kerberos por su cuenta (va por Negotiate) y cifra la sesión sin que tengamos que configurar nada. En caso de necesitar HTTPS por el 5986 (por ejemplo, si no hay Kerberos) habría que montar una CA en el AD.

1. **Crear la GPO** en OU Equipos: _Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Remote Management (WinRM) > WinRM Service > Allow remote server management through WinRM_ → **Enabled**. En el filtro IPv4 pon `*`.

![400](https://imgur.com/YLuSdhh.png)

2. Luego _Computer Configuration > Policies > Windows Settings > Security Settings > System Services > Windows Remote Management (WS-Management)_ → **Automatic**.

![400](https://imgur.com/PXqkuY5.png)

3. En el firewall, añade la regla predefinida **Windows Remote Management**, con el Scope limitado a DC1 y a los PCs de los técnicos.

![400](https://imgur.com/4oq9LCS.png)

4. Aplicar la GPO y reiniciar cliente para que levante el servicio.

> Se puede configurar una GPP (_Computer Configuration > Preferences > Control Panel Settings > Services_, marcando _Startup:_ **Automatic** y _Service action:_ **Start service**) para arrancar el servicio directamente, solo tener en cuenta que no bloquea el servicio (un admin puede detenerlo)

![400](https://imgur.com/Cd4GMgs.png)

6. Probar la conexión

- `Test-NetConnection PC-01 -Port 5985`
- `Test-WSMan PC-01`
- **Sesión interactiva**, como un SSH: `Enter-PSSession -ComputerName pc-02` (para salir, `Exit-PSSession`).
- **Enviar un comando a uno o varios equipos**: `Invoke-Command -ComputerName pc-01,pc-02 -ScriptBlock { Disable-LocalUser -Name admin }`

![596](https://imgur.com/fu9NxLM.png)

# servidor de impresión

## paso previo

1. Para simular el comportamiento de una impresora real, creamos un contenedor LXC de Debian (Para esta sección ver [instalacion-CT-proxmox](instalacion-ct-proxmox))

> Al crear el contenedor, en la pestaña Network, configurar el bridge **vmbr1** con IP fija **10.10.10.50/24** y gateway **10.10.10.1**

2. En el CT, ejecutamos `apt update && apt upgrade -y` y seguidamente `apt install netcat-openbsd`  

> **netcat** es una herramienta que abre conexiones TCP/UDP o escucha en un puerto. Hará de “impresora” escuchando en el 9100.

3. Abrimos el puerto y lo dejamos escuchando. Lo recibido irá a un .prn: `nc -lk 9100 >> /root/trabajos.prn` 

> `-l` lo deja en modo escucha (_listen_); `-k` hace que siga escuchando cuando se cierra una conexión, porque si no se cerraría tras el primer trabajo; `>>` añade lo recibido al final del archivo sin sobrescribirlo; `&` lo ejecuta en segundo plano y te devuelve el prompt.

> `.prn` es la extensión que se usa por convención para los datos “listos para imprimir”: lo que el driver genera y la impresora interpreta, en PCL, PostScript o texto plano según el driver.

4. Comprobar conexiones con `ss -tlnp` desde el CT, y `Test-NetConnection 10.10.10.50 -Port 9100` desde el DC

> Listar los puertos TCP en escucha y el proceso que los usa. `-t` TCP, `-l` en escucha, `-n` números en lugar de nombres, `-p` muestra el proceso. Debe aparecer `0.0.0.0:9100` con user `nc`.

![](https://imgur.com/H3Qt1PZ.png)

6. El siguiente paso es dejar el servicio de escucha fijo, para ello lo convertimos en un servicio de systemd (si se reinicia el CT, el servicio levanta solo)

> systemd es el gestor de servicios de Linux, el equivalente a `services.msc` en Windows. Un archivo _unit_ le dice qué ejecutar, cuándo arrancarlo y qué hacer si se cae.

- Matamos el proceso (`kill <pid>`) que levantamos manual
- Crear el archivo del servicio: `nano /etc/systemd/system/impresora.service`
- Pegamos dentro: 

```ini
[Unit]
Description=Impresora simulada RAW en el puerto 9100
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/bin/sh -c 'exec /usr/bin/nc -lk 9100 >> /root/trabajos.prn'
Restart=always

[Install]
WantedBy=multi-user.target
```

- Activar el servicio

|Comando|Para qué sirve|
|---|---|
|**systemctl daemon-reload**|Hace que systemd vuelva a leer los archivos de servicio y detecte el nuevo.|
|**systemctl enable --now** _impresora_|Lo habilita para que arranque con el sistema (`enable`) y lo inicia en ese momento (`--now`).|
|**systemctl status** _impresora_|Muestra si está activo, su PID y las últimas líneas de su registro. Debe aparecer `active (running)`.|
> Los servicios que instalan los paquetes (CUPS, sshd...) viven en `/lib/systemd/system/`. No se editan ahí, porque una actualización los pisa; usar `systemctl edit nombre`, que crea un override en `/etc`.

## instalar impresora

1. **Instalar el rol en DC1**: En el servidor, en PowerShell elevado:

```powershell
Install-WindowsFeature Print-Services -IncludeManagementTools
```

> En una empresa iría en un servidor miembro aparte, no en un DC. La impresora debe tener **IP fija o reserva DHCP**. Los clientes solo necesitan llegar al servidor (SMB, 445/TCP). El servidor necesita llegar a la impresora (RAW 9100/TCP y, si se usa, SNMP 161/UDP).

> Para administrarlo con RSAT, en PC cliente (powershell elevado): `Add-WindowsCapability -Online -Name Rsat.Print.Management.Tools~~~~0.0.1.0`

2. Instalado el rol, verificamos con `Get-WindowsFeature Print-* ` (muestra los componentes del rol y cuáles están instalados) y `Get-Service Spooler` (comprueba que el servicio de cola de impresión está en ejecución)

![](https://imgur.com/Sc66TMm.png)

3. **Abrir la consola Administración de impresión**. Ejecutar `printmanagement.msc` (o **Server Manager > Tools > Print Management**).
	- **Custom Filters:** vistas predefinidas, como _All Printers_ o _Printers Not Ready_. Son útiles para detectar colas con problemas.
	- **Print Servers > DC1 (local):** el servidor, con drivers, tamaños de papel, puertos por donde envía el trabajo, y las impresoras del servidor
	- **Deployed Printers:** las impresoras desplegadas por GPO desde esta consola.

![](https://imgur.com/FD4HBEv.png)

4. **Iniciar el asistente de nueva impresora**. Clic derecho en **Printers > Add Printer**. 
- **Printer Installation:** seleccionar **Add a TCP/IP or Web Services Printer by IP address or hostname**.
- **Printer Address:**
	- **Type of Device:** TCP/IP Device. Es una impresora que recibe los trabajos directamente por TCP/IP. _Web Services Printer_ es para impresoras que se anuncian con WSD.
	- **Host name or IP address:** la IP de la impresora. El **Port name** se rellena solo.
	- **Auto detect the printer driver to use:** marcar solo si se va a usar el driver que proponga Windows. Para elegir el driver a mano, desmarcar.
- **Additional Port Information Required:** solo aparece si Windows no identifica el dispositivo por SNMP.
	- Si la impresora soporta SNMP: **Standard > Generic Network Card**. Crea un puerto RAW 9100 con SNMP activado.
	- Si la impresora no responde a SNMP: **Custom > Settings…**, con **Protocol: Raw**, **Port Number: 9100** y **SNMP Status Enabled** desmarcado.

> Con SNMP activado, el servidor pregunta periódicamente a la impresora su estado (papel, tóner, errores). Si la impresora no responde, la cola aparece como _Offline_ aunque imprima.

- **Printer Driver:** **Install a new driver** y elegir un driver de **tipo 4**: el del fabricante si existe en versión tipo 4, o un driver de clase de Microsoft (**Microsoft > Microsoft PCL6 Class Driver** o **Microsoft PS Class Driver**).

> con un driver de tipo 4, el cliente no instala nada del servidor. Usa el _Microsoft enhanced Point and Print Compatibility Driver_, que ya trae Windows. Con un driver de **tipo 3**, el cliente necesita una copia **exacta** del driver del servidor. Si no la tiene, o la versión no coincide, intenta instalarla, y desde PrintNightmare (KB5005652) eso solo puede hacerlo un administrador. El resultado es que la impresora no se despliega. Un driver de tipo 3 obliga a preinstalarlo como administrador en cada equipo.

> El tipo se comprueba con `Get-PrinterDriver`, en la columna **MajorVersion** (3 o 4).

- **Printer Name and Sharing Settings:**
	- **Printer Name:** nombre de la cola, con una convención que indique qué es y dónde está (tipo, departamento, número). Ejemplo: `IMP-ECO-01`.
	- **Share this printer:** marcado. Publica la impresora como recurso compartido.
	- **Share Name:** nombre en red (`\\servidor\IMP-ECO-01`), sin espacios ni tildes.
	- **Location:** ubicación física, por ejemplo `Economía - Planta 1`. Los usuarios la ven al buscar impresoras.
	- **Comment:** descripción libre.
- **Printer Found:** revisar el resumen y pulsar **Next**.

> **Publish: No** indica que la impresora no se publica en AD. Publicarla permite buscarla desde _Agregar impresora_, y se activa en **Properties > Sharing > List in the directory**. No es necesario para desplegarla por GPO.

- **Completing:** marcar **Print test page** y pulsar **Finish**. La cola debe quedar vacía: si el trabajo se queda con error, el servidor no llega a la impresora.

![](https://imgur.com/YeuLzG8.png)

## compartir la impresora

Por defecto, **Everyone** tiene permiso **Print**. Se sustituye por un grupo local de dominio que contenga:

- el grupo global de **usuarios** que imprimirán (`G_Economia`);
- el grupo global de **equipos** donde se desplegará la impresora (`G_EQ_Economia`).

> Los equipos necesitan permiso porque, en el despliegue por equipo, la conexión la crea SYSTEM, que se presenta ante el servidor como la cuenta de equipo (`PC-01$`). Sin permiso, el despliegue falla. Para imprimir, Windows sigue comprobando el token del usuario.

1. Crear el grupo DL y añadir los miembros: `DL_IMP_ECO_01_Imprimir` -> Members -> Añadir -> `G_Economia`, `G_EQ_Economia` 

![](https://imgur.com/UcHYasa.png)

2. En **Printers > IMP-ECO-01 > Properties > Security**:
    - **Add** el DL con **Print** permitido.
    - **Remove** Everyone.
    - **Remove** la cuenta de usuario que creó la impresora, si aparece con permisos propios. Ya tiene control como miembro de Administrators, y los permisos deben ir siempre por grupos.
    - Dejar el resto de entradas:
        - **CREATOR OWNER:** cada usuario gestiona sus propios trabajos.
        - **ALL APPLICATION PACKAGES** y **Account Unknown (S-1-15-3-…)**: son los permisos para las aplicaciones de la Tienda. Ese _Account Unknown_ es un _capability SID_, no una cuenta borrada.
        - **Administrators**, **Server Operators** y **Print Operators**: administración.

> **Permisos de impresora:** **Print** permite enviar trabajos y pausar o cancelar los propios. **Manage documents** permite gestionar los trabajos de cualquier usuario y es el permiso típico de un técnico N1. **Manage this printer** da control total sobre propiedades, driver, puerto, uso compartido y permisos.


![](https://imgur.com/tzIp6Qy.png)

3. Desplegar la impresora por GPO: Clic derecho en **IMP-ECO-01 > Deploy with Group Policy**:
	- **GPO name > Browse…** Se abre el explorador de GPOs. En la pestaña **Domains/OUs**, entrar en la OU de los equipos (**LAB > Equipos**), pulsar **Create New Group Policy Object** y llamarla `EQ_Impresoras`. Al crearla dentro de la OU, queda vinculada a ella.
	- Marcar **The computers that this GPO applies to (per machine)**.
		- **Por equipo:** la impresora va ligada al puesto. Tiene sentido para equipos compartidos, salas o un puesto de recepción: cualquiera que se siente en ese PC la tiene.
		- **Por usuario:** la impresora va ligada a la persona y la sigue a cualquier equipo del dominio donde inicie sesión. Es lo coherente con AGDLP y con las unidades de red.
	- Pulsar **Add**, comprobar que aparece en la lista inferior y pulsar **OK**.
	- En `gpmc.msc`, comprobar que `EQ_Impresoras` está vinculada a **LAB/Equipos**. La impresora aparece en **Computer Configuration > Policies > Windows Settings > Deployed Printers**.

![](https://imgur.com/dyPqPhE.png)

4. Restringir Point and Print a los servidores aprobados: Editar `EQ_Impresoras` y configurar **Computer Configuration > Policies > Administrative Templates > Printers**:
	- **Package Point and print - Approved servers:** Enabled. En **Show…**, añadir el nombre corto y el FQDN del servidor (`DC1` y `DC1.november.lab`). Limita desde qué servidores se aceptan paquetes de drivers.
	- **Point and Print Restrictions:** Enabled.
		- **Users can only point and print to these servers:** marcado, con `DC1;DC1.november.lab`.
		- **Users can only point and print to machines in their forest:** desmarcado. La lista de servidores ya es más estricta.
		- **When installing drivers for a new connection:** _Do not show warning or elevation prompt_.
		- **When updating drivers for an existing connection:** _Do not show warning or elevation prompt_.
	- **Limits print driver installation to Administrators:** Enabled. Es la protección de PrintNightmare: los usuarios normales siguen sin poder instalar drivers. Quitar los avisos solo afecta a los administradores y a SYSTEM.
		    
> No se debe desactivar nunca esta protección (`RestrictDriverInstallationToAdministrators=0`) para que un despliegue funcione: reabre la vulnerabilidad.

5. Verificación en el cliente
	- En el cliente, como administrador: `gpupdate /force` y **reiniciar** el equipo.
	- `gpresult /r /scope computer`: en _Applied Group Policy Objects_ debe aparecer `EQ_Impresoras`, y en _security groups_, `G_EQ_Economia` y `DL_IMP_ECO_01_Imprimir`.
	- Iniciar sesión con un usuario del departamento y ejecutar `Get-Printer`: debe aparecer `\\DC1\IMP-ECO-01` con **Type: Connection** y **DriverName: Microsoft enhanced Point and Print Compatibility Driver**.
	- Imprimir una página y comprobar que la cola del servidor queda vacía.






