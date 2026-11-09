# configuración de windows server

> Previamente hay que [instalar windows server](instalacion-windows-server-proxmox.md)

## glosario básico

- **SYSVOL**: carpeta compartida de cada DC donde se guardan las GPO y los scripts. Se replica entre DCs para que todos tengan lo mismo.
- **NETLOGON**: es un recurso compartido dentro de SYSVOL donde van los scripts de inicio de sesión, y también el servicio que autentica a usuarios y equipos y registra en DNS los registros del DC.
- **KDC (Key Distribution Center)**: el servicio de Kerberos. Cuando inicias sesión, te da el “ticket” que demuestra quién eres para acceder a recursos sin volver a meter la contraseña.
- **Kerberos**: el protocolo de autenticación de AD. Es sensible a la hora: si los relojes difieren más de 5 minutos, falla.
- **Registros SRV**: entradas DNS que dicen “el servicio X está en tal servidor y puerto”. Es como los clientes encuentran el DC.
- **GC (Catálogo Global)**: copia resumida de todos los objetos del bosque, para búsquedas rápidas y para el inicio de sesión.
- **DFSR**: el mecanismo que replica SYSVOL entre DCs.
- **DSRM (Directory Services Restore Mode)**: un modo de arranque especial del DC con AD apagado, como el “modo seguro” de un controlador de dominio. Se usa para reparar o restaurar la base de datos de AD.
- **RODC (Read-Only Domain Controller)**: un DC de solo lectura, pensado para sitios poco seguros como una sucursal pequeña. Tiene una copia de AD que no se puede modificar ahí; los cambios se hacen en un DC normal y le llegan replicados. Por defecto no guarda las contraseñas de los usuarios, solo las de quienes autorices. 
- **Registros SRV**: son las entradas DNS que dicen dónde está el DC, por ejemplo “LDAP de november.lab está en dc1, puerto 389”. Los clientes los consultan para encontrar el DC al unirse al dominio o iniciar sesión. El servicio Netlogon los crea en DNS al arrancar. 
## primeros pasos

1. Definir un nombre de host en Server Manager -> Local Server -> Computer name

