# configuración avanzada

> Previamente es necesario [configurar windows server](configuracion-basica-windows-server.md)

## glosario básico

- **AGDLP** es la convención de Microsoft para dar acceso a recursos: los usuarios (Accounts) se meten en grupos **globales** que representan quiénes son (G_Economía), esos grupos se meten en grupos **locales de dominio**, uno por recurso y nivel de acceso (DL_Economía_Modificar, DL_Economía_Lectura), y los **permisos** se dan solo a los DL. Así, tras configurar los permisos una vez, las altas, bajas y cambios se gestionan moviendo usuarios entre grupos, sin volver a tocar las ACLs.
- **SMB (permisos de recurso compartido)**: solo se aplican al acceder por la red (\servidor\carpeta). Hay tres niveles: Lectura, Cambiar y Control total. Se configuran en Propiedades → Compartir → Uso compartido avanzado → Permisos. La práctica habitual es dejarlos abiertos (Usuarios autenticados → Control total) y controlar el acceso en NTFS.
- **NTFS (permisos del sistema de archivos)**: se aplican siempre, tanto por red como en local. Son granulares (Lectura, Lectura y ejecución, Escritura, Modificar, Control total) y se heredan de la carpeta padre a sus subcarpetas y archivos. Se configuran en Propiedades → Seguridad, y se asignan a los grupos DL_.
- **Access-Based Enumeration (ABE)**: cada usuario solo ve las carpetas y archivos a los que tiene acceso. Se activa en el recurso compartido. ABE solo oculta al acceder por la red, porque es una función de SMB. 
- **GPP (Group Policy Preferences)**: sección de una GPO que configura ajustes del día a día, como unidades de red, impresoras, accesos directos o claves de registro. A diferencia de las directivas, no se imponen: el usuario puede cambiarlos. Permiten _item-level targeting_, es decir, aplicar cada elemento solo si se cumple una condición (grupo, OU, equipo, IP…).
- **Default Domain Policy (DDP):** vinculada al **dominio**. Contiene las directivas de cuenta (contraseña, bloqueo, Kerberos) de todas las cuentas del dominio.
- **FGPP** (Fine-Grained Password Policies) son políticas de contraseña y bloqueo que se aplican a usuarios o grupos concretos y se saltan la regla general de la **DDP**.
- **RSAT**: Es el conjunto de consolas y módulos de PowerShell para administrar **cualquier rol** de Windows Server desde una PC (AD DS, Group Policy Management, DNS y DHCP, Server Manager, etc.) 
- **Relación de confianza**: Al unir un equipo al dominio, se crea en AD una cuenta de equipo con su propia contraseña, que el PC y el DC renuevan solos cada 30 días. Esa contraseña compartida es la **relación de confianza**. 
- **KDC** (_Key Distribution Center_): es el servicio que emite los tickets Kerberos. Corre en todos los DC. Si no puedes llegar a un DC, no consigues tickets nuevos.
- **TGT** (_Ticket Granting Ticket_): es el primer ticket que recibes al iniciar sesión (va cifrado con la clave krbtgt, la pc no lo descifra), y solo sirve para pedir otros tickets. En `klist` aparece como `krbtgt/NOVEMBER.LAB`. Dura 10 h y se puede renovar hasta 7 días.
- **krbtgt**: es una cuenta interna de AD que usa el KDC para crifrar los TGT. No es de ningún usuario y nadie inicia sesión con ella, no vive en el PC. Si su contraseña se ve comprometida, todos los tickets del dominio dejan de ser fiables.
	1. **Inicio de sesión:** PC le pide un TGT al **KDC** (DC1). El KDC se lo da, cifrado con la clave de krbtgt (y otros datos: usuario, grupos "congelados"). Esta es la razón por la que, cuando añades usuario a un grupo, debes purgar el klist.
	2. **Acceso a un servicio:** PC le presenta ese TGT al **KDC** y le pide un ticket para un SPN concreto, por ejemplo `cifs/DC1`. El KDC comprueba que la firma del TGT es buena (con la clave de krbtgt) y emite el ticket de servicio, cifrado con la clave de la cuenta que ejecuta ese servicio
	3. **Uso:** tu PC le entrega ese ticket al servicio, el SMB de DC1. El servicio lo descifra con su propia clave y te deja pasar **sin hablar con el KDC**.
