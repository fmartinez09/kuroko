---
title: "Galería con Minio, Eilisearch y HuggingFace en k8s"
description: ""
date: "May 27, 2025"
---

Mi gatito, _kai_ murió hace unas semanas. Pocas veces subía fotos de él a mis redes sociales, solamente las compartía con mi novia y mi madre. Su pérdida me afecto demasiado y no quiero que queden sus fotos en google imágenes. Me propuse hacerle una galería con dominio propio en internet, para que cuando quiera recordarlo, pueda visitarlo ahi, todos. 

Si bien pude hacer un álbum en google fotos con acceso a cualquiera que tenga el link, creo que mi hijo merece un poco más de dedicación que esto y se la puedo dar. 

La arquitectura que pensé es la siguiente:

```md
+-----------------------------+
|         Next.js Frontend    |
|   (Dynamic Gallery Viewer)  |
+-----------------------------+
            |
            v
+-----------------------------+
|       Backend API (quarkus) |
|  - Consultas a PostgreSQL   |
|  - Fetches de MinIO         |
|  - Queries a Meilisearch    |
|  - Experimentos con Apache  |
|    Trino                    |
+-----------------------------+
            |
            v
+-----------------------------+
|    PostgreSQL: Metadata     |
| (filename, fecha, tags...)  |
+-----------------------------+
            |
            v
+-----------------------------+
|         MinIO: Fotos        |
|       (original files)      |
+-----------------------------+
            |
            v
+-----------------------------+
|     Meilisearch: Index      |
|  (para buscar rápido)       |
+-----------------------------+
            |
            v
+-----------------------------+
|    Trino (opcional): SQL    |
|Federado: PostgreSQL + MinIO |
+-----------------------------+

```

El pipeline para poder ingresar las fotos a mi base de datos seria algo asi

```md
Google Photos -> JSON + imágenes
         |
         v
[Script ETL Python o Go]
         |
         |----> MinIO (fotos)
         |----> PostgreSQL (metadata)
         |----> Meilisearch (búsquedas)
         |----> (opcional) HuggingFace/BLIP para tags IA
```

La verdad pensé en asignarle los tags yo mismo a las imágenes, porque es mi hijo y solo yo se que significan esas cosas, pero al ser tantas (1200) la verdad tengo correos y trabajo, por lo que tenia tres alternativas gracias a la maravillosa IA: usar un modelo de computer vision (CV) para generar tags automáticos. 

- HuggingFace: BLIP, CLIP, Segment Anything.
- OpenAI: GPT-4 Vision (subiendo la foto como input).
- Google Vision API.

Pago GPT pero creo que sería más lindo usar HuggingFace. 

La infra quedó basica. Es una vm con k3s en tu cloud favorito. Al ser kubernetes, hay que configurar un poco el servidor (o la vm) pero no es para tanto. No me preocupa mucho la escalabilidad, escritura masiva, consistencia global, alta disponibilidad, multi-region, escalado horizontal, puesto que es algo personal. No necesito sharding ni CQRS. Además, esto no escala a billones, por lo que no necesito colas como nats o kafka ni caching con redis. No tengo autenticación capa 7, throttling masivo, ni control de versiones de APIs. 

Gitops y ci/cd? simple, ci con gitlab community y release tags y cd con argocd y deployment automático por si me da la cosa y le agrego apache trino o databricks con algo de ML, series temporales o clustering de fotos. Simplecito. Nada de OTEL, no me interesa instrumentar el backend o monitorear con grafana loki. 

Una vez declarada la infra, procedemos a crear los contenedores y subimos las imagenes al registry de tu preferencia. 

Generamos el par de claves ssh 
```bash
ssh-keygen -t rsa -b 4096 -f $env:USERPROFILE\.ssh\kai_vm_key
```

C:/Users/suusuario/.ssh/kai_vm_key.pub

Obtenemos nuestro ssh fingerprint y api token de DigitalOcean. Usaremos droplet de DO porque azure está muy caro. 

```bash
terraform init, plan, apply
```

```terraform
terraform {
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }

  required_version = ">= 1.0"
}

provider "digitalocean" {
  token = var.do_token
}

resource "digitalocean_droplet" "kai_vm" {
  name   = var.name
  region = var.region
  size   = var.size
  image  = var.image

  ssh_keys = [var.ssh_fingerprint]

  tags = ["development"]

  backups = false
  ipv6    = true

  monitoring = true
}

output "droplet_ip" {
  value = digitalocean_droplet.kai_vm.ipv4_address
}
```

Accedemos a nuestro droplet o vm.
```bash
ssh -i ~/.ssh/id_rsa root@<IP_DEL_DROPLET>
```

Segundo, instalamos las cosas necesarias en la vm. No la hice con ansible porque que me dio flojera. 

Actualizamos 
```bash
sudo apt update && sudo apt upgrade -y
```

Instalar dependencias básicas
```bash
sudo apt install -y curl wget vim git ufw htop
```

 Configurar hostname (opcional)
```bash
sudo hostnamectl set-hostname k3s-node
```

Firewall 
```bash
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 6443
sudo ufw enable
```

 Swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

 Sysctl 
