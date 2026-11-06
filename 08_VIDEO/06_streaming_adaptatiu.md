## 06. Streaming Adaptatiu (HLS)

### Què és l'Streaming Adaptatiu?

Amb la descàrrega progressiva (apartat 05), cada vídeo té una sola qualitat. A l'**streaming adaptatiu**:

1. El vídeo es codifica en **diverses qualitats** (una "escala": per exemple, 720p i 360p).
2. Cada qualitat es divideix en **fragments** de pocs segons.
3. Unes **llistes de reproducció** indiquen quins fragments hi ha de cada qualitat.
4. El reproductor mesura la velocitat de la connexió i, fragment a fragment, tria la millor qualitat que pot descarregar a temps. Si la xarxa s'alenteix, baixa de qualitat sense aturar el vídeo.

```
 master.m3u8 (llista mestra: quines qualitats hi ha)
   ├── 0/index.m3u8  → 0/seg000.ts  0/seg001.ts  0/seg002.ts ...    720p, 2.800 kbps
   └── 1/index.m3u8  → 1/seg000.ts  1/seg001.ts  1/seg002.ts ...    360p,   800 kbps

 Reproductor:  seg000 (720p) → seg001 (720p) → [la xarxa s'alenteix] → seg002 (360p) → ...
```

| Protocol | Qui el defineix         | Llistes                | Fragments                    | Reproductors                                   |
| -------- | ----------------------- | ---------------------- | ---------------------------- | ---------------------------------------------- |
| **HLS**  | Apple (RFC 8216)        | `.m3u8` (text)         | `.ts` (MPEG-TS) o `.m4s` (MP4) | Safari nativament; altres navegadors amb hls.js; VLC, mpv |
| **DASH** | MPEG (estàndard ISO)    | `.mpd` (XML)           | `.m4s` (MP4)                 | dash.js, Shaka Player; VLC, mpv                |

- Tots dos funcionen sobre **HTTP normal**: qualsevol servidor web els pot servir, i les xarxes de distribució de continguts (CDN) els poden guardar a la memòria cau.
- Inconvenient: el vídeo ocupa més disc, perquè es guarda una còpia per a cada qualitat.

### Les Llistes de Reproducció

**Llista mestra** (`master.m3u8`): una entrada per qualitat, amb la taxa de bits i la resolució. Exemple:

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=3220000,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
0/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1020000,RESOLUTION=640x360,CODECS="avc1.64001e,mp4a.40.2"
1/index.m3u8
```

**Llista d'una qualitat** (`0/index.m3u8`): els fragments, amb la durada de cadascun. `#EXT-X-ENDLIST` indica que el vídeo s'ha acabat (vídeo sota demanda). Un directe no en té, i la llista es va actualitzant (apartat 07).

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:4.000000,
seg000.ts
#EXTINF:4.000000,
seg001.ts
...
#EXT-X-ENDLIST
```

### Generar HLS amb Dues Qualitats

Fem servir un vídeo de prova de 2 minuts en 720p, perquè l'escala sigui de 720p i 360p (no té sentit generar una qualitat més alta que l'original):

```bash
cd ~/video
ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=30 -f lavfi -i sine=frequency=1000:sample_rate=48000 \
    -t 120 -c:v libx264 -pix_fmt yuv420p -c:a aac -b:a 128k prova-2min.mp4

mkdir -p hls-prova && cd hls-prova

ffmpeg -i ../prova-2min.mp4 \
    -filter_complex "[0:v]split=2[v1][v2];[v1]scale=-2:720[v720];[v2]scale=-2:360[v360]" \
    -map "[v720]" -map "[v360]" -map 0:a -map 0:a \
    -c:v libx264 -preset veryfast -pix_fmt yuv420p -g 120 -keyint_min 120 -sc_threshold 0 \
    -b:v:0 2800k -maxrate:v:0 2800k -bufsize:v:0 5600k \
    -b:v:1 800k -maxrate:v:1 800k -bufsize:v:1 1600k \
    -c:a aac -b:a 128k -ac 2 \
    -f hls -hls_time 4 -hls_playlist_type vod \
    -hls_segment_filename "%v/seg%03d.ts" \
    -master_pl_name master.m3u8 \
    -var_stream_map "v:0,a:0 v:1,a:1" "%v/index.m3u8"

ls -R | head -20
cat master.m3u8
```

| Opció                                     | Significat                                                                 |
| ----------------------------------------- | -------------------------------------------------------------------------- |
| `split=2`, `scale=-2:720`, `scale=-2:360` | Fa dues còpies de la imatge i les escala a 720 i 360 línies                |
| `-map ...`                                | Sortides: vídeo 720p (`v:0`), vídeo 360p (`v:1`) i dues còpies de l'àudio (`a:0` i `a:1`) |
| `-b:v:0 2800k`, `-b:v:1 800k`             | Taxa de cada qualitat, amb `-maxrate` i `-bufsize` per limitar-la          |
| `-g 120 -keyint_min 120 -sc_threshold 0`  | Un fotograma I exactament cada 4 segons (120 fotogrames a 30 fps). Cada fragment comença amb un fotograma I, i el reproductor pot canviar de qualitat en qualsevol fragment |
| `-hls_time 4`                             | Durada dels fragments, en segons                                           |
| `-hls_playlist_type vod`                  | Llista de vídeo sota demanda (acaba amb `#EXT-X-ENDLIST`)                  |
| `-var_stream_map "v:0,a:0 v:1,a:1"`       | Quines pistes formen cada qualitat                                         |
| `%v`                                      | Número de la qualitat: crea els directoris `0/` i `1/`                     |
| `-master_pl_name master.m3u8`             | Crea la llista mestra al directori actual                                  |