- **SPN** (_Service Principal Name_): es el nombre exacto de un servicio en el dominio, con el formato `servicio/servidor[/dominio]`. Kerberos emite los tickets para ese nombre exacto, así que si accedes con otro nombre (corto, FQDN o IP), el ticket cambia o no hay ticket.
- **Ticket de servicio**: es el ticket que pides con el TGT para acceder a un servicio concreto. Hay uno por cada SPN que usas, y nunca dura más que el TGT del que salió.
- **cifs**: es el nombre de servicio que usa SMB (carpetas compartidas) en los SPN; CIFS es el nombre antiguo de SMB. Lo pides al acceder a `\\servidor\recurso`, a SYSVOL o a una unidad mapeada.
- **ldap**: LDAP es el protocolo para consultar AD, porque AD es el directorio y LDAP es la forma de preguntarle. Lo usan las GPO, ADUC y el inicio de sesión para leer tus grupos y tus atributos.
- **NTLM**: es el método de autenticación antiguo que usa Windows cuando Kerberos no está disponible, por ejemplo al acceder por IP. No deja tickets en `klist`.
- **mDNS**: al igual que **LLMNR** son servicios de DNS por multicast (misma red local) a la `224.0.0.252`. NetBIOS hace lo mismo por broadcast. mDNS añade el sufijo `.local`. Es un riesgo de seguridad.
- **SRV**: es un tipo de registro DNS que indica **qué servidor ofrece un servicio concreto en un dominio, y por qué puerto**. Ejemplo: `_ldap._tcp.dc._msdcs.november.lab` es un SRV que anuncia un servicio en dc por tcp para ldap en el dominio november.lab.
## gpos

Las GPO's son paquetes de configuraciones que se definen en el DC y se almacenan en la carpeta SYSVOL que los clientes descargan. Se linkean a una OU (afecta a todos los equipos/usuarios de esa OU)

> Ojo: clientes aquí se refiere tanto a pcs como usuarios: la gpos de equipo se aplica al arancar el pc, la de usuario, cuando inicia sesión). También se aplican cada 90 minutos (aunque puede forzarse con `gpupdate /force`)

Otra cosa:

> En las GPOS, te encontrarás dos secciones:
> - **Directivas**: se imponen. El usuario ve la opción en gris y no puede cambiarla, y si quitas la GPO, el ajuste desaparece.
>- **Preferencias**: escriben el ajuste como si lo hubiera configurado el usuario, así que puede cambiarlo. Por defecto se vuelven a aplicar en cada actualización de directivas, salvo que marques “Aplicar una vez”. Si quitas la GPO, el ajuste se queda, salvo que marques “Quitar este elemento cuando ya no se aplique”.
### gpos de usuarios

1. Abrir **gpmc.msc** en DC -> OU del dominio -> click derecho en Usuarios -> _Create a GPO in this domain, and Link it here_ → nombrarla

> Para este ejemplo vamos a bloquear el acceso al panel de control

