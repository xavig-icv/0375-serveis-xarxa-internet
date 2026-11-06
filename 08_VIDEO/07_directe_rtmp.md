## 07. Vídeo en Directe: Nginx RTMP i OBS

### Com Funciona un Directe

Un directe té dues parts:

- **Ingesta:** la font (OBS Studio, una càmera o un mesclador de vídeo) envia una sola emissió al servidor.
- **Distribució:** el servidor la converteix a un format que els espectadors puguin reproduir, normalment HLS, i la reparteix.

```
 OBS Studio (client)  ──RTMP 1935──►  Nginx RTMP (0375)  ──escriu──►  /var/www/video/directe/
 (pantalla, càmera,                        │                        classe.m3u8 + fragments
  fitxer de vídeo)                         │ grava l'emissió                 │
                                           ▼                                 ▼
                                  /srv/video/gravacions/           Apache (HTTPS) ──► navegador (hls.js),
                                                                                       VLC, mpv
```

| Protocol   | Funció       | Transport | Latència habitual | Característiques                                                  |
| ---------- | ------------ | --------- | ----------------- | ----------------------------------------------------------------- |
| **RTMP**   | Ingesta      | TCP 1935  | 2-5 s             | Ve de l'antic Flash. Totes les plataformes i OBS el fan servir per rebre directes |
| **SRT**    | Ingesta      | UDP       | Menys d'1 s       | Modern; recupera els paquets perduts. Ideal per a connexions inestables |
| **WHIP**   | Ingesta      | WebRTC    | Menys d'1 s       | Envia el directe amb WebRTC                                       |
| **HLS / DASH** | Distribució | HTTP   | 10-30 s           | Arriba a milers d'espectadors amb servidors web normals i CDN      |
| **LL-HLS** | Distribució  | HTTP      | 2-5 s             | HLS de baixa latència                                             |
| **WebRTC** | Distribució  | UDP       | Menys d'0,5 s     | Interactiu (videoconferència, apartats 09 i 10)                   |

### Instal·lar i Configurar el Mòdul RTMP de Nginx

El Nginx del servidor 0375 (Tema 3) es converteix en servidor de directes amb el mòdul RTMP:

```bash
sudo apt install libnginx-mod-rtmp
ls /etc/nginx/modules-enabled/ | grep rtmp       # el paquet activa el mòdul
```

La configuració RTMP va en un bloc `rtmp { }`, que ha d'estar **fora del bloc `http { }`**. La posem en un fitxer propi:

```bash
sudo nano /etc/nginx/rtmp.conf
```

```nginx
# Servidor de directes: rep RTMP i el converteix a HLS
rtmp {
    server {
        listen 1935;

        application live {
            live on;

            # Qui pot emetre i qui pot mirar per RTMP
            allow publish 192.168.1.0/24;
            deny publish all;
            allow play 192.168.1.0/24;
            deny play all;

            # Conversió a HLS: fragments de 2 s; la llista conté els últims 20 s
            hls on;
            hls_path /var/www/video/directe;
            hls_fragment 2s;
            hls_playlist_length 20s;

            # Gravació de cada emissió, amb la data al nom del fitxer
            record all;
            record_path /srv/video/gravacions;
            record_unique on;
        }
    }
}
```

| Directiva                        | Funció                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------- |
| `listen 1935`                    | Port RTMP                                                                       |
| `application live`               | Nom de l'aplicació: l'adreça d'emissió serà `rtmp://video.educem.cat/live`       |
| `live on`                        | Mode directe: una font, molts espectadors                                       |
| `allow` / `deny publish`         | Només es pot emetre des de la xarxa interna. S'aplica la primera regla que coincideix |
| `hls on`, `hls_path`             | Genera l'HLS en aquest directori. Cada emissió hi crea una llista amb el seu nom (`classe.m3u8`) |
| `hls_fragment`, `hls_playlist_length` | Durada dels fragments i de la llista. Nginx esborra sol els fragments antics |
| `record all`, `record_path`      | Grava l'àudio i el vídeo de cada emissió en un fitxer FLV                       |
| `record_unique on`               | Afegeix l'hora al nom de la gravació, perquè no se sobreescrigui                |

