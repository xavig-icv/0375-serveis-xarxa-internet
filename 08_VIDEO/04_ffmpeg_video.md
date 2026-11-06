## 04. Conversió de Vídeo amb ffmpeg

### Preparació

Al servidor 0375 (ffmpeg ja s'hi va instal·lar al Tema 7):

```bash
sudo apt install ffmpeg unzip
mkdir -p ~/video && cd ~/video

# Codificadors de vídeo disponibles
ffmpeg -hide_banner -encoders | grep -E 'libx264|libx265|libvpx|libsvtav1|libaom'
```

### Vídeos de Proves

**1. Vídeo generat per ffmpeg:** 30 segons de 1280×720 a 30 fotogrames per segon, amb una imatge de prova en moviment i un to de 1 kHz:

```bash
ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=30 -f lavfi -i sine=frequency=1000:sample_rate=48000 \
    -t 30 -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k prova-720p.mp4
```

La imatge de prova mostra un comptador de temps. Ens servirà per mesurar el retard del directe (apartat 07).

**2. Pel·lícula lliure:** *Big Buck Bunny* (© Blender Foundation | peach.blender.org, llicència CC BY 3.0), en 640×360:

```bash
wget https://download.blender.org/peach/bigbuckbunny_movies/BigBuckBunny_640x360.m4v.zip
unzip BigBuckBunny_640x360.m4v.zip
ls -lh BigBuckBunny_640x360.m4v
```

### Informació d'un Vídeo

```bash
ffprobe -hide_banner BigBuckBunny_640x360.m4v

# Només les dades principals de cada pista
ffprobe -v error \
    -show_entries stream=codec_type,codec_name,width,height,r_frame_rate,bit_rate:format=duration \
    -of default=noprint_wrappers=1 BigBuckBunny_640x360.m4v
```

Fixeu-vos en el còdec de cada pista (`codec_name`), la resolució, els fotogrames per segon (`r_frame_rate`), la durada i la taxa de bits.

**Tipus de fotogrames** (apartat 02) dels 10 primers segons:

```bash
ffprobe -v error -select_streams v -read_intervals %+10 \
    -show_entries frame=pict_type -of csv=p=0 BigBuckBunny_640x360.m4v | sort | uniq -c
```

La sortida mostra quants fotogrames I, P i B hi ha. Els I són pocs: un a l'inici de cada GOP.

### Convertir a H.264 per a la Web

```bash
ffmpeg -i BigBuckBunny_640x360.m4v \
    -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p \
    -c:a aac -b:a 128k -movflags +faststart \
    bbb-360p.mp4
```

| Opció                   | Significat                                                                             |
| ----------------------- | -------------------------------------------------------------------------------------- |
| `-c:v libx264`          | Còdec de vídeo H.264                                                                   |
| `-crf 23`               | **Qualitat constant** (de 0 a 51): 18 gairebé no es distingeix de l'original, 23 és el valor per defecte i 28 ja és de qualitat baixa. Cada 6 punts més, el fitxer ocupa aproximadament la meitat |
| `-preset medium`        | Velocitat de codificació: de `ultrafast` a `veryslow`. Amb la mateixa qualitat, els més lents fan fitxers més petits |
| `-pix_fmt yuv420p`      | Mostreig de color 4:2:0, el que accepten tots els reproductors                          |
| `-c:a aac -b:a 128k`    | Àudio AAC a 128 kbps                                                                   |
| `-movflags +faststart`  | Posa l'índex del fitxer (l'àtom `moov`) al principi. Així, el navegador pot començar a reproduir abans de descarregar-lo tot. **Imprescindible per a la web** |

### Canviar la Resolució

```bash
# De 720p a 360p: amb -2, ffmpeg calcula l'amplada i la fa parella
ffmpeg -i prova-720p.mp4 -vf scale=-2:360 -c:v libx264 -preset veryfast -crf 23 \
    -c:a copy -movflags +faststart prova-360p.mp4
```

Amb `-c:a copy`, l'àudio es copia sense tornar-lo a codificar.

### Taxa de Bits Limitada (per al Streaming)

Per als directes i l'streaming adaptatiu, interessa que la taxa de bits no superi un màxim i que hi hagi un fotograma I cada 2 segons:

```bash
ffmpeg -i prova-720p.mp4 \
    -c:v libx264 -preset veryfast -b:v 2500k -maxrate 2500k -bufsize 5000k -g 60 \
    -c:a aac -b:a 128k -movflags +faststart prova-720p-2500k.mp4
```

| Opció                     | Significat                                                                  |
| ------------------------- | --------------------------------------------------------------------------- |
| `-b:v 2500k`              | Taxa de bits mitjana del vídeo                                              |
| `-maxrate`, `-bufsize`    | Taxa màxima i memòria intermèdia que la controla: la taxa no supera mai els 2.500 kbps de manera sostinguda |
| `-g 60`                   | Mida del GOP: un fotograma I cada 60 fotogrames (2 segons a 30 fps)         |

### Comparar Còdecs

Codifiquem el primer minut de la pel·lícula amb quatre còdecs, amb una qualitat semblant i sense àudio (`-an`):

```bash
time ffmpeg -i BigBuckBunny_640x360.m4v -t 60 -c:v libx264 -preset medium -crf 23 -an bbb-h264.mp4
time ffmpeg -i BigBuckBunny_640x360.m4v -t 60 -c:v libx265 -preset medium -crf 28 -an bbb-h265.mp4
time ffmpeg -i BigBuckBunny_640x360.m4v -t 60 -c:v libvpx-vp9 -crf 33 -b:v 0 -row-mt 1 -an bbb-vp9.webm
time ffmpeg -i BigBuckBunny_640x360.m4v -t 60 -c:v libsvtav1 -preset 8 -crf 35 -an bbb-av1.webm

ls -lh bbb-h264.mp4 bbb-h265.mp4 bbb-vp9.webm bbb-av1.webm
```

