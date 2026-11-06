## 05. Vídeo sota Demanda amb Apache

### Com Funciona

La manera més senzilla de servir vídeo sota demanda és la **descàrrega progressiva**: el vídeo és un fitxer MP4 en un servidor web normal.

- Gràcies a `faststart` (apartat 04), el navegador llegeix l'índex del fitxer i comença a reproduir de seguida.
- Per saltar a un altre punt del vídeo, el navegador demana només un tros del fitxer amb una **petició per rangs** (capçalera `Range`). El servidor respon amb el codi **206 Partial Content**.

```
 Navegador                                          Apache
    │ GET /videos/bbb-360p.mp4   Range: bytes=0-        │
    │───────────────────────────────────────────────────►│
    │ 206 Partial Content (des del principi)            │
    │◄───────────────────────────────────────────────────│
    │                 ... l'usuari salta al minut 5 ... │
    │ GET /videos/bbb-360p.mp4   Range: bytes=38000000- │
    │───────────────────────────────────────────────────►│
    │ 206 Partial Content (des d'aquell punt)           │
    │◄───────────────────────────────────────────────────│
```

Apache i Nginx accepten les peticions per rangs sense cap configuració. L'inconvenient d'aquest mètode és que cada vídeo té una sola qualitat: si la connexió de l'usuari és lenta, el vídeo s'atura per carregar. L'streaming adaptatiu ho resol (apartat 06).

### El Lloc Web `video.educem.cat`

**1. Directoris i fitxers:**

```bash
sudo mkdir -p /var/www/video/{videos,hls,directe,videopodcast/episodis,js}
sudo cp ~/video/bbb-360p.mp4 ~/video/bbb-poster.jpg ~/video/bbb.ca.vtt /var/www/video/videos/
```

Opcionalment, per oferir també un format lliure, codifiqueu la pel·lícula sencera en VP9. Pot tardar uns minuts:

```bash
cd ~/video
ffmpeg -i BigBuckBunny_640x360.m4v -c:v libvpx-vp9 -crf 33 -b:v 0 -row-mt 1 \
    -c:a libopus -b:a 96k bbb-360p.webm
sudo cp bbb-360p.webm /var/www/video/videos/
```

**2. Certificat i lloc web:**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/video.educem.cat.key \
    -out /etc/ssl/certs/video.educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=video.educem.cat" \
    -addext "subjectAltName=DNS:video.educem.cat"

sudo nano /etc/apache2/sites-available/video.educem.cat.conf
```

```apache
<VirtualHost *:443>
    ServerName video.educem.cat
    DocumentRoot /var/www/video

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/video.educem.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/video.educem.cat.key

    # Tipus MIME de l'streaming HLS (apartats 06 i 07)
    AddType application/vnd.apple.mpegurl .m3u8
    AddType video/mp2t .ts

    ErrorLog ${APACHE_LOG_DIR}/video.educem.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/video.educem.cat-access.log combined
</VirtualHost>
```

**Per què cal `AddType video/mp2t .ts`?** Apache treu els tipus MIME de `/etc/mime.types`, i a Ubuntu aquest fitxer assigna l'extensió `.ts` a un altre tipus de fitxer: les traduccions de Qt (`text/vnd.trolltech.linguist`). Sense aquesta línia, els fragments de vídeo de l'HLS s'enviarien com si fossin text.

**3. La pàgina amb el reproductor:**

```bash
sudo nano /var/www/video/index.html
```

```html
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Vídeos EDUCEM</title>
</head>
<body>
  <h1>Vídeos EDUCEM</h1>

  <h2>Big Buck Bunny</h2>
  <!-- El navegador fa servir el primer format que sap reproduir -->
  <video controls preload="metadata" width="640" poster="videos/bbb-poster.jpg">
    <source src="videos/bbb-360p.webm" type="video/webm">
    <source src="videos/bbb-360p.mp4" type="video/mp4">
    <track kind="subtitles" src="videos/bbb.ca.vtt" srclang="ca" label="Català" default>
    El navegador no pot reproduir vídeo.
  </video>
  <p>© Blender Foundation | peach.blender.org (CC BY 3.0)</p>

  <ul>
    <li><a href="hls.html">Streaming adaptatiu (HLS)</a></li>
    <li><a href="directe.html">En directe</a></li>
    <li><a href="videopodcast/feed.xml">Videopodcast (RSS)</a></li>
  </ul>