Afegiu el fitxer al **final** de `nginx.conf`, fora del bloc `http`, prepareu els directoris i reinicieu Nginx:

```bash
echo 'include /etc/nginx/rtmp.conf;' | sudo tee -a /etc/nginx/nginx.conf

# Nginx s'executa amb l'usuari www-data: ha de poder escriure-hi
sudo mkdir -p /srv/video/gravacions
sudo chown www-data:www-data /var/www/video/directe /srv/video/gravacions

sudo nginx -t
sudo systemctl restart nginx
sudo ss -tlnp | grep ':1935 '

sudo ufw allow from 192.168.1.0/24 to any port 1935 proto tcp
```

Els fragments HLS els escriu Nginx, però els serveix l'Apache del lloc `video.educem.cat` (apartat 05), que ja té el tipus MIME dels fitxers `.ts`.

### Emetre amb OBS Studio

**OBS Studio** és el programa lliure de referència per fer directes i gravacions.

```bash
sudo apt install obs-studio      # al client
obs &
```

1. **Fonts** (*Sources*): afegiu el que voleu emetre. **Screen Capture (PipeWire)** captura la pantalla; **Video Capture Device** captura una càmera web; i **Media Source** reprodueix un fitxer de vídeo, útil si la màquina virtual no té càmera.
2. **Settings** → **Stream**:

| Camp            | Valor                            |
| --------------- | -------------------------------- |
| **Service**     | Custom...                        |
| **Server**      | `rtmp://video.educem.cat/live`   |
| **Stream Key**  | `classe`                         |

3. **Settings** → **Video:** resolució de sortida 1280×720 i 30 fotogrames per segon.
4. **Settings** → **Output**, mode **Advanced**, pestanya **Streaming**: codificador x264, control de taxa **CBR**, **2500 Kbps** i **Keyframe Interval: 2 s**. L'interval de fotogrames clau ha de coincidir amb els fragments HLS.
5. **Start Streaming**.

La clau (*stream key*) és el nom de l'emissió: amb `classe`, l'HLS es diu `classe.m3u8`.

**Sense OBS** (o per simular una càmera), ffmpeg pot fer d'emissor des del client. La imatge de prova mostra un rellotge:

```bash
ffmpeg -re -f lavfi -i testsrc2=size=1280x720:rate=30 -f lavfi -i sine=frequency=1000:sample_rate=48000 \
    -c:v libx264 -preset veryfast -b:v 2500k -maxrate 2500k -bufsize 5000k -g 60 -pix_fmt yuv420p \
    -c:a aac -b:a 128k -ar 44100 \
    -f flv rtmp://video.educem.cat/live/classe
```

### Veure el Directe

**Pàgina web** amb hls.js (`/var/www/video/directe.html`):

```html
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>En directe - Vídeos EDUCEM</title>
  <script src="js/hls.min.js"></script>
</head>
<body>
  <h1>En directe</h1>
  <video id="video" controls autoplay muted width="640"></video>
  <p id="estat">Connectant...</p>

  <script>
    const video = document.getElementById('video');
    const font = 'directe/classe.m3u8';

    if (Hls.isSupported()) {
      const hls = new Hls();
      hls.loadSource(font);
      hls.attachMedia(video);
      hls.on(Hls.Events.MANIFEST_PARSED, () => {
        document.getElementById('estat').textContent = 'En directe';
      });
      hls.on(Hls.Events.ERROR, (esdeveniment, dades) => {
        if (dades.fatal) {
          document.getElementById('estat').textContent = 'Ara no hi ha cap emissió en directe.';
        }
      });
    } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
      video.src = font;
    }
  </script>
</body>
</html>
```

**Reproductors:**

