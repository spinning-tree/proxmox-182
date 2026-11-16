# Credenciales incorrectas al promocionar un DC adicional

| Campo              | Valor                               |
| ------------------ | ----------------------------------- |
| Ticket             | TS-008                              |
| Tipo               | Incidencia                          |
| Categoría          | Active Directory / Autenticación    |
| Urgencia / Impacto | Media / Media                       |
| Prioridad          | Media                               |
| Estado             | Resuelto                            |
| Abierto / Resuelto | 16/11/2026 22:28 / 16/11/2026 23:34 |
## Resumen

Al promocionar DC2 como controlador de dominio adicional de _november.lab_, el asistente de AD DS rechazaba las credenciales de _NOVEMBER\Administrator_ con el mensaje de usuario o contraseña incorrectos, aunque la contraseña era correcta.

**Causa raíz:** DC2 aún no estaba en el dominio. Con el formato NetBIOS (_NOVEMBER\usuario_) no podía localizar el KDC, así que no usaba Kerberos y recurría a NTLM sin avisar. Windows protege las rutas SYSVOL y NETLOGON (_Hardened UNC Paths_) exigiendo autenticación mutua, y NTLM no la ofrece. Resultado: acceso denegado, mostrado por el asistente como "credenciales incorrectas".

**Solución:** usar el formato UPN, _administrator@november.lab_.

## Entorno

|Elemento|Detalle|
|---|---|
|Hipervisor|Proxmox VE|
|Dominio|november.lab (NetBIOS: NOVEMBER)|
|DC1|10.0.0.10, AD DS + DNS + DHCP, todos los roles FSMO|
|DC2|10.0.0.11, clonado de una plantilla de Windows Server generalizada con sysprep, en grupo de trabajo, DNS apuntando a DC1|
|Operación|Add a domain controller to an existing domain|
## Síntoma

- El asistente de promoción rechaza las credenciales _NOVEMBER\Administrator_ con el aviso de que son incorrectas.
- La contraseña se comprobó escribiéndola en el Bloc de notas de ambos DC, e incluso se cambió por la misma. El fallo persistía.
- Desde DC2, el acceso a \dc1\sysvol también fallaba.
## Diagnóstico

El método fue descartar capas de abajo arriba: red, DNS, hora, identidad del equipo, credenciales, autenticación y autorización.

### Red y DNS: descartados

```
PS C:\> nslookup november.lab
Server:  dc1.november.lab
Address:  10.0.0.10

Name:    november.lab
Address:  10.0.0.10
```

DC2 resuelve el dominio a través de DC1 y tiene conectividad con él y con Internet.

### Hora: descartada

Al principio **Get-Date** mostraba 9 horas de diferencia entre DC1 y DC2:

```
DC2: Monday, November 16, 2026 1:31:50 PM
DC1: Monday, November 16, 2026 10:32:05 PM
```

Nueve horas exactas es la diferencia entre Pacific Time, la zona por defecto de una instalación en inglés, y Madrid. La zona horaria se había corregido a mano, pero esa sesión de PowerShell estaba abierta desde antes del cambio y seguía mostrando la hora vieja. Kerberos compara la hora en UTC, así que una zona horaria distinta no provoca desfase.

Medición real del desfase:

```
PS C:\> w32tm /stripchart /computer:dc1.november.lab /samples:3 /dataonly
23:01:53, +00.5176724s
23:01:55, +00.5173108s
23:01:57, +00.5171763s
```

> Medio segundo, muy por debajo del límite de 5 minutos de Kerberos.

### SID duplicado: descartado

Un clon sin sysprep tendría el mismo SID de máquina que DC1, y por tanto el mismo SID que el dominio, lo que rompe la autenticación.

```
DC1 - (Get-ADDomain).DomainSID : S-1-5-21-2406216165-4216629343-421595617
DC2 - SID local (whoami /user) : S-1-5-21-1952344775-1730565551-2401558512-500
```

Son distintos: el sysprep de la plantilla funcionó.

### Puertos: descartados

**Test-NetConnection** desde DC2 hacia DC1: los puertos 88 (Kerberos), 389 (LDAP) y 445 (SMB) responden con **TcpTestSucceeded: True**.

> _dc1_ se resolvía a una IPv6 de enlace local (fe80::). Como DC2 estaba en un grupo de trabajo, no tenía sufijo DNS; el nombre corto no lo resolvía el DNS sino **LLMNR**, y Windows prefiere IPv6. No era la causa, pero era una pista de que los nombres cortos no pasaban por el DNS.

### Credenciales: válidas

