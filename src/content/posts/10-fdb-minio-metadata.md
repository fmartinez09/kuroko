---
title: "FDB + MinIO: Metadatos (Java)"
description: ""
date: "Sep 29, 2025"
---

# FDB + MinIO: Metadatos (Java)

Crearemos un pequeño servicio HTTP que registre y consulte metadatos de blobs almacenados en MinIO. Los bytes estarán en MinIO; FoundationDB guardará el *catálogo de metadatos* (owner/tenant, bucket, object key, size, hash, timestamps, tags, etc.).

> Objetivo: algo mínimo para iterar luego a idempotencia avanzada, presigned URLs, composición con otros SNAPs, etc.
> 

# 1. Infraestructura local (docker-compose)

Creamos el docker-compose para nuestras tres aplicaciones base.

`docker-compose.yml`

```bash
version: "3.8"

services:
  fdb:
    image: foundationdb/foundationdb:7.4.0
    container_name: fdb
    command: >
      fdbserver
      --listen_address 0.0.0.0:4500
      --public_address fdb:4500
      --cluster_file /var/fdb/fdb.cluster
      --datadir /var/fdb/data
      --logdir /var/fdb/logs
    volumes:
      - fdb-data:/var/fdb/data
      - ./fdb.cluster:/var/fdb/fdb.cluster:ro
    ports:
      - "4500:4500"
    restart: unless-stopped

  minio:
    image: minio/minio
    container_name: minio
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio12345
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    restart: unless-stopped

  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: meta-app
    depends_on:
      - fdb
      - minio
    environment:
      MINIO_ENDPOINT: http://minio:9000
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio12345
      FDB_CLUSTER_FILE: /etc/foundationdb/fdb.cluster
    volumes:
      - ./fdb.cluster:/etc/foundationdb/fdb.cluster:ro
    ports:
      - "8080:8080"
    restart: unless-stopped

volumes:
  fdb-data:
  minio-data:
```

Obtener el clúster file para el cliente Java:

`docker cp fdb:/var/fdb/fdb.cluster ./fdb.cluster`

# 2) Esquema de claves en FDB (Tuple Layer)

Raíz: `("b")`

```bash
("b", tenant, objId, "meta") -> JSON { bucket, object, size, sha256, contentType, etag, createdAt, tags: {k:v} }
("b", tenant, objId, "state") -> BYTE (0=pending, 1=committed)
```

> Para el MVP escribiremos solo `meta` y `state=1`. Más adelante puedes agregar índices por prefijo, tags o owner.
> 

**Convenciones**

- `tenant`: string corto (p.ej. "acme").
- `objId`: identificador lógico dentro del tenant (UUIDv7 recomendado).

# 3) API mínima (Javalin)

- `PUT /o/{tenant}/{id}`  → Registra/actualiza metadatos de un objeto **ya existente** en MinIO. Body JSON:
    
    ```bash
    {
    "bucket": "metas",
    "object": "imgs/networking.png",
    "contentType": "image/png",
    "size": 12345,
    "sha256": "...",
    "tags": {"owner":"tú o yo","kind":"img"}
    }
    ```
    
    Valida que el objeto exista en MinIO (via `statObject`) y guarda `meta` + `state=1` en FDB (transacción). Idempotente por `(tenant,id)`.
    
- `GET /o/{tenant}/{id}`  → Devuelve JSON `meta`.
- `HEAD /o/{tenant}/{id}` → 200 si existe, 404 si no.
- `GET /o/{tenant}?prefix=abc` → Lista `objId` que comiencen con `abc` (scan acotado por rango). **MVP**: límite fijo (p.ej. 100).
- `DELETE /o/{tenant}/{id}?deleteBlob=true|false` → Borra metadatos; si `deleteBlob=true` intenta borrar también en MinIO.

# 4) Proyecto Java (Gradle)