- Les escales de qualitat (`-crf`) de cada còdec són diferents. Els valors d'aquests exemples donen una qualitat semblant.
- **VP9:** `-b:v 0` amb `-crf` activa el mode de qualitat constant, i `-row-mt 1` fa servir diversos nuclis.
- **AV1:** SVT-AV1 és el codificador més ràpid. `-preset` va de 0 (el més lent i eficient) a 13.

Ompliu aquesta taula i compareu també la qualitat, mirant els vídeos amb mpv o VLC al client:

| Còdec  | Mida | Temps de codificació | Qualitat percebuda |
| ------ | ---- | -------------------- | ------------------ |
| H.264  |      |                      |                    |
| H.265  |      |                      |                    |
| VP9    |      |                      |                    |
| AV1    |      |                      |                    |

Els còdecs moderns fan fitxers més petits, però tarden més a codificar. És el compromís entre la CPU del servidor, l'espai de disc i l'ample de banda.

### Portada i Miniatures

```bash
# Fotograma del segon 10, com a portada del vídeo a la web
ffmpeg -ss 10 -i bbb-360p.mp4 -frames:v 1 -q:v 2 bbb-poster.jpg

# Una miniatura cada 30 segons, de 320 píxels d'amplada
ffmpeg -i bbb-360p.mp4 -vf fps=1/30,scale=320:-2 miniatura-%03d.jpg
```

### Retallar i Extreure l'Àudio

```bash
# 30 segons a partir del minut 1, sense tornar a codificar
ffmpeg -ss 00:01:00 -i bbb-360p.mp4 -t 30 -c copy tros.mp4

# Només l'àudio, sense tornar-lo a codificar
ffmpeg -i bbb-360p.mp4 -vn -c:a copy bbb-audio.m4a
```

Amb `-c copy`, el tall només pot començar en un fotograma I. Per això el principi del tros pot no ser exactament el segon indicat. Per tallar exactament, cal tornar a codificar.

### Subtítols (WebVTT)

Els subtítols de la web es guarden en fitxers **WebVTT** (`.vtt`), separats del vídeo. Així es poden oferir en diversos idiomes i l'usuari els pot activar o desactivar. Creeu `bbb.ca.vtt`:

```
WEBVTT

00:00:01.000 --> 00:00:05.000
Big Buck Bunny - © Blender Foundation | peach.blender.org

00:00:10.000 --> 00:00:14.000
Subtítol de prova en català.
```

Els farem servir a l'apartat 05.

### Conversió per Lots

L'script `prepara-web.sh` converteix tots els vídeos d'una carpeta a MP4 per a la web i en treu una portada:

```bash
#!/bin/bash
# prepara-web.sh - Converteix els vídeos d'una carpeta a MP4 per a la web (H.264 + AAC)
# i en treu una imatge de portada.
# Ús: ./prepara-web.sh carpeta_origen carpeta_desti   (han de ser carpetes diferents)

origen="$1"
desti="$2"
mkdir -p "$desti"

for f in "$origen"/*.mp4 "$origen"/*.m4v "$origen"/*.mov "$origen"/*.mkv; do
    [ -e "$f" ] || continue
    nom=$(basename "${f%.*}")
    echo "Convertint $nom..."
    ffmpeg -nostdin -loglevel error -y -i "$f" \
        -c:v libx264 -preset medium -crf 23 -pix_fmt yuv420p \
        -c:a aac -b:a 128k -movflags +faststart "$desti/$nom.mp4"
    ffmpeg -nostdin -loglevel error -y -ss 5 -i "$desti/$nom.mp4" \
        -frames:v 1 -q:v 2 "$desti/$nom.jpg"
done

ls -lh "$desti"
```

### Resolució de Problemes

| Problema                                                    | Causa                                                      | Solució                                                     |
| ----------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| El navegador reprodueix el so però no la imatge             | Còdec de vídeo no suportat (per exemple, H.265)            | Convertir a H.264 (`libx264`)                               |
| El vídeo no comença fins que s'ha descarregat sencer        | L'índex (`moov`) és al final del fitxer                    | Tornar a generar-lo amb `-movflags +faststart`              |
| `width not divisible by 2`                                  | H.264 amb 4:2:0 necessita dimensions parelles              | Escalar amb `-2` (`scale=-2:360`)                           |
| El vídeo es veu en alguns reproductors però no en d'altres | Mostreig de color 4:4:4 o 4:2:2                            | Afegir `-pix_fmt yuv420p`                                    |
| La codificació és molt lenta                                | Preset lent, resolució alta o còdec AV1                    | Preset més ràpid (`veryfast`) o resolució més baixa          |
| `Unknown encoder 'libsvtav1'`                               | Versió d'ffmpeg sense aquest codificador                   | `ffmpeg -encoders \| grep av1`; fer servir `libaom-av1`     |

### Punt de Control

- [ ] Teniu `prova-720p.mp4` i la pel·lícula de prova, i n'heu consultat les dades amb `ffprobe`.
- [ ] Heu vist quants fotogrames I, P i B té un fragment de vídeo.
- [ ] `bbb-360p.mp4` està en H.264 + AAC, amb `faststart`.
- [ ] Heu omplert la taula comparativa dels quatre còdecs.
- [ ] Teniu la portada i el fitxer de subtítols `bbb.ca.vtt`.
