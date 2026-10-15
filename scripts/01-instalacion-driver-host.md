# Instalación del driver NVIDIA legacy 340.108 en Proxmox VE 9

Este documento recoge la instalación del driver NVIDIA 340.108 en el host Proxmox VE, necesaria para dar acceso a las GPUs desde contenedores LXC. La sección 3 documenta, en orden, los problemas encontrados y su solución; la sección 4 resume el procedimiento final, sin la traza de errores. La configuración del contenedor y de CUDA se describe en [`02-contenedor-lxc-cuda.md`](02-contenedor-lxc-cuda.md).

---

## 1. Contexto y objetivo

### 1.1 Motivación

El planteamiento inicial del trabajo era asignar dos GPUs NVIDIA de microarquitectura Tesla (una **ASUS ENGTS250** y una **NVIDIA Tesla C1060**) a máquinas virtuales QEMU/KVM mediante **VFIO passthrough**, sobre una placa base **ASUS TUF GAMING X670E-PLUS WIFI**, usando la iGPU del procesador como salida de vídeo.

El despliegue con VFIO resultó inviable: ambas tarjetas, por su antigüedad (2008-2009), presentan problemas de **FLR (Function Level Reset)** y su VBIOS carece de soporte UEFI (GOP), lo que impide un passthrough limpio y estable.

### 1.2 Cambio de enfoque

Se optó por **LXC (Linux Containers)** con paso de dispositivos, aprovechando que:

- LXC no requiere IOMMU/VFIO ni reiniciar el dispositivo, ya que el contenedor comparte el kernel del host.
- Ambas GPUs funcionan con la misma rama del driver de NVIDIA (**340.xx**), de modo que basta un único driver en el host, compartido con el contenedor mediante el montaje de sus ficheros de dispositivo.

### 1.3 Entorno de partida

| Componente | Valor |
|---|---|
| Hipervisor | Proxmox VE 9.2.0 |
| Kernel inicial | 7.0.2-6-pve |
| Kernel final utilizado | 6.14.11-6-bpo12-pve |
| Compilador inicial del sistema | gcc 14.2.0 |
| Compilador final del sistema | gcc 12.4.0 |
| GPU 1 | NVIDIA Tesla C1060 (GT200, *compute capability* 1.3) |
| GPU 2 | ASUS ENGTS250 (G92b, *compute capability* 1.1) |
| Driver | NVIDIA 340.108, última versión de la última rama con soporte para la microarquitectura Tesla |

---

## 2. Diagnóstico de la incompatibilidad

El driver 340.108 se publicó en **2019** y está diseñado para kernels Linux bastante más antiguos. El kernel de Proxmox VE 9 (2025-2026) presenta unos **7 años de evolución de la API interna del kernel** (gestión de memoria, `struct file_operations`, macros de compilación, `mm_struct`, etc.), por lo que el código fuente del driver no compila contra los *headers* actuales sin modificaciones.

**Nota comparativa**: en distribuciones más antiguas (por ejemplo, Ubuntu 18.04 con kernel 4.15), el paquete `nvidia-340` de los repositorios oficiales se instalaba sin problemas, porque la distribución mantenía binarios y parches adaptados. Para kernels tan recientes como los de este trabajo **ningún repositorio oficial mantiene el driver 340.xx**, por lo que hay que compilarlo manualmente o recurrir a parches mantenidos por la comunidad.

---

## 3. Problemas encontrados y su resolución

### 3.1 Configuración residual de VFIO

**Síntoma**: el driver NVIDIA no lograba tomar el control de las tarjetas en el host.

**Causa**: la configuración previa de VFIO reservaba las GPUs para el driver `vfio-pci` mediante:
```
options vfio-pci ids=10de:xxxx,10de:yyyy
```
en `/etc/modprobe.d/vfio.conf`, en conflicto con el nuevo enfoque (LXC necesita que el **host** cargue el driver NVIDIA).

**Solución**: eliminar o comentar las reservas de `vfio-pci` por identificador de dispositivo, comprobar que `blacklist-nouveau.conf` solo excluye `nouveau` (no `nvidia`) y regenerar el `initramfs`.

### 3.2 Headers del kernel no encontrados

**Error**:
```
ERROR: Unable to find the kernel source tree for the currently running kernel.
```

**Causa**: en Proxmox VE 9 el paquete de *headers* cambió de nombre, de `pve-headers-*` a `proxmox-headers-*`.

**Solución**:
```bash
apt install -y proxmox-headers-$(uname -r)
```

### 3.3 Kernel demasiado reciente (7.0.2)

