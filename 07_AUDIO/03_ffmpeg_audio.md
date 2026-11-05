## 03. Conversió i Edició d'Àudio amb ffmpeg

**ffmpeg** és l'eina de referència per convertir i processar àudio i vídeo. Funciona per línia d'ordres, de manera que es pot automatitzar amb scripts, i moltes plataformes de streaming el fan servir internament. En aquest tema també farem servir:

- **ffprobe** (del mateix paquet), per consultar la informació dels fitxers.
- **espeak-ng**, per generar veu sintètica en català.
- **Audacity**, un editor gràfic, al client.

### Instal·lació

Al servidor 0375 i al client:

```bash
sudo apt update
sudo apt install ffmpeg sox mediainfo espeak-ng

# Codificadors d'àudio disponibles
ffmpeg -hide_banner -encoders | grep -E 'mp3|aac|opus|vorbis|flac'
```

Han d'aparèixer `libmp3lame` (MP3), `aac`, `libopus`, `libvorbis` i `flac`.

### Àudio de Proves Sense Problemes de Drets

**1. Tons generats per ffmpeg.** Un to de 440 Hz (la nota la) de 30 segons:

```bash
mkdir -p ~/audio && cd ~/audio
ffmpeg -f lavfi -i "sine=frequency=440:duration=30" -ac 2 to-440.wav
```

`-f lavfi` indica que l'entrada no és un fitxer sinó un generador intern d'ffmpeg.

**2. Veu sintètica en català** amb espeak-ng:

```bash
espeak-ng -v ca -s 150 -w presentacio.wav "Benvinguts a Ràdio EDUCEM, la ràdio del laboratori d'ASIX."
```

- `-v ca`: veu catalana. `espeak-ng --voices=ca` mostra les veus disponibles.
- `-s 150`: velocitat, en paraules per minut.
- `-w`: desa el resultat en un fitxer WAV en lloc de reproduir-lo.

**3. Gravacions pròpies.** Amb Audacity al client (vegeu el final de l'apartat).

**4. Música amb llicència Creative Commons**, per exemple de Wikimedia Commons, Free Music Archive o Jamendo. Cal comprovar la llicència de cada obra i respectar-la. Per exemple, citar l'autor si la llicència és `BY` (apartat 09).

### Informació d'un Fitxer

```bash
ffprobe -hide_banner presentacio.wav
mediainfo presentacio.wav

# Només les dades principals
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bit_rate:format=duration \
    -of default=noprint_wrappers=1 presentacio.wav
```

**Sortida esperada** (espeak-ng genera WAV mono a 22.050 Hz i 16 bits):

```
codec_name=pcm_s16le
sample_rate=22050
channels=1
bit_rate=352800
duration=...
```

La taxa de bits coincideix amb la fórmula de l'apartat 02: 22.050 × 16 × 1 = 352.800 bps.

### Convertir entre Formats

A partir d'un fitxer original sense pèrdua (`original.wav`):

```bash
# MP3 a 128 kbps, taxa constant (CBR)
ffmpeg -i original.wav -c:a libmp3lame -b:a 128k sortida-128.mp3

# MP3 de qualitat variable (VBR): -q:a va de 0 (millor) a 9 (pitjor)
ffmpeg -i original.wav -c:a libmp3lame -q:a 2 sortida-vbr.mp3

# AAC a 96 kbps, en contenidor MP4
ffmpeg -i original.wav -c:a aac -b:a 96k sortida.m4a

# Opus a 64 kbps, en contenidor Ogg
ffmpeg -i original.wav -c:a libopus -b:a 64k sortida.opus

# Vorbis de qualitat 5 (uns 160 kbps)
ffmpeg -i original.wav -c:a libvorbis -q:a 5 sortida.ogg

# FLAC, sense pèrdua
ffmpeg -i original.wav -c:a flac sortida.flac

ls -lh original.wav sortida*
```

| Opció         | Significat                                                                  |
| ------------- | --------------------------------------------------------------------------- |
| `-i fitxer`   | Fitxer d'entrada                                                            |
| `-c:a còdec`  | Còdec d'àudio de sortida (`copy`: copiar-lo sense tornar a codificar)       |
| `-b:a 128k`   | Taxa de bits de l'àudio                                                     |
| `-q:a N`      | Qualitat variable (VBR), si el còdec ho permet                              |
| `-ar 44100`   | Freqüència de mostreig de sortida                                           |
| `-ac 1`       | Nombre de canals de sortida (1: mono, 2: estèreo)                           |
| `-vn`         | Sense vídeo (per extreure l'àudio d'un vídeo)                               |
| `-y`          | Sobreescriure el fitxer de sortida sense preguntar                          |

**Mida aproximada d'1 minut de música** (segons les fórmules de l'apartat 02):

| Fitxer               | Mida aproximada                            |
| -------------------- | ------------------------------------------ |
| WAV (CD)             | 10,6 MB                                    |
| FLAC                 | 5-7 MB (depèn del contingut)               |
| MP3 a 128 kbps       | 0,96 MB                                    |
| AAC a 96 kbps        | 0,72 MB                                    |
| Opus a 64 kbps       | 0,48 MB                                    |

Escolteu els fitxers amb auriculars i compareu-ne la qualitat. A partir de quina taxa no noteu la diferència amb l'original?

### Canviar els Paràmetres del So

```bash
# Veu: mono, 22,05 kHz i 48 kbps
ffmpeg -i original.wav -ac 1 -ar 22050 -c:a libmp3lame -b:a 48k veu.mp3

# Extreure l'àudio d'un vídeo sense tornar-lo a codificar (si és AAC)
ffmpeg -i video.mp4 -vn -c:a copy audio.m4a
```

