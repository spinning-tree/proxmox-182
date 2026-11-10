# Desfase horario en PC-01 y comportamiento de Kerberos

| Campo     | Valor                                      |
| --------- | ------------------------------------------ |
| Ticket    | TS-005                                     |
| Tipo      | Incidente (simulado en lab)                |
| Categoría | Active Directory / Kerberos / Hora         |
| Prioridad | Baja (no llegó a producir fallo)           |
| Estado    | Cerrado: el fallo esperado no se reprodujo |
| Fecha     | 2026-11-10                                 |

## Resumen

Se desfasa el reloj de PC-01 unos 30 minutos respecto a DC1 para provocar el fallo de Kerberos por desfase horario (límite teórico: 5 min). **El fallo no se reprodujo**: el cliente siguió obteniendo TGT y tickets de servicio, y el DC no registró ningún rechazo (sin evento 4771). Durante la prueba se analizaron los tickets con `klist`, lo que deja varias lecciones sobre Kerberos y sobre sesiones SMB.

## Entorno

|Elemento|Detalle|
|---|---|
|Dominio|november.lab|
|DC|DC1.november.lab — 10.10.10.10 — Windows Server 2025, todos los roles FSMO (PDC incluido)|
|Cliente|PC-01 — Windows 11, unido al dominio, acceso por RDP|
|Cuentas usadas|`NOVEMBER\cesar` (usuario estándar), `NOVEMBER\adm-cesar` (admin de dominio, ventanas elevadas)|

## Síntoma

Ninguno: con 30 minutos de desfase, el acceso a `\\november.lab\SYSVOL` y `\\DC1\RECURSOS` siguió funcionando.

## Contexto previo

### Jerarquía de hora (línea base)

En DC1:

![](https://imgur.com/P9ykBkn.png)

En PC1:

![](https://imgur.com/x5Dkoeq.png)

> Cadena: pool NTP externo → DC1 (PDC) → PC-01.
### Cómo se provocó el desfase

En DC1:

![](https://imgur.com/4YzBcLO.png)

En PC1:

![](https://imgur.com/Ewbw3cO.png)

> Resultado: PC-01 a las 12:56, DC1 a las 13:25.

## Diagnóstico paso a paso

1. **Sesión nueva tras el desfase**

![](https://imgur.com/Bl1YtnY.png)

> La hora de los tickets es la de **DC1** (12:59), no la de PC-01.

2. **Buscar el rechazo en DC1** (log Security, evento **4771** con Failure Code **0x25** = KRB_AP_ERR_SKEW): no aparece.

- Se activa la auditoría

![](https://imgur.com/bbRKmmN.png)

- Se repite con `klist purge` + `dir \\november.lab\SYSVOL`:

![](https://imgur.com/7lwQV5h.png)


- Se demuestra que la auditoría funciona y la petición llegó al DC. (**Ningún 4771**: el DC no rechazó la petición por desfase)

![](https://imgur.com/ieCb0N8.png)

## Causa raíz

No hubo fallo. Hechos comprobados:

- Con ~30 min de desfase, PC-01 obtuvo TGT y tickets de servicio de DC1 sin error.
- El DC emitió los tickets (4768/4769) sin registrar ningún rechazo previo (4771).
- Las horas de los tickets corresponden al reloj del DC.

> **Mecanismo no confirmado:** El cliente Kerberos de Windows compensa por su cuenta el desfase con el KDC. No verificado.

## Solución

Restaurar la hora:

![](https://imgur.com/XujthGB.png)

> Verificación: la hora de PC-01 coincide con la de DC1.

## Hallazgos adicionales durante la prueba

### 1. `klist purge` y `net use /delete` no cortan una sesión SMB ya autenticada

```powershell
net use * /delete         # desaparece R: \\DC1\RECURSOS
klist purge
dir \\DC1\Recursos        # funciona
klist                     # Cached Tickets: (0) — ni siquiera TGT
```

En DC1 no hubo un **4624** nuevo de cesar a esa hora: no hubo autenticación nueva, y el acceso reutilizó la sesión SMB que seguía viva por debajo. Más tarde, los tickets volvieron a aparecer.

**Consecuencia práctica:** para que un usuario estrene permisos (por ejemplo, tras añadirlo a un grupo), lo fiable es **cerrar sesión en Windows** (o reiniciar).

### 2. `klist sessions` frente a `quser`

|Comando|Qué muestra|
|---|---|
|`quser`|Sesiones de **escritorio** reales (quién está conectado ahora)|
|`klist sessions`|Sesiones de **inicio** de la LSA: servicios, red, ventanas elevadas y restos aún no liberados|

En PC-01, `klist sessions` mostraba dos sesiones de localadmin (Session 2), pero `quser` solo mostraba a cesar: eran sesiones de inicio no liberadas, no un escritorio abierto.
## Prevención y lecciones aprendidas

- **Comprobar siempre la hora** en problemas de autenticación, pero que funcione no demuestra que la hora esté bien.
- **No dar por buena una hipótesis sin evidencia:** buscar el evento que la confirme (4771 / 0x25) y verificar antes que la auditoría está activa (`auditpol`).
- **Dejar activada en el DC la auditoría de fallos de Kerberos:** será útil para los bloqueos de cuenta (tema 9).
- **Deshabilitar w32time, no solo detenerlo,** si se quiere congelar la hora; en Manual puede arrancar por evento.
- **Distinguir sesión de escritorio y sesión de inicio** (`quser` frente a `klist sessions`).