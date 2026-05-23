# Configuración

## docker-compose.yml base

```yaml title="docker-compose.yml"
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    container_name: mediamtx
    ports:
      - "8554:8554"
    restart: unless-stopped

  video-stream:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-stream
    depends_on:
      - mediamtx
    volumes:
      - ./videos:/videos
    command: >
      -re -stream_loop -1 -i /videos/demo.mp4
      -c copy -f rtsp rtsp://mediamtx:8554/mystream
```

---

## Múltiples streams (varias cámaras)

Duplica el servicio `video-stream` con un nombre de contenedor y ruta RTSP distintos por cada cámara simulada:

```yaml title="docker-compose.yml (múltiples cámaras)"
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    container_name: mediamtx
    ports:
      - "8554:8554"
    restart: unless-stopped

  camera-1:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-camera-1
    depends_on:
      - mediamtx
    volumes:
      - ./videos:/videos
    command: >
      -re -stream_loop -1 -i /videos/camera1.mp4
      -c copy -f rtsp rtsp://mediamtx:8554/camera1

  camera-2:
    image: jrottenberg/ffmpeg:4.4-ubuntu
    container_name: ffmpeg-camera-2
    depends_on:
      - mediamtx
    volumes:
      - ./videos:/videos
    command: >
      -re -stream_loop -1 -i /videos/camera2.mp4
      -c copy -f rtsp rtsp://mediamtx:8554/camera2
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
| `-i /videos/demo.mp4` | Archivo de video de entrada. |
| `-c copy` | Copia los streams sin recodificar (bajo consumo de CPU). |
| `-f rtsp` | Formato de salida RTSP. |

!!! note "Transcodificación"
    Si el video no es H.264/AAC, reemplaza `-c copy` por `-c:v libx264 -c:a aac` para que FFmpeg recodifique al vuelo. Esto consume más CPU.
