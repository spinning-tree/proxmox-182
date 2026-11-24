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
- **TPM**: (*Trusted Platform Module*) es un módulo de seguridad del equipo, que puede ser un chip físico o emulado. Guarda y cifra claves, como la de bitlocker.
- **Bitlocker**: Es una herramienta de cifrado de disco en Windows: windows genera la clave que cifra/descifra el disco -> se la entrega a TPM que la sella y la guarda en disco -> el equipo inicia sin problemas -> el gestor de arranque toma esta clave y se la entrega al TPM -> TPM la descifra y se la entrega a Windows que carga la clave en RAM durante el uptime para cifrar/descifrar.
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

| Código                   | Significado                                                                    |
| ------------------------ | ------------------------------------------------------------------------------ |
| 551 (SMBServer/Security) | Fallo de autenticación de sesión SMB                                           |
| 4604 (DFS Replication)   | SYSVOL inicializado en un DC nuevo                                             |
| 4624                     | Inicio de sesión correcto                                                      |
| 4625                     | Inicio de sesión fallido (en el equipo destino)                                |
| 4634                     | Cierre de sesión                                                               |
| 4648                     | Inicio de sesión con credenciales explícitas (runas, unidades con otra cuenta) |
| 4672                     | Inicio de sesión con privilegios de administrador                              |
| 4720 / 4726              | Usuario creado / eliminado                                                     |
| 4722 / 4725              | Cuenta habilitada / deshabilitada                                              |
| 4723 / 4724              | El usuario cambia su contraseña / un admin la resetea                          |
| 4728 / 4732 / 4756       | Miembro añadido a un grupo global / local de dominio / universal               |
| 4740                     | Cuenta bloqueada (en el PDC)                                                   |
| 4767                     | Cuenta desbloqueada                                                            |
| 4768 / 4769              | Kerberos: petición de TGT / de ticket de servicio                              |
| 4771                     | Fallo de preautenticación Kerberos (en el DC)                                  |
| 4776                     | Validación NTLM (en el DC)                                                     |
| 1102                     | Log de Security borrado                                                        |


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

# replicación dc
Para garantizar una alta disponibilidad, se implementa replicación del controlador de dominio. La idea aquí es que dos DC actúen como dos copias: escribes en cualquiera y el otro se pone al día solo. 

> AD es **multimaestro**: altas, bajas y cambios se hacen desde cualquier DC escribible y se replican al resto.

