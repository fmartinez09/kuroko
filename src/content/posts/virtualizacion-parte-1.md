---
title: "Parte 1: Arquitectura comparativa de Nested Virtualization en Hyper-V, KVM y VMware"
description: ""
date: "May 23, 2025"
---

¿Por qué Docker Desktop sí puede coexistir con Hyper-V y WSL2, pero VMware no?

Estaba intentando mostar un cluster de alta disponibilidad usando vmware, ya que irme a la rápida y usar una implementación de Kind (kubernetes in docker) no tendría un entorno real para probar técnicas de arquitectura de alta disponibilidad o sharding por ejemplo. Resulta que como buen desarrollador de windows, tenia habilitado docker y hyper-v, pero al momento de instalar proxmox ve (un sistema operativo en el cual puedo provisionar varias vm dentro de él) tenía un mensaje de que mi sistema no soporta KVM, algo esencial para hacer virtualización anidada. Pero ¿qué es KVM? para eso primero tenemos que remontarnos a los hipervisores. 

# Hipervisores

> Un hipervisor es un software que se puede utilizar para ejecutar varias máquinas virtuales en una única máquina física. [1](https://aws.amazon.com/es/what-is/hypervisor/#:~:text=Un%20hipervisor%20es%20un%20software,operativo%20y%20de%20aplicaciones%20propios.) 

Existen de dos tipos:
- Tipo 1 (bare-metal): corre directo sobre el hardware (ej: ESXi, KVM, Hyper-V Server). 
- Tipo 2 (hosted): corre como una app sobre un OS (ej: VirtualBox, VMware Workstation). 
Su función principal: virtualizar el hardware para que múltiples sistemas operativos puedan usarlo de forma aislada y compartida. 

Exactamente en ese momento con vmware estaba usando un hipervisor de tipo 2.  Preguntarnos por el origen de los hipervisores nos remontaría a los años 60' y la verdad, no viene al caso su funcionamiento ni arquitectura. Basta con saber algunos detalles.

## 1. Primera generación (1960s–2000): Virtualización base

- **Objetivo:** que *una sola máquina física* corra múltiples sistemas operativos simultáneamente.
- **Ejemplo:** IBM CP-40 (1967), VM/370 → corrían múltiples CMS sobre un solo mainframe.
- **Hipervisores tipo 1 nacen aquí**: controlan el hardware directamente.
- No se necesitaba nested virtualization porque el objetivo era **compartir el hardware**, no anidar mundos virtuales.

Entonces, estamos bien hasta este punto. Los hipervisores de tipo 1 son buenos, ya que:
- Los mainframes ya daban aislamiento fuerte.
- Nadie necesitaba correr un hipervisor *dentro* de una VM.
- Las VMs ya cumplían el objetivo de compartir recursos.

Pero surge la necesidad de correr un hipervisor dentro de un un so como nuestro querido virtualbox o vmware, asi que comienza la era del hipervisor de tipo 2. 

## 2. **Virtualización x86 (2000s)**

- **VMware** populariza la virtualización comercial sobre x86 sin soporte nativo del hardware.
- Luego aparecen Intel VT-x y AMD-V (~2005), que lo hacen eficiente.
- Empiezan los clouds (EC2 en 2006), y ahora sí: **hay razones para virtualizar dentro de máquinas virtualizadas**:
    - Testear tu propio cloud dentro de AWS
    - Correr Hyper-V dentro de una VM
    - Tener un laboratorio de clusters en un datacenter sin máquinas físicas adicionales

De hecho, si te das cuenta en este punto aparecieron los famosos clouds, como la maquina virtual de amazon ec2. Pero realmente esto nos ofrece una maquina virtual dentro del so y no tenemos un problema real. El tema aquí es cuando queremos crear una máquina virtual, dentro de otra. 

## 3. **Turtles (2010)**

- Hasta ese momento, **no existía una solución técnica real para permitir anidar virtualización** en x86.
- Turtles implementa nested virtualization en KVM, **antes de que el hardware lo soporte completamente**.
- Fue un **salto técnico** que abrió la posibilidad de tener *infraestructura dentro de la infraestructura*.

## 5. **NestCloud (2011) lo hace viable a escala cloud**

- Toma lo que hizo Turtles, pero le aplica mejoras usando las nuevas capacidades de **Intel VT-x con EPT**.
- Hace que la anidación tenga **bajo overhead** y pueda usarse en producción en entornos como OpenStack o Azure.




Aunque [NestCloud](/static/nestcloud-towards-practical-nested-virtualization.pdf) fue una de las primeras arquitecturas prácticas de virtualización anidada con bajo overhead, no fue la primera en implementar esta tecnología. Por ejemplo, el proyecto Turtles presentó en 2010 una solución de virtualización anidada en el hipervisor KVM, logrando un rendimiento cercano al de la virtualización de un solo nivel.

##  ¿Qué aportó NestCloud?
Antes de NestCloud, las soluciones de virtualización anidada en plataformas x86 se basaban principalmente en emulación, lo que resultaba en un rendimiento deficiente y una usabilidad limitada. NestCloud introdujo tres optimizaciones clave para reducir la sobrecarga de las máquinas virtuales anidadas:


1. Guest Page Fault Bypassing: Permitía que las máquinas virtuales anidadas manejaran fallos de página sin necesidad de una salida de máquina virtual (VM Exit).
2. Virtual EPT (Extended Page Table): Eliminaba fallos de página innecesarios introducidos por las tablas de páginas sombra en el monitor de máquina virtual anidado.
3. PV VMCS (Paravirtualized Virtual Machine Control Structure): Proporcionaba un acceso más eficiente a la estructura de control de la máquina virtual para el monitor de máquina virtual anidado.

Los resultados experimentales mostraron que el rendimiento de las máquinas virtuales anidadas en NestCloud era cercano al de las máquinas virtuales de un solo nivel en pruebas intensivas de CPU y memoria, con una sobrecarga de CPU del 5.22% y una sobrecarga de memoria del 5.69% 

En resumen:

## Turtles (2010)
- Autores: Ben-Ami et al., IBM Research y Technion.
- Hipervisor: KVM
- Paper: Turtles: Practical Nested Virtualization (ACM ASPLOS 2010)
- Aporte clave:
Primer diseño funcional y operativo de nested virtualization sobre Linux/KVM, permitiendo ejecutar un hipervisor completo (L1) sobre otro (L0) con rendimiento razonable.
- Impacto:
Fue la primera implementación seria y completa de nested virtualization sobre x86. Introdujo el concepto técnico que luego adoptarían otros sistemas.
- Reconocimiento:
Publicado en una conferencia de alto nivel (ASPLOS), con fuerte impacto en la comunidad académica y kernel de Linux.

## NestCloud (2011)
- Autores: Tsinghua University + Intel Asia Pacific
- Hipervisor: Intel VT-x + arquitectura propia
- Paper: NestCloud: Towards Practical Nested Virtualization
- Aporte clave:
Propuso optimizaciones concretas (Guest Page Fault Bypassing, Virtual EPT, PV VMCS) que mejoraron sustancialmente el rendimiento de la virtualización anidada en escenarios de nube.
- Impacto:
No fue pionero en implementar nested virtualization, pero sí mejoró su viabilidad práctica, sobre todo para nubes privadas/publicas que requerían overhead bajo.
- Reconocimiento:
Técnica sólida, pero con menos impacto general que Turtles. No apareció en una conferencia tan influyente como ASPLOS.

## Entonces, recapitulando

| Tiempo | Tecnología | Capacidad |
| --- | --- | --- |
| 1960s-90s | Hipervisores (IBM, VMware) | Crear VMs |
| 2000s | Clouds + Hypervisores modernos | Infraestructura virtualizada |
| 2010 | Turtles | Primera implementación funcional de nested virtualization |
| 2011 | NestCloud | Optimización y viabilidad productiva en cloud |


> 📌 Los hipervisores resolvieron la necesidad de compartir hardware. Nested virtualization resuelve la necesidad de virtualizar infraestructura completa dentro de infraestructura ya virtualizada.
 
Perfecto. Ya tenemos idea de cómo funcionan las VM y virtualización anidada. Windows desarrolló su alternativa comercial en Windows Server 2008: Hyper-V. Tenemos una comparativa técnica entre el hipervisor de windows y linux:

| Hypervisor   | Hyper-V (Windows) | QEMU + KVM (Linux) |
| --------- | -------- | ------ |
| Virtualizador | QEMU | No usa uno externo: es nativo |
| Interfaz Host-Guest | VirtIO | VMBus + VSC/VSP |
| CPU Extensions | VT-x/AMD-V | VT-x/AMD-V |
| Aceleración por hardware | Sí (KVM) | Sí (nativa) |
| Contenedores | Docker/LXC con namespaces | Windows Containers, WSL2 |

¿Qué sucedio entonces?

Vmware es un hipervisor de tipo 2 en el cual instalamos un SO que ejecuta un hipervisor de tipo 1. El problema aquí es que hyper-v, el hipervisor de tipo 1 de windows se apodera del hardware del so, lo que no permite funcionar.

Hyper-V Gen 2 permite UEFI y soporte para virtualización anidada solo si el host también es Hyper-V en Windows 10/11 Pro, Enterprise o Server y si habilitamos explícitamente la opción 

```powershell
ExposeVirtualizationExtensions
```

Esto permite que el sistema invitado detecte las extensiones VT-x/AMD-V necesarias para KVM, aunque no con el mismo rendimiento que en bare-metal.

Proxmox puede correr dentro de Hyper-V porque el kernel Linux detecta que se están exponiendo las extensiones de virtualización (si están habilitadas en la VM de Hyper-V).

Dentro de esa VM, Proxmox corre como si estuviera en una máquina física, y puede levantar contenedores LXC o VMs con QEMU/KVM.

Sin embargo, el rendimiento de KVM anidado en Hyper-V es mucho menor que en KVM sobre KVM o sobre bare metal.

VMware Workstation también permite virtualización anidada, pero:

- No siempre expone correctamente las extensiones de hardware.
- En ciertos casos, Linux no logra inicializar completamente KVM si las extensiones están mal emuladas.
- Hyper-V, al estar más cerca del hypervisor tipo 1, tiene mejor integración con el kernel de Windows y puede pasar mejor las extensiones al kernel Linux invitado.

El kernel de Linux detecta si hay soporte para virtualización de hardware leyendo los registros CPUID, que pueden estar presentes si Hyper-V los expone.

Si el kernel no ve las extensiones, entonces kvm no puede cargarse y Proxmox no podrá virtualizar nada dentro de esa VM.

# ¿WSL2 permite virtualizacion anidada?

WSL2 corre sobre una VM ligera de Hyper-V, y esa VM de WSL2 no tiene acceso directo a las extensiones de virtualización (VT-x/AMD-V) por defecto. Pero:

Si estás en Windows 11 (o Windows 10 Pro 21H2+), puedes habilitar virtualización anidada en tu máquina host (Windows), y luego levantar una VM completa dentro de WSL2 usando QEMU con ciertas flags especiales (-enable-kvm).

Esto es experimental y no tan estable como KVM en Linux bare metal o en una VM Linux sobre KVM.

Por lo tanto, no puedo correr Proxmox dentro de WSL2 directamente.

WSL2 no es un sistema completo con soporte para ```systemd``` (a menos que lo actives manualmente), ni puede exponer interfaces como ```/dev/kvm``` con facilidad. Pero sí puedes correr QEMU dentro de WSL2 con:

```--accel tcg``` (sin KVM): muy lento.

```--accel kvm``` (con acceso a ```/dev/kvm```): solo funciona si el host Windows expone virtualización anidada y la VM base de WSL2 la puede usar, lo cual es raro y poco fiable.

Entonces nos quedan tres alternativas.

- Instala Proxmox directamente en bare-metal.
- Usa Hyper-V Gen 2 con virtualización anidada activada para correr Proxmox como VM utilizando ```Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $true```
- Instala Ubuntu o NixOS en dualboot o en una VM con virtualización anidada activada, y ahí corres Proxmox o QEMU como se debe.

Se vería algo asi 

```plaintext
Intel CPU con VT-x
  └── Hyper-V Hypervisor (tipo 1)
      └── Hyper-V Gen 2
          ├── Kernel Linux con /dev/kvm
          └── Proxmox VE instalado
              └── VM Fedora 39
                  └── Kubernetes corriendo
                      └── Docker container
                          └── App Java con Quarkus y GraalVM

```
Cada capa ve sólo parte del sistema real, y se apoya en el subsuelo: hardware → hipervisor → kernel → namespaces → runtime → contenedor.
- Si /dev/kvm no existe, no hay aceleración de hardware.
- Si tu contenedor necesita /dev/kvm (como Firecracker o Kata Containers), no funcionará sin soporte del host.
- Si Docker corre sobre WSL2 y no hay VT-x expuesto, no puedes usar KVM dentro del contenedor.

# Mecanismos de virtualización

CPUID y MSR (Model-Specific Registers)

Cuando un hipervisor expone extensiones de virtualización (como VT-x), lo hace manipulando las respuestas al opcode CPUID, y permitiendo accesos a ciertos MSRs (Model-Specific Registers) como IA32_FEATURE_CONTROL. Si estas no son correctamente expuestas por el L0, el L1 no puede inicializar su hipervisor (L2).

EPT y shadow paging

Extended Page Tables (EPT) es una técnica de Intel VT-x que permite una segunda capa de traducción de direcciones virtuales a físicas sin intervención constante del hipervisor. Anteriormente, esto requería shadow page tables, que inducían muchos VM Exits. NestCloud explota EPT para mantener la eficiencia.

VMBus vs VirtIO

VMBus (de Microsoft) y VirtIO (de Red Hat) son buses de paravirtualización: el primero está acoplado a los VSP/VSC (Virtualization Service Provider/Client), mientras que VirtIO es un estándar abierto con dispositivos como virtio-blk, virtio-net, etc. La performance de I/O y el overhead dependen de cómo el host exponga estos dispositivos virtualizados.

| Hypervisor | KVM (módulo de kernel) | Hyper-V Hypervisor (propietario) |
| --- | --- | --- |
| Virtualizador | QEMU | No usa uno externo: es nativo |
| Interfaz Host-Guest | VirtIO | VMBus + VSC/VSP |
| CPU Extensions | VT-x/AMD-V | VT-x/AMD-V |
| Aceleración por hardware | Sí (KVM) | Sí (nativa) |
| Contenedores | Docker/LXC con namespaces | Windows Containers, WSL2 |

_Fernando Martínez – Infraestructura, virtualización y sistemas distribuidos_