El kernel por defecto de Proxmox VE 9.2 (7.0.2-6-pve) resultaba demasiado moderno para el driver. Se instaló y fijó como kernel de arranque una versión anterior, también soportada por Proxmox:

```bash
apt install -y proxmox-kernel-6.14.11-6-bpo12-pve proxmox-headers-6.14.11-6-bpo12-pve
proxmox-boot-tool kernel pin 6.14.11-6-bpo12-pve
reboot
```

### 3.4 Diferencia de versión de gcc (bloqueante en el instalador `.run`, no en `make`)

**Contexto**: el kernel se compiló con gcc 12.2.0, mientras que el sistema tenía instalado gcc 12.4.0 (misma versión principal, distinta revisión).

**Diagnóstico**: existen **dos comprobaciones distintas**:
1. La del instalador de NVIDIA (`.run`), que **bloquea** la instalación salvo que se indique expresamente que se ignore.
2. La del sistema de compilación del kernel (`kbuild`), que solo emite un **aviso no bloqueante** ante diferencias de revisión.

**Solución en la instalación con `.run`**: responder *"Ignore CC version check"* o usar la opción equivalente.

**Solución en la compilación con DKMS**: la propia comprobación del driver (`gcc-version-check.c`) admite la variable de entorno `IGNORE_CC_MISMATCH=1` (ver §3.9.3).

### 3.5 `stdarg.h` no encontrado (primer intento manual, sin parches)

**Error**:
```
os-interface.h:27:10: fatal error: stdarg.h: No such file or directory
```

**Causa**: al compilar con `-nostdinc`, la ruta de los *headers* propios del compilador (como `stdarg.h`, que pertenece a gcc y no al kernel) no se propagaba correctamente en un sistema con varias versiones de gcc instaladas a la vez (12 y 14).

**Solución provisional** (sustituida más adelante, ver §3.9):
```bash
EXTRA_CFLAGS="-I. -isystem $(gcc-12 -print-file-name=include)"
```

**Solución definitiva**: fijar gcc 12 como compilador por defecto del sistema con `update-alternatives`, lo que evita sobrescribir `EXTRA_CFLAGS` manualmente (algo que, como se vio en §3.7, interfiere con el mecanismo `conftest`).

### 3.6 `nvtypes.h` no encontrado

**Causa**: al pasar `EXTRA_CFLAGS` por línea de órdenes a `make`, se **sobrescribió** (en lugar de ampliarse) el valor que ya definía el Makefile de NVIDIA, que incluía `-I./.`, necesario para que el compilador encuentre los *headers* propios del driver.

**Solución**: añadir manualmente `-I.` al resto de opciones.

### 3.7 Tipos del driver no definidos (`nv_state_t`, `nv_stack_t`, etc.)

**Síntoma**: cientos de errores en cascada del tipo:
```
error: unknown type name 'nv_state_t'
error: request for member 'flags' in something not a structure or union
```

**Diagnóstico**: el driver emplea un mecanismo de detección llamado **`conftest`** (`conftest.sh`), que compila pequeños programas de prueba contra los *headers* del kernel de destino para detectar qué funciones y estructuras están disponibles, y genera las macros que el resto del código usa de forma condicional. Al invocar `make module` manualmente con `EXTRA_CFLAGS` sobrescritos, `conftest` no se ejecutó correctamente y esas macros y tipos quedaron sin definir.

**Conclusión**: compilar el driver contra un kernel tan reciente requiere, además de los *headers* correctos, un **código fuente parcheado** que actualice las llamadas a la API del kernel. Se abandonaron los ajustes manuales de opciones y se pasó a usar parches mantenidos por la comunidad.

### 3.8 Primer intento con parches de la comunidad (`dkosmari/nvidia-340.108-updated`)

