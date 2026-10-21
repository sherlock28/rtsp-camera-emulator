# Configuración

## Archivo `.env`

Todas las variables del servicio `video-stream` se definen en un archivo `.env` en la raíz del proyecto:

```env title=".env"
VIDEO_SOURCE=./videos
VIDEO_FILE=demo.mp4
VIDEO_CODEC=libx264
STREAM_NAME=mystream
```

| Variable | Descripción | Valores posibles |
|---|---|---|
| `VIDEO_SOURCE` | Ruta local de la carpeta con los videos | `./videos`, `/data/videos`, etc. |
| `VIDEO_FILE` | Nombre del archivo a transmitir | `demo.mp4`, `video.mkv`, `cam.avi` |
| `VIDEO_CODEC` | Codec de video para recodificación | `libx264` (H.264), `libx265` (H.265) |
| `STREAM_NAME` | Ruta del stream en el servidor RTSP | `mystream`, `cam1`, `entrada`, etc. |

!!! tip "¿Varias cámaras?"
    Estas variables configuran **una sola cámara** (`docker-compose.yml`). Para simular varias, cada cámara necesita sus propias variables en el `.env`. Ver [Múltiples streams](#multiples-streams-varias-camaras).

!!! note "Formatos soportados"
    El servicio acepta archivos `.mp4`, `.mkv` y `.avi`. El video siempre se recodifica al codec definido en `VIDEO_CODEC` y el audio se convierte a AAC estéreo para máxima compatibilidad.

---

## `docker-compose.yml` base

```yaml title="docker-compose.yml"
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    restart: unless-stopped
    container_name: mediamtx
    environment:
      - MTX_PROTOCOLS=tcp
    ports:
      - "8554:8554"

  video-stream:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-stream
    depends_on:
      - mediamtx
    volumes:
      - ${VIDEO_SOURCE}:/videos
    command: >
      -re -stream_loop -1 -i /videos/${VIDEO_FILE}
      -map 0:v:0 -map 0:a:0? -c:v ${VIDEO_CODEC} -preset veryfast
      -c:a aac -ac 2 -f
      rtsp rtsp://mediamtx:8554/${STREAM_NAME}

  docs:
    image: squidfunk/mkdocs-material:latest
    container_name: mkdocs-docs
    ports:
      - "8000:8000"
    volumes:
      - .:/docs
    profiles:
      - docs
```

---

## Múltiples streams (varias cámaras)

Para simular varias cámaras, cada una necesita **su propio servicio FFmpeg**, con su propio video y su propio `STREAM_NAME`. Todas publican en el mismo servidor `mediamtx`, cada una en una ruta distinta.

El proyecto incluye un archivo listo para esto: **`docker-compose.multi-camera.yml`**, con tres cámaras.

### Por qué no alcanza con duplicar el servicio

Lo intuitivo sería copiar `video-stream` y poner valores distintos en `environment:` de cada copia. **Eso no funciona:**

```yaml title="❌ Incorrecto"
  camera-1:
    environment:
      - VIDEO_FILE=camera1.mp4
      - STREAM_NAME=camera1
    command: >
      -re -stream_loop -1 -i /videos/${VIDEO_FILE}
      ... rtsp://mediamtx:8554/${STREAM_NAME}
```

Docker Compose reemplaza `${VIDEO_FILE}` y `${STREAM_NAME}` **antes de crear el contenedor**, tomando los valores del `.env`, no del bloque `environment:` del servicio. El resultado es que todas las cámaras leen el mismo `VIDEO_FILE` del `.env` y publican en el mismo `STREAM_NAME`, pisándose entre sí.

!!! warning "`environment:` no alimenta a `${...}` en el `command`"
    Las variables de `environment:` solo existen **dentro** del contenedor en ejecución. La sintaxis `${VAR}` del `docker-compose.yml` se resuelve antes, con el `.env` o las variables del sistema.

### La solución: variables por cámara en el `.env`

Se resuelve con dos cambios:

1. **Cada cámara tiene sus propias variables en el `.env`**, con un prefijo distinto (`CAMERA1_`, `CAMERA2_`, ...).
2. **El `command` lee las variables dentro del contenedor** (`$$VAR`, ejecutado con `/bin/sh -c`), así cada servicio usa los valores de su propio `environment:`.

#### 1. `.env`

```env title=".env"
VIDEO_SOURCE=./videos
VIDEO_CODEC=libx264

CAMERA1_FILE=camera1.mp4
CAMERA1_STREAM=camera1

CAMERA2_FILE=camera2.mp4
CAMERA2_STREAM=camera2

CAMERA3_FILE=camera3.mp4
CAMERA3_STREAM=camera3
```

| Variable | Descripción |
|---|---|
| `VIDEO_SOURCE` | Carpeta local con los videos. Compartida por todas las cámaras. |
| `VIDEO_CODEC` | Codec de salida. Compartido por todas las cámaras. |
| `CAMERAn_FILE` | Archivo de video de la cámara `n` (debe existir dentro de `VIDEO_SOURCE`). |
| `CAMERAn_STREAM` | Ruta RTSP de la cámara `n`. **Debe ser única** entre todas las cámaras. |

!!! info "¿Es obligatorio el `.env`?"
    Para `docker-compose.multi-camera.yml`, sí: el archivo lee `VIDEO_SOURCE`, `VIDEO_CODEC` y todas las `CAMERAn_*` desde el `.env`. Si falta alguna, Compose avisa con `The "CAMERA1_FILE" variable is not set` y la cámara arranca sin archivo o sin ruta.

    Técnicamente también se podrían escribir los valores fijos dentro de `environment:` (por ejemplo `VIDEO_FILE=camera1.mp4`), ya que el `command` usa `$$VIDEO_FILE`. Pero tenerlos en el `.env` deja **toda la configuración en un solo lugar** y no obliga a tocar el YAML para cambiar un video.

#### 2. Un servicio por cámara

Cada cámara de `docker-compose.multi-camera.yml` sigue este patrón:

```yaml title="docker-compose.multi-camera.yml (una cámara)"
  camera-1:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: camera1            # (1)
    depends_on:
      - mediamtx
    volumes:
      - ${VIDEO_SOURCE}:/videos
    environment:                       # (2)
      - VIDEO_FILE=${CAMERA1_FILE}
      - VIDEO_CODEC=${VIDEO_CODEC}
      - STREAM_NAME=${CAMERA1_STREAM}
    entrypoint: ["/bin/sh", "-c"]      # (3)
    command:
      - >
        ffmpeg -re -stream_loop -1 -i "/videos/$$VIDEO_FILE"
        -map 0:v:0 -map "0:a:0?" -c:v "$$VIDEO_CODEC" -preset veryfast
        -c:a aac -ac 2 -f
        rtsp "rtsp://mediamtx:8554/$$STREAM_NAME"
```

1. **`container_name` único**: Docker no permite dos contenedores con el mismo nombre.
2. **`environment:`**: Compose toma `${CAMERA1_FILE}` y `${CAMERA1_STREAM}` del `.env` y los pasa al contenedor con nombres genéricos (`VIDEO_FILE`, `STREAM_NAME`). Así el `command` es idéntico en todas las cámaras.
3. **`entrypoint` + `$$VAR`**: `$$` hace que Compose deje un `$` literal sin reemplazar. El shell dentro del contenedor expande `$VIDEO_FILE` con el valor de **esa** cámara. Por eso el comando empieza con `ffmpeg` y se ejecuta con `/bin/sh -c`.

!!! note "Por qué `0:a:0?` va entre comillas"
    Como ahora el comando pasa por `sh`, el `?` se interpretaría como comodín de archivos. Las comillas evitan eso.

#### 3. Levantar las cámaras

Coloca los videos en `./videos` con los nombres definidos en el `.env`:

```
videos/
├── camera1.mp4
├── camera2.mp4
└── camera3.mp4
```

Y levanta el compose de múltiples cámaras indicándolo con `-f`:

```bash
docker compose -f docker-compose.multi-camera.yml up -d
docker compose -f docker-compose.multi-camera.yml ps
```

Las URLs resultantes son `rtsp://<IP>:8554/<CAMERAn_STREAM>`:

- `rtsp://<IP>:8554/camera1`
- `rtsp://<IP>:8554/camera2`
- `rtsp://<IP>:8554/camera3`

Para detenerlas:

```bash
docker compose -f docker-compose.multi-camera.yml down
```

!!! warning "No levantar ambos compose a la vez"
    `docker-compose.yml` y `docker-compose.multi-camera.yml` definen el mismo contenedor `mediamtx` y el mismo puerto `8554`. Antes de levantar uno, baja el otro con `docker compose down` (o `docker compose -f docker-compose.multi-camera.yml down`).

### Agregar una cámara más

1. Agrega sus variables al `.env`:

    ```env
    CAMERA4_FILE=camera4.mp4
    CAMERA4_STREAM=camera4
    ```

2. Copia el bloque `camera-3` en `docker-compose.multi-camera.yml` y cambia:
    - el nombre del servicio → `camera-4`
    - `container_name` → `camera4`
    - `${CAMERA3_FILE}` → `${CAMERA4_FILE}`
    - `${CAMERA3_STREAM}` → `${CAMERA4_STREAM}`

3. Vuelve a ejecutar `docker compose -f docker-compose.multi-camera.yml up -d`. Solo se crea el contenedor nuevo.

### Errores comunes

| Síntoma | Causa |
|---|---|
| Todas las cámaras muestran el mismo video | El `command` usa `${VIDEO_FILE}` en lugar de `$$VIDEO_FILE`. |
| Solo funciona una cámara; las demás se reconectan en bucle | Dos cámaras con el mismo `CAMERAn_STREAM`: MediaMTX acepta un solo publicador por ruta. |
| `The "CAMERAn_FILE" variable is not set` | Falta la variable en el `.env`. |
| `Conflict. The container name "/cameraN" is already in use` | `container_name` repetido, o quedó un contenedor de un `up` anterior. |
| `No such file or directory` en los logs de la cámara | El archivo de `CAMERAn_FILE` no existe dentro de `VIDEO_SOURCE` (revisa mayúsculas y extensión). |

Para ver los logs de una cámara concreta:

```bash
docker compose -f docker-compose.multi-camera.yml logs -f camera-1
```

---

## Parámetros relevantes de FFmpeg

| Parámetro | Descripción |
|---|---|
| `-re` | Lee el archivo a velocidad de reproducción real (simula una cámara en vivo). |
| `-stream_loop -1` | Repite el video indefinidamente. |
| `-i /videos/${VIDEO_FILE}` | Archivo de video de entrada, tomado de la variable `VIDEO_FILE`. |
| `-map 0:v:0` | Selecciona el primer stream de video del archivo fuente. |
| `-map 0:a:0?` | Selecciona el primer stream de audio, si existe (el `?` lo hace opcional para videos sin audio). |
| `-c:v ${VIDEO_CODEC}` | Codec de video de salida (`libx264` o `libx265`), tomado de `VIDEO_CODEC`. |
| `-preset veryfast` | Velocidad de codificación: rápida con calidad aceptable. |
| `-c:a aac -ac 2` | Recodifica el audio a AAC estéreo para máxima compatibilidad. |
| `-f rtsp` | Formato de salida RTSP. |

!!! note "¿Por qué recodificar en lugar de `-c copy`?"
    La copia directa (`-c copy`) falla cuando el archivo fuente usa codecs incompatibles con RTSP (como AC3 o HEVC sin perfil compatible). La recodificación a H.264/AAC garantiza reproducción en VLC y la mayoría de clientes RTSP.
