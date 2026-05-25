# Emulación de Cámara IP con Docker Compose

Este proyecto permite **emular una cámara IP** utilizando [MediaMTX](https://github.com/bluenviron/mediamtx) como servidor RTSP y [FFmpeg](https://ffmpeg.org/) para enviar un video local.  
El resultado es un stream RTSP accesible desde cualquier cliente compatible (ej. VLC, ffprobe).

---

## Cómo funciona

- **Servicio `mediamtx`**: levanta un servidor RTSP en el puerto `8554` con protocolo TCP forzado.
- **Servicio `video-stream`**: usa FFmpeg para leer el archivo definido en `.env` (`VIDEO_FILE`), recodificarlo a H.264/AAC y publicarlo en MediaMTX.
- El codec (`VIDEO_CODEC`), el nombre del stream (`STREAM_NAME`) y la carpeta de videos (`VIDEO_SOURCE`) se configuran en el archivo `.env`.
- La URL resultante es: `rtsp://<IP_de_tu_maquina>:8554/<STREAM_NAME>` donde `<IP_de_tu_maquina>` es la IP real de tu host.

---

## Uso

### 1. Preparar el video

Colocar un archivo de video en la carpeta `./videos`. Los formatos soportados son `.mp4`, `.mkv` y `.avi`.

```
videos/
└── demo.mp4   ← tu archivo aquí
```

### 2. Configurar el archivo `.env`

Crear un archivo `.env` en la raíz del proyecto con las siguientes variables:

```env
VIDEO_SOURCE=./videos
VIDEO_FILE=demo.mp4
VIDEO_CODEC=libx264
STREAM_NAME=mystream
```

| Variable | Descripción | Ejemplo |
|---|---|---|
| `VIDEO_SOURCE` | Carpeta local que contiene los videos | `./videos` |
| `VIDEO_FILE` | Nombre del archivo de video a transmitir | `demo.mp4` |
| `VIDEO_CODEC` | Codec de video para recodificación | `libx264` (H.264) o `libx265` (H.265) |
| `STREAM_NAME` | Ruta del stream en el servidor RTSP | `mystream` |

### 3. Levantar los servicios

Ejecutar:
```bash
docker compose up -d
```

Esto inicia:

- `mediamtx` (servidor RTSP)
- `video-stream` (FFmpeg recodificando y enviando el video en bucle)

### 4. Probar el stream con ffprobe

Para verificar que el stream está activo (reemplazar `mystream` por el valor de `STREAM_NAME` si lo cambiaste):

```bash
ffprobe -rtsp_transport tcp rtsp://<IP_de_tu_maquina>:8554/mystream
```

Salida esperada (ejemplo):

```bash
Stream #0:0: Video: h264, 1280x720, 25 fps
Stream #0:1: Audio: aac, 48000 Hz, stereo
```

### 5. Visualizar el stream con VLC

Abrir VLC → Medio → Abrir ubicación de red → pegar la URL:

```bash
rtsp://<IP_de_tu_maquina>:8554/mystream
```

El video se reproducirá como si fuera una cámara IP.

## Notas importantes

- Usar `-rtsp_transport tcp` en pruebas para evitar problemas de pérdida de paquetes con UDP.
- El video se recodifica a **H.264/AAC** por defecto para máxima compatibilidad con VLC y clientes RTSP. Esto consume más CPU que `-c copy`, pero evita problemas de codec.
- Para usar H.265 basta con cambiar `VIDEO_CODEC=libx265` en el `.env`. Verificar que el cliente receptor soporte H.265.
- Si el video es muy corto, conviene usar uno más largo (1–2 minutos) o un patrón de prueba infinito.
- Se pueden definir múltiples streams duplicando el servicio `video-stream` con variables de entorno distintas por servicio.

## Ejemplo de configuración

**`.env`**

```env
VIDEO_SOURCE=./videos
VIDEO_FILE=demo.mp4
VIDEO_CODEC=libx264
STREAM_NAME=mystream
```

**`docker-compose.yml`**

```yaml
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
```

## Casos de uso

- Emular cámaras para pruebas de integración con sistemas de video.
- Validar pipelines de ingestión RTSP sin depender de hardware real.
- Simular múltiples cámaras con distintos videos.

---

## Documentación

La documentación completa está generada con [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) y puede levantarse localmente con Docker, sin instalar nada extra:

```bash
docker compose up docs
```

El sitio estará disponible en: `http://localhost:8000`

> El servicio `docs` no arranca junto a los demás servicios. Solo se levanta cuando se lo llama explícitamente por nombre.
