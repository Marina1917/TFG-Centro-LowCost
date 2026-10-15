# Arranque dual LXC / VFIO en Proxmox VE

Este documento describe cómo configurar el host para arrancar en uno de dos modos sin cambios en el hardware ni edición manual de ficheros antes de cada reinicio:

1. **Modo LXC (por defecto)**: el host carga el driver NVIDIA (`nvidia.ko`) y expone los dispositivos `/dev/nvidia*` a los contenedores (ver [`02-contenedor-lxc-cuda.md`](02-contenedor-lxc-cuda.md)).
2. **Modo VFIO**: arranca el mismo kernel con los parámetros necesarios para asignar las GPUs a máquinas virtuales, y un script desvincula las GPUs del driver del host y las vincula a `vfio-pci`.

La selección del modo VFIO dura **un solo arranque**: cualquier reinicio posterior vuelve al modo LXC, que es el más estable de los probados. Esto evita tener que recordar en qué modo quedó el sistema, algo especialmente útil si en el futuro se añaden más configuraciones o más tarjetas.

> **Precaución**: no conviene cambiar de modo con máquinas virtuales o contenedores en ejecución. Antes de reiniciar, detenlos con `qm shutdown <vmid>` o `pct shutdown <ctid>`.

---

## 1. Datos del sistema

Para escribir la entrada de GRUB hacen falta los parámetros actuales del kernel, la versión del kernel y el UUID de la partición raíz:

```bash
echo "=== 1. PARÁMETROS DEL KERNEL ACTUAL ==="
cat /proc/cmdline

echo "=== 2. KERNEL E INITRD EN USO ==="
uname -r

echo "=== 3. PARTICIÓN RAÍZ Y UUID ==="
findmnt -no SOURCE,UUID /
```

Valores en el sistema de referencia:
- Parámetros del kernel: `BOOT_IMAGE=/boot/vmlinuz-6.14.11-6-bpo12-pve root=/dev/mapper/pve-root ro quiet pcie_port_pm=off pcie_aspm=off amd_iommu=on iommu=pt`
- Kernel: `6.14.11-6-bpo12-pve`
- Partición raíz: `/dev/mapper/pve-root` (volumen LVM, que contiene también `/boot`)

---

## 2. Módulos VFIO

Para que `vfio-pci` esté disponible en el modo VFIO, se añaden los módulos de VFIO a `/etc/modules`:

```bash
nano /etc/modules
```

