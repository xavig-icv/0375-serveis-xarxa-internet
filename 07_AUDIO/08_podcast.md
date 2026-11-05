## 08. Podcast: Sindicació i Subscripció

### Què és un Podcast?

Un **podcast** és una sèrie d'episodis d'àudio publicats en un servidor web, juntament amb un fitxer de **sindicació** (*feed*) que en fa la llista. El nom ve d'*iPod* i *broadcast* (2004).

- **Sindicació:** publicar un resum del contingut en un format estàndard, com RSS, perquè altres programes el puguin consultar.
- **Subscripció:** l'oient afegeix l'adreça del *feed* al seu client de podcasts. El client el consulta periòdicament i descarrega sol els episodis nous.

```
 PRODUCCIÓ                    SERVIDOR WEB (Apache)                         OIENTS
 gravar, editar i    ──►  /podcast/episodis/episodi-01.mp3      ◄── consulta periòdica ── gPodder,
 publicar l'episodi       /podcast/feed.xml (llista d'episodis)       del feed               AntennaPod,
                          https://radio.educem.cat/podcast/      ──► descàrrega dels         Apple Podcasts...
                                                                      episodis nous
```

Els **directoris** de podcasts (Apple Podcasts, Spotify, iVoox, Podcast Index) no allotgen l'àudio: només indexen el *feed*. Es dona d'alta l'adreça del *feed* una vegada, i el directori el va consultant. La mateixa tècnica serveix per al vídeo (*videopodcast*, Tema 8).

### Formats de Sindicació

| Format        | Descripció                                                                          |
| ------------- | ----------------------------------------------------------------------------------- |
| **RSS 2.0**   | El més utilitzat. Els podcasts el fan servir amb les etiquetes d'Apple (`itunes:`)  |
| **Atom**      | Estàndard de l'IETF (RFC 4287). Més rigorós; és habitual als blocs                  |
| **OPML**      | No és un *feed*: és una llista de subscripcions, per passar-les d'un client a un altre |

### Estructura del Feed RSS d'un Podcast

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd">
  <channel>
    <title>El podcast de Ràdio EDUCEM</title>
    <link>https://radio.educem.cat/</link>
    <description>Programes de la ràdio del laboratori d'ASIX.</description>
    <language>ca</language>
    <itunes:author>Ràdio EDUCEM</itunes:author>
    <itunes:image href="https://radio.educem.cat/podcast/portada.jpg"/>
    <itunes:category text="Technology"/>
    <itunes:explicit>false</itunes:explicit>

    <item>
      <title>Episodi 1: Què és el streaming?</title>
      <enclosure url="https://radio.educem.cat/podcast/episodis/episodi-01.mp3"
                 length="1440000" type="audio/mpeg"/>
      <guid isPermaLink="false">episodi-01.mp3</guid>
      <pubDate>Thu, 05 Nov 2026 10:00:00 +0100</pubDate>
      <itunes:duration>180</itunes:duration>
    </item>
  </channel>
</rss>
```

| Etiqueta                    | Funció                                                                          |
| --------------------------- | ------------------------------------------------------------------------------- |
| `<channel>`                 | El podcast: títol, web, descripció i idioma                                     |
| `<itunes:image>`            | Portada: imatge quadrada JPEG o PNG, de 1.400 a 3.000 píxels de costat          |
| `<itunes:category>`         | Categoria als directoris, amb els noms en anglès que defineix Apple (`Technology`, `Education`...) |
| `<itunes:explicit>`         | Si té contingut per a adults (`true` o `false`)                                 |
| `<item>`                    | Un episodi                                                                      |
| `<enclosure>`               | El fitxer d'àudio: adreça (`url`), mida en bytes (`length`) i tipus MIME (`type`) |
| `<guid>`                    | Identificador únic de l'episodi, que no ha de canviar mai. Així el client sap quins episodis ja té |
| `<pubDate>`                 | Data de publicació, en format RFC 2822 (dia de la setmana i mes en anglès)      |
| `<itunes:duration>`         | Durada, en segons o en format `HH:MM:SS`                                        |

**Normes que comproven els directoris:**

- Els caràcters `&`, `<` i `>` dels textos s'han d'escriure `&amp;`, `&lt;` i `&gt;`.
- Els noms dels fitxers han de tenir només lletres sense accents, números i guions.
- El servidor ha d'acceptar peticions `HEAD` i per trossos (*byte ranges*). Apache les accepta.

### Preparar els Episodis

Un episodi es pot gravar amb Audacity, sortir del directe de la ràdio (`dump-file`, apartat 06) o, per a les proves, generar-se amb veu sintètica. En tots els casos, se segueixen els mateixos passos: normalitzar el volum, codificar a MP3 i posar-hi les metadades (apartat 03).

**Episodi 1, amb veu sintètica:**

```bash
cd ~/audio
espeak-ng -v ca -s 150 -w episodi-01.wav "Benvinguts al primer episodi del podcast de Ràdio EDUCEM. Avui expliquem què és el streaming d'àudio i com funciona un servidor Icecast."