![](https://imgur.com/qKFUF40.png)

2. Clic derecho en la GPO → **Edit** → _User Configuration → Policies → Administrative Templates → Control Panel_ → **Prohibit access to Control Panel and PC settings** → _Enabled_.

![](https://imgur.com/yQHYqKX.png)


3. Seguidamente, en la pc del usuario, ejecutamos `gpupdate /force` 

![](https://imgur.com/L0EkHGo.png)

4. Comprobamos que la GPO se aplicó correctamente con `gpresult /r`

![](https://imgur.com/N9md5dB.png)

5. Verificamos que no puede abrir el Panel de Control

![](https://imgur.com/oaU5hWa.png)

6. Esta GPO se aplica a todos los usuarios de la OU Usuarios, porque por defecto el filtrado de seguridad tiene a _Authenticated Users_ con Read + Apply. Si queremos que solo se aplique a un usuario tienes que hacer estos cambios:

- Quitar _Authenticated Users_ de _Security Filtering_. Mientras siga ahí, la GPO se aplica a todos.
- Añadir al usuario en _Scope → Security Filtering_. Así el usuario recibe Read + Apply y es el único al que se aplicará la GPO.

![](https://imgur.com/b6jdbEr.png)

- Añadir a _Domain Computers_ con Read en _Delegation → Advanced_. El PC descarga la GPO con su propia cuenta de equipo, y sin permiso de lectura no podría aplicarla ni al usuario.


![](https://imgur.com/kq3175G.png)


### gpos de grupos

1. **Creamos el grupo.** En dsa.msc, clic derecho en la OU **Grupos** → _New → Group_ → *nombre del grupo*, ámbito **Global**, tipo **Security**.

![](https://imgur.com/zFnJh0p.png)


2. **Añadir miembros.** Abrir el grupo → pestaña **Members** → añadir usuarios. 

![](https://imgur.com/sOmcep6.png)

3. **Cambiar el filtrado.** Finalmente, en la GPO, dentro de _Security Filtering_, quitar usuario específico y poner grupo. 

>_Domain Computers_ con Read se queda como estaba.

3. **Comprobar.** Cerrar la sesión los usuarios y volver a entrar (el token tiene que recoger el grupo nuevo).

### gpos de equipos

1. **Crear la GPO:** En gpmc.msc, clic derecho en **Equipos** → _Create a GPO in this domain, and Link it here_ → nombre -> *mensaje*

![](https://imgur.com/K1wOz8k.png)

2. **Configurar la GPO:** _Edit_ → _Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options_:
    - **Interactive logon: Message title for users attempting to log on** → por ejemplo _Aviso_.
    - **Interactive logon: Message text for users attempting to log on** → por ejemplo _Equipo propiedad de November Lab._


![](https://imgur.com/yGbK0l4.png)


3. **Aplicar.** En los clientes ejecutar **gpupdate /force**
4. **Comprobar.** Cerrar la sesión; el mensaje debe aparecer antes de la pantalla de inicio. 
5. Con **gpresult /r /scope computer** debe aparecer la GPO aplicada

![](https://imgur.com/0Gt3AVg.png)

### configuración avanzada gpos

#### Herencia
Si creas una sub-OU dentro de Equipos y modificas la misma GPO, el equipo hereda la de Equipos pero gana la que está más cerca del equipo, la de la sub-OU.

![](https://imgur.com/6Sbfs1U.png)

#### Block Inheritance
Se pone en una OU y corta todas las GPOs que vienen de arriba, menos las que están en _Enforced_.

![](https://imgur.com/WobfoyP.png)

#### Enforced
Se pone en el vínculo de una GPO (no en la OU), y gana a lo que digan las GPOs de las sub-OUs que haya debajo.

![](https://imgur.com/TcY3E6Q.png)

#### Loopback processing
Generalmente, las GPOs de usuario dependen de la OU del usuario. Con loopback que dependan solamente del equipo en el que inicia sesión, o del equipo más el usuario. 

> Tiene dos modos: **Merge**, el usuario recibe sus GPOS y las del equipo (si hay contradicción, gana el equipo). **Replace**, el usuario recibe las OUs del equipo e ignora las suyas.

1. **Crea la GPO.** Una nueva GPO vinculada a la sub-OU de equipos: _C-Loopback-Portatiles_.
2. **Activa loopback.** En _Computer Configuration → Policies → Administrative Templates → System → Group Policy → Configure user Group Policy loopback processing mode_, activa la directiva de loopback en modo **Merge**/**Replace**
3. **Añade la restricción.** En la misma GPO, activa la GPO deseada.
4. **Aplícala.** Ejecuta **gpupdate /force** en el equipo y cierra la sesión.
5. **Comprueba.** El mismo usuario, con un resultado distinto según el equipo.

![](https://imgur.com/bH6iVqz.png)

#### DDP/FGPP
En la Default Domain Policy (Una GPO que afecta a todo el dominio) se definen generalmente las políticas de contraseña y bloqueo, etc. 

1. En GPMC, clic derecho en **Default Domain Policy** > **Editar**
2. Ir a **Configuración del equipo > Directivas > Configuración de Windows > Configuración de seguridad > Directivas de cuenta > Directiva de contraseñas** y configurar:
    - **Exigir historial de contraseñas:** 24
    - **Vigencia máxima de la contraseña:** 90 días
    - **Vigencia mínima de la contraseña:** 1 día
    - **Longitud mínima de la contraseña:** 12
    - **La contraseña debe cumplir los requisitos de complejidad:** Habilitada
    - **Almacenar contraseñas con cifrado reversible:** Deshabilitada (no tocar)
3. En **Directiva de bloqueo de cuenta**:
    - **Umbral de bloqueo de cuenta:** 5 intentos
    - **Duración del bloqueo de cuenta:** 15 minutos
    - **Restablecer el contador de bloqueos de cuenta tras:** 15 minutos
4. Cerrar el editor y en DC1 ejecutar `gpupdate /force`.

![](https://imgur.com/MOX96zG.png)

Para **FGPP** (*Fine-Grained Password Policies*), definimos una política más robusta que sobreescribe los valores de la DDP (ideal para altos cargos o admins)

> Ojo: Una FGPP no es una GPO, sob objetos de AD llamados PSO (Password Settings Object)

1. Abrir el **Centro de administración de Active Directory**
2. En el panel izquierdo cambiar a **vista de árbol** e ir a **november (local) > System > Password Settings Container**.
3. En el panel derecho, **Nuevo > Configuración de contraseña**
4. Rellenar:
    - **Nombre:** PSO_Admins
    - **Precedencia:** 10 (si un usuario tiene varias reglas de contraseña, gana la más baja)
    - **Exigir longitud mínima:** 15
    - **Exigir historial:** 24
    - **Requisitos de complejidad:** marcado
    - **Exigir vigencia mínima:** 1 día
    - **Exigir vigencia máxima:** 60 días
    - **Exigir directiva de bloqueo:** 3 intentos, contador a 30 min, bloqueo durante 30 min
    - Dejar marcado **Proteger contra eliminación accidental**
5. En **Se aplica directamente a**, pulsar **Agregar** y elegir **Domain Admins**
6. Pulsar **Aceptar**. 

![](https://imgur.com/TpKhUkR.png)

>No hace falta `gpupdate`, porque no es una GPO: se aplica en cuanto replica.

7. Verificar:
    - En ADAC, clic derecho en un usuario Domain Admin > **Ver configuración de contraseña resultante**.
# SMB + NTFS

Microsoft utiliza una convención para dar acceso a recursos llamada AGDLP ((Accounts → Global → Domain Local → Permissions). Básicamente defines Grupos Globales (agupan usuarios por departamento, ej. Economía) y Grupos locales de dominio (DL) por cada recurso y nivel de acceso (quién puede hacer qué sobre qué), ejemplo: DL_Economía_Modificar, DL_Economía_Lectura, DL_Impresora_Imprimir, etc.

> Los usuarios van en los G_, los G_ van dentro de los DL_. Así, los permisos NTFS se dan solo a los DL_. **¿Por qué?** Porque tras configurar los permisos una vez, las altas, bajas y cambios se gestionan solo moviendo usuarios entre grupos.

Para este ejemplo, crearemos la siguiente estructura:

```
LAB
├── Equipos
│   ├── PC-01
│   └── PC-02
├── Servidores        
├── Usuarios
│   ├── Economía      → thiago, riri
│   └── Operaciones   → chachi
└── Grupos
    ├── G_Economía             (Global)       → thiago, riri
    ├── G_Operaciones          (Global)       → chachi
    ├── DL_Economía_Modificar  (Dominio local) → G_Economía
    └── DL_Economía_Lectura    (Dominio local) → G_Operaciones
```

![](https://imgur.com/0UPSLBl.png)

1. Creamos carpeta `C:\DPTOS\Economia` -> Propiedades de `Economia` → **Sharing** → **Advanced Sharing**.

> El botón Share… es un asistente simplificado que también toca los permisos NTFS sin avisarte claramente, y así no ves qué capa estás configurando.

2. Marcar **Share this folder**.
3. **Permissions** → quitar **Everyone** → **Add** → **Authenticated Users** → marcar **Full Control** -> Aceptar todo.

![](https://imgur.com/xHOnVLr.png)

> la práctica habitual es dejar SMB abierto (Usuarios autenticados con Control total o Cambiar) y hacer todo el control fino en NTFS, con los grupos DL_

4. Luego Propiedades de `Economia` → **Security** → **Advanced** -> **Disable inheritance** → **Convert inherited permissions into explicit permissions**. 

> Esto permite conservar los permisos actuales como base, pero ya no dependen de `C:\`.


5. En la lista, quitar **Users** y dejar **SYSTEM**, **Administrators** y **CREATOR OWNER**. Luego **Add** → **Select a principal** → **DL_Economia_Modificar** → marcar **Modify** → Aceptar y finalmente **Add** → **DL_Economia_Leer** → dejar **Read & execute** → Aplicar y Aceptar.

![](https://imgur.com/fA2k7Rc.png)

## abe
Access-Based Enumeration (ABE) permite que los usuarios solo vean la carpeta y archivos a los que tienen acceso. Se activa en el recurso compartido, no en NTFS:

1. **Server Manager** → **File and Storage Services** → **Shares**.
2. Clic derecho en **Economia** → **Properties** → **Settings**.
3. Marcar **Enable access-based enumeration** → Aceptar.

> En el ejemplo anterior, dentro de la carpeta compartida de Economía, creamos una subcarpeta `Presupuesto 2026` a la que deshabilitamos herencia y eliminamos el DL_Leer. De modo que los usuarios que no eran del grupo economía no podían acceder

![](https://imgur.com/QXV8al5.png)

## gpp

El siguiente escenario es crear una gpp para que cada usuario tenga una unidad de red mapeada a la carpeta de su departamento, además de una segunda unidad de red **recursos** donde todos pueden acceder, pero solo rh puede modificar. Para este ejemplo, hemos creado la siguiente estructura:

```
C:\DPTOS → contenedor local, no compartido 
├── Economia → \\DC1\Economia (ABE) 
├── RH → \\DC1\RH (ABE) 
└── OPS → \\DC1\OPS (ABE) 
C:\RECURSOS → \\DC1\RECURSOS (ABE)
```

| Grupo DL_                | Permiso NTFS | Recurso  | Miembros      |
| ------------------------ | ------------ | -------- | ------------- |
| DL_Economia_Modificar    | Modificar    | Economia | G_Economia    |
| DL_Operaciones_Modificar | Modificar    | OPS      | G_Operaciones |
| DL_RH_Modificar          | Modificar    | RH       | G_RH          |
| DL_Recursos_Modificar    | Modificar    | RECURSOS | G_RH          |
| DL_Recursos_Leer         | Lectura      | RECURSOS | Domain Users  |

1. **GPMC** → clic derecho en la OU **Usuarios** → **Create a GPO in this domain, and Link it here** → nombre: `U_Unidades_Red`.
2. Clic derecho en la GPO → **Edit** → **User Configuration** → **Preferences** → **Windows Settings** → **Drive Maps**.
3. Clic derecho → **New** → **Mapped Drive**:
    - **Action**: Update
    - **Location**: `\\DC1\Economia`
    - **Reconnect**: marcado
    - **Label as**: Economia
    - **Drive Letter**: Use → **S:**

![](https://imgur.com/1Cji1ng.png)

4. Pestaña **Common** → marcar **Item-level targeting** → **Targeting…** → **New Item** → **Security Group** → los tres puntos -> **G_Economia** -> Check → OK.

![](https://imgur.com/jL5hCYA.png)

5. Repetir los pasos 3 y 4 para `\\DC1\RH` (S:, G_RH) y `\\DC1\OPS` (S:, G_Operaciones).
6. Una última unidad para `\\DC1\RECURSOS`, letra **R:**, **sin** item-level targeting.

## redirección de carpetas

Consiste en que Documentos y Escritorio de cada usuario no se guarden en el disco del PC, sino en una carpeta suya del servidor, aunque el usuario las siga viendo en el sitio de siempre. 

> Por dentro, la GPO cambia la ruta de esas carpetas en el registro del usuario para que apunten a algo como `\\DC1\Perfiles$\%USERNAME%\Documents`. El resto del perfil (AppData, configuración) sigue siendo local.

1. Crear `C:\Redireccion`.
2. Compartir como **Redireccion$**

> El `$` permite ocultar la carpeta al navegar por la red. 

4. Compartir (Adavanced Sharing -> Autheticated Users -> Full Control), activar ABE, en pestaña Security -> Opciones avanzadas → Deshabilitar herencia → Quitar todos los permisos heredados 

> Aquí Domain Users debe tener permisos de _Recorrer carpeta_, _Mostrar carpeta / Leer datos_, _Leer atributos_ y _Crear carpetas / Anexar datos_ → **Solo esta carpeta** (Cada usuario puede crear su propia carepta pero no puede ver/entrar al resto)

5. En GPMC, crear **U_Redireccion_Carpetas** en **Usuarios**.
6. Luego _User Configuration -> Policies -> Windows Settings -> Folder Redirection -> Documents_ → clic derecho → Properties.
7. Pestaña _Target_:
    - Setting: **Basic**.
    - Target folder location: **Create a folder for each user under the root path**.
    - Root path: `\\DC1\Redireccion$`

> Ojo, poner la ruta de red, no la ruta local `C:\Redireccion`

8. Pestaña _Settings_:
    - Desmarcar **Grant the user exclusive rights** (para que los admins puedan entrar)
    - Dejar marcado **Move the contents…**.
9. Repetir los pasos 6–9 con **Desktop**.

![](https://imgur.com/Q40U9zo.png)

# RSAT 
Es importante tener en cuenta que la contraseña Administrator no debe usarse para el día. Lo ideal es crear una cuenta separada para cada admin (trazabilidad) y que el DC pueda gestionarse desde una estación a través de RSAT.

En este ejemplo, utilizaremos el usuario `cesar` y la cuenta `adm-cesar`

1. **Crear la OU:** en ADUC, dentro de **LAB**, crear la OU **Admins** y deja marcada la protección contra eliminación accidental.

> Es buena práctica crear una OU separada para los Domain Admins, o para los usuarios con privilegios administrativos en general.

2. **Crear usuario normal:** **cesar** en **LAB/Usuarios**, sin grupos de administración. 
3. **Crear la cuenta de admin:** **adm-cesar** en **LAB/Admins**. En la pestaña **Miembro de**, agregarla a **Domain Admins**.
4. **Instalar la GPMC en PC-01**  
    `Add-WindowsCapability -Online -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0`
5. **Iniciar sesión en pc como cesar**
6. **Abrir las consolas como adm-cesar sin cerrar sesión.** 

> Lo dicho: **Retirar Administrator del día a día es recomendable**: ponerle una contraseña larga, guardarla y, en lo adelante, administrar solo con adm-cesar desde pc.