Publiqueu-lo al lloc web de l'apartat 05:

```bash
sudo cp -r ~/video/hls-prova /var/www/video/hls/prova
```

### El Reproductor Web amb hls.js

Safari reprodueix HLS directament, però Firefox i Chrome necessiten la biblioteca de JavaScript **hls.js**. La descarreguem al servidor, en lloc de carregar-la d'un servidor extern cada vegada:

```bash
sudo curl -L -o /var/www/video/js/hls.min.js https://cdn.jsdelivr.net/npm/hls.js@1.7.3/dist/hls.min.js
sudo nano /var/www/video/hls.html
```

```html
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Streaming adaptatiu - Vídeos EDUCEM</title>
  <script src="js/hls.min.js"></script>
</head>
<body>
  <h1>Streaming adaptatiu (HLS)</h1>
  <video id="video" controls width="640"></video>
  <p>Qualitat actual: <span id="qualitat">-</span></p>

  <script>
    const video = document.getElementById('video');
    const font = 'hls/prova/master.m3u8';

    if (Hls.isSupported()) {
      // Firefox, Chrome i Edge: hls.js descarrega les llistes i els fragments
      const hls = new Hls();
      hls.loadSource(font);
      hls.attachMedia(video);
      // Cada vegada que canvia la qualitat, la mostra a la pàgina
      hls.on(Hls.Events.LEVEL_SWITCHED, (esdeveniment, dades) => {
        const nivell = hls.levels[dades.level];
        document.getElementById('qualitat').textContent =
          nivell.height + 'p (' + Math.round(nivell.bitrate / 1000) + ' kbps)';
      });
    } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
      // Safari reprodueix HLS sense biblioteca
      video.src = font;
    }
  </script>
</body>
</html>
```

### Proves

**Navegador** (client): obriu `https://video.educem.cat/hls.html`.

1. Amb la connexió normal, la qualitat puja a 720p.
2. A les eines de desenvolupament (`F12`) → **Network**, simuleu una connexió lenta. Al cap de pocs fragments, la pàgina mostra que ha baixat a 360p, i el vídeo no s'ha aturat.
3. A la llista de peticions de **Network**, es veu com el navegador demana els fragments `0/segNNN.ts` o `1/segNNN.ts`, segons la qualitat.

**Reproductors:**

```bash
mpv https://video.educem.cat/hls/prova/master.m3u8                      # la millor qualitat
mpv --hls-bitrate=min https://video.educem.cat/hls/prova/master.m3u8    # la qualitat més baixa
```

**Tipus MIME** (l'`AddType` de l'apartat 05):

```bash
curl -k -s -I https://video.educem.cat/hls/prova/master.m3u8 | grep -i content-type   # application/vnd.apple.mpegurl
curl -k -s -I https://video.educem.cat/hls/prova/0/seg000.ts | grep -i content-type   # video/mp2t
```

### DASH

ffmpeg també genera DASH (`-f dash`), amb una llista `.mpd` en XML i fragments `.m4s`. És l'estàndard internacional, i el fan servir moltes plataformes de vídeo. Els navegadors el reprodueixen amb biblioteques com dash.js o Shaka Player. Al laboratori ens quedem amb HLS, perquè és el que genera el servidor de directes de l'apartat 07.

### Resolució de Problemes

| Problema                                                  | Causa                                                    | Solució                                                       |
| --------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------- |
| El reproductor no carrega res i la consola diu `manifestLoadError` | Camí de la llista incorrecte (error 404)           | Comprovar que existeix `/var/www/video/hls/prova/master.m3u8` |
| La consola diu que `Hls` no està definit                  | No s'ha carregat `js/hls.min.js`                         | Comprovar que el fitxer s'ha descarregat                      |
| Els fragments arriben com a `text/vnd.trolltech.linguist` | Falta `AddType video/mp2t .ts`                           | Afegir-lo al lloc web (apartat 05) i recarregar Apache        |
| La qualitat no canvia mai                                 | Connexió massa ràpida o una sola qualitat               | Simular una connexió lenta; revisar la llista mestra          |
| Talls en canviar de qualitat                              | Els fragments no comencen amb un fotograma I            | Fer servir `-g`, `-keyint_min` i `-sc_threshold 0` coherents amb `-hls_time` |

### Punt de Control

- [ ] Heu generat l'HLS amb dues qualitats i la llista mestra en mostra les dues.
- [ ] Apache serveix les llistes i els fragments amb els tipus MIME correctes.
- [ ] `hls.html` reprodueix el vídeo amb hls.js i mostra la qualitat actual.
- [ ] En simular una connexió lenta, el reproductor baixa a 360p sense aturar-se.
