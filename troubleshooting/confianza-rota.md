# Relación de confianza rota entre PC-01 y el dominio

| Campo     | Valor                                                  |
| --------- | ------------------------------------------------------ |
| Ticket    | TS-004                                                 |
| Tipo      | Incidente                                              |
| Categoría | Active Directory / Inicio de sesión                    |
| Prioridad | Media (un usuario sin acceso, sin impacto en el resto) |
| Estado    | Resuelto                                               |
| Fecha     | 2026-11-10                                             |

## Resumen

Un usuario de dominio no puede iniciar sesión en PC-01: Windows muestra _"The trust relationship between this workstation and the primary domain failed"_. La contraseña de la cuenta de equipo (PC-01$) no coincide con la guardada en AD. Se restablece el canal seguro desde el propio equipo, sin sacarlo del dominio.
## Entorno

|Elemento|Detalle|
|---|---|
|Dominio|november.lab|
|DC|DC1.november.lab — 10.10.10.10 — Windows Server 2025, todos los roles FSMO|
|Cliente|PC-01 — Windows 11, unido al dominio|
|Hipervisor|Proxmox VE (ThinkPad T480)|
|Cuentas usadas|`localadmin` (admin local de PC-01), `NOVEMBER\adm-cesar` (admin de dominio)|

## Síntoma

- En la pantalla de inicio de sesión, con un usuario de dominio: _"The trust relationship between this workstation and the primary domain failed."_
- Hora del fallo: 10:37.

![](https://imgur.com/6be4uG4.png)
## Contexto previo

Al restaurar un snapshot o una imagen antigua del equipo, el PC volvió con una contraseña de equipo que AD ya había sustituido.
## Diagnóstico paso a paso

1. **Se entró al equipo.** Las cuentas de dominio fallan, así que se inicia sesión con la cuenta local `localadmin`.
2. **Se descartó DNS y red** antes de culpar a la confianza y luego se procedió a verificar el canal seguro

![](https://imgur.com/5fwyi86.png)

3. Con DNS y red correctos, `False` apuntó a la confianza y no a un problema de conectividad. Se verificó además el visor de eventos en DC.

![](https://imgur.com/iaNW3j8.png)

## Causa raíz

La contraseña de la cuenta de equipo PC-01$ guardada en PC-01 no coincide con la de AD (restablecida desde ADUC). Sin esa contraseña el equipo no puede abrir el canal seguro con el DC, y sin canal seguro el DC rechaza los inicios de sesión de dominio en ese equipo.
## Solución

1. En PC-01, con `localadmin`, se abrió PowerShell **elevado** y se restableció el canal seguro con credenciales de dominio:

![](https://imgur.com/qLn9TVY.png)

> Otra solución consistía en sacar el equipo a un grupo de trabajo, reiniciar, volver a unirlo al dominio y reiniciar. Sin embargo, habría que reiniciar dos veces y puede dar problemas con el perfil.

## Verificación

powershell

```powershell
Test-ComputerSecureChannel   # True
```

Cierre de sesión e inicio con un usuario de dominio: correcto.

## Prevención y lecciones aprendidas

- **Orden de diagnóstico:** primero DNS y red, después el canal seguro. `Test-ComputerSecureChannel` también devuelve `False` si el equipo no llega al DC.
- **Snapshots:** no restaurar snapshots antiguos de equipos del dominio sin contar con que la confianza puede romperse.
- **Reparar en lugar de sacar y volver a unir:** es más rápido y no afecta al perfil.