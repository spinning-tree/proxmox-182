# Disco de datos cifrado con BitLocker bloqueado tras moverlo a otro equipo

| Campo                  | Valor                                 |
| ---------------------- | ------------------------------------- |
| **Ticket**             | TS-009                                |
| **Tipo**               | Incidencia                            |
| **Categoría**          | Hardware / Almacenamiento – BitLocker |
| **Urgencia / Impacto** | Media / Bajo (un usuario)             |
| **Prioridad**          | Media                                 |
| **Estado**             | Resuelto                              |
| **Abierto / Resuelto** | 2026-11-20 / 2026-11-20               |

## Resumen

El equipo de un usuario (PC03) queda fuera de servicio y su disco de datos D: se instala en un equipo provisional (PC04). En PC04, D: aparece bloqueado por BitLocker. Se desbloquea con la clave de recuperación guardada en AD, buscándola por su ID, y se configura el desbloqueo automático en el equipo nuevo.

## Entorno

|Elemento|Detalle|
|---|---|
|Dominio|november.lab|
|DC|DC1 (Windows Server 2025) con _BitLocker Recovery Password Viewer_ instalado|
|Equipo origen|PC03 (Windows 11, VM en Proxmox VE con vTPM 2.0)|
|Equipo destino|PC04 (Windows 11, VM en Proxmox VE con vTPM 2.0)|
|Disco afectado|D: (30 GB, volumen de datos)|
|Cuentas usadas|Administrator del dominio|
|Requisito previo|GPO de BitLocker con guardado de claves en AD DS para _Operating System Drives_ y _Fixed Data Drives_|

## Síntoma

Tras instalar en PC04 el disco D: procedente de PC03, la unidad aparece en el Explorador con un candado. Al abrirla, BitLocker pide una contraseña para desbloquearla.
## Contexto previo

- GPO vinculada a la OU de equipos, en **Computer Configuration → Policies → Administrative Templates → Windows Components → BitLocker Drive Encryption**:
    - **Operating System Drives → Choose how BitLocker-protected operating system drives can be recovered**: guardar en AD DS y no activar BitLocker hasta que la clave esté en AD DS.
    - **Fixed Data Drives → Choose how BitLocker-protected fixed drives can be recovered**: los mismos ajustes para discos de datos.
- C y D: cifrado en PC03, con protectores **TPM** y **Numerical Password** y desbloqueo automático.

![](https://imgur.com/COsC0t2.png)

- C: cifrado en PC04 por comandos y con desbloqueo automático:
    Protectores de D: en PC03: **Numerical Password** y **External Key (Required for automatic unlock)**.

![](https://imgur.com/fYhj3eV.png)

- Simulación del traslado: con las dos VM apagadas, en Proxmox **PC03 → Hardware → disco → Disk Action → Reassign Owner → PC04**.
## Diagnóstico paso a paso

1. **Comprobar el estado de la unidad en PC04:** en el Explorador, doble clic en D: → **More options → Enter recovery key**. La ventana muestra el **Key ID**.

![](https://imgur.com/kMwPoJ2.png)

2. **Buscar la clave en AD:** en ADUC, la clave aparece en el objeto de equipo **PC03**, no en PC04, porque se guarda en el equipo donde se cifró la unidad.

![](https://imgur.com/iRIoGE0.png)

> Es buena práctica buscar por ID desde el dominio que revisar equipo por equipo.
## Causa raíz

El desbloqueo automático de D depende de una **External Key** que Windows guarda en el registro de **C de PC03**, protegida por el cifrado de ese C. En PC04 esa llave no existe. El único protector de D que se puede usar en otro equipo es la **Numerical Password** (clave de recuperación de 48 dígitos).

**Esto no es un fallo**: es el comportamiento esperado de BitLocker. Un disco de datos que sale de su equipo queda protegido hasta que alguien presente una llave válida.
## Solución

1. Desbloquear D: con la clave de recuperación obtenida de AD:
    - GUI: Explorador → D: → **More options → Enter recovery key** → introducir los 48 dígitos → **Unlock**.

![](https://imgur.com/asGJ7B5.png)

2. Finalmente, activar el desbloqueo automático en PC04 

![](https://imgur.com/fYhj3eV.png)

3. Eliminar la External Key de PC03:

![](https://imgur.com/3RKAEEm.png)

## Verificación

![](https://imgur.com/YKlxOn5.png)

- _Lock Status: Unlocked_
- _Automatic Unlock: Enabled_
- _Protection Status: Protection On_
- Una **Numerical Password** y una sola **External Key**, la de PC04.

Reiniciar PC04 y comprobar que D: se abre sin pedir clave.

## Prevención y lecciones aprendidas

- **Configurar _Fixed Data Drives_ en la GPO antes de cifrar discos de datos.** Con solo _Operating System Drives_, la clave de D: no sube a AD y el disco queda irrecuperable fuera de su equipo.
- **No borrar ni recrear el objeto de equipo de origen** hasta haber recuperado los datos. Las claves de BitLocker son objetos hijos del equipo y se van con él (salvo Papelera de reciclaje de AD).
- **Activar BitLocker en C: no cifra D:.** BitLocker funciona por volumen, y cada unidad se activa por separado.
- **El auto-unlock de un disco de datos requiere el C: del equipo cifrado con BitLocker**, porque la External Key se guarda protegida en ese C:.
- **Buena práctica:** una clave de recuperación usada o expuesta debería rotarse (añadir una nueva, subirla a AD con `-adbackup` y borrar la antigua). En AD local no rota sola.