```bash
plugins {
    id 'java'
    id 'application'
    id 'com.github.johnrengelman.shadow' version '8.1.1'
}

group = 'org.fmartinez'
version = '1.0-SNAPSHOT'

repositories { mavenCentral() }

java { toolchain { languageVersion = JavaLanguageVersion.of(21) } } // 21 LTS

dependencies {
    implementation "org.foundationdb:fdb-java:7.4.3"
    implementation "io.minio:minio:8.6.0"
    implementation "io.javalin:javalin:6.7.0"
    implementation "com.fasterxml.jackson.core:jackson-databind:2.20.0"
    runtimeOnly   "org.slf4j:slf4j-simple:2.0.16"

    testImplementation platform('org.junit:junit-bom:5.10.0')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

application {
    mainClass = 'Server'
}

tasks.test { useJUnitPlatform() }

```

te dejo el enlace al repo: https://github.com/fmartinez09/metadata

# 5) Contenerizar (docker)

¿Por qué? el binding Java necesita el cliente nativo de FDB instalado en el host (o disponible en PATH/LD_LIBRARY_PATH). En windows el binding Java intentará cargar `fdb_java.dll` (JNI) desde el JAR y luego `fdb_c.dll` del sistema. En **7.4.x** el JAR publicado en Maven (y gradle) no incluye el binario `fdb_java.dll` para Windows (de hecho, si quisiéramos deberíamos instalar cliente + usar versión con nativos), por ende lo tenemos que ejecutar en Linux (imagen Docker con JDK). 

Creamos un Dockerfile multistage para que corra con JDK 24 + foundationdb-clients 7.4 y se conecte a FDB y MinIO vía Docker network (sin dolores de DLL en Windows). 

```bash
# --- Etapa de build: compila y genera el fat-jar ---
FROM gradle:8.9-jdk21 AS build
WORKDIR /app
COPY . ./
RUN gradle shadowJar -x test

# --- Etapa FDB: solo para extraer libfdb_c.so ---
FROM foundationdb/foundationdb:7.4.0 AS fdb

# --- Etapa runtime: JRE y libfdb_c.so copiada ---
FROM eclipse-temurin:21-jre

# Copiamos la lib nativa del cliente FDB
COPY --from=fdb /usr/lib/libfdb_c.so /usr/lib/libfdb_c.so

# Variables por defecto
ENV MINIO_ENDPOINT=http://minio:9000 \
    MINIO_ACCESS_KEY=minio \
    MINIO_SECRET_KEY=minio12345 \
    FDB_CLUSTER_FILE=/etc/foundationdb/fdb.cluster

# Copiar el fat-jar construido
COPY --from=build /app/build/libs/*-all.jar /app/app.jar

# Entrypoint + preparación
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh && mkdir -p /etc/foundationdb

EXPOSE 8080
ENTRYPOINT ["/entrypoint.sh"]
```

Entrypoint.sh:

Nos ayudaremos de un script para automatizar tareas de consola. Necesitamos el fdb.cluster para nuestra aplicación en java. 

- Verificaremos si existe el archivo `fdb.cluster` en la ruta `/etc/foundationdb/fdb.cluster`.
    - Si no existe, lo creará con la línea `docker:docker@fdb:4500`.
- Mostrá su contenido (logs).
- Lanzará la aplicación Java (`java -jar /app/app.jar`).

```bash
#!/usr/bin/env bash
set -euo pipefail

# Si no existe el cluster file, crea uno apuntando al servicio 'fdb'
# Nota: la cadena "docker:docker" es el ID de coordinador por defecto.
if [ ! -f "${FDB_CLUSTER_FILE}" ]; then
  echo "docker:docker@fdb:4500" > "${FDB_CLUSTER_FILE}"
fi

echo "Using FDB_CLUSTER_FILE at ${FDB_CLUSTER_FILE}:"
cat "${FDB_CLUSTER_FILE}" || true

echo "Starting app..."
exec java -jar /app/app.jar
```

Pero ojo:

- Requiere que la variable `FDB_CLUSTER_FILE` esté definida (`set -u`). Si no está definida, el script revienta antes del `if`.
- Si `FDB_CLUSTER_FILE` existe, no lo sobrescribe.

