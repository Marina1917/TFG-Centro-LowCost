# Configuración del contenedor LXC con acceso a GPU (Tesla C1060 + ASUS ENGTS250)

## 1. Objetivo

Este documento recoge la creación de un contenedor LXC en Proxmox VE 9 con acceso directo a las dos GPUs NVIDIA del host y la instalación de CUDA 6.5 dentro del contenedor para validar el cómputo. Incluye también las pruebas de rendimiento y de estabilidad realizadas.

Se parte de que el driver **340.108** ya está instalado y funcionando en el **host** mediante DKMS, con `nvidia-smi` detectando ambas tarjetas (ver [`01-instalacion-driver-host.md`](01-instalacion-driver-host.md)).

> **Nota sobre el identificador del contenedor**: los comandos de este documento usan el VMID 200, con el que se creó el contenedor. El contenedor definitivo es el **202**, por el motivo que se explica en la sección 14.

---

## 2. Selección de almacenamiento

Antes de crear el contenedor se comprobó el almacenamiento disponible en Proxmox:

```bash
pvesm status
```

| Storage | Tipo | Uso recomendado |
|---|---|---|
| `local` | `dir` | Plantillas (`vztmpl`), ISOs, copias de seguridad |
| `local-lvm` | `lvmthin` | Discos de VMs y sistema de ficheros raíz de contenedores |

**Decisión**: `local-lvm` para el sistema de ficheros raíz del contenedor y `local` solo como origen de la plantilla.

---

## 3. Obtención de la plantilla del sistema

```bash
pveam update
pveam available --section system | grep debian-12
```

**Nota**: el nombre exacto de la plantilla (el número de *build* en `debian-12-standard_12.X-1_amd64.tar.zst`) cambia con el tiempo según lo que Proxmox tenga indexado. Debe consultarse siempre con `pveam available` antes de descargarla, en lugar de suponer una versión fija.

```bash
pveam download local <nombre-exacto-de-la-plantilla>
```

---

## 4. Creación del contenedor

```bash
pct create 200 local:vztmpl/<plantilla>.tar.zst \
  --hostname gpu-tfg \
  --cores 4 \
  --memory 4096 \
  --rootfs local-lvm:32 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0 \
  --features nesting=1
```

**Decisiones de diseño**:

- **`--unprivileged 0` (contenedor privilegiado)**: los contenedores no privilegiados remapean los identificadores de usuario y grupo (`idmap`) entre el contenedor y el host, lo que complica el acceso a los dispositivos de carácter de NVIDIA (habría que mapear también los grupos propietarios de `/dev/nvidia*`). Un contenedor privilegiado es la opción más sencilla, a costa de un aislamiento de seguridad menor, aceptable en un entorno experimental.
- **`--features nesting=1`**: permite anidar otros entornos (Docker, otro LXC) dentro del contenedor si se necesitan más adelante.

---

## 5. Identificación de los dispositivos NVIDIA en el host

```bash
ls -la /dev/nvidia*
cat /proc/devices | grep -i nvidia
```

Resultado relevante:
```
195 nvidia-frontend
```

El número mayor **195** corresponde al *frontend* del driver NVIDIA, que expone `/dev/nvidia0`, `/dev/nvidia1` y `/dev/nvidiactl`.

**Nota sobre UVM**: no aparece ningún número mayor para `nvidia-uvm` porque el módulo `nvidia-uvm.ko` **no se compiló**, de forma deliberada: estas GPUs no admiten direccionamiento unificado (ver §10). Por eso la configuración del contenedor no incluye los dispositivos de UVM.

Correspondencia de las GPUs, confirmada con `nvidia-smi` en el host:

| Dispositivo | GPU | Bus PCI |
|---|---|---|
| `/dev/nvidia0` | Tesla C1060 | `0000:01:00.0` |
| `/dev/nvidia1` | GeForce GTS 250 (ENGTS250) | `0000:08:00.0` |
| `/dev/nvidiactl` | (control, compartido) | — |