Amb `-c:a copy` no es perd qualitat i és instantani, però només funciona si el contenidor de sortida admet el còdec original.

### Retallar, Unir i Normalitzar el Volum

```bash
# Retallar: del segon 10 al 40
ffmpeg -i original.wav -ss 10 -to 40 -c copy tros.wav

# Unir 2 segons del to i la presentació (els dos passen a 44,1 kHz i estèreo)
ffmpeg -i to-440.wav -i presentacio.wav -filter_complex \
    "[0:a]atrim=0:2,aformat=sample_rates=44100:channel_layouts=stereo[a0]; \
     [1:a]aformat=sample_rates=44100:channel_layouts=stereo[a1]; \
     [a0][a1]concat=n=2:v=0:a=1[a]" \
    -map "[a]" unit.wav

# Normalitzar el volum a -16 LUFS
ffmpeg -i unit.wav -af loudnorm=I=-16:TP=-1.5:LRA=11 -ar 44100 normalitzat.wav
```

- **`-filter_complex`:** combina diverses entrades amb filtres. `atrim` retalla, `aformat` iguala el format i `concat` les uneix una darrere l'altra.
- **LUFS:** unitat de sonoritat percebuda. Si tots els episodis i les cançons tenen la mateixa sonoritat, l'oient no ha de tocar el volum. Valors habituals: -16 LUFS als podcasts i -23 LUFS a la televisió i la ràdio europees (EBU R128).
- El filtre `loudnorm` treballa internament a 192 kHz. Per això cal indicar `-ar 44100` a la sortida.

### Metadades

```bash
ffmpeg -i normalitzat.wav -c:a libmp3lame -b:a 128k \
    -metadata title="Presentació de Ràdio EDUCEM" \
    -metadata artist="Ràdio EDUCEM" \
    -metadata album="Proves del laboratori" \
    -metadata date="2026" \
    -id3v2_version 3 presentacio.mp3

ffprobe -v error -show_entries format_tags -of default=noprint_wrappers=1 presentacio.mp3
```

`-id3v2_version 3` escriu etiquetes ID3v2.3, que reconeixen tots els reproductors.

### Conversió per Lots

La ràdio automàtica de l'apartat 06 necessita que tots els fitxers tinguin el mateix format. Aquest script converteix tots els WAV i FLAC d'una carpeta a MP3 de 128 kbps, 44,1 kHz i estèreo:

```bash
#!/bin/bash
# converteix.sh - Converteix els WAV i FLAC d'una carpeta a MP3 (128 kbps, 44,1 kHz, estèreo)
# Ús: ./converteix.sh carpeta_origen carpeta_desti

origen="$1"
desti="$2"
mkdir -p "$desti"

for f in "$origen"/*.wav "$origen"/*.flac; do
    [ -e "$f" ] || continue              # cap fitxer d'aquest tipus
    nom=$(basename "${f%.*}")            # nom sense extensió
    echo "Convertint $nom..."
    ffmpeg -nostdin -loglevel error -y -i "$f" \
        -ar 44100 -ac 2 -c:a libmp3lame -b:a 128k "$desti/$nom.mp3"
done

ls -lh "$desti"
```

```bash
chmod +x converteix.sh
./converteix.sh ~/audio/originals ~/audio/mp3
```

`-nostdin` és important dins dels bucles: sense aquesta opció, ffmpeg llegeix l'entrada estàndard i pot interferir amb el bucle.

### Audacity (Editor Gràfic)

Al client, **Audacity** permet gravar amb el micròfon, retallar, aplicar efectes (normalitzar, reduir soroll) i exportar a WAV, FLAC, MP3 o Opus:

```bash
sudo apt install audacity
```

Per a una gravació de veu: grava en mono, a 44,1 kHz; normalitza amb **Effect** → **Loudness Normalization**; i exporta l'original en WAV i la versió per publicar en MP3.

### Resolució de Problemes

| Problema                                           | Causa                                                  | Solució                                                  |
| -------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------- |
| `Unknown encoder 'libmp3lame'`                     | Versió d'ffmpeg sense aquest codificador                | Fer servir el paquet d'Ubuntu; comprovar-ho amb `ffmpeg -encoders` |
| ffmpeg pregunta `File exists. Overwrite?`          | El fitxer de sortida ja existeix                       | Afegir `-y` o esborrar el fitxer                         |
| El bucle de l'script es comporta de manera estranya | ffmpeg llegeix l'entrada estàndard                    | Afegir `-nostdin`                                        |
| El fitxer normalitzat té 192 kHz                   | El filtre `loudnorm` treballa a 192 kHz                | Afegir `-ar 44100`                                       |
| `concat` dona un error de format                   | Les entrades tenen freqüències o canals diferents      | Igualar-les amb `aformat` abans d'unir-les               |
| espeak-ng no troba la veu                          | Nom de la veu incorrecte                               | `espeak-ng --voices=ca`                                  |

### Punt de Control

- [ ] `ffmpeg -encoders` mostra els codificadors de MP3, AAC, Opus, Vorbis i FLAC.
- [ ] Heu generat `presentacio.wav` amb espeak-ng i n'heu consultat els paràmetres amb `ffprobe`.
- [ ] Heu convertit un fitxer a MP3, AAC, Opus i FLAC, i n'heu comparat les mides.
- [ ] `presentacio.mp3` està normalitzat i té les metadades (títol i artista).
- [ ] L'script `converteix.sh` converteix una carpeta sencera.