ffmpeg -i episodi-01.wav -af loudnorm=I=-16:TP=-1.5:LRA=11 -ar 44100 -ac 1 \
    -c:a libmp3lame -b:a 64k \
    -metadata title="Episodi 1: Què és el streaming?" \
    -metadata artist="Ràdio EDUCEM" -metadata album="El podcast de Ràdio EDUCEM" \
    -metadata genre="Podcast" -id3v2_version 3 \
    episodi-01.mp3
```

**Episodi 2, a partir de la gravació d'un directe** (del minut 1 al 31):

```bash
ls /srv/radio/gravacions/        # el nom depèn de la data i l'hora del directe

ffmpeg -i /srv/radio/gravacions/directe-2026-11-05-1030.mp3 -ss 00:01:00 -to 00:31:00 \
    -af loudnorm=I=-16:TP=-1.5:LRA=11 -ar 44100 -ac 1 -c:a libmp3lame -b:a 64k \
    -metadata title="Episodi 2: Programa en directe" -metadata artist="Ràdio EDUCEM" \
    -id3v2_version 3 episodi-02.mp3
```

**Publicar-los i crear la portada:**

```bash
sudo cp episodi-0*.mp3 /var/www/radio/podcast/episodis/

# Portada de 1.400 × 1.400 píxels d'un sol color, generada amb ffmpeg
sudo ffmpeg -f lavfi -i color=c=0x1e5aa0:s=1400x1400 -frames:v 1 /var/www/radio/podcast/portada.jpg
```

### Generar el Feed amb un Script

Escriure el *feed* a mà és lent i s'hi fan errors: la mida i la durada de cada episodi s'han de copiar exactament. Aquest script el genera a partir dels MP3 de la carpeta d'episodis:

```bash
sudo nano /usr/local/bin/genera-feed.sh
```

```bash
#!/bin/bash
# genera-feed.sh - Genera el feed RSS del podcast a partir dels MP3 de la carpeta d'episodis
# Ús: sudo genera-feed.sh

DIR=/var/www/radio/podcast
URL=https://radio.educem.cat/podcast
FEED="$DIR/feed.xml"

# Escapa els caràcters especials de l'XML: & < >
escapa() {
    sed -e 's/&/\&amp;/g' -e 's/</\&lt;/g' -e 's/>/\&gt;/g' <<< "$1"
}

