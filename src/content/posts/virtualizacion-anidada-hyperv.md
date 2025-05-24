---
title: "Por qué VMware y Hyper-V no pueden coexistir cuando se requiere nested virtualization"
description: ""
date: "May 23, 2025"
---

# Por qué VMware y Hyper-V no pueden coexistir cuando se requiere nested virtualization

## Introducción

La virtualización anidada (nested virtualization) permite ejecutar un hipervisor dentro de una máquina virtual. Este tipo de arquitectura es fundamental para probar infraestructuras, laboratorios de Kubernetes, entornos de CI/CD e*n cloud, entre* otros. Sin embargo, en sistemas Windows modernos, surge un conflicto estructural: **VMware y Hyper-V no pueden coexistir** cuando se requiere acceso a extensiones de virtualización como Intel VT-x. Este artículo explora, con precisión técnica, por qué sucede esto, referenciando la arquitectura de Intel, papers como *Turtles* y *NestCloud*, y un ejemplo real de uso con Bareflank y la instrucción `VMXON`.

## Intel VT-x: el corazón de la virtualización por hardware

Intel proporciona a través del conjunto de instrucciones **VMX** (Virtual Machine Extensions) la capacidad de ejecutar código de hipervisores directamente sobre el hardware. Estas instrucciones están documentadas en el manual oficial: **Intel 64 and IA-32 Architectures Software Developer's Manual, Volumen 3C**.

La inicialización de virtualización se realiza con:

- Activación del bit `CR4.VMXE`
- Preparación de una **VMXON region** de 4 KB
- Ejecución de la instrucción `VMXON [phys_addr]`

Una vez activado el modo VMX en un core, **ningún otro software puede volver a ejecutar `VMXON` sobre ese core sin haber ejecutado `VMXOFF` antes**.

## ¿Qué hace cada hipervisor?

| Hipervisor | Acceso directo a VT-x | Usa VMBus / VSC/VSP | Instala driver ring 0 |
| --- | --- | --- | --- |
| **Hyper-V** | ✅ Sí | ✅ Sí | ✅ hvix64.sys |
| **VMware Workstation** | ✅ Sí | ❌ No | ✅ vmmon.sys |
| **VirtualBox** | ✅ Sí | ❌ No | ✅ VBoxDrv.sys |

Cuando **Hyper-V está habilitado**, se convierte en el **hipervisor de tipo 1** y toma control de todos los cores. Ejecuta `VMXON` en cada uno de ellos durante el boot, bloqueando el acceso a VMX para cualquier otro software.

## Turtles y NestCloud: el nacimiento del nested virtualization

### 🐢 Turtles (2010)

- Paper: *Turtles: Practical Nested Virtualization*, ASPLOS 2010
- Primer diseño funcional de nested virtualization en KVM
- Permite ejecutar un hipervisor dentro de una VM con overhead razonable

### ☁️ NestCloud (2011)

- Paper: *NestCloud: Towards Practical Nested Virtualization*, Tsinghua + Intel
- Usa Intel VT-x con EPT
- Introduce: Guest Page Fault Bypassing, Virtual EPT, PV VMCS
- Optimiza overhead en entornos como OpenStack o Azure

Ambos papers demostraron que, con el soporte adecuado del hardware y del hipervisor, es posible ejecutar múltiples niveles de hipervisores (L0, L1, L2) con overhead controlado.

## ¿Qué pasa cuando se activa Hyper-V?

Cuando Windows tiene habilitado Hyper-V (incluso si no hay VMs corriendo), el sistema ejecuta esto durante el boot:

```
bcdedit /set hypervisorlaunchtype auto
```

Esto causa que:

- `VMXON` se ejecute en todos los cores
- `CR4.VMXE` se establezca
- **VMware/VirtualBox ya no puedan ejecutar ****`VMXON`**, fallando con errores como "VMX unavailable"

## Diagrama de estados de VMCS (Intel SDM Vol. 3C)

[Se insertaría aquí la segunda imagen subida, que muestra el ciclo de vida de una VMCS]

Este diagrama muestra cómo una región de VMCS cambia de estado dependiendo de las instrucciones del VMM (`VMPTRLD`, `VMLAUNCH`, `VMRESUME`, `VMCLEAR`). Una vez que `VMXON` fue ejecutado, el procesador entra en modo VMX, y solo puede haber una región activa por core.

## Bareflank: ejecución real de `VMXON`

Bareflank es un hypervisor C++ moderno que implementa VMXON de forma portable. Sus archivos:

- `intrinsic_vmxon.S/.asm`: ejecutan la instrucción `vmxon (%rdi)`
- `enable_hve.c`: orquesta la verificación, asignación y ejecución de la VMXON region

Pseudocódigo:

```
if (cpu_supports_vmx()) {
    write_cr4(get_cr4() | CR4_VMXE);
    vmxon_region = alloc_page_aligned();
    *vmxon_region = get_revision_id();
    vmxon(phys_addr(vmxon_region));
}
```

Esto solo puede ejecutarse en ring 0 y si `IA32_FEATURE_CONTROL` lo permite.

## Conclusión

VMware y Hyper-V no pueden coexistir con nested virtualization porque **ambos requieren control exclusivo del estado VMX de los cores**. Una vez que Hyper-V está activo, **ningún otro hipervisor puede ejecutar `VMXON`**, incluso si hay cores disponibles, porque el CPU no permite múltiples inicializaciones VMXON sin un `VMXOFF` explícito.

Nested virtualization es posible (como lo demostraron *Turtles* y *NestCloud*), pero solo si el hipervisor principal coopera. Hyper-V no lo hace con VMware o VirtualBox.

Este límite no es un error de software, es una **consecuencia arquitectónica del diseño de Intel VT-x** y el control de privilegios del modo VMX.

> 📌 Para que VMware funcione correctamente en Windows, Hyper-V debe estar completamente deshabilitado con:
> 
> 
> ```
> bcdedit /set hypervisorlaunchtype off
> ```
> 

---

Referencias:

- Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volume 3C
- Turtles: Practical Nested Virtualization (ASPLOS 2010)
- NestCloud: Towards Practical Nested Virtualization (2011)
- https://github.com/Bareflank/hypervisor