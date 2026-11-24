# Regla local ssh de firewall habilitada que no se aplica (rule merging)

## Resumen

|Campo|Valor|
|---|---|
|Ticket|TS-010|
|Tipo|Incidencia|
|Categoría|Red > Firewall de Windows|
|Urgencia / Impacto|Baja / Bajo (un equipo)|
|Prioridad|Baja|
|Estado|Resuelto|
|Abierto / Resuelto|23/11/2026 / 23/11/2026|

Un usuario con permisos de administrador local instala el servidor OpenSSH en su equipo. El instalador crea una regla local de entrada para el TCP 22, pero no es posible conectarse desde otro equipo. Hay dos causas encadenadas: la regla se crea por defecto para el perfil Private solamente, y la GPO del dominio impide aplicar reglas locales (_rule merging_ desactivado). La solución es crear la regla en una GPO, limitada al equipo y al origen autorizados.

## Entorno

| Elemento             | Detalle                                                                                                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Dominio              | november.lab                                                                                                                                                                   |
| DC                   | DC1 (10.0.0.10, Windows Server 2025)                                                                                                                                           |
| Cliente afectado     | pc01 (10.0.0.105, Windows 11 para ARM, VM de Parallels)                                                                                                                        |
| Equipo de origen     | pc02 (10.0.0.106, reserva DHCP creada durante la resolución)                                                                                                                   |
| GPO de firewall base | EQ_FW_RULE_MERGING: _State_ configurado en los tres perfiles (firewall activado, entrada bloqueada y salida permitida) y _Apply local firewall rules: No_ en los tres perfiles |
| Cuentas usadas       | Administrador (elevación en pc01), usuario de dominio en pc02                                                                                                                  |
| Requisito previo     | El usuario tiene permisos de administrador local en pc01                                                                                                                       |

## Síntoma

"Soy desarrollador, tengo permisos de administrador en mi equipo. He activado el servidor SSH para conectarme desde otro equipo y no funciona."

## Contexto previo

- El usuario instala la característica OpenSSH Server en pc01 desde una consola elevada.
- El instalador crea la regla local de entrada `OpenSSH SSH Server (sshd)` (TCP 22, _Allow_), pero solo para el perfil **Private**.
- pc01 está en el perfil **Domain**.

## Diagnóstico paso a paso

1. **Primer intento de solución (técnico).** Como pc01 está en Domain, se añade el perfil Domain a la regla local. La regla queda con `Profile: Domain, Private`. Sigue sin funcionar.