---

## 6. Configuración del paso de dispositivos (`/etc/pve/lxc/200.conf`)

```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia1 dev/nvidia1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
```

Esta configuración da acceso a **ambas GPUs a la vez** desde el mismo contenedor. Para asignar una GPU a cada contenedor bastaría con omitir el `lxc.mount.entry` de la GPU no deseada en el `.conf` de cada uno.

**Comprobación previa al arranque**: los ficheros de dispositivo deben existir en el host. Si no existen, se crean ejecutando `nvidia-smi` una vez antes de arrancar el contenedor:
```bash
nvidia-smi >/dev/null
ls -la /dev/nvidia0 /dev/nvidia1 /dev/nvidiactl
```

---

## 7. Arranque y verificación del paso de dispositivos

```bash
pct start 200
pct enter 200
ls -la /dev/nvidia*
```

Los tres dispositivos (`nvidia0`, `nvidia1` y `nvidiactl`) aparecen dentro del contenedor con el mismo número mayor, 195.

---

## 8. Componentes de espacio de usuario del driver dentro del contenedor

### 8.1 Por qué no hacen falta los parches

A diferencia del **módulo del kernel** (`nvidia.ko`), que necesitó los parches descritos en el documento del host, los **componentes de espacio de usuario** (librerías como `libcuda`, `nvidia-smi`, etc.) son binarios precompilados por NVIDIA y **no dependen de esos parches**. Dentro del contenedor se usa el instalador `.run` oficial sin modificar, con la opción que omite la instalación del módulo del kernel (que aporta el host).

### 8.2 Procedimiento

```bash
apt update
apt install -y wget build-essential

cd ~
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/340.108/NVIDIA-Linux-x86_64-340.108.run
chmod +x NVIDIA-Linux-x86_64-340.108.run
./NVIDIA-Linux-x86_64-340.108.run --no-kernel-module --no-x-check
```

Durante la instalación se rechazaron las mismas opciones que en el host, por el mismo motivo (no hay entorno gráfico):
- *Librerías de compatibilidad de 32 bits* → **"Do not install compatibility libraries"**
- *`nvidia-xconfig` / configuración de X11* → **"No"**

### 8.3 Verificación

```bash
nvidia-smi
```

Ambas GPUs son visibles desde el contenedor, con la misma versión del driver (340.108) que el host. Esto confirma que el módulo del kernel del host y las librerías del contenedor son compatibles.

---

## 9. Instalación de CUDA Toolkit 6.5 dentro del contenedor

### 9.1 Elección de la versión

*Compute capability* de las GPUs:

| GPU | *Compute capability* |
|---|---|
| Tesla C1060 | 1.3 (sm_13) |
| GeForce GTS 250 | 1.1 (sm_11) |

CUDA Toolkit **6.5** es la última versión con soporte para *compute capabilities* inferiores a 2.0 (microarquitectura Tesla). Las versiones posteriores eliminan ese soporte.

### 9.2 Descarga

```bash
cd ~
wget https://developer.download.nvidia.com/compute/cuda/6_5/rel/installers/cuda_6.5.14_linux_64.run
chmod +x cuda_6.5.14_linux_64.run
```

### 9.3 Primer obstáculo: compilador no soportado

**Error**:
```
Error: unsupported compiler: 12.2.0. Use --override to override this check.
```

**Causa**: CUDA Toolkit 6.5 (2014) declara soporte oficial hasta gcc 4.8. El contenedor, con Debian 12, tiene gcc 12.2.0 por defecto.

**Solución**: usar la opción `--override` del instalador para omitir la comprobación.

### 9.4 Segundo obstáculo: fallo silencioso del instalador (`--silent`)

Ejecutar el instalador en modo silencioso con `--override` terminó sin errores visibles, pero el *toolkit* **no quedó instalado** (`/usr/local/cuda-6.5/bin/` vacío salvo el script de desinstalación). El registro (`/tmp/cuda_install_*.log`) mostró la causa real:

```
Can't locate InstallUtils.pm in @INC (...) at ./install-linux.pl line 6.
BEGIN failed--compilation aborted at ./install-linux.pl line 6.
```

**Diagnóstico**: el `.run` de CUDA 6.5 es un **meta-instalador** que contiene tres instaladores independientes (driver, *toolkit* y ejemplos), cada uno autoextraíble (formato *makeself*). El script Perl `install-linux.pl` del instalador del *toolkit* no encontraba su propio módulo auxiliar `InstallUtils.pm` al ejecutarse mediante la extracción y ejecución automáticas encadenadas: se extrae en un directorio temporal que se borra al terminar, y Perl no incluye ese directorio en su ruta de búsqueda de módulos (`@INC`).

### 9.5 Solución: extracción manual en dos niveles

**Nivel 1: extraer los instaladores del meta-instalador**:
```bash
mkdir -p ~/cuda-extract
./cuda_6.5.14_linux_64.run --extract=$HOME/cuda-extract
```
Resultado:
```
cuda-linux64-rel-6.5.14-18749181.run   (toolkit)
cuda-samples-linux-6.5.14-18745345.run (ejemplos)
NVIDIA-Linux-x86_64-340.29.run         (driver incluido; no se usa, ya está el 340.108)
```

**Nivel 2: extraer el instalador del *toolkit* a una ruta fija**, con las opciones propias de *makeself*:
```bash
cd ~/cuda-extract
chmod +x cuda-linux64-rel-6.5.14-18749181.run
./cuda-linux64-rel-6.5.14-18749181.run --keep --target ~/cuda-toolkit-extracted --noexec
```

`--noexec` evita la ejecución automática tras la extracción, y `--keep --target` deja el contenido en una ruta fija en lugar de un directorio temporal. Así se puede comprobar que `InstallUtils.pm` está presente:
```bash
find ~/cuda-toolkit-extracted -iname "InstallUtils.pm"
```

**Ejecución manual del script Perl, indicando la ruta de sus módulos**:
```bash
cd ~/cuda-toolkit-extracted
perl -I. ./install-linux.pl -noprompt -prefix=/usr/local/cuda-6.5
```

La opción `-I.` de Perl añade el directorio actual a `@INC`, de modo que el script encuentra su módulo auxiliar.

### 9.6 Instalación de los ejemplos (*samples*)

El instalador de los ejemplos tiene la misma estructura que el del *toolkit*:

```bash
cd ~/cuda-extract
chmod +x cuda-samples-linux-6.5.14-18745345.run
./cuda-samples-linux-6.5.14-18745345.run --keep --target ~/cuda-samples-extracted --noexec
ls ~/cuda-samples-extracted/
```

Contenido: `cuda-install-samples-6.5.sh`, `install-sdk-linux.pl`, `InstallUtils.pm`, `cuda-samples/` (código fuente) y `uninstall_cuda.pl`.

**Intento con el script auxiliar** (`cuda-install-samples-6.5.sh`): falló porque espera una estructura de directorios distinta de la real del paquete (`../samples/*` en lugar de `./cuda-samples/*`):
```bash
./cuda-install-samples-6.5.sh $HOME/NVIDIA_CUDA-6.5_Samples
# cp: cannot stat './../samples/*': No such file or directory
```

**Solución adoptada: copia manual**, ya que el paso consiste únicamente en copiar un árbol de directorios:
```bash
mkdir -p ~/NVIDIA_CUDA-6.5_Samples
cp -r ~/cuda-samples-extracted/cuda-samples/* ~/NVIDIA_CUDA-6.5_Samples/
```

### 9.7 Configuración del entorno

```bash
echo 'export PATH=/usr/local/cuda-6.5/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-6.5/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
nvcc --version
```

Resultado: `nvcc` operativo, con `release 6.5, V6.5.14`.

---

## 10. Validación del cómputo: `deviceQuery`

