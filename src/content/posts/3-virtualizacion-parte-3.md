---
title: "Parte 3:  GPU-PV en contenedores Hyper-V y testing hcsshim"
description: ""
date: "May 23, 2025"
weight: 1
---

## Introducción

Durante nuestra exploración sobre GPU-PV y Windows Server 2025, identificamos una limitación crítica: los contenedores de Windows con aislamiento Hyper-V aún no pueden acceder a la GPU. Sin embargo, al revisar el repositorio [hcsshim](https://github.com/microsoft/hcsshim) de Microsoft, descubrimos algo inesperado: **existen pruebas internas que sugieren que esto sí fue intentado o planeado**.

Este artículo narra ese descubrimiento, analiza el estado actual de la arquitectura y plantea posibles caminos futuros para lograr contenedores Hyper-V con acceso real a GPU.

---

## El rastro: una prueba oculta en `hcsshim`

En el issue [#1902](https://github.com/microsoft/hcsshim/issues/1902), un usuario reporta que intenta lanzar un contenedor de Windows con aislamiento Hyper-V y acceso a GPU utilizando `--device vpci://gpu-id`. Acompaña su reporte con un test case existente dentro del repositorio de `hcsshim`:

- [container_virtual_device_test.go](https://github.com/microsoft/hcsshim/blob/27df1b95b69faaeca97de86d25d68f48f89bc0b9/test/cri-containerd/container_virtual_device_test.go#L497C1-L543C2)


```java
import org.junit.Test;
import static org.junit.Assert.*;
import java.util.concurrent.TimeUnit;

public class ContainerTest {

    @Test
    public void testRunContainerVirtualDeviceGPUWCOWHypervisor() {
        requireFeatures(featureWCOWHypervisor, featureGPU);

        if (osversion.build() < osversion.V20H1) {
            System.out.println("Requires build +20H1");
            return;
        }

        String testDeviceInstanceID;
        try {
            testDeviceInstanceID = findTestNvidiaGPUDevice();
        } catch (Exception e) {
            fail("skipping test, failed to retrieve assignable device on host with: " + e.getMessage());
            return;
        }
        if (testDeviceInstanceID.isEmpty()) {
            fail("skipping test, host has no assignable devices");
            return;
        }

        pullRequiredImages(new String[]{imageWindowsNanoserver});
        TestRuntimeClient client = newTestRuntimeClient();

        Context podctx = new Context();
        SandboxRequest sandboxRequest = getRunPodSandboxRequest(
                wcowHypervisorRuntimeHandler,
                new SandboxAnnotations(Map.of(
                        annotations.FullyPhysicallyBacked, "true"
                ))
        );

        String podID = runPodSandbox(client, podctx, sandboxRequest);
        try {
            removePodSandbox(client, podctx, podID);
            stopPodSandbox(client, podctx, podID);
        } catch (Exception e) {
            e.printStackTrace();
        }

        Runtime.Device device = new Runtime.Device();
        device.setHostPath("vpci://" + testDeviceInstanceID);
        ContainerRequest containerRequest = getGPUContainerRequestWCOW(podID, sandboxRequest.getConfig(), device);
        Context ctx = new Context();
        ctx.setTimeout(20, TimeUnit.SECONDS);

        String containerID = createContainer(client, ctx, containerRequest);
        try {
            removeContainer(client, ctx, containerID);
            startContainer(client, ctx, containerID);
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (!isGPUPresentWCOW(client, ctx, containerID)) {
            fail("expected to see a GPU device on container " + containerID + ", none present");
        }
    }
}
```

Este test intenta agregar un dispositivo virtual (como una GPU) al spec de contenedor usando `vpci://GPU-1`, y se conecta al dispositivo vía `\\.\GPU0`, exactamente como se haría para un contenedor con acceso a hardware acelerado.


## ¿Qué respondió Microsoft?

Una desarrolladora del equipo fue clara:

> “We do not support this scenario… There is no matching code in cri or ctr to setup the device information on the spec as hcsshim expects… Please do not take any dependencies on this.”

En resumen:

- Sí, el código existe, pero **no está cableado al runtime** (containerd o CRI).
- `hcsshim` espera información de dispositivos y drivers que **no está siendo pasada** por los runtimes.
- Los tests eran parte de un trabajo anterior, no una funcionalidad completa.


## ¿Qué implicaciones tiene esto?

- **La arquitectura para GPU en contenedores Hyper-V *existe en partes*.**
- El soporte está *fragmentado*: el runtime (ctr/containerd) no integra los pasos requeridos.
- Microsoft y NVIDIA **no han liberado imágenes base ni drivers compatibles** para Hyper-V Containers con GPU.

Esto significa que aunque `hcsshim` tenga el código para manejar dispositivos como GPU via VPCI, **la cadena de ejecución está incompleta.**


## ¿Es posible completarla?

Teóricamente sí, pero implica trabajo profundo:

1. Modificar `containerd` y su CRI plugin para aceptar `-device vpci://...` y construir correctamente el `spec`.
2. Extender `hcsshim` para manejar mejor el registro de drivers (`DriverPackagePath`) en contenedores con aislamiento Hyper-V.
3. Crear imágenes base Windows con el driver GPU adecuado preinstalado o auto-instalable.

Esto sería el primer paso para lograr contenedores Hyper-V con GPU-PV real.

## Conclusión

Dentro de `hcsshim` se sugiere que **el soporte para GPU en contenedores con aislamiento Hyper-V fue explorado internamente, pero no completado**. Esta es una oportunidad abierta para aportar a la evolución del ecosistema de contenedores en Windows.

**Un flujo ideal seria ir de `ctr` a `containerd`, a `hcsshim`, y de ahí a la GPU real.** Sería un salto enorme en compatibilidad de contenedores para entornos empresariales con aceleración de hardware.

**El stack está roto, pero las piezas están ahí.**


_Fernando Martínez – Infraestructura, virtualización y sistemas distribuidos_