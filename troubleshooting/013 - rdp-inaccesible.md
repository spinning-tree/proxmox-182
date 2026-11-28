# Escritorio Remoto (RDP) inaccesible por múltiples bloqueos de configuración

| Campo     | Valor                                |
| --------- | ------------------------------------ |
| Ticket    | TS-013                               |
| Tipo      | Incidente (simulado en lab)          |
| Categoría | Red / Firewall / Servicios / Windows |
| Prioridad | Media                                |
| Estado    | Resuelto                             |
| Fecha     | 2026-11-28                           |

## Resumen

Al intentar conectar por Escritorio Remoto (RDP) desde macOS (`192.168.1.100`) con Windows App hacia un equipo Windows 10 Enterprise LTSC (`192.168.1.141`), la conexión falla por *timeout* en el puerto 3389. La conectividad de red básica y el acceso remoto administrativo por SSH (puerto 22) funcionan con normalidad. El bloqueo responde a tres capas deshabilitadas de forma independiente en el destino: las reglas de entrada del Firewall de Windows, los servicios del sistema detenidos y el interruptor lógico `fDenyTSConnections` bloqueado en el Registro de Windows. Se resuelve habilitando las reglas TCP/UDP en el perfil Privado, levantando los servicios asociados y estableciendo `fDenyTSConnections` a `0`.

## Entorno

| Elemento      | Detalle                                                    |
| ------------- | ---------------------------------------------------------- |
| Red           | Subred local `192.168.1.0/24` (Perfil de red: Privado)     |
| Equipo remoto | Host Windows (`192.168.1.141`) — Windows Enterprise LTSC   |
| Cliente       | Mac mini (`192.168.1.100`) — macOS con cliente Windows App |
| Acceso aux.   | OpenSSH Server activo en puerto `22/TCP` (sesión elevada)  |

## Síntoma

| Prueba                                      | Resultado                                                                                   |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Conexión desde Windows App (macOS)          | Falla al iniciar sesión: error de tiempo de espera agotado al conectar con `192.168.1.141`  |
| `nc -vz 192.168.1.141 3389` (desde macOS)   | Falla: _`nc: connectx to 192.168.1.141 port 3389 (tcp) failed: Operation timed out`_        |
| `nc -vz 192.168.1.141 22` (desde macOS)     | Funciona: _`Connection to 192.168.1.141 port 22 [tcp/ssh] succeeded!`_                      |

## Contexto previo

El equipo remoto con Windows Enterprise LTSC dispone de soporte nativo para servidor RDP, pero viene con la característica de Escritorio Remoto completamente desactivada por defecto (servicios detenidos, firewall cerrado y clave del registro en estado de denegación). 

Antes de diagnosticar por consola, se intentan habilitar manualmente las reglas en el Firewall de Windows (`TCP-In` y `UDP-In` en perfil Privado) y arrancar los servicios desde `services.msc`, pero el cliente sigue sin poder conectar.

## Diagnóstico paso a paso

1. **Comprobación de conectividad en capa 4 y estado de escucha del puerto:**
   Desde el cliente macOS se lanza un escaneo de puertos con `netcat` para descartar un corte total de red y confirmar que el host responde en otros servicios:

