# Emulación de Cámara IP con Docker Compose

Este proyecto permite **emular una cámara IP** utilizando [MediaMTX](https://github.com/bluenviron/mediamtx) como servidor RTSP y [FFmpeg](https://ffmpeg.org/) para enviar un video local.  
El resultado es un stream RTSP accesible desde cualquier cliente compatible (ej. VLC, ffprobe).

---

## Cómo funciona

- **Servicio `mediamtx`**: levanta un servidor RTSP en el puerto `8554`.
- **Servicio `video-stream`**: usa FFmpeg para leer un archivo de video (`demo.mp4`) y lo publica en el servidor bajo la ruta `/mystream`.
- La URL resultante es: `rtsp://<IP_de_tu_maquina>:8554/mystream` donde `<IP_de_tu_maquina>` es la IP real de tu host.

---

## Uso

### 1. Preparar el video

Colocar un archivo de prueba en la carpeta `./videos` con el nombre `demo.mp4`.  
Ejemplo:

`./videos/demo.mp4`

### 2. Levantar los servicios

Ejecutar:
```bash
docker compose up -d
```

Esto inicia:

- `mediamtx` (servidor RTSP)
- `video-stream` (FFmpeg enviando el video)

### 3. Probar el stream con ffprobe

Para verificar que el stream está activo:

```bash
ffprobe -rtsp_transport tcp rtsp://<IP_de_tu_maquina>:8554/mystream
```

Salida esperada (ejemplo):

```bash
Stream #0:0: Video: h264, 1280x720, 25 fps
Stream #0:1: Audio: aac, 48000 Hz, stereo
```

### 4. Visualizar el stream con VLC

Abrir VLC → Medio → Abrir ubicación de red → pegar la URL:

```bash
rtsp://<IP_de_tu_maquina>:8554/mystream
```

El video se reproducirá como si fuera una cámara IP.

## Notas importantes

- Usar `-rtsp_transport tcp` en pruebas para evitar problemas de pérdida de paquetes con UDP.
- Si el video es muy corto, conviene usar uno más largo (1–2 minutos) o un patrón de prueba infinito.
- Se pueden definir múltiples streams duplicando el servicio video-stream con distinto nombre y ruta RTSP.

## Ejemplo de configuración

```yaml
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
