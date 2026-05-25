# Primeros Pasos

## 1. Preparar el video

Coloca un archivo de video en la carpeta `./videos`. Los formatos soportados son `.mp4`, `.mkv` y `.avi`:

```
videos/
└── demo.mp4   ← tu archivo aquí
```

!!! tip "¿No tienes un video de prueba?"
    Puedes descargar **Big Buck Bunny** (el video de prueba estándar de la industria) desde [peach.blender.org](https://peach.blender.org/download/) o buscar clips gratuitos en [Pexels Videos](https://www.pexels.com/videos/).  
    También puedes generar un patrón de color infinito con `ffmpeg`:
    ```bash
    ffmpeg -f lavfi -i testsrc=duration=60:size=1280x720:rate=25 -c:v libx264 videos/demo.mp4
    ```

---

## 2. Configurar el archivo `.env`

Crea un archivo `.env` en la raíz del proyecto con el siguiente contenido:

```env
VIDEO_SOURCE=./videos
VIDEO_FILE=demo.mp4
VIDEO_CODEC=libx264
STREAM_NAME=mystream
```

| Variable | Descripción |
|---|---|
| `VIDEO_SOURCE` | Carpeta local donde están los videos (montada en `/videos` dentro del contenedor). |
| `VIDEO_FILE` | Nombre del archivo a transmitir. |
| `VIDEO_CODEC` | Codec de salida: `libx264` (H.264) o `libx265` (H.265). |
| `STREAM_NAME` | Ruta del stream en el servidor RTSP. |

---

## 3. Levantar los servicios

```bash
docker compose up -d
```

Esto inicia:

- **`mediamtx`** — servidor RTSP en el puerto `8554`
- **`video-stream`** — FFmpeg recodificando el video a H.264/AAC y enviańdolo en bucle

Verifica que ambos contenedores estén corriendo:

```bash
docker compose ps
```

---

## 4. Verificar el stream con ffprobe

```bash
ffprobe -rtsp_transport tcp rtsp://<IP_de_tu_maquina>:8554/mystream
```

> Reemplaza `mystream` por el valor de `STREAM_NAME` si lo cambiaste en el `.env`.

Salida esperada:

```
Stream #0:0: Video: h264, 1280x720, 25 fps
Stream #0:1: Audio: aac, 48000 Hz, stereo
```

!!! warning "Usa `-rtsp_transport tcp`"
    Con UDP pueden ocurrir pérdidas de paquetes en algunas redes. TCP es más confiable para pruebas locales.

---

## 5. Visualizar con VLC

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
