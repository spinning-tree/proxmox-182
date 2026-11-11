# servicios de infraestructura de windows server

> Para este módulo consultar las secciones 
> - [configuracion basica de windows server](configuracion-basica-windows-server) 
> - [configuracion avanzada de windows server](configuracion-avanzada-windows-server)

# glosario básico

- **rsop.msc**: es la versión gráfica y antigua de gpresult. Abre una consola con el aspecto del editor de GPO, pero solo de lectura, y muestra la configuración final que tiene aplicada ese equipo y ese usuario, junto con la GPO que ganó cada ajuste.
- **eventvwr.msc**: Es el libro de registro del sistema. Cada componente de Windows (el kernel, los servicios, AD, DNS, las GPO) apunta lo que hace en su propio libro, y el visor te deja leerlos, filtrarlos y buscar en ellos

# herramientas de diagnóstico

## event Viewer

Para este ejemplo, simularemos dos escenarios: el usuario inserta mal la contraseña, y finalmente, el usuario bloquea su cuenta tras superar el límite de intentos configurado en la DDP.

Antes de proceder, tener en cuenta lo siguiente:
- **Firewall de los clientes por GPO**: habilitar el grupo **Remote Event Log Management** (NP-In, RPC, RPC-EPMAP), con Scope limitado a DC1 y a los PCs de administración.
- **Cuenta**: Al conectarse por otra pc, la MMC le exige elevación, así que se lanza con `Start-Process powershell -Verb RunAs` y desde ahí `eventvwr.msc`.
- En **Connect as another user**, poner el nombre completo del equipo (**Check Names** → PC-02.november.lab).
- Para actualizar las GPOs en clientes -> click derecho en la OU Equipos -> Group Policy Updates. Esto puede dar fallo si la GPO del Firewall no tiene habilitado **Remote Scheduled Tasks Management (RPC, RPC-EPMAP)** y **WMI (WMI-In)**.

![](https://imgur.com/8rE1MXM.png)

1. Simulamos el intento fallido, tres veces y consultamos desde el eventvwr.msc para la máquina en concreto
	- El 4625 (Inicio de sesión fallido) se registra en la máquina donde se intentó iniciar la sesión, no en el DC.
	- Sub Status **0xC000006A**: el usuario existe y la contraseña es incorrecta.

![](https://imgur.com/JUhZHeG.png)

2. Comprobamos el evento en DC-1. 
Solo registraba los eventos con éxito del 4776. Se activaron las auditorías para las categorías necesarias. Ver [activar auditoría en DDCP](#activar-auditoria-en-la-ddcp)

- Activadas las auditoría, el DC registra correctamente los intentos fallidos con código `0xC000006A`

![](https://imgur.com/TIHog60.png)

2. Seguidamente, provocamos el bloqueo de cuenta por exceder el número de intentos.

- Se crea la custom view cuentas_bloqueadas y se filtra por seguridad y 4740 como event id.

![](https://imgur.com/U0LaStY.png)

> Conclusión: Para seguir el rastro: **4740** (cuenta bloqueada) → **4776** en el DC (_Source Workstation_) → **4625** en el equipo destino (_Source Network Address_ = .100).
## activar auditoria en la DDCP
1. Abrir GPMC y edita **Default Domain Controllers Policy**.
2. Ir a **Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies**.
3. Configurar las subcategorías de la tabla.
4. En el DC, ejecuta `gpupdate /force` y comprueba con `auditpol /get /category:*`.

![](https://imgur.com/6VjHmnf.png)

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
## principales event ids

### security

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

### system

|Código|Significado|
|---|---|
|41 (Kernel-Power)|Reinicio sin apagado limpio|
|6008|Apagado inesperado anterior|
|1074|Apagado o reinicio iniciado por un usuario o proceso|
|1030 / 1058 / 1129|Fallos de GPO (no obtiene la directiva / no lee gpt.ini / sin red hacia el DC)|
|5719 (Netlogon)|No puede establecer el canal seguro con un DC|
|3210 (Netlogon)|No puede autenticarse con el DC (relación de confianza rota)|

