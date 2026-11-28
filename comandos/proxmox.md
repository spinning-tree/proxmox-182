# Proxmox

## Nodo

|Comando|Qué hace|
|---|---|
|**pveversion -v**|Versión de Proxmox y paquetes|
|**pvesh get /nodes**|Lista nodos y estado: RAM, CPU, online|
|**apt update && apt full-upgrade -y**|Actualizar el sistema (nunca apt upgrade a secas)|
|**systemctl status pveproxy**|Estado de la interfaz web|
|**systemctl status pvedaemon**|Estado del daemon|
|**systemctl restart pveproxy**|Reiniciar la web (puerto 8006) si no carga|
|**journalctl -xe**|Logs recientes del sistema|
|**pvereport**|Informe completo del nodo (diagnóstico)|
|**pvesm status**|Almacenamientos y espacio libre|
|**pvesm list local**|Contenido de un almacenamiento|

## Máquinas virtuales

|Comando|Qué hace|
|---|---|
|**qm list**|Lista VMs con estado|
|**qm status** _vmid_|Estado de una VM|
|**qm config** _vmid_|Ver configuración (CPU, RAM, discos, red)|
|**cat** _/etc/pve/qemu-server/vmid.conf_|Archivo de configuración de la VM|
|**qm start** _vmid_|Arrancar|
|**qm shutdown** _vmid_|Apagado ordenado (ACPI)|
|**qm stop** _vmid_|Apagado forzado (como quitar corriente)|
|**qm reboot** _vmid_|Reiniciar|
|**qm set** _vmid_ **--name** _nombre_|Cambiar nombre|
|**qm set** _vmid_ **--memory** _MB_|Cambiar RAM (MB)|
|**qm set** _vmid_ **--cores** _n_|Cambiar núcleos|
|**qm set** _vmid_ **--onboot 1**|Arranque automático con el nodo|
|**qm clone** _vmid_ _nuevo_id_ **--name** _nombre_|Clonar VM|
|**qm template** _vmid_|Convertir en plantilla|
|**qm destroy** _vmid_ **--purge**|Borrar VM y sus discos|
|**qm unlock** _vmid_|Quitar bloqueo (tras backup/snapshot fallido)|
|**qm terminal** _vmid_|Consola serie (si está configurada)|

## Snapshots

|Comando|Qué hace|
|---|---|
|**qm snapshot** _vmid_ _snapshot_name_|Crear snapshot|
|**qm listsnapshot** _vmid_|Listar snapshots|
|**qm rollback** _vmid_ _snapshot_name_|Volver a un snapshot|
|**qm delsnapshot** _vmid_ _snapshot_name_|Borrar snapshot|

## Backups y restauración

|Comando|Qué hace|
|---|---|
|**vzdump** _vmid_ **--storage local --mode snapshot --compress zstd**|Backup en caliente|
|**vzdump --all --storage local**|Backup de todo|
|**qmrestore** _archivo.vma.zst_ _vmid_|Restaurar VM|