Se probó el repositorio [dkosmari/nvidia-340.108-updated](https://github.com/dkosmari/nvidia-340.108-updated), que reúne todas las modificaciones en un único parche (`updated.patch`) sobre el código del `.run` oficial. **El parche se aplicó sin fallos**, pero al compilar manualmente con `make module` y `EXTRA_CFLAGS` forzados (el mismo problema de fondo que en §3.7), `conftest` volvió a fallar. Se descartó esta vía en favor de una compilación gestionada por DKMS.

### 3.9 Solución definitiva: `MichelBoucey/nvidia-340xx` + DKMS

Se adoptó el repositorio [MichelBoucey/nvidia-340xx](https://github.com/MichelBoucey/nvidia-340xx), derivado de `archlinux-jerry/nvidia-340xx` y mantenido activamente. Ofrece parches incrementales por versión del kernel (de la 5.7 a la 7.0) y parches de compatibilidad con gcc 14 y 15. La compilación se delegó en **DKMS**, que ejecuta el proceso tal como lo define el propio driver y, por tanto, deja que `conftest` funcione como se espera.

#### 3.9.1 Aplicación de los parches

El repositorio contiene 21 parches incrementales, que se aplican en orden:

| Parches | Destino |
|---|---|
| `0001` a `0016` | Kernels de la 5.7 a la 6.8 |
| `0017` y `0018` | gcc 14 y gcc 15 |
| `0019` a `0021` | Kernels 6.15, 6.19 y 7.0 |

En un primer momento se aplicaron los 21, lo que produjo errores: los parches `0019` a `0021` adaptan el código a cambios de kernels posteriores al 6.14 que el kernel instalado todavía no incluye. Se rectificó aplicando **solo los necesarios hasta la versión del kernel instalado**, es decir, del `0001` al `0018`.

Los parches deben aplicarse desde el **directorio padre** de `kernel/`, ya que sus rutas son del tipo `a/kernel/Makefile`:

```bash
cd NVIDIA-Linux-x86_64-340.108/   # directorio extraído del .run
for p in ~/nvidia-340xx/00{01..18}-*.patch; do
  patch -p1 < "$p"
done
```

Los 18 parches se aplicaron **sin ningún fallo**. Con otra versión del kernel habría que ajustar el rango a los parches que le correspondan.

#### 3.9.2 Registro en DKMS

```bash
mkdir -p /usr/src/nvidia-340.108
cp -r kernel/* /usr/src/nvidia-340.108/
```

El parche `0001-kernel-5.7.patch` ya deja preparado el fichero `dkms.conf` del driver (`PACKAGE_NAME`, `MAKE[0]`, `CLEAN`, etc.).

#### 3.9.3 Ajuste del `dkms.conf` para omitir la comprobación de gcc

Se añadió `IGNORE_CC_MISMATCH=1` a la línea `MAKE[0]` para que la variable llegue a la compilación gestionada por DKMS:

```bash
sed -i 's|MAKE\[0\]="make -j\$num_cpu_cores module KERNEL_UNAME=\${kernelver}"|MAKE[0]="IGNORE_CC_MISMATCH=1 make -j$num_cpu_cores module KERNEL_UNAME=${kernelver}"|' /usr/src/nvidia-340.108/dkms.conf
```

#### 3.9.4 Opción de compilación no reconocida

**Error**:
```
gcc: error: unrecognized command-line option '-Wunused-but-set-variable=3'
```

**Causa**: el parche `0018-gcc-15.patch` introduce la forma `-Wunused-but-set-variable=3` (con nivel numérico), que solo reconoce **gcc 15 o posterior**. El sistema usaba gcc 12.4, que solo admite la forma sin nivel.

**Solución**: sustituir la opción en los tres ficheros donde aparece:
```bash
sed -i 's/-Wunused-but-set-variable=3/-Wunused-but-set-variable/' \
  /usr/src/nvidia-340.108/conftest.sh \
  /usr/src/nvidia-340.108/uvm/conftest.sh \
  /usr/src/nvidia-340.108/nvidia-modules-common.mk
```

#### 3.9.5 Compilación e instalación

```bash
dkms remove -m nvidia -v 340.108 --all
dkms add -m nvidia -v 340.108
dkms build -m nvidia -v 340.108
dkms install -m nvidia -v 340.108
modprobe nvidia
```

#### 3.9.6 Componentes de espacio de usuario

Con el módulo del kernel gestionado por DKMS, se instaló el resto del driver (librerías, `nvidia-smi`) sin recompilar el módulo:

```bash
./NVIDIA-Linux-x86_64-340.108.run --no-kernel-module --no-x-check
```

Durante la instalación se rechazaron dos opciones innecesarias en un host sin entorno gráfico:
- **Librerías de compatibilidad de 32 bits**: no se ejecutan aplicaciones de 32 bits.
- **`nvidia-xconfig` / configuración de X11**: el host no tiene entorno gráfico.

#### 3.9.7 Verificación

```bash
nvidia-smi
```

Se confirmó la detección de ambas GPUs (Tesla C1060 y ASUS ENGTS250). Un reinicio posterior confirmó que DKMS carga el módulo correctamente en cada arranque.

---

## 4. Procedimiento final (sin la traza de errores)

```bash
# 1. Kernel y headers
apt update
apt install -y proxmox-kernel-6.14.11-6-bpo12-pve proxmox-headers-6.14.11-6-bpo12-pve
proxmox-boot-tool kernel pin 6.14.11-6-bpo12-pve
reboot   # arrancar ya con el kernel fijado

# 2. Confirmar el kernel activo
uname -r   # debe mostrar 6.14.11-6-bpo12-pve

# 3. Desactivar nouveau (si no se hizo ya en un intento VFIO previo)
echo "blacklist nouveau" > /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u -k all

# 4. Revertir cualquier configuración VFIO previa que reserve las GPUs
#    (eliminar o comentar las líneas "options vfio-pci ids=..." en /etc/modprobe.d/)

# 5. Dependencias de compilación
apt install -y build-essential git wget patch dkms gcc-12 cpp-12

# 6. Fijar gcc 12 como compilador por defecto (evita los problemas de -nostdinc / stdarg.h)
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 20
update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-12 20
update-alternatives --set gcc /usr/bin/gcc-12

# 7. Descargar y extraer el driver oficial
mkdir -p ~/nvidia-build && cd ~/nvidia-build
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/340.108/NVIDIA-Linux-x86_64-340.108.run
chmod +x NVIDIA-Linux-x86_64-340.108.run
./NVIDIA-Linux-x86_64-340.108.run --extract-only

# 8. Clonar los parches de compatibilidad
git clone https://github.com/MichelBoucey/nvidia-340xx.git ~/nvidia-340xx

# 9. Aplicar los parches 0001-0018 (hasta el kernel 6.14), desde el directorio PADRE de kernel/
cd NVIDIA-Linux-x86_64-340.108
for p in ~/nvidia-340xx/00{01..18}-*.patch; do
  patch -p1 < "$p"
done

# 10. Corregir la opción -Wunused-but-set-variable=3 (incompatible con gcc < 15)
sed -i 's/-Wunused-but-set-variable=3/-Wunused-but-set-variable/' \
  kernel/conftest.sh kernel/uvm/conftest.sh kernel/nvidia-modules-common.mk

# 11. Preparar el código fuente para DKMS
mkdir -p /usr/src/nvidia-340.108
cp -r kernel/* /usr/src/nvidia-340.108/

# 12. Añadir IGNORE_CC_MISMATCH al dkms.conf
sed -i 's|MAKE\[0\]="make -j\$num_cpu_cores module KERNEL_UNAME=\${kernelver}"|MAKE[0]="IGNORE_CC_MISMATCH=1 make -j$num_cpu_cores module KERNEL_UNAME=${kernelver}"|' \
  /usr/src/nvidia-340.108/dkms.conf

# 13. Compilar e instalar con DKMS
dkms add -m nvidia -v 340.108
dkms build -m nvidia -v 340.108
dkms install -m nvidia -v 340.108
modprobe nvidia

# 14. Instalar los componentes de espacio de usuario (sin recompilar el módulo)
cd ~/nvidia-build
./NVIDIA-Linux-x86_64-340.108.run --no-kernel-module --no-x-check
#   -> rechazar las librerías de compatibilidad de 32 bits
#   -> rechazar la configuración de nvidia-xconfig / X11

# 15. Verificación
nvidia-smi
```

Este procedimiento puede servir de base para un script de instalación automatizada, cuyo desarrollo y validación quedan como trabajo futuro.

---

## 5. Resumen de causas raíz

| # | Problema | Causa raíz | Categoría |
|---|---|---|---|
| 1 | Conflicto de configuración | Restos de la configuración VFIO que reservaban las GPUs | Configuración del host |
| 2 | *Headers* no encontrados | Cambio de nombre de los paquetes en Proxmox VE 9 (`pve-*` → `proxmox-*`) | Empaquetado / documentación desactualizada |
| 3 | Kernel demasiado nuevo | Unos 7 años de diferencia entre el driver (2019) y el kernel por defecto (2025-2026) | Obsolescencia del driver |
| 4 | Diferencia de versión de gcc | Distinta revisión de gcc entre el kernel y el sistema | Comprobaciones estrictas del instalador de NVIDIA |
| 5-6 | *Headers* del compilador no encontrados | Interacción entre `-nostdinc` y varias versiones de gcc instaladas | Configuración del compilador |
| 7 | Tipos del driver no definidos | Fallo de `conftest` al sobrescribir manualmente las opciones de compilación | Metodología de compilación (manual frente a gestionada) |
| 8-9 | Necesidad de parches de terceros | API del kernel muy distinta de la que espera el código de 2019 | Falta de mantenimiento oficial |
| 10 | Opción de compilación no reconocida | Parche pensado para gcc 15 aplicado con gcc 12.4 | Desfase entre el parche y el compilador del sistema |

**Conclusión**: que este hardware funcione con un kernel y un compilador de 2025-2026 depende por completo de parches mantenidos por la comunidad (Arch Linux / AUR), ya que NVIDIA no ofrece soporte oficial para esta combinación. La reproducibilidad del proceso depende, por tanto, de que esos parches sigan disponibles.