A diferencia de `nvidia-smi`, que solo confirma que el hardware se *detecta*, el ejemplo `deviceQuery` de CUDA se ejecuta contra el *runtime* de CUDA. Es, por tanto, la prueba de que el cómputo en GPU funciona de extremo a extremo dentro del contenedor.

```bash
find ~/NVIDIA_CUDA-6.5_Samples -iname "deviceQuery" -type d
cd <ruta-encontrada>/1_Utilities/deviceQuery
make
./deviceQuery
```

### Resultado

```
Detected 2 CUDA Capable device(s)
deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 6.5, CUDA Runtime Version = 6.5,
NumDevs = 2, Device0 = Tesla C1060, Device1 = GeForce GTS 250
Result = PASS
```

| Especificación | Tesla C1060 | GeForce GTS 250 |
|---|---|---|
| *Compute capability* | 1.3 | 1.1 |
| Multiprocesadores | 30 | 16 |
| Núcleos CUDA (8 por multiprocesador) | 240 | 128 |
| Memoria global | 4096 MB | 1024 MB |
| Ancho del bus de memoria | 512 bits | 256 bits |
| Reloj de la GPU | 1296 MHz | 1620 MHz |
| Direccionamiento unificado (UVA) | No | No |
| ECC | Desactivado | Desactivado |

Ambas GPUs indican `Device supports Unified Addressing (UVA): No`, lo que confirma con datos reales la decisión de no compilar el módulo `nvidia-uvm.ko` en el host (§5): el hardware no admite esa característica, independientemente del software instalado.

Con `Result = PASS` queda validado el cómputo CUDA de extremo a extremo dentro del contenedor LXC.

---

## 11. Resumen de causas raíz: contenedor y CUDA

| # | Problema | Causa raíz | Categoría |
|---|---|---|---|
| 1 | Plantilla no encontrada | El nombre de la plantilla en el índice de Proxmox cambia con el tiempo | Documentación desactualizada |
| 2 | Compilador no soportado (instalador de CUDA) | CUDA 6.5 (2014) declara soporte hasta gcc 4.8; el sistema tiene gcc 12.2 | Obsolescencia del software |
| 3 | Instalación silenciosa fallida sin aviso claro | El modo `--silent` oculta el fallo real de un instalador anidado | Instalador poco transparente |
| 4 | `InstallUtils.pm` no localizado | Ruta de módulos de Perl (`@INC`) incorrecta en la extracción automática encadenada de *makeself* | Fragilidad del empaquetado de 2014 |
| 5 | El script auxiliar de los ejemplos falla | Espera una estructura de directorios (`../samples/`) distinta de la del paquete (`./cuda-samples/`) | Inconsistencia interna del paquete |
| 6 | `nvcc` no compila con los *headers* actuales | El analizador C++ de `nvcc` 6.5, anterior a C++11, frente a las cabeceras de Debian 12 (ver §13.1) | Obsolescencia del compilador |

**Conclusión**: igual que con el driver del kernel, instalar software de NVIDIA/CUDA de hace más de diez años en un sistema actual exige **extraer y ejecutar los instaladores manualmente, por capas**, en lugar de usar sus flujos automáticos o silenciosos, que fallan de forma poco informativa ante entornos que sus autores no podían prever.

---

## 12. Estado final del sistema

- [x] Contenedor LXC privilegiado con Debian 12 sobre `local-lvm` (VMID 202, ver §14).
- [x] Dispositivos `/dev/nvidia0`, `/dev/nvidia1` y `/dev/nvidiactl` accesibles mediante `lxc.mount.entry` y `lxc.cgroup2.devices.allow`.
- [x] Componentes de espacio de usuario del driver 340.108 instalados en el contenedor (sin módulo del kernel, que aporta el host).
- [x] `nvidia-smi` operativo dentro del contenedor, detectando ambas GPUs.
- [x] CUDA Toolkit 6.5.14 instalado en `/usr/local/cuda-6.5`, con `nvcc` operativo.
- [x] Ejemplos de CUDA 6.5.14 copiados en `~/NVIDIA_CUDA-6.5_Samples`.
- [x] `deviceQuery` compilado y ejecutado con éxito (`Result = PASS`) en ambas GPUs.

