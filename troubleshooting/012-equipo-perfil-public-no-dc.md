# Equipo sin red al arrancar con el DC apagado

## Resumen

| Campo              | Valor                            |
| ------------------ | -------------------------------- |
| Ticket             | TS-012                           |
| Tipo               | Incidencia                       |
| Categoría          | Red > DHCP / Firewall de Windows |
| Urgencia / Impacto | Alta / Bajo (un equipo)          |
| Prioridad          | Media                            |
| Estado             | Resuelto                         |
| Abierto / Resuelto | 21/11/2026 / 21/11/2026          |

El usuario enciende su equipo mientras DC1, el único DC disponible, está apagado. Como DC1 también es el servidor DHCP, el equipo arranca con una dirección APIPA y se queda sin red. Al restablecer la conectividad, el equipo queda en el perfil de firewall Public y no vuelve a Domain aunque DC1 esté de nuevo en línea, hasta que se fuerza una nueva detección de la red.

## Entorno

| Elemento                 | Detalle                                                                                                                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dominio                  | november.lab                                                                                                                                                                                      |
| DC                       | DC1 (10.0.0.10, Windows Server 2025): AD DS, DNS y DHCP, con DHCP failover configurado con DC2. DC2 (10.0.0.11) apagado durante toda la incidencia                                                |
| Cliente afectado         | pc01 (Windows 11, VM de Parallels con adaptador VirtIO)                                                                                                                                           |
| Equipo de administración | DC1                                                                                                                                                                                               |
| Puerta de enlace         | OpenWrt (10.0.0.1, opción 003 del ámbito)                                                                                                                                                         |
| DNS del cliente          | 10.0.0.10 y 10.0.0.11                                                                                                                                                                             |
| GPO implicada            | EQ_ADMINISTRACION_REMOTA: reglas ICMPv4-In, Remote Desktop, Windows Remote Management (HTTP-In) y File and Printer Sharing (SMB-In), solo para el perfil Domain y con origen limitado a 10.0.0.10 |
| Cuentas usadas           | Administrador del dominio (elevación en pc01)                                                                                                                                                     |
| Requisito previo         | El usuario de dominio ha iniciado sesión antes en pc01, por lo que tiene credenciales en caché                                                                                                    |

## Síntoma

> "He encendido el ordenador y no tengo Internet."

El usuario inicia sesión con credenciales en caché, pero el equipo no tiene red.

## Contexto previo

- DC1 amaneció apagado. La causa no se ha determinado; una posibilidad es un reinicio por actualizaciones. DC2 también estaba apagado.
- Antes de la incidencia, pc01 tenía una concesión DHCP vigente (10.0.0.100), el perfil `DomainAuthenticated` y la administración remota desde DC1 funcionando (ping, RDP, `Enter-PSSession` y `\\pc01\c$`).

## Diagnóstico paso a paso

1. **Comprobar la configuración IP del cliente:**

