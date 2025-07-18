---
title: "Parte 2: Virtualización anidada y el origen de Hyper-V vs KVM"
description: ""
date: "May 23, 2025"
weight: 2
---

Con el lanzamiento de [Windows Server 2025](https://techcommunity.microsoft.com/blog/windowsosplatform/introducing-gpu-innovations-with-windows-server-2025/4161879), Microsoft reforzó su apuesta por la virtualización moderna, incluyendo soporte oficial para GPU Partitioning (GPU-PV), migración en vivo de VMs con GPU y nuevas capacidades para clústeres sin Active Directory. Sin embargo, hay un área crítica que sigue sin resolverse: los contenedores con aislamiento Hyper-V (Hyper-V Windows Containers) aún no pueden usar GPU-PV.


# ¿Qué es GPU-PV?

GPU-PV permite compartir una GPU física entre múltiples máquinas virtuales, asignando fracciones de la GPU a cada una. Esta tecnología es clave para escenarios de alta densidad, como VDI, inferencia de modelos ML o renderizado paralelo.

Ya disponible en Windows 10/11 y Server 2022, GPU-PV se consolida en Server 2025 con mejoras como:

- Soporte oficial en entornos productivos.
- Integración con Windows Admin Center.
- Live migration de VMs con GPU activa.

# Contenedores Windows: dos modos de aislamiento

Windows permite ejecutar contenedores en dos modos:

1. Process Isolation: comparte el kernel del host.
2. Hyper-V Isolation: ejecuta el contenedor dentro de una mini-VM completamente aislada.

El segundo modo ofrece compatibilidad binaria entre versiones de Windows y mayor seguridad, pero también limita el acceso a recursos físicos.

# El problema actual: sin GPU en contenedores Hyper-V

Mientras que los contenedores en modo "process" pueden usar la GPU (siempre que coincidan con la versión del host y los drivers estén bien configurados), los contenedores con aislamiento Hyper-V no pueden acceder a la GPU, ni siquiera usando GPU-PV.

Esto se debe a:

- Falta de drivers compatibles dentro de las mini-VMs.
- Limitaciones del aislamiento de Hyper-V que impiden exponer /dev/dxg o dispositivos DirectX.
- Ausencia de soporte oficial de NVIDIA y Microsoft para contenedores Hyper-V + GPU.

# Lo que sí funciona: Easy-GPU-PV

El proyecto [Easy-GPU-PV](https://github.com/jamesstringerparsec/Easy-GPU-PV) permite usar GPU-PV en VMs Hyper-V comunes (no contenedores), incluso con GPUs de consumo como la 3080 Ti. Esto ha demostrado que la tecnología funciona, pero no ha sido trasladada a la arquitectura de los Hyper-V Containers.

# Windows Server 2025

Aunque Windows Server 2025 incluye mejoras en gestión de GPU-PV, no menciona nada sobre soporte para contenedores Hyper-V con GPU. Usuarios han manifestado esta preocupación, y todo indica que esta funcionalidad sigue pendiente o relegada a versiones futuras.

Un área crítica y poco documentada en el ecosistema Windows: la imposibilidad actual de usar GPU en contenedores con aislamiento Hyper-V, a pesar de los avances en GPU-PV.

Esto abre la puerta a:

- Trabajos de investigación sobre virtualización de dispositivos en entornos aislados.
- Proyectos open source que exploren drivers ligeros para mini-VMs.
- Demandas empresariales para incorporar esta capacidad en futuras builds.

El mundo Windows contenedorizado con GPU-PV aún no está completo.

_Fernando Martínez – Infraestructura, virtualización y sistemas distribuidos_

Ahora, hablemos de VMXON

[Implementación de VMX](https://web.archive.org/web/20250523204652/https://revers.engineering/day-2-entering-vmx-operation/)