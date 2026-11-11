# Conflicto de sincronización en carpetas redirigidas (Archivos sin conexión)
## Resumen

| Campo              | Valor                                 |
| ------------------ | ------------------------------------- |
| Ticket             | TS-002                                |
| Tipo               | Incidencia                            |
| Categoría          | Almacenamiento > Carpetas redirigidas |
| Urgencia / Impacto | Media / Bajo                          |
| Prioridad          | Baja                                  |
| Estado             | Cerrado                               |
| Abierto / Resuelto | 2026-11-09 12:17 / 2026-11-09 12:23   |
Un archivo del Escritorio redirigido (`incidencias.txt`) muestra contenido distinto en PC-01 y PC-02. Las dos copias se editaron mientras el servidor de archivos no estaba disponible. Al volver el servidor, Archivos sin conexión detectó un conflicto que solo se ve en el Centro de sincronización del equipo que sincronizó en segundo lugar.
## Entorno
- Dominio: `november.lab`
- DC1 (10.10.10.10): AD DS, DNS, DHCP, servidor de archivos. Recurso compartido oculto `\\DC1\Redireccion$` (ABE).
- Clientes: PC-01 (10.10.10.100) y PC-02 (10.10.10.101), Windows 11, unidos al dominio.
- GPO `U_Redireccion_Carpetas`, vinculada a `LAB/Usuarios`: Folder Redirection Basic de Documents y Desktop → `\\DC1\Redireccion$\%USERNAME%`.
- Archivos sin conexión: activado por defecto en los clientes. Las carpetas redirigidas quedan disponibles sin conexión automáticamente.
- Usuario afectado: `grisel`.
## Síntoma
- El usuario ve un contenido distinto en `incidencias.txt` según la PC desde la que inicia sesión.
- Desde el punto de vista del usuario: "he perdido cambios" o "en cada PC tengo un archivo diferente".
- Otros cambios hechos en el mismo periodo (una carpeta y un archivo nuevo dentro de ella) sí aparecen en las dos PCs.

> En PC-01

![](https://imgur.com/GNL5YGH.png)

> En PC-02

![](https://imgur.com/3gR6hnU.png)
## Contexto previo

1. DC1 se apagó (`qm stop 102`) con `grisel` con la sesión iniciada en PC-01 y PC-02.
2. Con el servidor no disponible, en las dos PCs:
    - se editó `incidencias.txt` con un texto distinto en cada una;
    - en una de ellas se creó una carpeta con un archivo dentro.
3. Se volvió a encender DC1 (`qm start 102`) y las dos PCs pasaron a estar en línea.
## Diagnóstico paso a paso
1. **¿El cambio afecta a todo o solo a un archivo?** La carpeta nueva y su contenido aparecen en las dos PCs, así que la sincronización funciona en general. El problema está limitado a `incidencias.txt`.
2. **¿Qué tiene de distinto ese archivo?** La carpeta nueva solo existía en un lado, así que no hay nada que comparar. El txt ya existía y se modificó en dos sitios a la vez desde la última sincronización.
3. **Centro de sincronización en la primera PC:** _Panel de control → Centro de sincronización → Ver conflictos de sincronización_. No hay conflictos.
4. **Centro de sincronización en la segunda PC:** aparece un conflicto en `incidencias.txt`:
    - _On this computer_: 58 bytes, modificado a las 3:10.
    - `\\DC1\Redireccion$\grisel\Desktop`: 75 bytes, modificado a las 3:12.
5. **Interpretación:** solo ve el conflicto la PC que sincronizó en segundo lugar. Cuando la primera conectó, el servidor aún tenía el archivo original, así que subió su versión sin problema. Cuando conectó la segunda, el archivo había cambiado tanto en el servidor como en su caché local.

![](https://imgur.com/XfJkTDc.png)
## Causa raíz
Archivos sin conexión compara **archivos completos**, no líneas, y no combina contenidos. Si el mismo archivo se modifica en dos cachés locales mientras el servidor no está disponible:

- la **primera** PC que vuelve a conectar sube su versión al servidor sin conflicto;
- la **segunda** encuentra cambios en los dos lados y genera un conflicto que el usuario tiene que resolver.

La versión que se queda en el servidor depende del **orden de sincronización**, no de cuál es más reciente.
## Solución
1. En la PC con el conflicto, se abrió _Centro de sincronización → Ver conflictos de sincronización_.
2. Se seleccionó el archivo y se pulsó **Resolver**.
3. Se eligió **Keep both versions**. La versión local se renombró a `incidencias (grisel v1).txt` y se conservaron ambas.
4. El usuario revisó las dos versiones, combinó a mano lo que necesitó y borró la que sobraba.
## Verificación
- El Centro de sincronización de las dos PCs no muestra conflictos.
- En `\\DC1\Redireccion$\grisel\Desktop` están los archivos que espera el usuario.
- Después de pulsar F5 en el Explorador, las dos PCs muestran el mismo contenido.
## Prevención y lecciones aprendidas
- Evitar editar el mismo archivo desde varias PCs mientras el servidor no está disponible.
- Ante un "he perdido cambios" en carpetas redirigidas, revisar el Centro de sincronización **en todas las PCs** del usuario: el conflicto solo aparece en la que sincronizó en segundo lugar.
- Las horas de los archivos aparecían en horario del Pacífico (3:10 frente a las 12:10 de Madrid): las VMs tienen mal la zona horaria. Se corrigió (`tzutil /s "Romance Standard Time"`) para que las horas de los archivos y de los eventos puedan leerse correctamente.