```text
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

> **Notas**:
> - Desde el kernel 6.2, `vfio_virqfd` está integrado en `vfio` y ya no existe como módulo independiente. En el kernel 6.14 esta línea solo genera un aviso durante el arranque y puede omitirse.
> - Estos módulos se cargan en **ambos** modos. Cargarlos no impide que el driver NVIDIA tome el control de las tarjetas: solo deja `vfio-pci` disponible para que el script de la sección 3.2 pueda vincular las GPUs a él.

---

## 3. Entrada de GRUB para el modo VFIO

### 3.1 Creación de la entrada

Se añade una entrada personalizada al final de `/etc/grub.d/40_custom`, sin modificar el resto del fichero:

```bash
nano /etc/grub.d/40_custom
```

Sustituye `<UUID>` por el obtenido en la sección 1, y la versión del kernel si es distinta:

```bash
menuentry 'Proxmox VE - Modo VFIO Passthrough (Aislamiento GPU para VMs)' --class proxmox --class gnu-linux --class gnu --class os {
    load_video
    insmod gzio
    insmod lvm
    insmod ext2
    search --no-floppy --fs-uuid --set=root <UUID>
    linux /boot/vmlinuz-6.14.11-6-bpo12-pve root=/dev/mapper/pve-root ro quiet pcie_port_pm=off pcie_aspm=off amd_iommu=on iommu=pt pci=noaer vfio_pci.disable_idle_d3=1 vfio_pci.disable_vga=1 vfio_dynamic
    initrd /boot/initrd.img-6.14.11-6-bpo12-pve
}
```

La entrada apunta a una versión concreta del kernel. Si se cambia de kernel, hay que actualizarla.

**Parámetros**:
- `amd_iommu=on iommu=pt`: activa la IOMMU, necesaria para asignar dispositivos a máquinas virtuales. `iommu=pt` deja sin traducción los dispositivos que sigue usando el host, para no penalizar su rendimiento.
- `pcie_port_pm=off pcie_aspm=off`: desactivan la gestión de energía de los puertos y enlaces PCIe, para evitar que las tarjetas antiguas se desconecten al cambiar de estado.
- `pci=noaer`: desactiva el informe avanzado de errores del bus PCIe (*Advanced Error Reporting*), que con estas tarjetas genera avisos por negociaciones anómalas del enlace.
- `vfio_pci.disable_idle_d3=1`: impide que `vfio-pci` pase las tarjetas a un estado de bajo consumo (D3) mientras no las usa una máquina virtual, ya que el único método de reinicio que admiten se basa en esa transición.
- `vfio_pci.disable_vga=1`: impide que `vfio-pci` gestione las regiones VGA heredadas de las tarjetas.
- `vfio_dynamic`: no es una opción del kernel, que la ignora. Es una marca propia: indica al script de la sección 3.2 que debe vincular las GPUs a `vfio-pci`, y permite a `check-mode` (sección 5.2) saber en qué modo ha arrancado el sistema.

### 3.2 Vinculación de las GPUs a `vfio-pci`

La entrada de GRUB y los módulos de la sección 2 no vinculan las GPUs a `vfio-pci` por sí solos. De eso se encarga un script que se ejecuta durante el arranque y que solo actúa si encuentra la marca `vfio_dynamic` en `/proc/cmdline`.

#### Antecedente: reserva estática por identificadores

En la primera fase del trabajo, las GPUs se reservaban para `vfio-pci` desde el inicio del arranque mediante sus identificadores PCI (`vendor:device`), en `/etc/modprobe.d/vfio.conf`:

```text
options vfio-pci ids=10de:05e3,10de:0615
softdep nvidia pre: vfio-pci
softdep nouveau pre: vfio-pci
```

Las líneas `softdep` obligan a cargar `vfio-pci` antes que los drivers gráficos, para que sea el primero en reclamar las tarjetas. Además, se añadían los mismos identificadores y la exclusión de los drivers gráficos a la línea del kernel en `/etc/default/grub` (`vfio-pci.ids=...` y `modprobe.blacklist=nvidia,nouveau`), y se regeneraba el `initramfs`.

Este enfoque presentaba tres limitaciones:
1. **Incompatible con LXC**: con `nvidia` excluido, el host no puede crear los dispositivos `/dev/nvidia*`, así que pasar a contenedores obligaba a revertir toda la configuración.
2. **Rígido ante cambios de hardware**: cambiar una GPU de ranura o sustituirla obliga a editar los ficheros, actualizar GRUB y regenerar el `initramfs`.
3. **No selectivo**: la reserva por identificador afecta a todas las tarjetas del mismo modelo, por lo que no permite destinar una a LXC y otra a una máquina virtual si son idénticas.

#### Vinculación durante el arranque del modo VFIO

En el arranque dual, el host carga siempre sus drivers con normalidad. Solo cuando el sistema arranca con la entrada VFIO, el script desvincula las GPUs NVIDIA del driver del host y las vincula a `vfio-pci` a través de la interfaz `sysfs` del kernel. Para cada GPU:

1. Obtiene sus identificadores de `/sys/bus/pci/devices/<BDF>/vendor` y `device`, sin fijarlos de antemano.
2. Registra ese par de identificadores en `vfio-pci` (`/sys/bus/pci/drivers/vfio-pci/new_id`), para que acepte la tarjeta.
3. La desvincula de su driver actual (`.../driver/unbind`).
4. La vincula a `vfio-pci` (`/sys/bus/pci/drivers/vfio-pci/bind`).

Así no hay que tocar ficheros ni regenerar el `initramfs` al cambiar de modo o de hardware, y el modo LXC no se ve afectado.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ------------------------------------------------------------------------------
# Script de asignación dinámica VFIO para GPUs NVIDIA (GT200 / G92b)
# Detecta la bandera 'vfio_dynamic' en el cmdline y realiza el re-binding
# ------------------------------------------------------------------------------

CMDLINE="/proc/cmdline"
FLAG="vfio_dynamic"

log() {
    echo "[VFIO-DYNAMIC] $1"
}

# 1. Comprobar si la bandera de activación está presente en el arranque
if ! grep -q "\b${FLAG}\b" "$CMDLINE"; then
    log "Flag '${FLAG}' no detectado en ${CMDLINE}. Conservando drivers del host."
    exit 0
fi

log "Flag '${FLAG}' detectado. Iniciando aislamiento de GPUs..."

# Asegurar que el módulo vfio-pci esté cargado
if ! lsmod | grep -q vfio_pci; then
    log "Cargando módulo kernel vfio-pci..."
    modprobe vfio-pci
fi

# 2. Escanear el bus PCI en busca de GPUs NVIDIA (vendor 10de, clases 0300/0302).
#    Se procesan todas: la salida de vídeo del host la proporciona la iGPU.
PCI_DEVICES=$(lspci -nn | grep -E "0300|0302" | grep "10de:" | awk '{print $1}')

for BDF_SHORT in $PCI_DEVICES; do
    BDF="0000:${BDF_SHORT}"
    SYS_PATH="/sys/bus/pci/devices/${BDF}"

    if [ ! -d "$SYS_PATH" ]; then
        continue
    fi

    # Extraer los identificadores de fabricante y dispositivo
    VENDOR=$(cat "${SYS_PATH}/vendor" | sed 's/0x//')
    DEVICE=$(cat "${SYS_PATH}/device" | sed 's/0x//')

    log "Procesando dispositivo PCI: ${BDF} [${VENDOR}:${DEVICE}]"

    # Verificar si el dispositivo ya está controlado por vfio-pci
    CURRENT_DRIVER=""
    if [ -L "${SYS_PATH}/driver" ]; then
        CURRENT_DRIVER=$(basename "$(readlink "${SYS_PATH}/driver")")
    fi

    if [ "$CURRENT_DRIVER" == "vfio-pci" ]; then
        log "El dispositivo ${BDF} ya se encuentra vinculado a vfio-pci."
        continue
    fi

    # Registrar el ID en vfio-pci
    echo "${VENDOR} ${DEVICE}" > /sys/bus/pci/drivers/vfio-pci/new_id 2>/dev/null || true

    # Desvincular del driver actual, si lo hay
    if [ -n "$CURRENT_DRIVER" ]; then
        log "Desvinculando ${BDF} del driver host '${CURRENT_DRIVER}'..."
        echo "${BDF}" > "${SYS_PATH}/driver/unbind"
    fi

    # Vincular a vfio-pci
    log "Vinculando ${BDF} a vfio-pci..."
    echo "${BDF}" > /sys/bus/pci/drivers/vfio-pci/bind

    log "Dispositivo ${BDF} aislado correctamente."
done

exit 0
```