```bash
sudo tee /etc/sysctl.d/k3s.conf <<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF

sudo sysctl --system
```

Aputamos el dominio del proveedor DNS a la IP pública de la VM.

K3s
```bash
curl -sfL https://get.k3s.io | sh -
```

- K3s server (API server, controller, scheduler).
- Traefik (Ingress Controller).
- Flannel (CNI).
- Containerd.
- K3s se habilita como servicio (k3s.service).
- Configuración en /etc/rancher/k3s/k3s.yaml


Verificamos
```bash
kubectl get nodes
kubectl get pods -A
```

Creamos los manifiestos (YAML) para:
- El namespace del proyecto.
- StatefulSets para PostgreSQL y MinIO.
- Deployments para Next.js y Meilisearch.
- PVCs para la persistencia (StorageClass de la nube).
- Ingress para exponer servicios con dominios.
- Secrets para contraseñas y claves API.
- Recursos (requests, limits) para cada pod.
- RBAC para permisos y seguridad del clúster.

Opcionales:
- Cert-Manager para TLS automático con Let's Encrypt.
- Helm para facilitar despliegues y manejar versiones.

Desplegamos
```bash
mkdir -p ~/k3s-manifests
cd ~/k3s-manifests
kubectl apply -f namespace.yaml
kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml
kubectl apply -f .
```

Verificamos
```bash
kubectl get pods -A
kubectl get ingressroutes -A
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
```

Para ejecutar el script que inserte la metadata en postgresql y asocie el blob de la imagen en minio, creamos unas variables temporales y hacemos un port forward del k3s. No queremos exponer nuestros puertos.

Este pipeline procesa fotos y videos locales, generando descripciones automáticas, extrayendo metadata, subiéndolos a MinIO, registrando en PostgreSQL, e indexando en Meilisearch, todo corriendo en un clúster Kubernetes (K3s) en DigitalOcean.

### **Flujo de procesamiento (pipeline)**:

**Iteración de archivos**:

- El script recorre cada archivo del directorio especificado.

**Clasificación de archivo**:

- Si es una **foto** (`.jpg`, `.jpeg`, `.png`, `.heic`):
    - Extrae metadata **EXIF** (fecha de captura, cámara, etc.).
    - Genera un **caption** usando el modelo **BLIP** (`Salesforce/blip-image-captioning-base`).
    - Determina la fecha de captura (`taken_at`) a partir de EXIF.
- Si es un **video** (`.mp4`):
    - Marca `description` como `"Video file"`.
    - No extrae metadata ni genera caption (por ahora, eso es para una futura mejora).

**Subida a MinIO** (en el clúster K3s):

- Conecta al servicio MinIO (via `kubectl port-forward` o acceso directo).
- Sube el archivo al bucket `photos`, organizándolo por año (ej: `2025/foto1.jpg`).

**Registro en PostgreSQL** (en el clúster):

- Inserta un registro en la tabla `photos` con:
    - `filename`
    - `title` (igual al nombre del archivo)
    - `description` (caption o `"Video file"`)
    - `taken_at` (fecha extraída de EXIF o `null`)
    - `minio_path` (ruta en MinIO)
    - `metadata` (JSON con datos EXIF o vacío)

**Indexado en Meilisearch** (en el clúster):

- Crea un documento indexado con:
    - `id`
    - `filename`
    - `title`
    - `description`
    - `minio_path`
- Permite búsquedas rápidas por estos campos.

Entonces, tenemos que hacer un port-forward entre el cluster y la maquina local para correr el ETL. 
Instalamos [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) usando choco

Creamos la carpeta .kube/k3s.yaml y copiamos la configuración de nuestro cluster y ajustamos la ip interna del cluster por la publica
```bash
cat /etc/rancher/k3s/k3s.yaml
```

Cargamos la variable de entorno

```bash
$env:KUBECONFIG="C:\Users\rferm\.kube\k3s-config.yaml"
```

Y corremos el port-forward localmente
```bash
kubectl port-forward svc/postgres 5432:5432 -n photo-gallery
kubectl port-forward svc/minio 9000:9000 -n photo-gallery
kubectl port-forward svc/meilisearch 7700:7700 -n photo-gallery
```
Y en el cluster obviamente
```bash
kubectl port-forward svc/postgres 5432:5432 -n photo-gallery
kubectl port-forward svc/minio 9000:9000 -n photo-gallery
kubectl port-forward svc/meilisearch 7700:7700 -n photo-gallery
```


Alimentando al clúster en tiempo real 
```python
python process_photos.py C:/Users/rferm/Downloads/kai/kai
```

Algunas imágenes de digitalocean y pycharm, porque no agregué prometheus ni grafana.

![blog placeholder](/public/static/loading1.png)

![blog placeholder](/public/static/loading2.png)

![blog placeholder](/public/static/loading3.png)

![blog placeholder](/public/static/loading4.png)

Este pipeline lleva los .jpg, .jpeg, .png, .heic y .mp4 al clúster.

Si queremos ver el bucket
```bash
kubectl get pods -n photo-gallery
kubectl logs -n photo-gallery minio-7b9f488855-dqhvk
```

Querido kai, este es tu recuerdo. 

[Kai.gold](https://www.kai.gold)