```bash
mpv https://video.educem.cat/directe/classe.m3u8      # HLS, a través d'Apache
mpv rtmp://video.educem.cat/live/classe               # RTMP directe (menys latència)
```

**La llista del directe es va actualitzant.** Executeu aquesta ordre unes quantes vegades:

```bash
curl -k -s https://video.educem.cat/directe/classe.m3u8
```

El número de `#EXT-X-MEDIA-SEQUENCE` augmenta i apareixen fragments nous. No hi ha `#EXT-X-ENDLIST`, perquè el vídeo encara no s'ha acabat.

### Latència

Compareu el rellotge de la imatge de prova a l'emissor i als reproductors:

| Reproducció                         | Latència aproximada | Motiu                                                     |
| ----------------------------------- | ------------------- | --------------------------------------------------------- |
| RTMP directe (mpv)                  | 2-5 s               | El servidor reenvia el flux tal com arriba                |
| HLS (navegador, mpv)                | 10-20 s             | El reproductor espera tenir uns quants fragments descarregats |

Amb fragments més curts baixa la latència, però augmenten les peticions i la mida de les capçaleres. Per a una conversa en temps real calen altres tecnologies (WebRTC, apartat 09).

### Gravacions

Quan acaba l'emissió, la gravació queda en FLV. Es converteix a MP4 sense tornar-la a codificar i es pot publicar com a vídeo sota demanda o al videopodcast (apartat 08):

```bash
ls -lh /srv/video/gravacions/
ffmpeg -i /srv/video/gravacions/classe-XXXXXXXXXX.flv -c copy -movflags +faststart ~/video/classe.mp4
```

Substituïu `XXXXXXXXXX` pel número que hi ha al nom del fitxer.

### Seguretat del Directe

- **Qui pot emetre:** amb `allow publish` només s'emet des de la xarxa interna. La clau de l'emissió no és una contrasenya, sinó el nom de l'emissió. Per exigir una clau secreta, el mòdul pot consultar una aplicació web abans d'acceptar cada emissió (`on_publish`).
- **Xifratge:** RTMP no va xifrat. Per emetre des de fora de la xarxa interna, cal RTMPS (RTMP sobre TLS) o SRT amb xifratge.
- **Tallafoc:** el port 1935 només s'obre a la xarxa interna.

### Resolució de Problemes

| Problema                                                     | Causa                                                       | Solució                                                       |
| ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------- |
| `nginx -t`: `unknown directive "rtmp"`                       | El mòdul no està carregat                                   | `sudo apt install libnginx-mod-rtmp`; comprovar `modules-enabled` |
| `nginx -t`: `"rtmp" directive is not allowed here`           | L'`include` és dins del bloc `http`                         | L'`include` ha d'anar al final de `nginx.conf`, fora de `http { }` |
| OBS diu `Failed to connect to server`                        | Port 1935 tancat, Nginx aturat o IP no permesa              | `sudo ss -tlnp \| grep 1935`, `sudo ufw status`, `allow publish` |
| No es crea `classe.m3u8`                                     | Nginx no pot escriure a `hls_path`                          | `chown www-data` al directori; `/var/log/nginx/error.log`     |
| El reproductor es queda carregant                            | Encara no hi ha prou fragments, o OBS no emet               | Esperar uns segons; comprovar `ls /var/www/video/directe/`    |
| La latència és molt alta                                     | Fragments llargs o interval de fotogrames clau massa gran   | Fragments de 2 s i fotograma clau cada 2 s a OBS              |

### Punt de Control

- [ ] Nginx escolta al port 1935 i `nginx -t` no dona errors.
- [ ] OBS (o ffmpeg) emet a `rtmp://video.educem.cat/live` amb la clau `classe`.
- [ ] `https://video.educem.cat/directe.html` mostra el directe.
- [ ] Heu comparat la latència de l'HLS i de l'RTMP amb el rellotge de la imatge de prova.
- [ ] L'emissió queda gravada i l'heu convertit a MP4.