> **Nota**: el script debe ejecutarse en cada arranque, con permisos de `root`. Como solo actúa cuando encuentra la marca `vfio_dynamic` en `/proc/cmdline`, puede lanzarse también en el modo LXC sin ningún efecto. La forma concreta de integrarlo en la secuencia de arranque depende de cada instalación y no se detalla en esta guía.

> **Limitación**: el script vincula a `vfio-pci` **todas** las GPUs NVIDIA del sistema. Resuelve la rigidez de la reserva estática, pero no permite repartir las tarjetas entre LXC y máquinas virtuales en un mismo arranque. Para ello habría que filtrar por dirección PCI (BDF).

---

## 4. Arranque por defecto en modo LXC

El modo LXC debe ser el que arranque por defecto. En `/etc/default/grub`:

```bash
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
```

> **Comprobación recomendada**: `GRUB_DEFAULT=0` arranca la primera entrada del menú, que corresponde al kernel **más reciente** instalado. En el sistema de referencia, esa entrada es la del kernel 7.0.2, para el que no está compilado el driver. El sistema arranca aun así con el 6.14 porque este se fijó con `proxmox-boot-tool kernel pin` (ver [`01-instalacion-driver-host.md`](01-instalacion-driver-host.md)), que prevalece sobre este ajuste. Para comprobar qué entrada se arranca realmente por defecto:
> ```bash
> grep -r GRUB_DEFAULT /etc/default/grub /etc/default/grub.d/
> grub-editenv list
> ```

