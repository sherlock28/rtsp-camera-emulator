# Emulación de Cámara IP con Docker Compose

Este proyecto permite **emular una cámara IP** utilizando [MediaMTX](https://github.com/bluenviron/mediamtx) como servidor RTSP y [FFmpeg](https://ffmpeg.org/) para transmitir un video local como si fuera una cámara real.

---

## ¿Cómo funciona?

```mermaid
graph LR
    A[demo.mp4] -->|FFmpeg lee y reenvía| B(MediaMTX RTSP Server)
    B -->|rtsp://host:8554/mystream| C[VLC / ffprobe / cliente RTSP]
```

| Servicio | Descripción |
|---|---|
| `mediamtx` | Servidor RTSP que recibe y redistribuye el stream, expuesto en el puerto `8554`. |
| `video-stream` | Contenedor FFmpeg que lee `demo.mp4` en bucle y lo publica en MediaMTX. |

La URL resultante del stream es:

```
rtsp://<IP_de_tu_maquina>:8554/mystream
```

!!! info "Reemplaza `<IP_de_tu_maquina>`"
    Usa la IP real del host donde corre Docker, no `localhost`, para que clientes externos puedan conectarse.

---

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) y [Docker Compose](https://docs.docker.com/compose/) instalados.
- Un archivo de video `demo.mp4` colocado en la carpeta `./videos`.