- Arrancar windows server, configurar ip fija, hostname y DNS preferido (IP de DC1), de lo contrario DC2 no encuentra el dominio y la promoción falla. Comprobar con `ping dominio` y `nsloookup dominio`
- Instalar el rol AD DS: Administrador del servidor → Agregar roles → Servicios de dominio de Active Directory -> siguiente, siguiente
- Al terminar, promocionar a DC: Elegir **Agregar un controlador de dominio a un dominio existente** -> Credenciales de un administrador del dominio. (Dejar marcados Servidor DNS y Catálogo global -> Replicar desde: dejar _cualquier DC_ o elegir _DC1_.

> OJO: A la hora de ingresar las credenciales de administrador, utilizar el FQDN (dotted domain\user o bien user@dotted domain). En este ejemplo: november.lab\administrator

- Delegación DNS desmarcada. Instalar desde medios (IFM) desmarcado. Replicar desde: en producción se elige uno concreto (DC cercano o con poca carga). Para este ejemplo, vale cualquier opción.

## próximos pasos

- Comprobar con `net share` que SYSVOL y NETLOGON aparecen, verificar el DNS preferido de DC2 (que sea DC1), y comprobar que las zonas DNS se replicaron correctamente.
- **DHCP:** en la consola DHCP de DC1 → ámbito → **Opciones de ámbito** → **006 Servidores DNS**. Añadir 10.0.0.11 debajo de 10.0.0.10 y, en un cliente, ejecutar **ipconfig /renew** y luego **ipconfig /all** para que reciba ambos dcs como servidores dns

## dhcp failover

Por defecto, el rol DHCP se queda en dc1. Si queremos que dc2 también actúe como servidor DHCP, configuramos el failover (**TCP 647**).

- **Instalar el rol DHCP en DC2:** luego la bandera amarilla con Completar la configuración de DHCP para crear los grupos de seguridad y autorizar el servidor en AD. (**No crear ningún ámbito en DC2.** Se lo pasa DC1)
- En la consola DHCP de DC1 -> click derecho en scope (o en ipv4 para incluir todos los ámbitos) -> Configurar Failover -> Servidor asociado: dc2.november.lab -> Nombre de la relación -> MLCT: 1 hora -> Modo: Equilibrio de carga al 50% -> Intervalo -> opcional -> secreto compartido -> Finalizar.

> OJO: Las opciones de ámbito se replican, las de servidor no (hay que configurar a mano). Si modificamos algo -> click derecho en -ambito -> Replicar Failover scopes.

> Sobre el **Switchover Interval**: es el tiempo que espera en _Communication Interrupted_ antes de pasar solo a _Partner Down_. Viene **desactivado** por defecto (si se activa, propone 60 minutos). Mientras está desactivado, el paso a _Partner Down_ es manual.

- Probar en cliente:
	- `nltest /dsgetdc:november.lab /force` para ver que DC encuentra el cliente
	- **DNS**: Ojo, nslookup pregunta solo al primer dns y entra en timeout. Mejor usar `Resolve-DnsName` o `ping` con el FQDN.
	- **DHCP**: `ipconfig /release` y `ipconfig /renew`. Comprobar en DC2 con `Get-DhcpServer4Failover` 
	- **ADDS**: Crear, modificar o restablecer contraseñas
	- Al encender DC1, consultar con `repadmin /replsummary` 

# proteccion de datos
## shadow copies (instantáneas)

Las **shadow copies** hace fotos programadas de un volumen del servidor. La primera foto marca el estado de partida, y a partir de ahí solo se guardan los bloques que cambian. El usuario recupera él mismo versiones antiguas o archivos borrados desde Previous Versions. 

> Es como un “ctrl + z” a nivel de disco, pero no sustituye a la copia de seguridad: si se pierde el disco, se pierden también las fotos.

1. Abrir el explorador de archivos, ir a **This PC**, hacer clic derecho en el volumen de los datos (C:) y elegir **Configure Shadow Copies...**
2. Seleccionar el volumen y pulsar **Settings...**:
    - **Located on this volume:** dejar el mismo volumen.
    - **Maximum size:** **Use limit**, por ejemplo 2–5 GB en el lab. El mínimo es 300 MB.
3. Pulsar **Schedule...** y revisar la programación. Por defecto son las 7:00 y las 12:00, de lunes a viernes. En el lab se puede dejar así o añadir otra hora. Aceptar.
4. Seleccionar el volumen y pulsar **Enable**. Al confirmar se crea la primera instantánea.
5. Comprobarlo desde PowerShell o CMD como administrador con `vssadmin list shadows`.

![](https://imgur.com/XtUlomm.png)

6. Probar en cliente: crear un archivo de texto, pulsar **create now** en el servidor, luego borrarlo o modificarlo y restaurarlo con click derecho sobre el recurso -> Previous Versions.

![](https://imgur.com/ofPux84.png)


> Cuidado con **Restore** sobre una carpeta entera, porque sobrescribe todo su contenido actual. Lo más seguro es **Open** y copiar solo lo que haga falta.

## file server

> Antes de arrancar la VM, añadir en Proxmox un segundo disco (Hardware > Add > Hard Disk, por ejemplo de 20 GB)

![](https://imgur.com/JDMCaa8.png)

- Levantado el servidor, abrir `ncpa.cpl` y definir ip (gateway, dns) y añadir al dominio como una pc cliente

>  En ADUC, crear la OU **Servidores** dentro de NOVEMBER, mover el objeto FS01 desde **Computers** a **Servidores**.

- En FS01, abrir `diskmgmt.msc`, inicializar (online) el disco nuevo en **GPT** y crear un volumen simple NTFS con la letra **D:** y la etiqueta **Datos**.

![](https://imgur.com/JWx6UKX.png)

- Instalar el rol **File Server**, desde Server Manager en FS1

![](https://imgur.com/zQXNCfw.png)

8. Crear `D:\Recursos` con las subcarpetas `Economia` y `Operaciones` -> Propiedades -> Compartir Recursos: **Advanced Sharing**, Authenticated Users con **Change**. 
9. Permisos NTFS. En Recursos: Authenticated Users con **Read & execute**, **This folder only**. Subcarpetas: quitar la herencia y añadir su DL con **Modify**.
10. Activar ABE para el recurso compartido en Server Manager.
11. Probar en cliente

## windows server backup
Es la herramienta de copias de seguridad que trae Windows Server, y se instala como característica. Puede copiar archivos sueltos, volúmenes enteros, el **System State** (incluye la base de datos de AD y SYSVOL) o  **Bare Metal Recovery** para reconstruir el servidor desde cero. 

Para este caso, configuraremos las copias de seguridad en un servidor de archivos.

- En Proxmox, añadir un disco de 40 GB a FS1 (se puede añadir en caliente)

![](https://imgur.com/xxbllRF.png)

- Abrir `diskmgmt.msc`, poner el disco nuevo **Online** e inicializarlo en **GPT**. No hace falta crear ningún volumen, Windows Server Backup lo formatea.
- Instalar la característica en los dos servidores: **Add Roles and Features > Features > Windows Server Backup**.

![](https://imgur.com/uE0DI6j.png)

- Abrir Tools -> Windows Server Backup (`wbadmin.msc`) > Local Backup > click derecho -> Backup Schedule -> Full server -> Once a day y una hora -> elegir Back up to a hard disk that is dedicated for backups -> Show All Available Disks -> marcar el disco nuevo y aceptar el aviso de formateo.

> Cuando eliges un disco dedicado, Windows Server Backup lo formatea y **le quita la letra de unidad** para que no aparezca en el explorador. Así ningún usuario ni programa escribe ahí por error, y un malware que recorre las unidades tampoco lo encuentra tan fácilmente.

- Terminar el asistente.

![](https://imgur.com/eOw5HYa.png)

- Para no esperar a las 21:00, lanzar **Backup Once...** > **Scheduled backup options** > **Backup** y esperar a que acabe.

![](https://imgur.com/OcI77FT.png)

- Simular el incidente: borrar `D:\Recursos\Economia` desde PC Cliente con acceso
- Pulsar **Recover...** > **This server** > fecha de la copia > **Files and folders**, y seleccionar `D:\Recursos\Economia`.
- Elegir **Original location** y dejar marcado **Restore access control list (ACL) permissions**. Pulsar **Recover**.

![](https://imgur.com/IxJjUum.png)

- Comprobarlo con riri y con chachi: la carpeta vuelve, riri entra y modifica, y chachi no la ve.

## papelera ad

Con la Papelera activada, un objeto borrado se guarda entero (grupos, atributos e incluso la contraseña) durante 180 días en el contenedor **Deleted Objects**. Se activa una vez para todo el bosque, no se puede desactivar y solo protege lo que se borre a partir de ese momento.

A modo de ejemplo, simularemos la eliminación/restauración de un usuario en AD.

- En **Active Directory Administrative Center** (`dsac.exe`) -> seleccionar el dominio -> Panel **Tasks** a la derecha -> **Enable Recycle Bin...** y aceptar el aviso de que es irreversible.

![](https://imgur.com/rSO1lve.png)

- Pulsar **F5**. Si la opción aparece en gris, ya está activada. Si no, esperar a la replicación y refrescar otra vez.
- Crear un usuario de prueba, por ejemplo `prueba01`, en la OU Usuarios. Añadirlo a un grupo, definir **Description** (para poder comprobar después que vuelve con todo)
- Borrar `prueba01` desde ADUC.
- En ADAC, abrir **november (local) > Deleted Objects**. Si no aparece, pulsar F5.
- Clic derecho en `prueba01` > **Restore**. Lo devuelve a su OU original. **Restore To...** sirve para elegir otra.
- Comprobar que el usuario vuelve a la OU Usuarios, con la descripción, como miembro del GG de Economía.

> Si alguien borra una OU entera, se restaura primero la OU y luego su contenido: un usuario no puede volver a una OU que no existe.
## backup gpo

GPMC tiene su propio sistema de copias para las GPO. Guarda en una carpeta las dos partes de cada GPO (la configuración en AD y los archivos de SYSVOL), junto con su security filtering y su delegation. 

> OJO: Lo que **no** guarda son los **vínculos**: esos pertenecen a la OU o al dominio, no a la GPO. Después de restaurar una GPO borrada, hay que volver a vincularla.


- Crear una carpeta para las copias, por ejemplo `C:\GPOBackups`, a la que solo tengan acceso los administradores.
- Abrir `gpmc.msc` > **Group Policy Objects**:
    - Una GPO: clic derecho en la GPO > **Back Up...** > carpeta y descripción > **Back Up**.
    - Todas: clic derecho en **Group Policy Objects** > **Back Up All...**

![](https://imgur.com/Tpbd0Mv.png)

- Borrar esa GPO en **Group Policy Objects** (así se eliminan también sus vínculos)
- En un cliente, ejecutar `gpupdate /force`, cerrar sesión y volver a entrar. Comprobar que no se aplica la GPO.
- Clic derecho en **Group Policy Objects** > **Manage Backups...**, seleccionar la copia y pulsar **Restore**.

![](https://imgur.com/ivo4Fc0.png)

- Volver a vincularla: clic derecho en la OU > **Link an Existing GPO...** > elegir la GPO restaurada.
- En el cliente, `gpupdate /force` y volver a iniciar sesión. La unidad vuelve.

# wsus (anillos de parches)
WSUS (*Windows Server Update Services*) es un servicio que descarga los parches de Microsoft una sola vez y los reparte a los equipos del dominio. Funciona como un almacén central: WSUS se sincroniza con Microsoft Update y mediante GPO, los clientes se apuntan al servidor local en lugar de ir a Internet.

> Los anillos de parches consisten en no parchear todos los equipos a la vez. Primero se parchea un grupo piloto y, si no se rompe nada, el resto.

> OJO: wsus es legacy a partir de 2025. Lo sustituye Windows Update for Business con Intune o Autopatch.
## cómo funciona

| Paso            | Quién                            | Qué pasa                                                                                                                                                                                                                 |
| --------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Sincronización  | WSUS ↔ Microsoft Update          | A la hora programada en **Options > Synchronization Schedule**, WSUS descarga los metadatos de los parches nuevos. **No descarga los archivos.**                                                                         |
| Aprobación      | Administrador o regla automática | Dos formas: automatic approvals o manual. En la primera van las pcs pilotos. En la segunda las pcs produccion. Como configuras aprobación automática, a la hora indicada, wsus descarga metadatos y las actualizaciones. |
| Búsqueda (scan) | Cliente → WSUS                   | El cliente pregunta "¿qué hay aprobado para mi grupo?". Lo hace solo, cada ~22 h o al arrancar. Los pilotos descargan entonces automáticamente.                                                                          |
| Descarga        | Cliente ← WSUS                   | El cliente descarga el parche desde el servidor, en segundo plano. No tiene hora fija. Los pilotos descargan solos, los produccion cuando se aprueben                                                                    |
| Instalación     | Cliente                          | A la hora configurada en la GPO. Si a esa hora aún no lo ha descargado, lo instala al día siguiente.                                                                                                                     |

> **Conclusión**: con sincronización a las 02:00 wsus descarga las actualizaciones. Pasadas 22 horas, los clientes preguntan: los pilotos descargan ahí mismo, los de producción esperan aprobación. Se descarga la actualización en pc client, pero no se actualiza hasta hora programada en GPO.

- **Aprovisionar un disco** con al menos 40 GB libres. Ahí se guardarán los archivos de las actualizaciones. Luego crear la carpeta `U:\WSUS`.
- **Instalar el rol:** Server Manager > Manage > Add Roles and Features > Windows Server Update Services > Role services: WID Connectivity y WSUS Services > Content location: `U:\WSUS` > Install

> WID es la base de datos interna de Windows y basta para entornos pequeños y medianos.

- **Ejecutar Launch Post-Installation tasks** desde la notificación de Server Manager. Esto crea la base de datos y el sitio de IIS en el puerto 8530.
- **WSUS Configuration Wizard**: Server Manager -> Tools -> wsus -> Choose Upstream Server: Synchronize from Microsoft Update -> Specify Proxy Server: sin proxy -> Start Connecting -> Choose Languages: _Download updates only in these languages_ > English -> Choose Products: desmarcar todo y marcar solo Windows 11. Incluye x64 y ARM64 -> Choose Classifications: Critical Updates y Security Updates -> Set Sync Schedule: _Synchronize automatically_, una vez al día a las 02:00 -> Marcar Begin initial synchronization y pulsar Finish.

![](https://imgur.com/6vt8vIv.png)

- **Consola de Update Services** -> En Options -> Computers -> **Use Group Policy or registry settings on computers**. Así cada equipo se coloca en su grupo según su GPO.

> Esperar a que actualice. Tarda un poco

![](https://imgur.com/G2kUm0W.png)

- En **Computers > All Computers**, clic derecho > **Add Computer Group**: crear **Piloto** y **Produccion**. Los nombres tienen que coincidir exactamente con los de las GPO.
- **Crear OUS:** En la OU Grupos, crear dos grupos globales de seguridad: GG_WSUS_PILOTO y GG_WSUS_PRODUCCION, y añadir las cuentas de equipo. Piloto: PC01 y PC03; Producción: PC02 y PC04.
- **Crear GPOs:** EQ_WSUS_PILOTO y EQ_WSUS_PRODUCCION, ambas vinculadas a la OU Equipos -> Security Filtering: quitar _Authenticated Users_ y añadir el grupo del anillo -> Pestaña Delegation: añadir _Authenticated Users_ con permiso Read. 
	- Ruta para ambas GPOS: Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Update
	- Manage updates offered from Windows Server Update Service_ > Specify intranet Microsoft update service location -> `http://fs1.november.lab:8530` en los dos primeros campos
	- Manage updates offered from Windows Server Update Service > Enable client-side targeting (Piloto o produccion)
	- Manage end user experience > Configure Automatic Updates 4 > Auto download and schedule the install, Every day, 03:00
	- Manage end user experience > Turn off auto-restart for updates during active hours -> 8-18

> La cadena es esta: el security filtering aplica la GPO al equipo → la GPO escribe `TargetGroup = Piloto` en el Registro → el cliente le dice a WSUS "soy de Piloto" → WSUS lo coloca en ese grupo. **WSUS no lee AD ni aplica GPO. Solo aplica aprobaciones.**

- **Gestionar aprobaciones automáticas para piloto**: Options > Automatic Approvals > New Rule -> When an update is in a specific classification -> _Set a deadline_ desmarcado -> _any classification_ → Critical Updates y Security Updates -> _all computers_ → Piloto -> nombre, por ejemplo `Auto - Piloto - Criticas y Seguridad`.

![](https://imgur.com/UxDi88B.png)

> Para **Produccion**, a mano, cuando Piloto esté validado: Updates > All Updates > _Approval: Approved_, _Status: Any_ > Refresh > seleccionar los parches > clic derecho > **Approve** > Produccion > **Approved for Install**. 

> Revisar **Supersedence** en los detalles del parche: _Updates superseding this update: None_ → es el más reciente y se aprueba. Desde la 24H2, Windows 11 usa **acumulativas de referencia (checkpoint)**. Si un parche no aparece como reemplazado, no se rechaza: puede ser la base que necesitan los siguientes.

- **Probar en cliente**: Actualizar GPOs (`gpupdate /force`) -> verificar que se aplica la GPO del anillo (`gpresult /r`) -> Verificar que existe la clave `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate` con los valores `WUServer` y `TargetGroup` -> Settings -> Windows Update -> Check for updates -> En la consola de WSUS, el equipo debe aparecer en su grupo

> Para Produccion, aprobar a mano pasados unos días. En Updates > Security Updates, filtrar por _Approval: Unapproved_ y _Status: Needed_, y luego clic derecho > Approve > Produccion > _Approved for Install_.

![](https://imgur.com/rwMqCja.png)

> Realizar mantenimiento mensual: **Options > Server Cleanup Wizard**. Rechaza las actualizaciones reemplazadas (superseded) y borra archivos sin usar.

# bitlocker
Para este caso, vamos a activar Bitlocker en clientes y activar una GPO vinculada a la OU de Equipos que almacene estas claves de recuperación en el DC.

- Comprobar estado de bitlocker en cliente con `tpm.msc` (o `tpmtool getdeviceinformation`).

![](https://imgur.com/bgDQ0ND.png)

- En DC1 instalar el rol: **Server Manager → Manage → Add Roles and Features** -> **Features** → **Remote Server Administration Tools → Feature Administration Tools → BitLocker Drive Encryption Administration Utilities** → marcar **BitLocker Recovery Password Viewer**

> En un cliente RSAT: **Settings → System → Optional features → Add** → _RSAT: BitLocker Drive Encryption Administration Utilities_.

![](https://imgur.com/ZARvRrv.png)

- En GPMC, crear una GPO en OU Equipos -> **Computer Configuration → Policies → Administrative Templates → Windows Components → BitLocker Drive Encryption → Operating System Drives** -> **Choose how BitLocker-protected operating system drives can be recovered** → _Enabled_: 
	- _Save BitLocker recovery information to AD DS for operating system drives_
	- _Store recovery passwords and key packages_
	- _Do not enable BitLocker until recovery information is stored to AD DS for operating system drives_

> Si algún cliente tiene D:, configurar también el mismo ajuste en **Fixed Data Drives** (_Choose how BitLocker-protected fixed drives can be recovered_)

![](https://imgur.com/QMyVJ2V.png)

- Comprobar en cliente: `gpupdate /force` y comprobar con `gpresult /r /scope computer`.
- Luego activar BitLocker: **Control Panel → BitLocker Drive Encryption → Turn on BitLocker** -> Guardar la clave -> New encryption mode -> Run Bitlocker system check -> Start Encrypting -> Reiniciar si lo pide.

> El orden importa: si se cifra antes de que la GPO se aplique, la clave no sube a AD.

- Verificar en cliente: `manage-bde -status C:` y luego `manage-bde -protectors -get C:`. Aparecerá **TPM** y **Numerical Password**. 

![](https://imgur.com/YhNjjtZ.png)

- En ADUC, abrir las propiedades del **equipo** → pestaña **BitLocker Recovery**. Comparar los IDs.

> Nota: si un equipo ya estaba cifrado antes de la GPO, la clave no sube sola. Hay que subirla con `manage-bde -protectors -adbackup C: -id {ID}`

![](https://imgur.com/M2UM1Fy.png)

# gpos avanzadas

## bloquear SMB saliente
Para este caso, la idea es impedir que los equipos de la empresa se conecten a servidores SMB que estén en Internet (puerto 445 TCP saliente).

> Cuando Windows abre una ruta `\\servidor\recurso`, **intenta autenticarse automáticamente** con las credenciales del usuario (NTLM), lo que puede aprovechar un atacante para descifrarlas.

1. **Crear la GPO.** En GPMC, clic derecho sobre la OU de los equipos cliente › _Create a GPO in this domain, and Link it here_ › nombre: `EQ_FW_SMB_SALIENTE`.
2. **Editar la GPO** _Edit_ › _Computer Configuration › Policies › Windows Settings › Security Settings › Windows Defender Firewall with Advanced Security › Windows Defender Firewall with Advanced Security › Outbound Rules_ › _New Rule_.
3. **Rule Type:** _Port_.
4. **Protocol and Ports:** _TCP_, _Specific remote ports_: `445`. En una regla saliente, el puerto que importa es el del destino.
5. **Action:** _Block the connection_.
6. **Profile:** marcar _Domain_, _Private_ y _Public_. La protección tiene que seguir activa cuando el portátil sale de la oficina.
7. **Name:** `Bloquear SMB saliente a Internet`.
8. **Limitar el destino a Internet.** En las propiedades de la regla › _Scope_ › _Remote IP address_ › _These IP addresses_ › _Add_ › _This IP address range_, añadir estos cuatro rangos:
    - `0.0.0.0` – `9.255.255.255`
    - `11.0.0.0` – `172.15.255.255`
    - `172.32.0.0` – `192.167.255.255`
    - `192.169.0.0` – `255.255.255.255`

> Es todo el espacio de direcciones excepto los rangos privados (10.x, 172.16–31.x y 192.168.x). 
    
9. **Aplicar GPO y comprobar en pc01:**
```powershell
gpupdate /force
Get-NetFirewallRule -PolicyStore ActiveStore -DisplayName "Bloquear SMB saliente a Internet" |
  Format-Table DisplayName, Enabled, Direction, Action, Profile
```

11. **Comprobar que el SMB interno sigue funcionando:** abrir `\\dc1\sysvol` desde pc01.

## rule merging
El firewall de cada equipo aplica dos listas de reglas (*ActiveStore*): las **locales** (*PersistentStore*) y las de **GPO** (*RSOP*). El ajuste _Apply local firewall rules_ decide si se aplican las dos o solo las de GPO. 

> Recordar: Bloqueo gana a permiso, y si no hay regla, se aplica la acción por defecto: entrante bloqueado y saliente permitido.

- En GPMC, editar la GPO › _Computer Configuration › Policies › Windows Settings › Security Settings › Windows Defender Firewall with Advanced Security_.
- Clic derecho en _Windows Defender Firewall with Advanced Security_ › _Properties_.
- En la pestaña del perfil (_Domain Profile_, _Private Profile_ o _Public Profile_): _Settings › Customize_ › _Rule merging_ › _Apply local firewall rules_: **No**.
- En la misma pestaña: State -> Firewall State: On -> Inbound connections: Block -> Outbound connections: Allow.

> Hacer esto para los tres perfiles

![](https://imgur.com/ppvePeI.png)

- Aplicar la directiva con `gpupdate /force`.
- Comprobar: Crear una regla local (por ejemplo, que permita ping desde cualquier origen) y verifica que existe pero no se aplica.
## firewall log
Es un documento que registra paquetes descartados. Se almacena generalmente en `C:\System32\LogFiles\Firewall\pfirewall.log`

> Se configura por perfil. Si solo se activa en Dominio y el equipo cae en Público, no registra nada. Además, el log registre **qué** se bloque, pero no la regla.

Para este caso emplearemos la regla [SMB saliente](servicios-infraestructura-windows-server#bloquear-smb-saliente) implementada anteriormente.

- Editar la GPO -> _Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security_.
2. Clic derecho en el nodo **Windows Defender Firewall with Advanced Security** > **Properties**.
3. Pestaña **Domain Profile** > sección **Logging** > **Customize**.
4. Configurar:
    - **Name**: dejar la ruta por defecto, `%systemroot%\system32\LogFiles\Firewall\pfirewall.log`. Si se cambia, el servicio del firewall tiene que tener permiso de escritura en la nueva carpeta, y si no lo tiene no se crea el log.
    - **Size limit (KB)**: `16384`.
    - **Log dropped packets**: **Yes**.
    - **Log successful connections**: **No**. (Con Yes el log se llena enseguida y cuesta encontrar nada)

![700](https://imgur.com/a0LNwD1.png)

5. Repetir el paso 4 en las pestañas **Private Profile** y **Public Profile** > **OK**.
6. En un cliente, ejecutar `gpupdate /force` y comprobar el resultado con `Get-NetFirewallProfile -PolicyStore ActiveStore | Select Name, LogBlocked, LogFileName, LogMaxSizeKilobytes`. 

![](https://imgur.com/HLE254t.png)

7. Provocar el bloqueo del caso 2 desde el cliente: `Test-NetConnection` _IP_externa_ `-Port 445`.

![](https://imgur.com/U4GTzYf.png)

8. Buscarlo en el log desde una consola elevada: `Select-String -Path C:\Windows\System32\LogFiles\Firewall\pfirewall.log -Pattern " 445 "`. Tiene que salir una línea **DROP TCP** con la IP del cliente como origen, el 445 como puerto de destino y **SEND** como dirección.

![](https://imgur.com/twisJNQ.png)

## user rights assignment
Definen qué puede hacer una cuenta en un equipo: iniciar sesión en local, por RDP, apagarlo, etc. Cada derecho es una lista de usuarios y grupos, locales o de dominio, que se guarda en el propio equipo. 

> Si se define por GPO, la lista **sustituye** a la local, no se suma a ella. **Deny** prevalece sobre **Allow**.

En este caso, limitaremos el acceso por RDP a los administradores del dominio solamente.

1. En un cliente, añadir un usuario normal del dominio al grupo local **Remote Desktop Users** y comprobar que puede entrar por RDP. Así la prueba final demuestra que el derecho manda sobre la pertenencia al grupo.
2. En GPMC, editar la GPO de administración remota vinculada a la OU de los clientes.
3. Ir a _Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment_.
4. Abrir **Allow log on through Remote Desktop Services** > marcar **Define these policy settings** > **Add User or Group**:
    - `Administrators`
5. **OK** y cerrar el editor.

![](https://imgur.com/NexMNCQ.png)

6. En el cliente: `gpupdate /force`.
7. Comprobar en el cliente con `secpol.msc` > _User Rights Assignment_: el derecho aparece en gris con solo esos dos grupos.
8. Probar por RDP:
    - Con el usuario normal del paso 1 → rechazado (_the user account is not authorized for remote login_).
    - Con un miembro de Administrators → entra.

![](https://imgur.com/y9AL6wF.png)


## administradores locales
Es una buena práctica meter en el grupo de administradores locales a los técnicos administradores del dominio. De esta manera los técnicos son administradores en todos los PCs sin tener que tocarlos uno a uno ni conocer la contraseña de LAPS. 

En este caso, crearemos un grupo G_Helpdesk, y lo añadiremos por AGDLP al grupo de administradores locales con GPP.

1. En la OU de grupos: **New > Group** > `G_Helpdesk_Admins`, ámbito **Global**, tipo **Security** -> añadimos miembros (adm-nix)
2. **New > Group** > `DL_PC_Admins`, ámbito **Domain local**, tipo **Security**. _Members_: `G_Helpdesk_Admins`.

![](https://imgur.com/UVow1C3.png)

3. En GPMC, clic derecho en la OU de los equipos > **Create a GPO in this domain, and Link it here** > `EQ_Admins_Locales`.
4. Editar > _Computer Configuration > Preferences > Control Panel Settings > Local Users and Groups_ > **New > Local Group**.
5. Configurar:
    - **Action**: Update.
    - **Group name**: `Administrators (built-in)`, elegir en el desplegable.
    - **Delete all member users** y **Delete all member groups**: sin marcar. Si los marcas, se vacía el grupo y se pierden Domain Admins y la cuenta de LAPS.
    - **Add** > `NOVEMBER\DL_PC_Admins` > acción **Add to this group** > OK.

![](https://imgur.com/YUnSdS6.png)

6. En PC-01: `gpupdate /force` y luego `net localgroup Administrators`. Debe aparecer `NOVEMBER\DL_PC_Admins`.
7. Pruebas:
    - RDP a PC-01 con `adm-nix` → entra, porque tiene el derecho por ser de Administrators. Dentro, `whoami /groups` 
    - Debe mostrar `BUILTIN\Administrators`.


![](https://imgur.com/17qPKgz.png)

# security options
Se trata de los ajustes de seguridad de cada equipo: cómo se comporta la pantalla de inicio de sesión, qué protocolos antiguos se permiten, qué pasa si dejas el equipo desatendido.

> Están en la misma GPO que los derechos de usuario (_Local Policies > Security Options_). 

Para este caso, configuraremos el bloqueo por inactividad, un mensaje de bienvenida antes de iniciar sesión, que la pantalla no muestre quién fue el último que inició sesión, y desactivaremos SMBv1.

1. En GPMC, clic derecho en la OU **Equipos** > **Create a GPO in this domain, and Link it here** > `EQ_Seguridad_Base`.
2. Editar > _Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > **Security Options**_.
	- **Interactive logon: Machine inactivity limit** > `900` segundos (15 minutos). Para la prueba puedes poner `60` y subirlo después.
	- **Interactive logon: Message title for users attempting to log on** > por ejemplo, `Aviso legal`.
	- **Interactive logon: Message text for users attempting to log on** > por ejemplo, `Este equipo es propiedad de November. Su uso está restringido a personal autorizado y puede ser monitorizado.`
	- **Interactive logon: Don't display last signed-in** > **Enabled**.

![](https://imgur.com/kOgnqZB.png)

- Comprobar en cliente con `gpupdate /force`

![](https://imgur.com/CniXG4b.png)





