## 06. Fonts d'Emissió i Punts de Muntatge

### Punts de Muntatge

Cada emissió d'Icecast té un **punt de muntatge** (*mountpoint*), un camí com `/radio.mp3`. La font s'hi connecta per emetre, i els oients s'hi connecten per escoltar: `http://radio.educem.cat:8000/radio.mp3`.

No cal declarar els punts de muntatge: Icecast en crea un quan s'hi connecta una font. Però declarar-los a `icecast.xml` permet posar-hi límits, noms, contrasenyes pròpies i emissions de reserva. La ràdio del laboratori en tindrà tres:

| Punt de muntatge | Font                                  | Format                 | Contingut                                                   |
| ---------------- | ------------------------------------- | ---------------------- | ----------------------------------------------------------- |
| `/radio.mp3`     | ffmpeg al servidor (servei `radio-auto`) | MP3, 128 kbps       | Programació automàtica, les 24 hores                        |
| `/radio.opus`    | La mateixa font                       | Opus, 64 kbps          | La mateixa programació, per a connexions lentes             |
| `/directe.mp3`   | BUTT al client (micròfon)             | MP3, 128 kbps          | Programa en directe. Quan no n'hi ha, sona `/radio.mp3`     |

### Preparar els Directoris i la Música

```bash
# Usuari del sistema per al servei d'emissió automàtica (directori /srv/radio)
sudo adduser --system --group --home /srv/radio radio
sudo chmod 755 /srv/radio

# Música de la programació i gravacions dels directes (les escriu Icecast)
sudo mkdir -p /srv/radio/musica /srv/radio/gravacions
sudo chown icecast2:icecast /srv/radio/gravacions
```

La programació automàtica necessita fitxers amb el mateix format. Convertiu-los amb l'script `converteix.sh` de l'apartat 03 (desat a `~/audio` del servidor) i copieu-los a `/srv/radio/musica`. Per exemple, amb falques de veu i tons generats:

```bash
mkdir -p ~/audio/originals && cd ~/audio/originals
espeak-ng -v ca -w falca1.wav "Esteu escoltant Ràdio EDUCEM."
espeak-ng -v ca -w falca2.wav "Ràdio EDUCEM, la ràdio del laboratori d'ASIX."
ffmpeg -f lavfi -i "sine=frequency=440:duration=20" -ac 2 to1.wav
ffmpeg -f lavfi -i "sine=frequency=660:duration=20" -ac 2 to2.wav

cd ~/audio
./converteix.sh ~/audio/originals ~/audio/mp3
sudo cp ~/audio/mp3/*.mp3 /srv/radio/musica/

# Llista de reproducció en el format del demultiplexor concat d'ffmpeg
for f in /srv/radio/musica/*.mp3; do echo "file '$f'"; done | sudo tee /srv/radio/llista.txt
```

Si teniu música amb llicència Creative Commons, convertiu-la i afegiu-la també a la llista.

### Declarar els Punts de Muntatge

Afegiu aquests blocs a `/etc/icecast2/icecast.xml`, just abans de `</icecast>`:

```xml
    <!-- Programació automàtica en MP3 -->
    <mount type="normal">
        <mount-name>/radio.mp3</mount-name>
        <stream-name>Ràdio EDUCEM</stream-name>
        <stream-description>La ràdio del laboratori d'ASIX</stream-description>
        <genre>Diversos</genre>
        <max-listeners>50</max-listeners>
    </mount>

    <!-- La mateixa programació en Opus -->
    <mount type="normal">
        <mount-name>/radio.opus</mount-name>
        <stream-name>Ràdio EDUCEM (Opus)</stream-name>
        <max-listeners>50</max-listeners>
    </mount>

    <!-- Programa en directe, amb contrasenya pròpia -->
    <mount type="normal">
        <mount-name>/directe.mp3</mount-name>
        <password>CONTRASENYA_DIRECTE</password>
        <stream-name>Ràdio EDUCEM - En directe</stream-name>
        <max-listeners>50</max-listeners>
        <fallback-mount>/radio.mp3</fallback-mount>
        <fallback-override>1</fallback-override>
        <dump-file>/srv/radio/gravacions/directe-%F-%H%M.mp3</dump-file>
    </mount>
```