![](https://imgur.com/oN4YaDg.png)

> Resultado: dirección APIPA (169.254.x.x). El equipo no ha obtenido una dirección del servidor DHCP.

2. **Revisar el registro del cliente DHCP.** En _Event Viewer › Applications and Services Logs › Microsoft › Windows › Dhcp-Client › Admin_ aparece el evento **1001** con el error **0x79** (_timeout_): el equipo no ha recibido respuesta del servidor DHCP. No hay ningún evento de validación con la puerta de enlace.

![](https://imgur.com/iIQpYvA.png)

3. **Comprobar si se mantiene la concesión al reiniciar el adaptador** (sin reiniciar el equipo). Se deshabilita y se vuelve a habilitar el adaptador de red: el equipo sigue en APIPA.
4. **Aislar el problema con una IP estática temporal** (10.0.0.100/24, puerta de enlace 10.0.0.1, DNS 10.0.0.10) y comprobar la puerta de enlace:

> Resultado: el OpenWrt responde. La red física y la puerta de enlace funcionan, así que el problema está en el servicio DHCP (DC1).

5. **Comprobar el perfil de red con la IP estática:**

![](https://imgur.com/9CAql0m.png)

> Resultado: `NetworkCategory: Public`, `DomainAuthenticationKind: None` y nombre de red `Network` en lugar de `november.lab`. Con este perfil, las reglas de administración remota de la GPO (solo Domain) no se aplican.

6. **Encender DC1** sin tocar pc01 y volver a comprobar el perfil al cabo de unos minutos. El equipo sigue en Public.

![](https://imgur.com/x5o5mot.png)

7. **Descartar un problema de conectividad con el DC:**
 
![](https://imgur.com/7UYqG1x.png)

> Resultado: localiza `\\dc1.november.lab` (10.0.0.10) con los flags `LDAP`, `KDC` y `DNS_DC`. El DNS, la localización del DC y la conectividad funcionan, pero el perfil sigue siendo Public. NLA (Network Location Awareness) no ha vuelto a evaluar la red.

8. **Forzar una nueva detección de red** desconectando y volviendo a conectar el adaptador de red de la VM en Parallels (equivale a desconectar el cable) y comprobar el perfil:

![](https://imgur.com/ZUoTJZ9.png)

> Resultado: `NetworkCategory: DomainAuthenticated`, `DomainAuthenticationKind: Ldap` y nombre de red `november.lab`.

![](https://imgur.com/x5o5mot.png)

## Causa raíz

Hay dos causas encadenadas:

1. **Sin DHCP, el equipo se queda sin IP.** DC1 concentra DHCP, DNS y AD, y DC2 (el socio de failover) también estaba apagado. Al arrancar, el cliente no recibe respuesta DHCP y pasa a APIPA, aunque su concesión seguía vigente y la puerta de enlace respondía.

> Según [documentación consulatada](https://learn.microsoft.com/en-us/windows-server/troubleshoot/how-to-use-automatic-tcpip-addressing-without-a-dh), el comportamiento clásico documentado por Microsoft es conservar la IP si responde la puerta de enlace. Este comportameiento no se cumplió en este entorno.

2. **NLA no vuelve a evaluar la red cuando el DC regresa.** NLA determina el perfil cuando detecta un cambio en la red, y para asignar el perfil Domain necesita autenticarse contra un DC. Sin DC disponible, la red quedó como Public. Cuando DC1 volvió, para el cliente no había cambiado nada en la red, así que NLA no repitió la detección y el equipo siguió en Public, con las reglas de la GPO sin aplicarse.

## Solución

1. Encender DC1 y comprobar que los servicios DHCP y DNS están en ejecución.
2. Forzar una nueva detección de red en el cliente. De menor a mayor impacto:
    - **Sin permisos de administrador** (instrucciones al usuario por teléfono): desconectar el cable de red y volver a conectarlo (en un portátil, desactivar y activar el Wi-Fi) y, si no se recupera, reiniciar el equipo.
    - **Con permisos de administrador** (por ejemplo, tras entrar con Asistencia rápida, que funciona en Public porque la conexión sale desde el equipo del usuario): `Restart-Service NlaSvc -Force` o deshabilitar y habilitar el adaptador.
3. Devolver el adaptador a DHCP para retirar la IP estática temporal.

> En el lab solo se comprobó la desconexión y reconexión del adaptador. El reinicio del servicio NLA no se probó.

## Verificación

- `Get-NetConnectionProfile` devuelve `DomainAuthenticated`, `Ldap` y `november.lab`.
- Con el adaptador de nuevo en DHCP, el equipo obtiene una concesión de DC1 (10.0.0.104).

![](https://imgur.com/sz8sLAL.png)

## Prevención y lecciones aprendidas

- **No apagar nunca todos los DC a la vez.** El parcheo de los DC se hace de forma escalonada (anillos de parches).
- **Concentrar DHCP, DNS y AD en un único servidor es un punto único de fallo.** El DHCP failover solo protege si el servidor socio está encendido.
- **Una concesión vigente no garantiza conservar la IP en el arranque.** En Windows 11 (en este caso, con un adaptador VirtIO de Parallels) el equipo pasó a APIPA con la concesión vigente y la puerta de enlace respondiendo. No se descarta que el adaptador influya.
- **Si el perfil es Public pero `nltest` encuentra el DC, el problema es la detección de NLA, no la conectividad.**
- **No poner un DNS público como secundario en los clientes.** Mantendría la navegación cuando caen los DC, pero rompería la localización del DC a diario. La salida a Internet se resuelve con reenviadores en los DC.
- **No administrar clientes con una cuenta Domain Admin.** Al elevar en pc01 con la cuenta de administrador del dominio, sus credenciales quedaron en la caché del equipo. Por eso la elevación funcionó con los DC apagados. En los clientes se usa una cuenta de administración de equipos sin privilegios de dominio o la cuenta local gestionada por Windows LAPS.