Aplicar los cambios:

```bash
update-initramfs -u -k all
update-grub
```

Para comprobar el nombre exacto de cada entrada del menú:

```bash
grep -E "menuentry ['\"]" /boot/grub/grub.cfg | cut -d"'" -f2 | cut -d'"' -f2
```

---

## 5. Comandos de conmutación y comprobación

### 5.1 `boot-vfio`: arrancar una vez en modo VFIO

Fija la entrada VFIO **solo para el próximo reinicio** y reinicia el sistema. Si GRUB no acepta la selección, cancela el reinicio.

```bash
nano /usr/local/bin/boot-vfio
```

```bash
#!/bin/bash
# Nombre de la entrada de GRUB del modo VFIO
ENTRY_NAME="Proxmox VE - Modo VFIO Passthrough (Aislamiento GPU para VMs)"

echo "[+] Configurando próximo arranque único en: $ENTRY_NAME"
grub-reboot "$ENTRY_NAME"

if [ $? -eq 0 ]; then
    echo "[+] Parámetros aplicados correctamente. Reiniciando servidor..."
    systemctl reboot
else
    echo "[-] Error al fijar la entrada de GRUB. Reinicio cancelado."
    exit 1
fi
```

### 5.2 `check-mode`: comprobar el modo activo

Muestra en qué modo ha arrancado el sistema y qué driver tiene asignado cada GPU.

```bash
nano /usr/local/bin/check-mode
```

```bash
#!/bin/bash
echo "=== Estado del Modo de Arranque ==="
if grep -qw "vfio_dynamic" /proc/cmdline; then
    echo "MODO ACTUAL: [ VFIO Passthrough / KVM ]"
else
    echo "MODO ACTUAL: [ Default / LXC Host Drivers ]"
fi

echo -e "\n=== GPUs detectadas en el sistema y driver activo ==="
lspci -nnk | grep -E "VGA compatible controller|3D controller" -A 3
```

Permisos de ejecución:

```bash
chmod +x /usr/local/bin/boot-vfio /usr/local/bin/check-mode
```

---

## 6. Retorno automático al modo LXC

`grub-reboot` guarda la entrada elegida en la variable `next_entry` del fichero de entorno de GRUB (`/boot/grub/grubenv`). Normalmente GRUB borra esa variable al arrancar, pero **no puede escribir en un volumen LVM**, que es donde Proxmox instala `/boot` por defecto. Sin un mecanismo adicional, la variable seguiría definida y el sistema arrancaría en modo VFIO en cada reinicio.

Para evitarlo se crea un servicio de un solo uso (*oneshot*) que borra `next_entry` en cada arranque:

```bash
nano /etc/systemd/system/grub-clear-next-entry.service
```

```ini
[Unit]
Description=Limpiar next_entry de GRUB para forzar retorno al perfil por defecto
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/bin/grub-editenv /boot/grub/grubenv unset next_entry
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now grub-clear-next-entry.service
```

> **Limitación**: el servicio solo se ejecuta si el arranque llega a montar los sistemas de ficheros. Si el modo VFIO se bloquea antes, `next_entry` sigue definida y el siguiente reinicio vuelve a entrar en modo VFIO. En ese caso hay que elegir la entrada LXC a mano en el menú de GRUB, para lo que hace falta acceso físico o una consola remota.

**Comprobación**: tras arrancar con `boot-vfio`, un reinicio normal (`reboot`) debe volver al modo LXC.

---

## 7. Uso

1. **Uso habitual (contenedores LXC)**:
   - El sistema arranca por defecto en modo LXC.
   - `check-mode` debe mostrar `[ Default / LXC Host Drivers ]`.
2. **Cambio a máquinas virtuales (VFIO)**:
   - Detener los contenedores y máquinas virtuales en ejecución.
   - Ejecutar `boot-vfio`. El sistema se reinicia en modo VFIO.
   - Comprobar con `check-mode` el modo activo y el driver de cada GPU.
3. **Vuelta al modo LXC**:
   - Basta con un reinicio normal (`reboot`). El servicio de la sección 6 garantiza que el sistema arranque de nuevo en modo LXC.
