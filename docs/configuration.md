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
      -map 0:v:0 -map 0:a:0 -c:v ${VIDEO_CODEC} -preset veryfast
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

Para simular varias cámaras, duplica el servicio `video-stream` y sobreescribe las variables de entorno por servicio. Cada uno puede apuntar a un video y stream distintos:

```yaml title="docker-compose.yml (múltiples cámaras)"
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    restart: unless-stopped
    container_name: mediamtx
    environment:
      - MTX_PROTOCOLS=tcp
    ports:
      - "8554:8554"

  camera-1:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-camera-1
    depends_on:
      - mediamtx
    volumes:
      - ./videos:/videos
    environment:
      - VIDEO_FILE=camera1.mp4
      - VIDEO_CODEC=libx264
      - STREAM_NAME=camera1
    command: >
      -re -stream_loop -1 -i /videos/${VIDEO_FILE}
      -map 0:v:0 -map 0:a:0 -c:v ${VIDEO_CODEC} -preset veryfast
      -c:a aac -ac 2 -f
      rtsp rtsp://mediamtx:8554/${STREAM_NAME}

  camera-2:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-camera-2
    depends_on:
      - mediamtx
    volumes:
      - ./videos:/videos
    environment:
      - VIDEO_FILE=camera2.mkv
      - VIDEO_CODEC=libx264
      - STREAM_NAME=camera2
    command: >
      -re -stream_loop -1 -i /videos/${VIDEO_FILE}
      -map 0:v:0 -map 0:a:0 -c:v ${VIDEO_CODEC} -preset veryfast
      -c:a aac -ac 2 -f
      rtsp rtsp://mediamtx:8554/${STREAM_NAME}
```

Las URLs resultantes serían:

- `rtsp://<IP>:8554/camera1`
- `rtsp://<IP>:8554/camera2`

---

## Parámetros relevantes de FFmpeg

| Parámetro | Descripción |
|---|---|
| `-re` | Lee el archivo a velocidad de reproducción real (simula una cámara en vivo). |
| `-stream_loop -1` | Repite el video indefinidamente. |
| `-i /videos/${VIDEO_FILE}` | Archivo de video de entrada, tomado de la variable `VIDEO_FILE`. |
| `-map 0:v:0` | Selecciona el primer stream de video del archivo fuente. |
| `-map 0:a:0` | Selecciona el primer stream de audio del archivo fuente. |
| `-c:v ${VIDEO_CODEC}` | Codec de video de salida (`libx264` o `libx265`), tomado de `VIDEO_CODEC`. |
| `-preset veryfast` | Velocidad de codificación: rápida con calidad aceptable. |
| `-c:a aac -ac 2` | Recodifica el audio a AAC estéreo para máxima compatibilidad. |
| `-f rtsp` | Formato de salida RTSP. |

!!! note "¿Por qué recodificar en lugar de `-c copy`?"
    La copia directa (`-c copy`) falla cuando el archivo fuente usa codecs incompatibles con RTSP (como AC3 o HEVC sin perfil compatible). La recodificación a H.264/AAC garantiza reproducción en VLC y la mayoría de clientes RTSP.
