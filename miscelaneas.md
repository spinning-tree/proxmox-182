# ajuste de batería con tlp
tlp es una app para gestionar la energía de un equipo: umbrales de batería, frecuencia y modo de la CPU, ahorro de energía en PCIe, USB, SATA, Wi-Fi, Bluetooth y audio, etc.

En este ejemplo lo utilizaremos para gestionar los umbrales de carga de la batería de un t480.

| Comando                                                                                                        | Para qué sirve                                                           |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **apt install tlp**                                                                                            | Instala TLP                                                              |
| **nano /etc/tlp.d/**_01-bateria.conf_                                                                          | Crea un fichero de configuración propio (no toca el `tlp.conf` original) |
| **systemctl enable --now tlp**                                                                                 | Activa TLP en el arranque y lo arranca ya, aplicando los umbrales        |
| **tlp start**                                                                                                  | Vuelve a aplicar la configuración si la cambia                           |
| **tlp-stat -b**                                                                                                | Muestra el estado de las baterías y los umbrales activos                 |
| **cat /sys/class/power_supply/**{**BAT0** \| **BAT1**}**/charge_control_**{**start** \| **end**}**_threshold** | Comprueba un valor directamente en el kernel                             |

En `/etc/tlp.d/01-bateria.conf`:

```
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
START_CHARGE_THRESH_BAT1=75
STOP_CHARGE_THRESH_BAT1=80
```

Para forzar la descarga sin desenchufar:
```
echo force-discharge > /sys/class/power_supply/BAT1/charge_behaviour
```
Luego restableces la configuración:
```
echo auto > /sys/class/power_supply/BAT1/charge_behaviour
```
Finalmente, confirmar con 
```
tlp-stat -b
```

> No puedes forzar la descarga en ambas baterías. Descargas una y luego descargas la otra. Para mantener un seguimiento: `watch -n 60 cat /sys/class/power_supply/BAT0/capacity`

# monitorización con lm-sensors
lm-sensors es una herramienta para leer los sensores de hardware del equipo: temperaturas de CPU, NVMe y chipset, velocidad de ventiladores y voltajes, según lo que exponga cada placa.

En este ejemplo lo utilizaremos para vigilar temperaturas y ventilador de un t480.

| Comando                             | Para qué sirve                                                          |
| ----------------------------------- | ----------------------------------------------------------------------- |
| **apt install lm-sensors**          | Instala lm-sensors                                                      |
| **sensors-detect** [**--auto**]     | Busca chips de sensores y propone qué módulos cargar                    |
| **sensors**                         | Muestra todas las lecturas actuales                                     |
| **watch -n** _segundos_ **sensors** | Refresca las lecturas cada _segundos_ en tiempo real                    |

En la salida de `sensors`:

- `coretemp-isa-0000`: temperatura de cada núcleo de la CPU
- `thinkpad-isa-0000`: ventilador (RPM) y sensores de la placa
- `nvme-pci-*`: temperatura del SSD
- `acpitz-acpi-0`: zona térmica ACPI