La directiva de auditoría de DC1 solo registraba los **éxitos** en _Credential Validation_ y _Kerberos Authentication Service_, así que los fallos no dejaban rastro. Se activó la auditoría de fallos:

```
auditpol /set /subcategory:"Credential Validation" /failure:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /failure:enable
```

Tras un nuevo intento, DC1 registró:

```
Event 4776 - The computer attempted to validate the credentials for an account.
Logon Account:       administrator
Source Workstation:  WIN-HSNJPK1MKEI
Error Code:          0x0
```

**Error Code 0x0 = validación correcta.** La contraseña estaba bien y DC1 la aceptaba.

### Autenticación: correcta, pero con NTLM

```
PS C:\> net use \\dc1.november.lab\sysvol /user:november\Administrator
Enter the password for 'november\Administrator' to connect to 'dc1.november.lab':
System error 5 has occurred.

Access is denied.
```

No dice "contraseña incorrecta", dice **acceso denegado**. En DC1 apareció el inicio de sesión:

```
Event 4624 - An account was successfully logged on.
Logon Type:              3
Account Name:            Administrator
Account Domain:          NOVEMBER
Workstation Name:        WIN-HSNJPK1MKEI
Source Network Address:  10.0.0.11
Authentication Package:  NTLM
Package Name:            NTLM V2
```

La sesión se crea, pero **con NTLM, no con Kerberos**. Además, **klist** en DC2 mostraba 0 tickets.

### Autorización: el fallo solo afecta a SYSVOL y NETLOGON

|Recurso|Resultado|
|---|---|
|\dc1.november.lab\c$|Acceso correcto|
|\dc1.november.lab\netlogon|Acceso denegado|
|\dc1.november.lab\sysvol|Acceso denegado|

Las GPO del dominio quedaron descartadas: estaban vinculadas a la OU _Equipos_, no a _Domain Controllers_, y DC2 aún no estaba en el dominio.
## Causa raíz

- **Hardened UNC Paths.** Desde Windows 10 y Server 2016, el cliente exige por defecto **autenticación mutua e integridad** para \*\SYSVOL y \*\NETLOGON. Con NTLM el servidor no demuestra su identidad, así que el cliente rechaza la conexión aunque la sesión sea válida. Por eso C$ funcionaba y SYSVOL no.
- **¿Por qué NTLM?** El reino de Kerberos es el nombre DNS del dominio (NOVEMBER.LAB), y el KDC se localiza por DNS con el registro SRV **_kerberos._tcp.november.lab**. Un equipo unido al dominio sabe que NOVEMBER equivale a november.lab. Un equipo en grupo de trabajo no. Con NOVEMBER\Administrator, DC2 no encontraba el KDC y recurría a NTLM.
- **El asistente de AD DS** necesita acceder a SYSVOL y chocaba con lo mismo, pero lo mostraba como "credenciales incorrectas".

## Solución

Usar el formato **UPN**:

```
net use \\dc1.november.lab\sysvol /user:administrator@november.lab *
```

El acceso a SYSVOL funcionó y **klist** ya mostraba tickets de Kerberos. En el asistente se pulsó **Change...**, se usaron las credenciales _administrator@november.lab_ y la promoción terminó sin errores.

## Verificación posterior

```
PS C:\> repadmin /replsummary
Source DSA          largest delta    fails/total %%   error
 DC1                       09m:11s    0 /   5    0
 DC2                          :12s    0 /   5    0

PS C:\> w32tm /query /source
dc1.november.lab
```

Replicación de las 5 particiones sin errores, DC2 marcado como GC (IS_GC) y tomando la hora de DC1.

## Lecciones aprendidas

- **Desde un equipo que no está en el dominio, usar UPN** (_usuario@dominio.fqdn_) **o FQDN**, nunca el nombre NetBIOS.
- **"Credenciales incorrectas" no siempre significa contraseña incorrecta.** Distinguir el mensaje real: _password is invalid_ frente a _access denied_ (error 5).
- **Mirar el problema desde el servidor.** Los eventos 4776, 4624 y 4625 de DC1 dijeron más que cualquier mensaje del cliente.
- **Comprobar qué se audita antes de concluir que no hay eventos.** Por defecto, varios fallos de autenticación no se registran.
- **Una zona horaria distinta no es un desfase de hora.** Medir con **w32tm /stripchart** y no fiarse de una sesión de PowerShell abierta antes del cambio.
- **Usar siempre el FQDN en un dominio.** Un nombre corto puede resolverse por LLMNR y llevar a NTLM.