</body>
</html>
```

| Atribut de `<video>`   | Funció                                                                          |
| ---------------------- | ------------------------------------------------------------------------------- |
| `controls`             | Mostra els botons de reproducció, volum, pantalla completa i subtítols          |
| `preload`              | Què es descarrega abans de prémer el botó: `none` (res), `metadata` (durada i dimensions) o `auto` |
| `poster`               | Imatge que es mostra abans de començar                                          |
| `autoplay muted`       | Reproducció automàtica. Els navegadors només la permeten si el vídeo comença sense so |
| `<source>`             | Formats alternatius, per ordre de preferència                                   |
| `<track>`              | Subtítols en WebVTT. Se'n poden posar diversos, un per idioma                   |

Si `bbb-360p.webm` no existeix, el navegador passa al MP4.

**4. Activar el lloc:**

```bash
sudo a2ensite video.educem.cat.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Proves

**Des del client:** obriu `https://video.educem.cat`, accepteu l'avís del certificat i reproduïu el vídeo. Activeu els subtítols i salteu a diferents punts.

**Peticions per rangs amb curl:**

```bash
# Capçaleres: tipus MIME, mida i acceptació de rangs
curl -k -s -I https://video.educem.cat/videos/bbb-360p.mp4 \
    | grep -i -E '^HTTP|content-type|content-length|accept-ranges'

# Demanar només 1 MB a partir del byte 1.000.000
curl -k -s -o /dev/null -r 1000000-1999999 -w '%{http_code} %{size_download}\n' \
    https://video.educem.cat/videos/bbb-360p.mp4
# 206 1000000
```

**On és l'índex del fitxer:** amb `faststart`, l'àtom `moov` (l'índex) va abans de l'`mdat` (les dades):

```bash
ffprobe -v trace ~/video/bbb-360p.mp4 2>&1 | grep -E "type:'(moov|mdat)'" | head -2
```

**Connexió lenta:** a les eines de desenvolupament del navegador (`F12`), la pestanya **Network** permet simular una connexió lenta (*throttling*). Amb una connexió més lenta que la taxa de bits del vídeo, la reproducció s'atura cada poc per carregar.

**Al registre d'Apache** hi ha les peticions amb codi 206:

```bash
sudo tail /var/log/apache2/video.educem.cat-access.log
```

### Accés Restringit

Si els vídeos són només per a l'alumnat d'un curs, es pot protegir el directori amb usuari i contrasenya o per adreça IP, igual que al Tema 3 (apartats 13 i 14):

```apache
<Directory /var/www/video/videos>
    AuthType Basic
    AuthName "Vídeos del curs"
    AuthUserFile /etc/apache2/.htpasswd-video
    Require valid-user
</Directory>
```

### Resolució de Problemes

| Problema                                                  | Causa                                                        | Solució                                                      |
| --------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| El vídeo no comença fins que s'ha descarregat sencer      | Fitxer sense `faststart`                                     | Tornar a generar-lo amb `-movflags +faststart` (apartat 04)  |
| Es veu el reproductor buit, amb un error de format        | Còdec no suportat, o no hi ha cap `<source>` compatible      | Oferir MP4 amb H.264 + AAC                                    |
| Els subtítols no surten                                   | Error al fitxer VTT o camí incorrecte                        | El fitxer ha de començar per `WEBVTT`; comprovar el camí     |
| No es pot saltar a un altre punt del vídeo                | El servidor no accepta rangs                                 | `curl -I` ha de mostrar `Accept-Ranges: bytes`               |
| El vídeo s'atura cada poc per carregar                    | La connexió és més lenta que la taxa de bits                 | Oferir una resolució més baixa o streaming adaptatiu (apartat 06) |

### Punt de Control

- [ ] `https://video.educem.cat` mostra el vídeo amb la portada i els subtítols.
- [ ] `curl -r` retorna el codi 206 i la mida demanada.
- [ ] `ffprobe -v trace` mostra el `moov` abans que l'`mdat`.
- [ ] Heu vist l'efecte d'una connexió lenta amb les eines del navegador.