---

## 13. Pruebas de rendimiento y estabilidad

### 13.1 Obstáculo: `nvcc` 6.5 frente a las cabeceras C++ actuales

A diferencia de `deviceQuery` (un fichero `.cpp` que compila directamente el `g++` del sistema), `bandwidthTest` y `matrixMul` son ficheros `.cu` con código para la GPU. Estos pasan por el **analizador C++ propio de `nvcc`**, anterior a C++11, que resultó incompatible con las cabeceras de glibc y libstdc++ del contenedor (Debian 12, gcc 12.2):

```
error: identifier "nullptr" is undefined
error: variable "std::constexpr" is not a type name
error: this declaration may not have extern "C" linkage
```

**Diagnóstico**: desactivar el `#error` de comprobación de la versión de gcc en `host_config.h` solo elimina el aviso; no resuelve la incompatibilidad real entre `nvcc` y unas cabeceras que ya asumen C++11.

### 13.2 Solución: *chroot* con Debian 8 (gcc 4.9.2) solo para compilar

Se creó un entorno mínimo de Debian 8 (Jessie, 2015) con `debootstrap`, usado únicamente para compilar. Los binarios resultantes se ejecutan directamente en el contenedor principal.

```bash
apt install -y debootstrap
debootstrap --no-check-gpg --arch=amd64 jessie /root/old-chroot http://archive.debian.org/debian/

cat > /root/old-chroot/etc/apt/sources.list << 'EOF'
deb http://archive.debian.org/debian/ jessie main
deb http://archive.debian.org/debian-security/ jessie/updates main
EOF

chroot /root/old-chroot /bin/bash -c "apt-get update"
chroot /root/old-chroot /bin/bash -c "apt-get install -y --force-yes build-essential"
chroot /root/old-chroot /bin/bash -c "gcc --version"   # gcc 4.9.2
```

> `--no-check-gpg` y `--force-yes` son necesarios porque las claves de firma de Debian 8 ya han caducado. Solo deben usarse con los repositorios archivados oficiales (`archive.debian.org`).

**Montajes necesarios** para que el *chroot* vea el *toolkit* de CUDA y el código fuente del contenedor:

```bash
mount --bind /proc /root/old-chroot/proc
mount --bind /dev /root/old-chroot/dev
mount --bind /sys /root/old-chroot/sys
mkdir -p /root/old-chroot/usr/local/cuda-6.5
mount --bind /usr/local/cuda-6.5 /root/old-chroot/usr/local/cuda-6.5
mkdir -p /root/old-chroot/root/samples
mount --bind /root/NVIDIA_CUDA-6.5_Samples /root/old-chroot/root/samples
```

**Compilación**, invocada de forma no interactiva con `chroot ... -c "..."`. Si se pega una sesión interactiva del *chroot* junto con más órdenes y un `exit` final, puede cerrarse también la sesión del contenedor y no solo la del *chroot*:

```bash
chroot /root/old-chroot /bin/bash -c "cd /root/samples/1_Utilities/bandwidthTest && \
  export PATH=/usr/local/cuda-6.5/bin:\$PATH && \
  nvcc -ccbin g++ -I../../common/inc -m64 \
    -gencode arch=compute_11,code=sm_11 \
    -gencode arch=compute_13,code=sm_13 \
    -o bandwidthTest.o -c bandwidthTest.cu"

chroot /root/old-chroot /bin/bash -c "cd /root/samples/1_Utilities/bandwidthTest && \
  export PATH=/usr/local/cuda-6.5/bin:\$PATH && \
  nvcc -ccbin g++ -m64 \
    -gencode arch=compute_11,code=sm_11 \
    -gencode arch=compute_13,code=sm_13 \
    -o bandwidthTest bandwidthTest.o"
```

