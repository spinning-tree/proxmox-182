# Acceso directo de Edge que el usuario no puede borrar del escritorio

| Campo              | Valor                                                         |
| ------------------ | ------------------------------------------------------------- |
| Ticket             | TS-007                                                        |
| Tipo               | Petición                                                      |
| Categoría          | Escritorio > Directivas de grupo (plantillas administrativas) |
| Urgencia / Impacto | Baja / Bajo                                                   |
| Prioridad          | Baja                                                          |
| Estado             | Resuelto                                                      |
| Abierto / Resuelto | 2026-11-11 11:16 / 2026-11-11 12:15                           |

## Resumen

El usuario quiere quitar el acceso directo de Microsoft Edge de su escritorio. Al enviarlo a la papelera, Windows le pide credenciales de administrador. El acceso directo no está en su escritorio, sino en el **escritorio público** (`C:\Users\Public\Desktop`), donde los usuarios estándar solo tienen lectura. Además, si no se hace nada más, Edge puede volver a crearlo al reinstalarse o actualizarse.

La solución combina dos cosas. Primero, una GPO con las plantillas administrativas de Edge Update, que impide que el acceso directo se cree o vuelva a aparecer. Segundo, el borrado manual en remoto del `.lnk` para que desaparezca, sin esperar a la próxima actualización.

## Entorno

| Elemento         | Detalle                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| Dominio          | november.lab                                                                                    |
| DC               | DC1.november.lab — 10.10.10.10 — Windows Server 2025 (AD DS, DNS, DHCP, archivos)               |
| Cliente afectado | PC-02 — 10.10.10.101 — Windows 11, equipo de usuario                                            |
| Usuario          | `NOVEMBER\chachi`, con redirección de Documentos y Escritorio a `\\DC1\Redireccion$\%USERNAME%` |
| Cuentas usadas   | `NOVEMBER\adm-cesar` (admin de dominio)                                                         |
## Síntoma

- El usuario arrastra el acceso directo de Edge a la papelera y aparece el aviso "Necesitas permiso de administrador para eliminar este archivo".
- El usuario no sabe que, aunque se borre, Edge puede volver a crearlo al reinstalarse o actualizarse.

## Contexto previo

- El usuario no quiere desinstalar Edge, solo quitar el acceso directo.
- Ese equipo solo lo usa el usuario.
- En el dominio todavía no existía un almacén central de plantillas administrativas (`PolicyDefinitions` en SYSVOL).

## Diagnóstico paso a paso

1. **Dónde vive el `.lnk`.** Clic derecho sobre el acceso directo → _Propiedades → General → Ubicación_. Devuelve `C:\Users\Public\Desktop`.