![](https://imgur.com/mZfgRXt.png)

2. **Comprobar la conectividad desde pc02:**

![](https://imgur.com/BlLPs9z.png)

> Resultado: el ping falla, pero pc01 (10.0.0.105) aparece en la tabla ARP con su MAC (`00-1c-42-79-1e-f9`). El equipo está encendido y es alcanzable en capa 2, así que el tráfico se filtra en pc01. 

3. Comprobar conectividad en puerto 22

![](https://imgur.com/M7XShoO.png)


> `Test-NetConnection` devuelve `TcpTestSucceeded: False`. (El ping falla porque la regla ICMP de la GPO solo admite el origen 10.0.0.10.)

4. **Descartar que el servicio no esté escuchando.** En pc01:

![](https://imgur.com/QyB8BOX.png)

> Resultado: `sshd` en estado _Running_ y escuchando en el 22 en IPv4 (`0.0.0.0`) y en IPv6 (`::`). El servicio funciona.
  
5. **Comparar la regla en el almacén local y en la directiva efectiva:**

![](https://imgur.com/fc2lNOp.png)



| Campo                 | PersistentStore | ActiveStore                       |
| --------------------- | --------------- | --------------------------------- |
| Enabled               | True            | True                              |
| Profile               | Domain, Private | Domain, Private                   |
| PrimaryStatus         | OK              | **Inactive**                      |
| EnforcementStatus     | NotApplicable   | **LocalFirewallRulesNotAllowed**  |
| PolicyStoreSourceType | Local           | Local                             |

> La regla existe, está habilitada, tiene el perfil correcto y la acción correcta, pero no se aplica.

6. **Comprobar si los perfiles admiten reglas locales:**

![](https://imgur.com/iMVx3xS.png)


> Resultado: `AllowLocalFirewallRules: False` en Domain, Private y Public. La GPO tiene el _rule merging_ desactivado en los tres perfiles.
## Causa raíz

Dos causas encadenadas:

1. **La regla del instalador se crea solo para el perfil Private.** pc01 está en Domain, así que la regla nunca habría coincidido con el perfil activo, ni siquiera con el _rule merging_ activado.
2. **La GPO desactiva el _rule merging_** (_Apply local firewall rules: No_). Las reglas locales siguen guardadas en el equipo (PersistentStore), pero no se aplican. Por eso, al corregir el perfil de la regla local, la conexión sigue fallando.

> Sin ninguna regla de permiso aplicada, el tráfico entrante al TCP 22 cae en la acción por defecto del perfil: bloquear.

## Solución

Crear una GPO limitada al equipo de destino y al origen autorizado.

1. **Reserva DHCP para el equipo de origen** (pc02, 10.0.0.106), para que la IP usada en la regla no cambie.

![](https://imgur.com/np9zEt6.png)

2. **Grupo de equipos** `G_EQ_SSH_ENTRANTE` en la OU de grupos, con pc01 como miembro.

![](https://imgur.com/O027epc.png)


3. **Nueva GPO** `EQ_PUERTO_22_PC01`, vinculada a la OU de equipos:
    - _Security Filtering:_ `G_EQ_SSH_ENTRANTE` (se quita _Authenticated Users_).
    - _Delegation:_ _Authenticated Users_ con permiso de lectura.

![](https://imgur.com/gtnDEg4.png)

4. **Regla de entrada en la GPO** (_Computer Configuration › Policies › Windows Settings › Security Settings › Windows Defender Firewall with Advanced Security › Inbound Rules_):
    - _Rule Type:_ Port, TCP 22.
    - _Action:_ Allow the connection.
    - _Profile:_ Domain.
    - _Scope › Remote IP address:_ 10.0.0.106.
    - _Name:_ `SSH entrante de pc02 hacia pc01`.

![](https://imgur.com/40FjQx8.png)


![](https://imgur.com/7VehTuF.png)

5. **Actualizar la directiva y la pertenencia al grupo en pc01:** `gpupdate /force` y reinicio del equipo. El reinicio renueva el ticket Kerberos de la cuenta del equipo, que pasa a incluir el nuevo grupo. Sin reiniciar, se consigue lo mismo con `klist purge -li 0x3e7` antes del `gpupdate /force`.

## Verificación

- En pc01, la regla de la GPO aparece en la directiva efectiva y se aplica:

![](https://imgur.com/OmCk9lD.png)

- Desde pc02:

![](https://imgur.com/6OQy954.png)

> `TcpTestSucceeded: True`, con origen 10.0.0.106.

- Inicio de sesión por SSH desde pc02 a pc01 correcto.

![](https://imgur.com/S4ETRUw.png)
## Prevención y lecciones aprendidas

- **`Enabled: True` no significa que la regla funcione.** Lo que importa es `PrimaryStatus` y `EnforcementStatus` en el ActiveStore. `EnforcementStatus` indica la causa exacta (`LocalFirewallRulesNotAllowed`, `ProfileInactive`).
- **PersistentStore y ActiveStore no son lo mismo.** El primero contiene solo las reglas locales guardadas en el equipo. El segundo es la directiva efectiva: GPO, reglas locales y temporales.
- **El _rule merging_ controla las reglas, no la configuración del perfil.** Sin el _State_ configurado en la GPO, un administrador local podría poner la entrada en _Allow_ o apagar el firewall. Hay que fijar las dos cosas en los tres perfiles.
- **Configurar el _rule merging_ en los tres perfiles.** Si se deja solo en Domain, una regla local volvería a funcionar en cuanto el equipo cayera en Private o Public.
- **Al desactivar el _rule merging_ también se ignoran las reglas predefinidas de Windows** (_Core Networking_ y otras). Las necesarias deben recrearse en la GPO.
- **Los instaladores pueden crear reglas limitadas a ciertos perfiles.** Hay que revisar siempre el perfil de la regla frente al perfil activo del equipo (`Get-NetConnectionProfile`).
- **Antes de culpar al firewall, comprobar que el servicio escucha.** Usar `Get-Service` y `Get-NetTCPConnection -State Listen`.
- **ARP para saber si un equipo está vivo cuando no responde al ping.** Si aparece en `arp -a` con su MAC, está en la red y el filtrado es local.
- **Una regla con origen por IP necesita una IP fija** para ese origen (reserva DHCP).