La compilación terminó sin errores, solo con avisos de obsolescencia por usar `sm_11` y `sm_13`. El mismo procedimiento se repitió para `matrixMul`.

**Arquitecturas de destino**: `-gencode` se limitó a las arquitecturas reales del hardware (`sm_11` para la GTS 250 y `sm_13` para la Tesla C1060), en lugar de las 6-7 que genera por defecto el Makefile original (hasta `sm_50`), que no aportan nada con este hardware.

**Ejecución**: los binarios compilados en el *chroot* (con glibc y libstdc++ de Debian 8) se ejecutaron sin problemas en el contenedor principal (Debian 12). Esto confirma que la incompatibilidad se limitaba al analizador de `nvcc` y no al binario final.

### 13.3 Resultados

**matrixMul** (rendimiento de cómputo):

| GPU | Rendimiento | Tiempo por iteración | Resultado |
|---|---|---|---|
| Tesla C1060 | 155.26 GFLOP/s | 0.106 ms | PASS |
| GeForce GTS 250 | 80.49 GFLOP/s | 0.204 ms | PASS |

**bandwidthTest** (modo rápido, memoria *pinned*):

| Transferencia | Tesla C1060 | GeForce GTS 250 |
|---|---|---|
| Host → Device | 1603.3 MB/s | 778.0 MB/s |
| Device → Host | 1635.6 MB/s | 817.6 MB/s |
| Device → Device | 75 661.9 MB/s | 54 603.4 MB/s |

**Cocientes Tesla C1060 / GTS 250**:

| Métrica | Cociente |
|---|---|
| Cómputo (GFLOP/s) | ~1.93× |
| Host → Device | ~2.06× |
| Device → Host | ~2.00× |
| Device → Device | ~1.39× |
| Núcleos CUDA | 1.875× (240 frente a 128) |
| Memoria | 4× (4096 MB frente a 1024 MB) |

**Interpretación** (analizada en detalle en el capítulo de resultados de la memoria):
- **Host ↔ Device**: la diferencia de unas 2× se debe a la **ranura**, no a la tarjeta. La Tesla negoció un enlace PCIe Gen 1 x8 en la ranura principal y la GTS 250 un enlace Gen 1 x4 en la secundaria. Con la Tesla en la ranura secundaria se obtienen prácticamente los mismos valores que con la GTS 250 (777.5 y 815.0 MB/s).
- **Device → Device**: depende de la memoria de cada tarjeta, no del bus. La Tesla tiene el doble de ancho de bus, pero la memoria de la GTS 250 funciona a mayor frecuencia (1000 MHz frente a 800 MHz). Por eso el cociente teórico es de 1.6× (102.4 frente a 64.0 GB/s) y no de 2×. El medido es algo menor porque la GTS 250 aprovecha mejor su memoria (≈85 % de su máximo frente a ≈74 %).

### 13.4 Ejecución concurrente en ambas GPUs

Se lanzaron `matrixMul --device=0` y `--device=1` a la vez, en segundo plano, para comprobar si compartir contenedor y bus PCIe provoca contención.

**Rendimiento con ejecución concurrente** (prácticamente idéntico al de cada GPU por separado):
- Tesla C1060: ~155.2-155.3 GFLOP/s (155.26 GFLOP/s por separado)
- GTS 250: ~81.3 GFLOP/s (80.49 GFLOP/s por separado)

**Conclusión**: no se observa contención apreciable entre las dos GPUs en ejecuciones cortas. Cada una tiene su propio enlace PCIe y, a esta escala de carga, no compiten por el ancho de banda.

#### Condición de carrera en el primer lanzamiento concurrente