![](https://imgur.com/8PpSjwx.png)

2. Definir IP: Local Server -> Ethernet -> Properties -> IPV4

![](https://imgur.com/qVBOIoz.png)

> En este punto, sería buena idea instalar windows app y gestionar el server desde escritorio remoto. Hay que habilitarlo en Local Server -> Remote Desktop -> Enable

3. Ahora instalamos el rol. Server Manager → **Manage → Add Roles and Features** → Role-based → este servidor → marcar **Active Directory Domain Services** (aceptar las herramientas que añade) → Install.

![](https://i.ibb.co/mF2py1Sy/Screenshot-2026-11-05-at-12-34-14.png)

4. Promover a DC. Pulsar la bandeja de aviso -> **Promote this server to a domain controller**

![](https://imgur.com/r7Dmg5q.png)
> Esperar al reinicio posterior a la promoción y unos 10-15 minutos, para que DNS, Netlogon y SYSVOL terminen de inicializarse y registrarse, luego ejecutar `dcdiag /q` para verificar que no haya errores.

5. **Configurar el asistente**
	- **Deployment:** _Add a new forest_.
	- **Root domain name:** evitar `.local` e `.internal` (reservado para uso privado)
	- **Niveles funcionales:** Windows Server 2025.
	- Dejar marcados **DNS server** y **Global Catalog** (el DC hará de DNS Server, lo recomendado. Además, el primer DC debe tener activado el catálogo global: es la guía telefónica de todo el bosque, sabe quién está en cada dominio sin tener que preguntarle a cada uno)
	- **RODC** queda desmarcado (esto se utiliza para crear DC con una copia de AD de solo lectura, ideal en entornos de poca seguridad)
	- **Contraseña DSRM:** sirve para la recuperación de AD.
	- Ignorar el aviso de delegación DNS, el nombre NetBIOS sugerido y las rutas por defecto.
	- Install. Se reinicia solo.

> Sobre DNS delegation: Sirve para que el DNS de la zona padre (`lab` en este ejemplo) redirija las consultas de `november.lab` hacia el DC. Como nadie gestiona `lab`, no hay dónde crear esa delegación, y por eso sale el aviso.

![](https://imgur.com/IBY6e8G.png)

6. **Ajustar DNS**
	- Durante la promoción a DC, el asistente configura la 127.0.0.1 como dns preferido, hay que cambiar esto y poner la ip del DC como preferida: Config -> Red -> tarjeta Ethernet -> DNS server assignment -> Edit -> preferido: ip del DC / alternativo: 127.0.0.1
	- Además, hay que configurar un dns de internet (el dc solo responde consultas de su dominio `november.lan`. Para consultas externas hay que pasarle un dns de internet): Server Manager -> Tools -> DNS -> click derecho en DC1 -> Properties -> Forwarders -> Edit -> 1.1.1.1 

> Comprobamos con `nslookup dc1.november.lab` y luego `nslookup google.com`

![](https://imgur.com/HZFokvj.png)

7. **Ajustar la zona de búsqueda inversa**

	- ya tienes la zona directa configurada (tus servidores dns traducirán nombres a ip). La zona inversa hace lo contrario (asocia una ip a un nombre). Sirve para que `nslookup` muestre el nombre del servidor en vez de “UnKnown”; logs, herramientas y algunos servicios muestren nombres en lugar de IPs, etc.
	- En Server Manager -> Tools -> DNS -> Reverse Lookup Zones -> New Zone
	- Primary Zone (con "Store the zone in Active Directory" marcado) -> IPv4 Reverse Lookup Zone -> Network ID: 10.10.10 (creará la zona `10.10.10.in-addr.arpa`) -> Dynamic updates: Allow only secure dynamic updates -> Finish
	- Comprobamos:

```powershell
PS C:\WINDOWS\system32> nslookup 10.10.10.10
Server:  UnKnown
Address:  ::1

*** UnKnown can't find 10.10.10.10: Non-existent domain
PS C:\WINDOWS\system32> ipconfig /registerdns

Windows IP Configuration

Registration of the DNS resource records for all adapters of this computer has been initiated. Any errors will be reported in the Event Viewer in 15 minutes.
PS C:\WINDOWS\system32> nslookup 10.10.10.10
Server:  UnKnown
Address:  ::1

Name:    DC1.november.lab
Address:  10.10.10.10

PS C:\WINDOWS\system32>
```

8. **Configurar NTP**
Es buena práctica configurar un servidor NTP externo

```powershell
w32tm /config /manualpeerlist:"hora.roa.es pool.ntp.org" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /query /status

# si falla forzar la sincronización
w32tm /resync /rediscover
w32tm /query /status
```
## configuración de DHCP

1. **Instalar el rol:** Server Manager → Manage → Add Roles and Features → Role-based → DC1 → marcar **DHCP Server** (aceptar añadir las herramientas) → Next hasta Install.

![](https://imgur.com/0go0zy3.png)

2. Al terminar, clic en la bandera amarilla → **Complete DHCP configuration** → Commit (Esto crea los grupos de seguridad y autoriza el servidor en AD con las credenciales del dominio)

![](https://imgur.com/Lnt42op.png)

3. **Crear el ámbito**: Tools -> DHCP -> desplegar dc1 -> click derecho en IPV4 -> New scope

![](https://imgur.com/LlSE6nf.png)


4. **Crear el ámbito:** Tools → DHCP → desplegar dc1 → clic derecho en **IPv4** → **New Scope**:
    - Nombre: LAB
    - Rango: 10.10.10.100 – 10.10.10.200, máscara 255.255.255.0
    - Exclusiones: ninguna (el pool ya no incluye .1 ni .10)
    - Duración de la concesión: por defecto (8 días)
    - Configure options now: **Yes**
    - Router (003): 10.10.10.1 → Add
    - Domain name (015) y DNS (006): el asistente rellena el dominio. En DNS dejar solo 10.10.10.10 y eliminar cualquier otra IP que proponga.
    - WINS: saltar
    - Activate scope now: **Yes**

> Verificar en la consola DHCP, IPv4 debe tener el icono verde (autorizado y activo). Si sale una flecha roja, clic derecho en dc1 → **Authorize** y luego refrescar.

## configuración OUs
En Server Manager -> Tools -> Active Directory Users and Computers:
1. Clic derecho en **november.lab** → New → Organizational Unit → **LAB** → OK.
2. Clic derecho en **LAB** → New → Organizational Unit, y repetir para cada una: **Equipos**, **Servidores**, **Usuarios**, **Grupos**.
3. Dejar marcada “Protect container from accidental deletion” en todas.
4. Redirigir los nuevos equipos/usuarios a las carpetas creadas

```
PS C:\WINDOWS\system32> redircmp "OU=Equipos,OU=LAB,DC=november,DC=lab"
Redirection was successful.
PS C:\WINDOWS\system32> redirusr "OU=Usuarios,OU=LAB,DC=november,DC=lab"
Redirection was successful.
PS C:\WINDOWS\system32>
```

![](https://imgur.com/WyhwApt.png)

### Delegación de control
En vez de hacer Domain Admin a todo el que tenga que resetear contraseñas, es buena práctica crear un grupo con  permisos (resetear contraseñas, desbloquear cuentas), y solo sobre una OU concreta. 

> El asistente de delegación añade automáticamente entradas a la ACL de esa OU.

1. **Crear el grupo:** en LAB/Grupos, un grupo global de seguridad **G_Helpdesk**.
2. **Crear un técnico de prueba:** en LAB/Usuarios, un usuario **soporte1**, y añadirlo a G_Helpdesk.
3. **Primera delegación, para resetear contraseñas:**
    - Clic derecho en la OU **LAB/Usuarios** > **Delegar control**.
    - Agregar **G_Helpdesk**.
    - Marcar **Restablecer contraseñas de usuario y forzar el cambio de contraseña en el siguiente inicio de sesión**.
    - Finalizar.
4. **Segunda delegación, para desbloquear.** Desbloquear no viene en las tareas comunes. Hay que hacerlo aparte:
    - Otra vez **Delegar control** sobre LAB/Usuarios, con **G_Helpdesk**.
    - Elegir **Crear una tarea personalizada**.
    - **Sólo los siguientes objetos de la carpeta** > marcar **Objetos Usuario**
    - En permisos, desmarcar **General** y marcar **Específicos de propiedades**
    - Marcar **Leer lockoutTime** y **Escribir lockoutTime**.
    - Finalizar.
5. **Instalar RSAT en PC-01:** Configuración -> Sistema -> Características opcionales -> Agregar -> **RSAT: Herramientas de Active Directory Domain Services**.

> Para este paso ver la sección [RSAT](configuracion-avanzada-windows-server#RSAT)

> Para comprobar, en ADUC -> **Ver > Características avanzadas**. En LAB/Usuarios -> Propiedades -> **Seguridad** -> **Opciones avanzadas**. Aparecerán las entradas de G_Helpdesk.


## Unir equipo al dominio

> Antes de unir un equipo al dominio, conviene crear una cuenta de usuario en AD. 

1. Ejecutar `sysdm.cpl` → pestaña Computer Name → **Change**.
2. Ahí cambiar el nombre y unir en el mismo paso: marcar **Domain** -> escribir el dominio → OK.
3. Credenciales: Administrador del dominio y su contraseña.
4. Esperar confirmación -> reiniciar.

![](https://imgur.com/kr5lIQS.png)

> Seguir con [configuracion avanzada windows server](configuracion-avanzada-windows-server.md)


