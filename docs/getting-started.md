# Primeros Pasos

## 1. Preparar el video

Coloca un archivo de video en la carpeta `./videos` con el nombre `demo.mp4`:

```
video/
└── videos/
    └── demo.mp4   ← tu archivo aquí
```

!!! tip "¿No tienes un video de prueba?"
    Puedes descargar uno de muestra desde [sample-videos.com](https://sample-videos.com/) o usar `ffmpeg` para generar un patrón de color:
    ```bash
    ffmpeg -f lavfi -i testsrc=duration=60:size=1280x720:rate=25 -c:v libx264 videos/demo.mp4
    ```

---

## 2. Levantar los servicios

```bash
docker compose up -d
```

Esto inicia:

- **`mediamtx`** — servidor RTSP en el puerto `8554`
- **`video-stream`** — FFmpeg enviando el video en bucle

Verifica que ambos contenedores estén corriendo:

```bash
docker compose ps
```

---

## 3. Verificar el stream con ffprobe

```bash
ffprobe -rtsp_transport tcp rtsp://<IP_de_tu_maquina>:8554/mystream
```

Salida esperada:

```
Stream #0:0: Video: h264, 1280x720, 25 fps
Stream #0:1: Audio: aac, 48000 Hz, stereo
```

!!! warning "Usa `-rtsp_transport tcp`"
    Con UDP pueden ocurrir pérdidas de paquetes en algunas redes. TCP es más confiable para pruebas locales.

---

## 4. Visualizar con VLC

1. Abre VLC.
2. Ve a **Medio → Abrir ubicación de red**.
3. Pega la URL:

```
rtsp://<IP_de_tu_maquina>:8554/mystream
```

El video se reproducirá como si fuera una cámara IP en vivo.

---

## Detener los servicios

```bash
docker compose down
```
