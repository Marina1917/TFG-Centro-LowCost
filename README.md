# Sistema de Computación Paralelo Low-cost

Trabajo Fin de Grado · Grado en Ingeniería de Computadores · Universidad de Sevilla

**Autora:** Marina Delgado Trinidade
**Tutor:** D. Fernando Sancho Caparrini
**Curso:** [2025/2026]

## De qué trata

¿Es viable montar un sistema de cómputo multi-GPU reaprovechando tarjetas gráficas antiguas y de distinta generación? El trabajo responde a esta pregunta con un caso real: dos GPUs NVIDIA de 2008–2009 (Tesla C1060 y GeForce GTS 250) sobre una plataforma actual (AMD Ryzen 9 7950X, placa X670E) con Proxmox VE.

Sin limitarnos a la parte empírica, se propone a su vez para ello un **marco de viabilidad** con cinco filtros, que se aplica después al caso de estudio:

- **Soporte del driver**: el driver puede compilarse y cargarse en un kernel actual.
- **Homogeneidad del driver y aislamiento**: las GPUs comparten driver o requieren aislamiento en máquinas virtuales.
- **Perfil de carga**: la carga está limitada por el cálculo y no por el bus PCIe.
- **Coste**: el despliegue compensa en energía, horas de trabajo y riesgo.
- **Estabilidad**: los fallos quedan contenidos en el dispositivo.

## Resultados principales

- El driver 340.108, sin soporte oficial desde hace años, funciona en el kernel 6.14 gracias a parches de la comunidad y DKMS.
- El aislamiento mediante máquinas virtuales (VFIO) no es viable con estas tarjetas: no admiten un reinicio por software (FLR).
- Con contenedores LXC, ambas GPUs funcionan a la vez con CUDA 6.5 en pruebas breves.
- Bajo carga sostenida en las dos GPUs, el host completo se reinició, por lo que el sistema no supera el filtro de estabilidad.
- Las conclusiones aplican el marco en sentido inverso para proponer una **configuración mínima realista**: GPUs Pascal (viable, con fecha de caducidad) o Turing y posteriores (sostenible), validando el reinicio y la estabilidad de cada tarjeta.

## Contenido del repositorio

| Ruta | Contenido |
|---|---|
| [`MemoriaTFG.pdf`](MemoriaTFG.pdf) | Memoria completa del trabajo |
| [`scripts/`](scriptsTFG/) | Guías técnicas paso a paso: instalación del driver en el host, contenedor LXC con CUDA 6.5 y arranque dual LXC/VFIO |

Las guías recogen las órdenes completas de cada fase, los problemas encontrados y su solución. Su [README](scriptsTFG/README.md) detalla la configuración de referencia.

## Advertencia

Las guías describen un entorno experimental con hardware de más de quince años y un driver sin soporte oficial. El sistema no se recomienda para servicio continuo. Revisa los comandos antes de aplicarlos en otro equipo: están pensados para ejecutarse como `root`.
