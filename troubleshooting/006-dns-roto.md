# DNS mal configurado

| Campo     | Valor                        |
| --------- | ---------------------------- |
| Ticket    | TS-006                       |
| Tipo      | Incidente (simulado en lab)  |
| Categoría | Red / DNS / Active Directory |
| Prioridad | Media                        |
| Estado    | Resuelto                     |
| Fecha     | 2026-11-10                   |

## Resumen

Se cambia el DNS de PC-01 de DC1 (10.10.10.10) a un DNS público (8.8.8.8). El equipo deja de localizar el DC del dominio: fallan `gpupdate` y `\\november.lab\SYSVOL`, mientras que el acceso por nombre corto (`\\DC1`) y por IP sigue funcionando. Esto da la falsa impresión de que "la red va". La causa es que el DNS público no conoce los registros SRV de november.lab. Se resuelve volviendo a apuntar el DNS del cliente a DC1.

## Entorno

|Elemento|Detalle|
|---|---|
|Dominio|november.lab|
|DC / DNS|DC1.november.lab — 10.10.10.10 — Windows Server 2025|
|Cliente|PC-01 — Windows 11, unido al dominio, acceso por RDP (por IP)|
|Cuentas usadas|`NOVEMBER\adm-cesar` (ventana elevada)|

## Síntoma

|Prueba|Resultado|
|---|---|
|`gpupdate /force`|Falla (equipo y usuario): _"Windows could not resolve the computer name / user name… Name Resolution failure on the current domain controller"_|
|`dir \\november.lab\SYSVOL`|Falla: _"Cannot find path… because it does not exist"_|
|`dir \\DC1\RECURSOS`|Funciona|
|`dir \\10.10.10.10\Recursos`|Funciona|
![](https://imgur.com/suVlTBD.png)
## Contexto previo

Configuración original antes de provocar el fallo:

![](https://imgur.com/B0NX61i.png)

Fallo provocado: Se configura un nuevo servidor DNS, y se purga la caché dns y los tickets de Kerberos:

![](https://imgur.com/xcrwCR3.png)

![](https://imgur.com/n7bsRXv.png)
## Diagnóstico paso a paso

1. **¿Por qué `\\DC1` funciona sin un DNS válido?** Se compara una consulta que solo pregunta al DNS con la resolución completa de Windows:

![](https://imgur.com/3vLYSZJ.png)

> El DNS falla, pero Windows resuelve el nombre corto por **mDNS** (sufijo `.local`): DC1 responde en la red local con sus IP, y Windows prefiere la IPv6 link-local. El acceso a `\\DC1` también pudo reutilizar una sesión SMB abierta anteriormente, no se determinó cuál de las dos lo hizo posible.
    
2. **¿Puede el cliente localizar el DC del dominio?** Consulta del registro SRV a cada servidor DNS y se localiza el DC (NETLOGON)

![](https://imgur.com/UkptNxe.png)

> DC1 tiene los registros correctos; el DNS configurado en el cliente no.

> `nltest` no es capaz de encontrar el DC del dominio. OJO: utilizar `/force` par evitar datos cacheados de Netlogon
## Causa raíz

PC-01 usaba como DNS un servidor público que no conoce la zona november.lab. Sin los registros **SRV** (`_ldap._tcp.dc._msdcs.november.lab`), el DC Locator no encuentra ningún DC, y fallan todas las funciones que lo necesitan: GPO, rutas de dominio (`\\november.lab`), localización del KDC para Kerberos e inicios de sesión nuevos (sin caché).

## Solución

Vover a configurar el servidor DNS en cliente

![](https://imgur.com/iew8I1K.png)

## Verificación
Salida de `nltest` con el DNS correcto (flags de DC1): `PDC GC DS LDAP KDC TIMESERV GTIMESERV WRITABLE DNS_DC DNS_DOMAIN DNS_FOREST CLOSE_SITE …`

| Flag                 | Significado                                               |
| -------------------- | --------------------------------------------------------- |
| PDC                  | Tiene el rol FSMO de emulador de PDC                      |
| KDC                  | Emite tickets Kerberos                                    |
| TIMESERV / GTIMESERV | Fuente de hora / fuente de hora de referencia del dominio |
| GC                   | Catálogo global                                           |
| WRITABLE             | DC de escritura (no RODC)                                 |
| CLOSE_SITE           | Está en el mismo sitio de AD que el cliente               |
## Prevención y lecciones aprendidas

- **En un equipo del dominio, el DNS siempre debe apuntar a los DC**, nunca al router ni a un DNS público. La salida a internet se configura en el DC con reenviadores DNS.
- **Que funcione por nombre corto o por IP no demuestra que el DNS esté bien.** Para probar el DNS: `nslookup` o `Resolve-DnsName`, nunca `ping`.
- **`ERROR_NO_SUCH_DOMAIN` (1355)** con `nltest`: lo primero que hay que mirar es el DNS del cliente.
- **Usar `nltest /dsgetdc /force`**: sin `/force` puede devolver un DC en caché y ocultar el problema.
- **No nombrar nunca un dominio de AD con `.local`:** conflicto con mDNS.
- **Desactivar LLMNR y NetBIOS** en entornos corporativos, por seguridad.
- **"Me va a medias"** (acceso por IP o nombre corto sí, GPO o rutas de dominio no): sospechar del DNS.