En el primer lanzamiento simultáneo, sin espera entre procesos, falló el proceso de la GPU 1. En la primera reproducción el error fue:
```
cudaGetDevice returned error code 38, line(396)
cudaGetDeviceProperties returned error code 38, line(409)
cudaMalloc d_A returned error code 38, line(164)
```
(código 38 = `cudaErrorNoDevice`). En una reproducción posterior del mismo escenario (primer lanzamiento concurrente tras un reinicio limpio), el código fue distinto:
```
cudaGetDevice returned error code 10, line(396)
cudaGetDeviceProperties returned error code 10, line(409)
cudaMalloc d_A returned error code 10, line(164)
```
(código 10 = `cudaErrorInvalidDevice`). En ambos casos la GPU 0 terminó correctamente. Que el código de error cambie entre reproducciones del mismo escenario apunta a una condición de carrera en el registro interno de dispositivos del driver, y no a un fallo determinista. El registro del kernel (`dmesg`) mostraba, en el momento del fallo:

```
WARNING: Flushing system-wide workqueues will be prohibited in near future.
os_flush_work_queue+0x4a/0x70 [nvidia]
rm_disable_adapter+0x7b/0x110 [nvidia]
nvidia_close+0x461/0x4d0 [nvidia]
```

**Diagnóstico**: al cerrar el contexto de una GPU (`nvidia_close`), el driver 340.108 vacía la **cola de trabajo global del sistema** en lugar de una cola propia del dispositivo, una práctica que el kernel actual ya señala como obsoleta. Si el cierre de un proceso coincide con la inicialización de otro proceso en la segunda GPU, ese vaciado global puede interferir con la inicialización.

**Reproducibilidad**: se repitió el lanzamiento concurrente sin espera:
- 5 repeticiones inmediatas tras el primer fallo → ningún fallo más.
- 15 repeticiones tras un reinicio limpio del contenedor (`pct reboot`) → **el fallo se reprodujo exactamente en el primer intento tras el reinicio** y no volvió a aparecer en los 14 restantes.

**Conclusión**: la condición de carrera aparece en el **primer** lanzamiento concurrente tras cada arranque del contenedor. Los lanzamientos concurrentes posteriores, dentro de la misma sesión del driver, son estables.

**Mitigación**: escalonar ligeramente (`sleep 2`) el primer lanzamiento concurrente tras cada arranque, o reintentar la ejecución si falla. El problema no reaparece una vez que el driver ha inicializado al menos un contexto en cada GPU.

### 13.5 Consumo eléctrico

Se comprobó si el consumo podía consultarse por *software*:

```bash
nvidia-smi -q -d POWER
```

**Resultado**: todos los campos (`Power Management`, `Power Draw`, `Power Limit`, etc.) devuelven `N/A` en ambas GPUs. Estas generaciones de chips (GT200 y G92b) carecen del subsistema de gestión de energía que `nvidia-smi` necesita para informar del consumo. No es un fallo de configuración del driver ni del contenedor, sino una limitación de estas tarjetas.

**Implicación**: un sistema con este hardware no puede informar de su propio consumo, y conocer su coste energético exige instrumentación externa.

**Protocolo de medición** (pendiente de ejecución, ya que requiere un vatímetro o enchufe inteligente):

1. Medir el consumo del servidor completo en reposo, sin carga en las GPUs, como línea base.
2. Generar carga sostenida en una GPU con `matrixMul` en bucle y matrices grandes, para que cada ejecución dure más:
   ```bash
   cd /root/NVIDIA_CUDA-6.5_Samples/0_Simple/matrixMul
   while true; do ./matrixMul --device=0 --wA=4096 --hA=4096 --wB=4096 --hB=4096 > /dev/null 2>&1; done &
   ```
3. Medir el consumo durante varios minutos de carga sostenida.
4. `consumo_bajo_carga − consumo_reposo` ≈ consumo atribuible a esa GPU.
5. Repetir con la segunda GPU y con ambas a la vez.
6. Detener el bucle: `kill %1`.