Entonces, lanza `java -jar /app/app.jar` ⇒ dentro de un contenedor. Esto implica que  debemos tener un uber-jar (fat jar) o un jar auto-contenedor (ShadowJar / Spring Boot `bootJar`). Si no, faltarán dependencias.

Con todo esto, el directorio queda más menos así:

```bash
metadata/                 # raíz del proyecto Gradle
├── build.gradle
├── settings.gradle
├── src/
│   └── main/java/...      # Server.java, FdbStore.java, etc.
├── fdb.cluster            # cluster file compartido (1 línea: docker:docker@fdb:4500)
├── docker/
│   ├── Dockerfile
│   ├── entrypoint.sh
├── docker-compose.yml
```

# 6) Construcción de imagen (genera el fat-jar en la etapa build)

Creamos **`fdb.cluster`:**

powershell:

```bash
"docker:docker@fdb:4500" | Out-File -NoNewline -Encoding ascii .\fdb.cluster
```

bash:

```bash
echo docker:docker@fdb:4500 > fdb.cluster
```

Construimos:

```bash
docker compose build 
```

Levantamos todo:

```bash
docker compose up -d
```

Vemos logs de la app (para confirmar que tomó el cluster file):

```bash
docker compose logs -f app
```

## 7) Inicializar la base de datos FDB (solo la 1ª vez)

FoundationDB arranca el proceso, pero el cluster está “unconfigured”.

```powershell
docker compose exec fdb fdbcli --cluster_file /var/fdb/fdb.cluster --exec "configure new single ssd"

docker compose exec fdb fdbcli --cluster_file /var/fdb/fdb.cluster --exec "status minimal"

```

Cuando veamos `Database available` estamos.

## 8) Crear el bucket en MinIO (una vez)

El servicio valida que el objeto exista en MinIO; necesitamos un bucket. Puedemos hacerlo por la consola web ([http://localhost:9001](http://localhost:9001/), user `minio`, pass `minio12345`) o con el cliente `mc`:

```powershell
# (si tienes mc instalado en tu host)
mc alias set local http://localhost:9000 minio minio12345
mc mb local/metadata
mc ls local

```

# 9) Registro de metadatos en FDB

Accedemos a fdb:

```bash
docker compose exec fdb fdbcli --cluster_file /var/fdb/fdb.cluster
```

Dentro podemos hacer una pequeña prueba para ver los rangos k-v

```bash
getrangekeys "" \xff 
```

PUT:

```bash
$body = @{ bucket="metadata"; object="networking.jpg"; size=(Get-Item .\networking.jpg).Length; sha256="DF47E064EFC7A9ABC31E3E556BC0E43CF933738D4C843D9F41139194F28638EC"; contentType="image/jpeg"; tags=@{ owner="fernando"; tipo="foto" } } | ConvertTo-Json -Depth 3; Invoke-RestMethod -Method PUT -Uri "http://localhost:8080/o/tenant1/foto-001" -ContentType "application/json" -Body $body

```

**GET (leer metadata):**

```powershell
Invoke-RestMethod -Method GET -Uri "http://localhost:8080/o/tenant1/foto-001"
```

**HEAD (existencia):**

```powershell
$resp = Invoke-WebRequest -Method HEAD -Uri "http://localhost:8080/o/tenant1/foto-001"; $resp.StatusCode  # 200 o 404
```

**LIST (IDs por tenant):**

```powershell
Invoke-RestMethod -Method GET -Uri "http://localhost:8080/o/tenant1"
```

**DELETE (solo metadata):**

```powershell
Invoke-RestMethod -Method DELETE -Uri "http://localhost:8080/o/tenant1/foto-001" -SkipHttpErrorCheck
```

**DELETE (metadata + blob en MinIO):**

```powershell
Invoke-RestMethod -Method DELETE -Uri "http://localhost:8080/o/tenant1/foto-001?deleteBlob=true" -Sk

```

Listo. Estás trabajando con la base de datos más potente del mercado. 

Eliminar todo:

```bash
 docker compose down -v  
```