| Element                  | Funció                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------ |
| `<stream-name>`, `<stream-description>`, `<genre>` | Informació que veuen els oients. Té prioritat sobre la que envia la font |
| `<max-listeners>`        | Oients màxims d'aquesta emissió: limita l'ample de banda (apartat 04)                      |
| `<password>`             | Contrasenya només per a aquest punt de muntatge. Qui fa el directe no coneix la contrasenya general de les fonts |
| `<fallback-mount>`       | Si `/directe.mp3` no té font, els seus oients escolten `/radio.mp3`                        |
| `<fallback-override>`    | Quan comença el directe, els oients tornen automàticament a `/directe.mp3`                 |
| `<dump-file>`            | Grava tot el que s'emet en directe. `%F` és la data i `%H%M`, l'hora                       |

**L'emissió de reserva ha de tenir el mateix format.** `/radio.mp3` i `/directe.mp3` són tots dos MP3 a 128 kbps, 44,1 kHz i estèreo. Si no, el canvi d'una emissió a l'altra talla el so dels reproductors.

```bash
sudo xmllint --noout /etc/icecast2/icecast.xml
sudo systemctl restart icecast2
```

### Emissió Automàtica: ffmpeg com a Servei

La programació automàtica ha de funcionar sempre: s'ha d'engegar amb el servidor i reiniciar-se si falla. Per això la configurem com un servei de systemd que s'executa amb l'usuari `radio`, sense privilegis.

**1. La contrasenya, fora de l'script.**

```bash
sudo mkdir -p /etc/radio
sudo nano /etc/radio/radio.env
```

```
ICECAST_PASSWORD=CONTRASENYA_FONTS
```

```bash
sudo chmod 600 /etc/radio/radio.env      # només el pot llegir root
```

**2. L'script d'emissió.**

```bash
sudo nano /usr/local/bin/radio-auto.sh
```

```bash
#!/bin/bash
# radio-auto.sh - Emet la llista de reproducció en bucle a Icecast, en MP3 i en Opus.
# La contrasenya arriba a la variable ICECAST_PASSWORD (fitxer /etc/radio/radio.env).

LLISTA=/srv/radio/llista.txt
SERVIDOR=localhost:8000

exec ffmpeg -nostdin -hide_banner -loglevel warning \
    -re -stream_loop -1 -f concat -safe 0 -i "$LLISTA" \
    -map 0:a -c:a libmp3lame -b:a 128k -ar 44100 -ac 2 \
        -f mp3 "icecast://source:${ICECAST_PASSWORD}@${SERVIDOR}/radio.mp3" \
    -map 0:a -c:a libopus -b:a 64k -ar 48000 -ac 2 -content_type audio/ogg \
        -f ogg "icecast://source:${ICECAST_PASSWORD}@${SERVIDOR}/radio.opus"
```

```bash
sudo chmod 755 /usr/local/bin/radio-auto.sh
```

- **Una entrada i dues sortides:** ffmpeg llegeix la llista una sola vegada i la codifica en dos formats. Cada sortida té les seves opcions i el seu punt de muntatge.
- `-stream_loop -1` repeteix la llista indefinidament, i `-f concat` llegeix els fitxers de la llista l'un darrere l'altre.
- `-content_type audio/ogg`: ffmpeg envia l'MP3 com a `audio/mpeg` per defecte; per a l'Ogg cal indicar-ho.
- `exec` substitueix l'script per ffmpeg, perquè systemd controli directament el procés.

**3. El servei.**

```bash
sudo nano /etc/systemd/system/radio-auto.service
```

```ini
[Unit]
Description=Ràdio EDUCEM - emissió automàtica cap a Icecast
After=network-online.target icecast2.service
Wants=network-online.target

[Service]
User=radio
EnvironmentFile=/etc/radio/radio.env
ExecStart=/usr/local/bin/radio-auto.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now radio-auto
systemctl status radio-auto
journalctl -u radio-auto -n 20 --no-pager
```

systemd llegeix `radio.env` abans de canviar a l'usuari `radio`. Per això el fitxer pot ser de root amb permisos 600. Amb `Restart=always`, si Icecast es reinicia i ffmpeg perd la connexió, el servei es torna a connectar al cap de 10 segons.

**Comprovació:** la pàgina d'estat (`http://radio.educem.cat:8000/`) mostra `/radio.mp3` i `/radio.opus`. Escolteu-los des del client:

```bash
mpv http://radio.educem.cat:8000/radio.mp3
mpv http://radio.educem.cat:8000/radio.opus
```

### Emissió en Directe des del Client (BUTT)

**BUTT** (*Broadcast Using This Tool*) és una font d'emissió gràfica: envia a Icecast el so del micròfon de l'equip.

```bash
sudo apt install butt       # al client
butt &
```

1. Obriu **Settings** i, a la configuració del servidor, feu clic a **ADD**:

| Camp                    | Valor                   |
| ----------------------- | ----------------------- |
| **Name**                | `Ràdio EDUCEM`          |
| **Type**                | IceCast                 |
| **Address**             | `radio.educem.cat`      |
| **Port**                | `8000`                  |
| **Password**            | `CONTRASENYA_DIRECTE`   |
| **IceCast mountpoint**  | `directe.mp3`           |
| **IceCast user**        | `source`                |

2. A la configuració de l'àudio i de l'emissió, trieu el còdec **MP3**, **128** kbps, **44100** Hz i **estèreo**: el mateix format que `/radio.mp3`.
3. A la finestra principal, el botó de reproducció (▶) comença l'emissió. L'indicador de nivell mostra el so del micròfon.

**Sense micròfon** (per exemple, si la màquina virtual del client no té entrada d'àudio), podeu simular el directe enviant un fitxer amb ffmpeg:

```bash
ffmpeg -re -i programa.mp3 -c:a libmp3lame -b:a 128k -ar 44100 -ac 2 \
    -f mp3 icecast://source:CONTRASENYA_DIRECTE@radio.educem.cat:8000/directe.mp3
```

**Prova de l'emissió de reserva:**

1. Sense directe, escolteu `http://radio.educem.cat:8000/directe.mp3`: sona la programació automàtica.
2. Comenceu el directe amb BUTT: el reproductor passa al directe sense que l'oient faci res.
3. Atureu BUTT: el reproductor torna a la programació automàtica.
4. A `/srv/radio/gravacions/` hi ha la gravació del directe, que podreu publicar com a podcast (apartat 08).

### Actualitzar el Títol del que Sona

Els reproductors mostren el títol de la cançó o del programa. Per a les emissions MP3, l'administrador el pot canviar amb una petició a la interfície d'administració:

```bash
curl -u admin:CONTRASENYA_ADMIN \
    "http://localhost:8000/admin/metadata?mount=/directe.mp3&mode=updinfo&song=Entrevista+amb+el+professorat"
```

Als paràmetres de l'adreça, els espais s'escriuen com a `+`. BUTT també pot enviar el títol des del client.

### Resolució de Problemes

| Problema                                                   | Causa                                                        | Solució                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| `radio-auto` es reinicia contínuament                      | Error d'ffmpeg                                               | `journalctl -u radio-auto` mostra el motiu                       |
| `Impossible to open '/srv/radio/llista.txt'`               | La llista no existeix o l'usuari `radio` no la pot llegir   | `ls -l /srv/radio/`                                              |
| `Unsafe file name`                                         | Falta `-safe 0` (la llista té camins absoluts)               | Revisar l'script                                                 |
| `401 Unauthorized`                                         | Contrasenya incorrecta a `radio.env` o a BUTT                | Comprovar `<source-password>` i la `<password>` del punt de muntatge |
| `403 Forbidden`                                            | Punt de muntatge ocupat o límit de `<sources>`               | `<sources>4</sources>`; comprovar que no hi ha dues fonts al mateix punt |
| El so es talla en passar del directe a la programació      | Formats diferents entre `/directe.mp3` i `/radio.mp3`        | El mateix còdec, taxa, freqüència i canals a totes dues fonts    |
| No es crea la gravació                                     | Icecast no pot escriure al directori                         | `sudo chown icecast2:icecast /srv/radio/gravacions`; mirar `error.log` |

### Punt de Control

- [ ] El servei `radio-auto` està actiu i la pàgina d'estat mostra `/radio.mp3` i `/radio.opus`.
- [ ] El client escolta els dos formats.
- [ ] BUTT (o ffmpeg) emet a `/directe.mp3` amb la contrasenya pròpia del punt de muntatge.
- [ ] Un oient de `/directe.mp3` passa del directe a la programació automàtica, i a l'inrevés, sense fer res.
- [ ] El directe queda gravat a `/srv/radio/gravacions/`.