if ! ls "$DIR"/episodis/*.mp3 > /dev/null 2>&1; then
    echo "No hi ha episodis a $DIR/episodis" >&2
    exit 1
fi

{
    cat <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd">
  <channel>
    <title>El podcast de Ràdio EDUCEM</title>
    <link>https://radio.educem.cat/</link>
    <description>Programes de la ràdio del laboratori d'ASIX.</description>
    <language>ca</language>
    <itunes:author>Ràdio EDUCEM</itunes:author>
    <itunes:image href="$URL/portada.jpg"/>
    <itunes:category text="Technology"/>
    <itunes:explicit>false</itunes:explicit>
EOF

    # Un element <item> per episodi, del més recent al més antic
    ls -t "$DIR"/episodis/*.mp3 | while read -r f; do
        nom=$(basename "$f")
        mida=$(stat -c %s "$f")
        durada=$(ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 "$f" | cut -d. -f1)
        titol=$(ffprobe -v error -show_entries format_tags=title -of default=nw=1:nk=1 "$f")
        data=$(LC_ALL=C date -R -r "$f")
        cat <<EOF
    <item>
      <title>$(escapa "${titol:-$nom}")</title>
      <enclosure url="$URL/episodis/$nom" length="$mida" type="audio/mpeg"/>
      <guid isPermaLink="false">$nom</guid>
      <pubDate>$data</pubDate>
      <itunes:duration>$durada</itunes:duration>
    </item>
EOF
    done

    cat <<EOF
  </channel>
</rss>
EOF
} > "$FEED"

echo "Feed generat: $FEED"
```

| Dada             | D'on la treu l'script                                                   |
| ---------------- | ----------------------------------------------------------------------- |
| Títol            | Etiqueta `title` de l'MP3 (`ffprobe`). Si no en té, el nom del fitxer   |
| Mida (`length`)  | `stat -c %s`: mida del fitxer en bytes                                  |
| Durada           | `ffprobe`, en segons (sense decimals)                                   |
| Data             | Data de modificació del fitxer, en format RFC 2822 (`date -R`). `LC_ALL=C` fa que el dia i el mes surtin en anglès |
| Identificador    | El nom del fitxer, que no canvia                                        |

```bash
sudo chmod 755 /usr/local/bin/genera-feed.sh
sudo genera-feed.sh

# Validar l'XML
xmllint --noout /var/www/radio/podcast/feed.xml

# Comprovar-ho des del client
curl -k -s https://radio.educem.cat/podcast/feed.xml | head -20
curl -k -s -I https://radio.educem.cat/podcast/episodis/episodi-01.mp3 | grep -i -E 'content-type|content-length|accept-ranges'
```

Cada vegada que publiqueu un episodi, torneu a executar `genera-feed.sh`.

### Subscriure's al Podcast

**Confiar en el certificat.** El certificat de `radio.educem.cat` és autosignat. Al client, afegiu-lo als certificats de confiança, com vam fer amb el de missatgeria (Tema 6, apartat 04):

```bash
openssl s_client -connect radio.educem.cat:443 -servername radio.educem.cat </dev/null 2>/dev/null \
    | openssl x509 -out radio.educem.cat.crt
openssl x509 -in radio.educem.cat.crt -noout -fingerprint -sha256    # compareu-la amb la del servidor
sudo cp radio.educem.cat.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**gPodder (client de podcasts):**

```bash
sudo apt install gpodder
gpodder &
```

1. **Subscriptions** → **Add podcast via URL** → `https://radio.educem.cat/podcast/feed.xml`.
2. Apareixen el podcast, la portada i els episodis. Descarregueu-ne un i escolteu-lo.
3. Publiqueu un episodi nou, torneu a generar el *feed* i, a gPodder, feu **Check for new episodes**: apareix l'episodi nou.

**Altres clients:**

- **VLC:** a la llista de reproducció, **Internet** → **Podcasts** → **+** i l'adreça del *feed*.
- **Mòbil:** AntennaPod (Android) o Apple Podcasts (iOS) permeten afegir un podcast per adreça. Al laboratori no hi arriben, perquè el domini només existeix a la xarxa interna.
- **Thunderbird:** també llegeix *feeds* (**File** → **New** → **Feed Account**); mostra cada episodi com un missatge amb l'àudio adjunt.

**OPML.** gPodder pot exportar les subscripcions a un fitxer OPML, per importar-les en un altre client.

### Publicar als Directoris

Per aparèixer a Apple Podcasts, Spotify o iVoox, el podcast ha de ser accessible des d'Internet amb un domini real i un certificat vàlid (Let's Encrypt, Tema 3). A més, ha de complir les normes del *feed* que hem vist: portada, identificadors únics, dates RFC 2822 i peticions per trossos. El *feed* es dona d'alta una vegada al portal de cada directori.

### Estadístiques de Descàrregues

Les descàrregues queden al registre d'Apache. Els reproductors demanen els fitxers per trossos (codi `206`), de manera que un mateix oient pot generar diverses línies. Per això es compten les parelles diferents d'adreça IP i episodi:

```bash
sudo awk '$7 ~ /^\/podcast\/episodis\/.*\.mp3$/ {print $1, $7}' \
    /var/log/apache2/radio.educem.cat-access.log \
    | sort -u | awk '{print $2}' | sort | uniq -c | sort -rn
```

### Resolució de Problemes

| Problema                                             | Causa                                                       | Solució                                                          |
| ---------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| gPodder diu `certificate verify failed`              | El client no confia en el certificat autosignat             | Afegir-lo amb `update-ca-certificates`                           |
| `xmllint` dona `parser error`                        | Hi ha un `&` o un `<` sense escapar, o una etiqueta mal tancada | Corregir el text o l'script; `xmllint` indica la línia      |
| L'episodi nou no surt al client                      | No s'ha tornat a generar el *feed*, o el client no l'ha consultat | `sudo genera-feed.sh`; **Check for new episodes**            |
| La durada surt buida                                 | `ffprobe` no pot llegir el fitxer                           | `ffprobe episodi.mp3`; tornar a codificar l'episodi              |
| Els accents es veuen malament                        | El fitxer no és UTF-8                                       | `file feed.xml`; desar els textos en UTF-8                       |
| L'script diu `No hi ha episodis`                     | No hi ha cap MP3 a la carpeta                               | Copiar-los a `/var/www/radio/podcast/episodis/`                  |

### Punt de Control

- [ ] Hi ha almenys dos episodis normalitzats, amb metadades i noms sense accents.
- [ ] `genera-feed.sh` genera el *feed* i `xmllint` no mostra errors.
- [ ] Apache serveix els episodis com a `audio/mpeg` i amb `Accept-Ranges: bytes`.
- [ ] gPodder se subscriu al podcast, descarrega els episodis i detecta un episodi nou.
- [ ] Sabeu comptar les descàrregues de cada episodi al registre d'Apache.