> **Precaución**: la carga sostenida en ambas GPUs provocó el reinicio del host (§13.6). Si se ejecuta este protocolo, conviene hacerlo con una sola GPU cada vez y vigilando el sistema.

### 13.6 Prueba de estabilidad bajo carga sostenida

Para comprobar el comportamiento con un uso prolongado, se lanzó `matrixMul` en bucle continuo en ambas GPUs a la vez, con matrices de 4096 × 4096, y se programó una monitorización de diez minutos que registraba cada diez segundos la temperatura y la utilización de cada tarjeta con `nvidia-smi`.

**Resultado: reinicio completo del host.** La prueba no llegó a completarse. Durante su ejecución se perdió la conexión con el sistema y, al recuperarla, tanto el host como el contenedor llevaban encendidos apenas siete minutos: se había reiniciado la máquina física entera. El registro del kernel del arranque anterior mostraba, en los minutos previos al reinicio, mensajes como:

```
watchdog: BUG: soft lockup - CPU#4 stuck for 231s! [kworker/4:4:124258]
Workqueue: pm pm_runtime_work
RIP: 0010:pci_mmcfg_read+0xcb/0x110
```

**Interpretación**: la tarea bloqueada pertenecía a la gestión de energía de los dispositivos PCI, que estaba leyendo el espacio de configuración de un dispositivo (`pci_mmcfg_read`). Esa lectura nunca terminó, y el valor leído era `0xffffffff`, que es lo que se obtiene al leer un dispositivo PCI que no responde. Otras trazas del mismo periodo muestran funciones del driver NVIDIA, lo que sitúa el bloqueo en plena actividad de las GPUs. Lo más probable es que, bajo carga sostenida, una de las GPUs dejara de responder. El registro no permite identificar cuál.

**Implicación**: a diferencia de la condición de carrera (§13.4), este fallo no quedó contenido en un proceso ni en una GPU: provocó el reinicio del host completo. El sistema funciona en ejecuciones cortas, pero **no es apto para carga continua**.

### 13.7 Uso y desarrollo

Para quien vaya a usar el sistema conviene distinguir tres casos:
- **Ejecutar programas ya compilados**: no requiere pasos adicionales. Basta con el driver 340.108 en el host y los componentes de espacio de usuario y el *runtime* de CUDA 6.5 en el contenedor.
- **Desarrollar código CUDA nuevo**: requiere el entorno de compilación del §13.2 (*chroot* con gcc 4.9), bastante más complejo.
- **Ejecución concurrente en ambas GPUs**: funciona sin pérdida de rendimiento en ejecuciones cortas, con la precaución de escalonar el primer lanzamiento tras cada arranque (§13.4). No se recomienda para cargas sostenidas, que provocaron el reinicio del host (§13.6).

---

## 14. Nota sobre la gestión del contenedor

Durante el proceso se crearon por error contenedores duplicados con el mismo VMID (200), al volver a ejecutar `pct create` sin advertir que el contenedor ya existía. El disco original (`vm-200-disk-0`, 32 GB, con toda la instalación funcional) quedó huérfano pero intacto en `local-lvm`. Se recuperó asignándolo a un nuevo VMID, el **202**, que es el contenedor definitivo a partir de ese momento.

**Lección**: antes de volver a ejecutar `pct create` con un VMID ya usado, conviene comprobar su estado con `pct list` y `pct config <vmid>`. Proxmox no impide crear un disco nuevo con un identificador existente si la orden cambia ligeramente (por ejemplo, en el tamaño de `--rootfs`), lo que puede dejar volúmenes huérfanos y generar confusión sobre cuál es el entorno activo.

**Además**: `hostnamectl set-hostname` debe ejecutarse siempre *dentro* del contenedor o VM cuyo nombre se quiere cambiar, nunca en el host de Proxmox. Cambiar el nombre del propio nodo rompe temporalmente las rutas de `pmxcfs` (`/etc/pve/nodes/<hostname>/...`) y deja inaccesibles las configuraciones de VMs y contenedores hasta revertirlo.
