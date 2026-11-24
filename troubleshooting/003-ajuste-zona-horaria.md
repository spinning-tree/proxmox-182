# Hora y zona horaria incorrectas en el dominio
## Resumen

|Campo|Valor|
|---|---|
|Ticket|TS-003|
|Tipo|Incidencia|
|Categoría|Sistema > Hora y fecha|
|Urgencia / Impacto|Baja / Medio|
|Prioridad|Baja|
|Estado|Cerrado|
|Abierto / Resuelto|2026-11-09 12:19 / 2026-11-09 13:15|

Las horas de modificación de los archivos y de la sincronización aparecen 9 horas por detrás de la hora real (3:10 AM en lugar de 12:10). El análisis encontró dos problemas independientes:

1. Todas las VMs tienen configurada la zona horaria del Pacífico (UTC-8) en lugar de la de Madrid.
2. DC1, que tiene el rol de emulador de PDC, no sincroniza con ninguna fuente externa: usa su propio reloj (`Local CMOS Clock`).

Se corrigió la fuente de hora del PDC con NTP público y la zona horaria de los clientes mediante GPO.

## Entorno

- Dominio: `november.lab`
- DC1 (10.10.10.10): emulador de PDC, AD DS, DNS, DHCP y servidor de archivos. VM 102 en Proxmox VE.
- Clientes: PC-01 (10.10.10.100) y PC-02 (10.10.10.101), Windows 11, en `LAB/Equipos`.
- Salida a Internet del lab: NAT de Proxmox (vmbr1 → vmbr0).
- Todas las VMs se crearon a partir de plantillas.

## Síntoma

- El Centro de sincronización muestra las horas de modificación de los archivos 9 horas por detrás de la hora real.
- `w32tm /query /status` en el cliente muestra _Last Successful Sync Time: 3:19 AM_ cuando en Madrid son las 12:19.
- El reloj de la barra de tareas también está mal. No se notó antes porque las VMs se usan por consola.

## Contexto previo

- Las plantillas se instalaron con la zona horaria por defecto (Pacific Standard Time), y todas las VMs creadas a partir de ellas la heredaron.
- Desde que se promovió, DC1 nunca se ha configurado con una fuente NTP externa.

## Diagnóstico paso a paso

1. **¿Es la hora o la zona horaria?** La diferencia es de exactamente 9 horas, que es la que hay entre UTC-8 (Pacífico) y UTC+1 (Madrid). Eso apunta a la zona horaria, no a un reloj desfasado.
2. **Fuente de hora del cliente:**
    - `w32tm /query /source` con un usuario normal devuelve _Access is denied (0x80070005)_, porque este comando necesita permisos de administrador.
    - `w32tm /query /status` funciona sin permisos de administrador:
        - _Source: DC1.november.lab_ (ReferenceId 10.10.10.10): el cliente toma la hora del DC, como es normal en un dominio (NT5DS).
        - _Stratum: 4_.
3. **Fuente de hora de DC1** (consola de administrador):
    - `w32tm /query /source` devuelve **Local CMOS Clock**.
    - `w32tm /query /status` devuelve **Stratum 1**.
    - DC1 se anuncia como reloj de referencia usando el reloj de la VM, que depende del reloj del host Proxmox.
4. **Incoherencia en el Stratum:** si DC1 es Stratum 1, el cliente debería ser 2, no 4.
    - En PC-01, como administrador: `w32tm /resync` y después `w32tm /query /status`.
    - Resultado: **Stratum 2**. El 4 era un valor de una sincronización anterior.

## Causa raíz

- **Zona horaria:** es una configuración local de cada equipo y no se sincroniza por el dominio. Las plantillas tenían Pacific Standard Time y todas las VMs la heredaron.
- **Fuente de hora:** el emulador de PDC es la raíz de la jerarquía de hora del dominio. Sin una fuente externa configurada, todo el dominio depende del reloj de una sola máquina, que en este caso es una VM.

> NTP y Kerberos trabajan en UTC, así que una zona horaria mal configurada no rompe la autenticación. Sí la rompe un desfase de hora real superior a 5 minutos entre cliente y DC.

## Solución

**1. NTP externo en el PDC (DC1)**

```
w32tm /config /manualpeerlist:"0.es.pool.ntp.org,0x8 1.es.pool.ntp.org,0x8 2.es.pool.ntp.org,0x8 3.es.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
w32tm /resync
```

- **0x8**: modo cliente, el que esperan los servidores NTP públicos. La primera vez se usó **0x1** (intervalo fijo), que no define el modo de consulta y con el que algunos servidores del pool no responden. Las dos flags se pueden combinar (0x9).
- `/reliable:yes`: DC1 se anuncia como fuente de hora fiable del dominio.

**2. Zona horaria**

- DC1, a mano (está en Domain Controllers y no recibe la GPO de LAB/Equipos). Primero se consultó la correlación entre el nombre y el id interno

```
tzutil /l
```

-  Brussels, Copenhagen, Madrid, Paris (UTC+01:00) corresponde a
Romance Standard Time. Se ajustó en DC1:


```
  tzutil /s "Romance Standard Time"
```


- Seguidamente, se configuró una GPO de Equipos:
    1. Se creó la GPO **C_Zona_Horaria** y se vinculó a `LAB/Equipos`.
    2. En se configuró lo siguiente: _Computer Configuration > Preferences > Control Panel Settings > Scheduled Tasks_ → _New > Immediate Task (At least Windows 7)_.
    3. Usuario: `NT AUTHORITY\System`, con **Run with highest privileges** marcado.
    4. Acción: `tzutil.exe` con los argumentos `/s "Romance Standard Time"`.

![](https://imgur.com/Js6ROuZ.png)

![](https://imgur.com/IxXjMlh.png)

- En el cliente se ejecutó `gpupdate /force` 

![](https://imgur.com/o4kbFvy.png)

- Finalmente, se confirmó la zona horaria

![](https://imgur.com/I31GycC.png)

## Prevención y lecciones aprendidas

- **La hora se sincroniza; la zona horaria no.** NTP iguala la UTC de todos los equipos, pero cada uno muestra esa hora según su zona horaria local.
- Configurar la zona horaria **en la plantilla**, antes de clonar, y mantenerla además por GPO para corregir cambios manuales.
- El **emulador de PDC** debe tener siempre una fuente NTP externa. Si se usa Local CMOS Clock, todo el dominio depende del reloj de una sola máquina.
- `w32tm /query /source` necesita permisos de administrador; `/status` no.
- El Stratum que muestra un cliente puede corresponder a una sincronización anterior. Forzar `w32tm /resync` antes de sacar conclusiones.
- Para fuentes NTP públicas, usar la flag **0x8** y no 0x1.