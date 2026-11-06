## 08. Videopodcast: Sindicació i Subscripció de Vídeo

### Sindicació de Vídeo

La sindicació de vídeo funciona igual que la d'àudio (Tema 7, apartat 08). Hi ha un *feed* RSS que fa la llista dels episodis, i els clients s'hi subscriuen i descarreguen els episodis nous. Només canvien els fitxers (MP4 en lloc de MP3) i el tipus MIME de l'element `<enclosure>`:

```xml
<enclosure url="https://video.educem.cat/videopodcast/episodis/videoepisodi-01.mp4"
           length="18734512" type="video/mp4"/>
```

**Altres formes de sindicació de vídeo:**

- **Plataformes de vídeo:** cada canal de YouTube té un *feed* RSS amb els seus vídeos nous (`https://www.youtube.com/feeds/videos.xml?channel_id=...`). Els lectors de *feeds* el poden seguir sense compte.
- **PeerTube:** cada canal té *feeds* RSS i Atom. A més, els servidors PeerTube es federen amb el protocol **ActivityPub**: un usuari d'un servidor pot seguir un canal d'un altre servidor, com al correu o a XMPP (Tema 6).
- **Subscripció a les plataformes:** el botó de subscriure's d'una plataforma fa el mateix dins del seu sistema, però només funciona amb els seus usuaris.

### Preparar els Episodis

Els episodis han de ser MP4 amb H.264 i AAC, amb `faststart`, i han de tenir el títol a les metadades. Per exemple, la gravació del directe de l'apartat 07 i un fragment de la pel·lícula de prova:

```bash
cd ~/video

ffmpeg -i classe.mp4 -c copy -movflags +faststart \
    -metadata title="Classe 1: Introducció al vídeo digital" videoepisodi-01.mp4

ffmpeg -i bbb-360p.mp4 -t 60 -c copy -movflags +faststart \
    -metadata title="Episodi 2: Big Buck Bunny (fragment)" videoepisodi-02.mp4

sudo cp videoepisodi-0*.mp4 /var/www/video/videopodcast/episodis/

# Portada quadrada de 1.400 × 1.400 píxels
sudo ffmpeg -f lavfi -i color=c=0xa0321e:s=1400x1400 -frames:v 1 /var/www/video/videopodcast/portada.jpg
```

### Generar el Feed

Fem servir l'script `genera-feed.sh` del Tema 7, adaptat al vídeo. `sed` en canvia els camins, l'adreça, l'extensió dels fitxers i el tipus MIME:

```bash
sudo cp /usr/local/bin/genera-feed.sh /usr/local/bin/genera-videofeed.sh

sudo sed -i \
    -e 's#/var/www/radio/podcast#/var/www/video/videopodcast#' \
    -e 's#https://radio.educem.cat/podcast#https://video.educem.cat/videopodcast#' \
    -e 's#\.mp3#.mp4#g' \
    -e 's#audio/mpeg#video/mp4#' \
    -e 's#https://radio.educem.cat/#https://video.educem.cat/#' \
    /usr/local/bin/genera-videofeed.sh

# Comprovar els canvis respecte de l'script original
diff /usr/local/bin/genera-feed.sh /usr/local/bin/genera-videofeed.sh
```

Canvieu també el títol, la descripció i l'autor del canal (`<title>`, `<description>` i `<itunes:author>`):

```bash
sudo nano /usr/local/bin/genera-videofeed.sh
```

```xml
    <title>El videopodcast d'EDUCEM</title>
    <link>https://video.educem.cat/</link>
    <description>Classes i vídeos del laboratori d'ASIX.</description>
    <language>ca</language>
    <itunes:author>EDUCEM</itunes:author>
```

Genereu el *feed* i valideu-lo:

```bash
sudo genera-videofeed.sh
xmllint --noout /var/www/video/videopodcast/feed.xml
curl -k -s https://video.educem.cat/videopodcast/feed.xml | head -25
```

### Subscriure's

Al client, primer cal confiar en el certificat de `video.educem.cat`, com al Tema 7 (`openssl s_client`, comparar l'empremta i `update-ca-certificates`).

- **gPodder:** **Subscriptions** → **Add podcast via URL** → `https://video.educem.cat/videopodcast/feed.xml`. Descarregueu un episodi i obriu-lo: gPodder fa servir el reproductor de vídeo del sistema.
- **VLC:** a la llista de reproducció, **Internet** → **Podcasts** → **+** i l'adreça del *feed*. Reprodueix els episodis sense descarregar-los.
- **Thunderbird:** com a *feed* (**File** → **New** → **Feed Account**).

**Prova:** publiqueu un tercer episodi, torneu a executar `genera-videofeed.sh` i comproveu que gPodder el detecta amb **Check for new episodes**.

### Ample de Banda i Emmagatzematge

Un episodi de vídeo ocupa molt més que un d'àudio. Per exemple, 30 minuts a 2,5 Mbps són 2.500.000 × 1.800 / 8 = 562,5 MB. Si se'l descarreguen 200 persones, són 112,5 GB de trànsit. Per això, els videopodcasts acostumen a oferir una versió de resolució més baixa o a publicar-se en plataformes de vídeo.

### Punt de Control

- [ ] Hi ha dos episodis MP4 amb títol i `faststart` a `/var/www/video/videopodcast/episodis/`.
- [ ] `genera-videofeed.sh` genera un *feed* vàlid amb `type="video/mp4"`.
- [ ] gPodder o VLC se subscriuen al videopodcast i reprodueixen els episodis.