![](https://imgur.com/z4HW5WN.png)

> El puerto `22` responde al instante, lo que confirma conectividad IP correcta y descarta problemas de enrutamiento o aislamiento en el switch/router.

2. **Verificación del perfil de red y sockets activos en el host Windows (vía SSH):** Al haber habilitado las reglas del firewall únicamente para el perfil **Privado**, se comprueba si Windows tiene catalogada la interfaz de red como Pública y si algún proceso está realmente escuchando en el `3389`

![](https://imgur.com/YMtb5Jv.png)

> `netstat` no devuelve ninguna línea en estado `LISTENING`. Aunque el firewall dejara pasar el tráfico, no hay ningún proceso atendiendo peticiones en el puerto `3389`.

3. **Inspección de los servicios de Escritorio Remoto:** Se verifica el estado de los servicios implicados en la pila RDP (`TermService`, `UmRdpService` y `SessionEnv`)

![](https://imgur.com/09rmasi.png)

> Inicialmente estaban detenidos. Tras iniciarlos manualmente, pasan a estado `Running`, pero `netstat -ano | findstr :3389` sigue sin mostrar el puerto en escucha.

4. **Comprobación del bloqueo lógico en el Registro de Windows:** Se consulta la clave que controla si el servicio `TermService` debe abrir o no el socket de red

![](https://imgur.com/ludZGQC.png)

> El valor `fDenyTSConnections` está en `1` (conexiones denegadas). Por diseño, `TermService` puede estar en ejecución (`Running`), pero no abrirá el puerto `3389` mientras esta clave valga `1`.

## Causa raíz

El acceso por Escritorio Remoto estaba bloqueado en tres niveles distintos que deben estar operativos de forma simultánea:

1. **Registro de Windows:** La propiedad `fDenyTSConnections` estaba establecida en `1`, impidiendo que el servicio de Escritorio Remoto abriera el socket en el puerto `3389`.
    
2. **Servicios del sistema:** Los servicios `TermService` (Servicios de Escritorio remoto) y `UmRdpService` (Redirector de puertos en modo usuario) estaban detenidos.
    
3. **Firewall de Windows:** Las reglas de entrada para el puerto `3389` (`Escritorio remoto - Modo de usuario (TCP-In)` y `(UDP-In)`) estaban deshabilitadas para el perfil de red Privado.

## Solución

Se aplican los cambios en las tres capas desde una consola elevada (PowerShell como Administrador vía SSH):

1. Habilitar RDP en el Registro y reiniciar el servicio para abrir el puerto

![](https://imgur.com/6vnfqgk.png)

2. Asegurar el inicio automático de los servicios dependientes:

![](https://imgur.com/5kZJiGU.png)

3. Habilitar las reglas de entrada TCP y UDP en el Firewall (perfil Privado):

![](https://imgur.com/bco523S.png)

> En la consola gráfica: habilitar únicamente `TCP-In` y `UDP-In` en el perfil Privado; descartar `TCP-WS-In`, `WSS-In` y `Shadow`

## Verificación

1. **Comprobación del puerto en escucha en Windows:** Al ejecutar `netstat -ano | findstr :3389`, el sistema ya muestra el proceso escuchando tanto en TCP como en UDP

![](https://imgur.com/34a1ivk.png)

2. Comprobación de conectividad desde macOS:

![](https://imgur.com/hvwjDg1.png)

3. **Validación de transporte UDP en Windows App:** Al iniciar sesión gráfica desde Windows App en macOS y pulsar sobre el icono de calidad de conexión en la barra superior, se confirma que el transporte rápido está activo (**UDP habilitado**).

![](https://imgur.com/fdTcGqD.png)

## Prevención y lecciones aprendidas

- **Activar RDP desde la interfaz gráfica (`sysdm.cpl`) ahorra pasos:** al marcar la casilla en la GUI, Windows modifica automáticamente `fDenyTSConnections = 0`, arranca los servicios y abre el grupo de reglas en el firewall. Si se configura por script o consola, hay que tocar las tres cosas.
- **Un servicio en estado `Running` no garantiza que haya un puerto abierto:** `TermService` lee el registro antes de enlazar el socket; comprobar siempre con `netstat -ano | findstr :3389` o `Get-NetTCPConnection -LocalPort 3389`.
- **El firewall no abre puertos, solo filtra tráfico:** crear o activar reglas en el firewall es inútil si no hay un proceso en estado `LISTENING` detrás.
- **Revisar siempre el perfil de red (`Get-NetConnectionProfile`):** si Windows detecta la red como `Public` en lugar de `Private`, bloqueará el puerto `3389` aunque las reglas del perfil privado estén activas.
- **Ediciones de Windows:** Windows Home no incluye servidor RDP de forma nativa; es necesario contar con ediciones Pro, Enterprise (incluyendo LTSC), Education o Server.