# TFG-Centro-LowCost
# GPUs NVIDIA legacy en una plataforma AM5 con Proxmox VE

Material complementario del Trabajo Fin de Grado Sistema de Computación Paralelo Low-cost, Grado en Ingeniería de Computadores, Universidad de Sevilla.

El trabajo estudia la viabilidad de reutilizar dos GPUs NVIDIA de microarquitectura Tesla (una Tesla C1060 y una GeForce GTS 250, de 2008-2009) como aceleradores de cómputo en un host moderno con Proxmox VE 9.

## Contenido

| Documento | Descripción |
|---|---|
| [`01-instalacion-driver-host.md`](01-instalacion-driver-host.md) | Instalación del driver NVIDIA 340.108 en el host con los parches de la comunidad y DKMS |
| [`02-contenedor-lxc-cuda.md`](02-contenedor-lxc-cuda.md) | Creación del contenedor LXC, acceso a las GPUs, instalación de CUDA 6.5 y pruebas de rendimiento y estabilidad |
| [`03-arranque-dual-lxc-vfio.md`](03-arranque-dual-lxc-vfio.md) | Arranque dual del host entre el modo LXC (por defecto) y el modo VFIO, con los comandos `boot-vfio` y `check-mode` |

Cada documento recoge los problemas encontrados en el orden en que aparecieron, con su causa y su solución, y termina con un resumen de causas raíz. El documento del driver incluye además el procedimiento final sin la traza de errores (sección 4).

## Configuración de referencia

| Componente | Versión |
|---|---|
| Placa base | ASUS TUF GAMING X670E-PLUS WIFI |
| CPU | AMD Ryzen 9 7950X (su iGPU proporciona la salida de vídeo) |
| Hipervisor | Proxmox VE 9.2 |
| Kernel del host | 6.14.11-6-bpo12-pve |
| Driver NVIDIA | 340.108 + parches de [MichelBoucey/nvidia-340xx](https://github.com/MichelBoucey/nvidia-340xx) |
| Contenedor | LXC privilegiado con Debian 12 |
| CUDA | 6.5 (compilación en un *chroot* con Debian 8 y gcc 4.9) |

Es la única configuración validada en el trabajo. El apéndice A de la memoria indica qué componentes vienen impuestos por el hardware y cuáles podrían sustituirse.

## Advertencias

- El sistema funciona en ejecuciones cortas, pero **la carga sostenida en ambas GPUs provocó el reinicio completo del host** (sección 13.6 del documento del contenedor). No se recomienda para servicio continuo.
- El driver 340.108 no recibe soporte de NVIDIA. Su funcionamiento en kernels actuales depende de parches mantenidos por la comunidad.
- Los comandos están pensados para ejecutarse como `root` en un entorno experimental. Revísalos antes de aplicarlos en otro sistema.