![400](https://imgur.com/a0ZO42k.png)

2. **De dónde sale el escritorio que ve el usuario.** El escritorio que se muestra es la suma de dos carpetas:
    - El escritorio del usuario: en este caso, redirigido al servidor. Ahí el usuario es propietario y tiene control total.
    - El escritorio público, `C:\Users\Public\Desktop`, que se muestra a **todos los usuarios** que inician sesión en ese equipo.
    - La redirección de carpetas no mezcla las dos. Solo mueve la carpeta del usuario al servidor.
3. **Permisos.** En `C:\Users\Public\Desktop`, los usuarios estándar solo tienen lectura; modificar requiere ser administrador. Eso explica la ventana de UAC.
4. **Qué pasa si se borra.** Solo hay un archivo, así que borrarlo lo quita para todos los usuarios del equipo, no solo para el usuario. Como el equipo es solo suyo, el impacto es aceptable.
5. **Si volverá a aparecer.** El acceso directo lo crea el instalador de Edge. La directiva de Edge Update que lo controla indica que, sin configurar, se crea en cada instalación. Y existe otra directiva específica para eliminarlo al actualizar, lo que indica que las actualizaciones pueden volver a ponerlo. 

> Conclusión: borrarlo a mano no basta; hay que controlarlo por GPO.

## Causa raíz

El acceso directo de Edge se instala en el **escritorio público** (`C:\Users\Public\Desktop`), que es común a todos los usuarios del equipo y en el que un usuario estándar no puede escribir. Además, Edge puede volver a crearlo al reinstalarse o actualizarse si ninguna directiva lo impide.

## Solución

### Descargar las plantillas de directiva de Edge

1. En la página de **Microsoft Edge para empresas**, se descargaron los archivos de directiva (_policy files_). Se descargó un `.cab`.

![](https://imgur.com/KRplkNk.png)

2. Dentro del `.cab` hay un `.zip`, `MicrosoftEdgePolicyTemplates`. Se extrajeron.
3. Dentro hay dos carpetas:
    - `admx`: es el formato actual y la que se usa.
    - `adm`: es el formato antiguo (XP / Server 2003). Se ignora.
4. Contenido de `admx`:
    - En la raíz, los archivos `.admx`: `msedge.admx` (navegador), `msedgeupdate.admx` (actualizador) y `msedgewebview2.admx` (WebView2).
    - Subcarpetas de idioma (`en-US`, `es-ES`…) con los `.adml`.

> **Qué son las plantillas administrativas.** Un `.admx` describe un conjunto de directivas: qué opciones aparecen en el editor de GPO y qué clave del registro escribe cada una en el cliente. Un `.adml` contiene los textos de esas directivas en un idioma concreto. Hacen falta los dos. Por sí solas no aplican nada; son como la carta de un restaurante. Lo que eliges al editar la GPO se guarda en su `Registry.pol`, en SYSVOL.
> 
> **No todas las aplicaciones tienen plantillas**: solo las publican los fabricantes de software orientado a empresas (Edge, Chrome, Firefox, Office, Adobe Reader, Zoom…). Si una aplicación no las tiene, se configura con GPP de registro o con scripts.

### Crear el almacén central en SYSVOL

Sin almacén central, el editor de GPO usa las plantillas locales del equipo desde el que se edita (`C:\Windows\PolicyDefinitions`). Así, cada administrador ve directivas distintas según su máquina. El almacén central es una única carpeta en SYSVOL que usan todos.

1. En DC1, se copió la carpeta entera `C:\Windows\PolicyDefinitions` dentro de `C:\Windows\SYSVOL\sysvol\november.lab\Policies\`.

> Si el cliente desde el que se administra tiene una versión de Windows más reciente que el DC, es mejor copiarla desde el cliente, porque sus plantillas son más nuevas.

2. Se comprobó que existe `\\november.lab\SYSVOL\november.lab\Policies\PolicyDefinitions`.
### Añadir las plantillas de Edge al almacén central

1. Se copió `msedgeupdate.admx` a `...\Policies\PolicyDefinitions\`.
2. Se copió `msedgeupdate.adml` a `...\Policies\PolicyDefinitions\en-US\`. 

> Si los equipos de administración están en español, copiar también a `es-ES`.

![](https://imgur.com/2ni5ekA.png)

### Crear el grupo de equipos

1. En ADUC, en `LAB/Grupos`, se creó `G_EQ_Edge_SinAccesoDirecto`, con ámbito **Global** y tipo **Seguridad**.
2. En _Miembros → Agregar → Tipos de objeto_, se marcó **Equipos** y se añadió `PC-02`.

![400](https://imgur.com/UNJ6BAm.png)

### Crear la GPO

1. En la GPMC, se creó `C_Edge_SinAccesoDirecto` y se vinculó a `LAB/Equipos`.

![](https://imgur.com/4pf9Nio.png)

2. Se editó la GPO y se comprobó que el nodo aparece como _Plantillas administrativas: definiciones de directiva (archivos ADMX) recuperadas del almacén central_. 

![](https://imgur.com/HRTwACI.png)

> Si no lo pone, el editor no está leyendo `PolicyDefinitions` de SYSVOL.

2. Luego en _Configuración del equipo → Directivas → Plantillas administrativas → Microsoft Edge Update → Applications_ se configuró:

| Directiva                                    | Valor                                                          | Efecto                                                                                           |
| -------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Create desktop shortcut upon install default | Deshabilitada                                                  | No se crea el acceso directo al instalar Edge. No afecta si Edge ya está instalado.              |
| Remove Desktop Shortcuts upon update default | Habilitada → _Force delete system-level Desktop Shortcuts_ (1) | Borra el acceso directo del escritorio público cuando Edge se actualiza o el equipo se reinicia. |

- No usar la opción (2) _system-level and user-level_: borraría también los accesos directos a Edge que los usuarios se hayan creado ellos mismos en su escritorio, y eso va más allá de lo que pide el ticket.
- Comprobar que las directivas equivalentes del canal (_Applications → Microsoft Edge_) están en _No configurada_, porque si lo están mandan sobre las _default_.

![](https://imgur.com/UqYpUMX.png)

4. **Filtrado de seguridad** (pestaña _Ámbito_): se quitó _Usuarios autentificados_ y se añadió `G_EQ_Edge_SinAccesoDirecto`.
5. Pestaña _Delegación_: _Usuarios autentificados_ se conservó **Lectura**, sin "Aplicar directiva". Sin ese permiso, ningún equipo puede leer la GPO y no se aplica a nadie.

![](https://imgur.com/GPrUQvd.png)

> Se filtra por **equipo**, no por el usuario, porque son directivas de _Configuración del equipo_: las procesa el equipo al arrancar, antes de que nadie inicie sesión.
### Aplicar la directiva

1. Se reinició PC-02 para que su cuenta de equipo recoja la pertenencia a `G_EQ_Edge_SinAccesoDirecto`.
2. En PC-02, se ejecutó `gpupdate /force`.
### Borrar el acceso directo en remoto

El usuario quiere que desaparezca ya, sin esperar a la próxima actualización de Edge. Por lo tanto, en DC se ejecutó lo siguiente:

![](https://imgur.com/nnQZusL.png)

> `Public\Desktop` es el escritorio común a **todos los usuarios** del equipo. Borrar de ahí quita el acceso directo a todo el que inicie sesión en PC-02. En este caso es correcto, porque el equipo solo lo usa el usuario. Si fuera un equipo compartido, habría que confirmarlo antes.

## Verificación

1. En PC-02, `gpresult /r /scope computer`: `C_Edge_SinAccesoDirecto` aparece en las GPO aplicadas y `G_EQ_Edge_SinAccesoDirecto` en los grupos del equipo.
2. `gpresult /h informe.html` (elevado): en la sección _Microsoft Edge Update_ aparecen las dos directivas con su valor.
3. En el registro de PC-02 están los valores en `HKLM\SOFTWARE\Policies\Microsoft\EdgeUpdate`.
4. `C:\Users\Public\Desktop` ya no contiene `Microsoft Edge.lnk`, y el usuario no lo ve en su escritorio.
5. Edge sigue instalado y funciona.

## Prevención y lecciones aprendidas

- El escritorio que ve un usuario es la suma de su escritorio y de `C:\Users\Public\Desktop`. Si no puede borrar un icono, lo primero es mirar dónde vive el `.lnk` (_Propiedades → General → Ubicación_), no a qué apunta.
- `Public\Desktop` es de todos los usuarios del equipo: lo que se borra ahí desaparece para todos.
- Borrar a mano algo que un instalador vuelve a crear es una solución temporal. Hay que buscar la directiva que lo controla.
- Las plantillas administrativas son `.admx` (las directivas) más `.adml` (los textos por idioma). Hacen falta las dos. La carpeta `adm` es legacy.
- Solo publican plantillas los fabricantes de software orientado a empresas. Para el resto: GPP de registro o scripts.
- Crea el almacén central (`PolicyDefinitions` en SYSVOL) para que todos los administradores editen con las mismas plantillas. Se comprueba con el texto "recuperadas del almacén central" en el editor.
- Lee siempre la ayuda de la directiva. Aquí revela que "Create desktop shortcut…" está en positivo (para impedirlo se deshabilita) y que no afecta si Edge ya está instalado.
- Las directivas de _Configuración del equipo_ se filtran por cuentas de equipo, y la nueva pertenencia a un grupo requiere reiniciar el equipo.
- Al quitar _Usuarios autentificados_ del filtrado de seguridad, hay que dejarle **Lectura** en _Delegación_.