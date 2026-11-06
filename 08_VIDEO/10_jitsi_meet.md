## 10. Videoconferència amb Jitsi Meet

**Jitsi Meet** és una plataforma de videoconferència lliure que funciona al navegador, sense comptes ni instal·lacions als clients. La instal·lem a la màquina virtual nova `192.168.1.3` (apartat 01).

### Components de Jitsi Meet

```
 Navegador ──HTTPS 443──► Nginx ──► web de Jitsi Meet (HTML i JavaScript)
     │                      └──► /xmpp-websocket ──► Prosody (XMPP) ◄──► Jicofo
     │                                                                     │ tria el videobridge
     └──── àudio i vídeo (UDP 10000) ──────────────────────────────► Jitsi Videobridge (SFU)
```

| Component                     | Funció                                                                        |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **Nginx**                     | Serveix la web i passa la senyalització (WebSocket) a Prosody                 |
| **Prosody**                   | Servidor XMPP: les sales i els participants són sales i usuaris XMPP, com al Tema 6 |
| **Jicofo** (*Conference Focus*) | Gestiona les conferències i assigna un videobridge a cada sala              |
| **Jitsi Videobridge** (JVB)   | L'SFU: rep l'àudio i el vídeo de cada participant i el reenvia als altres (UDP 10000) |
| **coturn**                    | Servidor TURN per als participants que no poden fer servir UDP                |

### Preparar la Màquina Virtual

1. A VirtualBox, creeu una màquina amb Ubuntu Server 24.04, 4 GB de memòria, 2 CPU i 20 GB de disc. Ha de tenir **un sol adaptador de xarxa**, a la xarxa interna `intnet`, com el client.
2. Instal·leu Ubuntu Server amb el servidor OpenSSH.
3. Configureu la xarxa amb netplan (`/etc/netplan/50-cloud-init.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                       # única interfície: xarxa interna
      addresses: [192.168.1.3/24]
      routes:
        - to: default
          via: 192.168.1.1        # servidor 0374 (NAT)
      nameservers:
        addresses: [192.168.1.2, 192.168.1.1]
        search: [educem.cat]
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

4. Poseu-li el nom complet i afegiu-lo a `/etc/hosts`:

```bash
sudo hostnamectl set-hostname meet.educem.cat
sudo nano /etc/hosts
```

```
127.0.0.1     localhost
192.168.1.3   meet.educem.cat meet
```

Esborreu la línia `127.0.1.1` amb el nom antic, si n'hi ha.

5. Comproveu la xarxa:

```bash
hostname                      # meet.educem.cat
ping -c1 192.168.1.1          # porta d'enllaç (0374)
ping -c1 www.google.com       # Internet a través del NAT
dig +short meet.educem.cat    # 192.168.1.3 (registre de l'apartat 01)
```

### Afegir els Repositoris

Jitsi Meet no és als repositoris d'Ubuntu. S'instal·la des del repositori del projecte i, a més, la documentació recomana una versió recent de Prosody, del repositori de Prosody. Cada repositori té una **clau** que signa els paquets: `apt` només n'accepta els paquets si la signatura és correcta (`signed-by`).

```bash
sudo apt update
sudo apt install apt-transport-https curl gnupg2
sudo add-apt-repository universe

# Repositori de Prosody
sudo curl -sL https://prosody.im/files/prosody-debian-packages.key \
    -o /usr/share/keyrings/prosody-debian-packages.key
echo "deb [signed-by=/usr/share/keyrings/prosody-debian-packages.key] http://packages.prosody.im/debian $(lsb_release -sc) main" \
    | sudo tee /etc/apt/sources.list.d/prosody-debian-packages.list

# Repositori de Jitsi
curl -sL https://download.jitsi.org/jitsi-key.gpg.key \
    | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" \
    | sudo tee /etc/apt/sources.list.d/jitsi-stable.list

sudo apt update
```

### Tallafoc

```bash
sudo ufw allow 22/tcp                                              # SSH
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp       # redirecció a HTTPS
sudo ufw allow from 192.168.1.0/24 to any port 443 proto tcp      # web i senyalització
sudo ufw allow from 192.168.1.0/24 to any port 10000 proto udp    # àudio i vídeo (JVB)
sudo ufw allow from 192.168.1.0/24 to any port 3478 proto udp     # STUN/TURN
sudo ufw allow from 192.168.1.0/24 to any port 5349 proto tcp     # TURN sobre TCP
sudo ufw enable
sudo ufw status verbose
```

### Instal·lació

```bash
sudo apt install jitsi-meet
```

| Pregunta                                         | Resposta                                    |
| ------------------------------------------------ | ------------------------------------------- |
| **The hostname of the current installation**     | `meet.educem.cat`                           |
| **SSL certificate**                              | **Generate a new self-signed certificate**  |

- Let's Encrypt no és possible: el domini `educem.cat` no existeix a Internet.
- Si l'instal·lador pregunta per altres opcions, com serveis de pagament, responeu que no.

**Comprovacions:**

```bash
systemctl is-active prosody jicofo jitsi-videobridge2 nginx
sudo ss -tulnp | grep -E ':(443|10000) '
```

**Fitxers de configuració i registres:**

| Fitxer                                                | Contingut                                                  |
| ----------------------------------------------------- | ---------------------------------------------------------- |
| `/etc/jitsi/meet/meet.educem.cat-config.js`           | Opcions de la web: idioma, resolució, comportament de les sales |
| `/etc/prosody/conf.avail/meet.educem.cat.cfg.lua`     | Dominis i components XMPP de Jitsi                         |
| `/etc/jitsi/jicofo/jicofo.conf`                       | Jicofo                                                     |
| `/etc/jitsi/videobridge/jvb.conf`                     | Videobridge                                                |
| `/etc/nginx/sites-available/meet.educem.cat.conf`     | Lloc web                                                   |
| `/var/log/jitsi/jvb.log`, `/var/log/jitsi/jicofo.log`, `/var/log/prosody/prosody.log` | Registres                  |

### La Primera Videoconferència

**1. Primer participant (client, Firefox):** obriu `https://meet.educem.cat`, accepteu l'avís del certificat i creeu una sala amb un nom difícil d'endevinar, per exemple `ClasseASIX-Video-2026`. Doneu permís per fer servir la càmera i el micròfon.

**2. Segon participant:** un altre navegador del client, com Chromium (`sudo snap install chromium`), o un altre equip de la xarxa interna, a la mateixa adreça de la sala.

**Sense càmera ni micròfon** (habitual en màquines virtuals), els navegadors poden simular-los:

- **Chromium:** `chromium --use-fake-device-for-media-stream`. Mostra una imatge de prova i emet un to.
- **Firefox:** a `about:config`, posar `media.navigator.streams.fake` a `true`.

VirtualBox també pot passar la càmera de l'equip amfitrió a la màquina virtual (**Devices** → **Webcams**, amb l'Extension Pack).

**3. Funcions que cal provar:**

| Funció                                  | On                                                       |
| --------------------------------------- | -------------------------------------------------------- |
| Silenciar el micròfon i aturar la càmera | Botons de la part inferior                              |
| Compartir la pantalla                   | Botó de compartir pantalla                               |
| Xat i aixecar la mà                     | Botons de xat i de mà                                    |
| Vista de mosaic                         | Botó de canviar la vista                                 |
| Contrasenya i sala d'espera (*lobby*)   | Opcions de seguretat de la sala (escut). Només el moderador |
| Qualitat del vídeo                      | Menú de configuració → qualitat                          |
| Estadístiques de la connexió            | Icona de senyal de cada vídeo: resolució, taxa de bits i paquets perduts |

El primer participant de la sala n'és el **moderador**: pot posar-hi contrasenya, activar la sala d'espera, silenciar o expulsar participants.

### P2P o SFU: Veure-ho a la Xarxa

Amb dos participants, Jitsi fa servir el mode P2P: l'àudio i el vídeo van directament d'un navegador a l'altre. Amb tres o més, passen pel Videobridge (apartat 09). Ho podem veure al servidor Jitsi:

```bash
sudo tcpdump -ni enp0s3 udp port 10000 -c 20
```

1. Amb **dos participants**, gairebé no passen paquets pel port 10000.
2. Obriu un **tercer participant** (una altra pestanya o finestra privada): el `tcpdump` mostra un flux continu de paquets UDP cap al port 10000 i des d'aquest port.

### Configurar Jitsi Meet

La configuració de la web és a `/etc/jitsi/meet/meet.educem.cat-config.js`. Moltes opcions hi són comentades (`//`): per activar-ne una, cal treure el comentari i posar-hi el valor.

```bash
sudo cp /etc/jitsi/meet/meet.educem.cat-config.js /etc/jitsi/meet/meet.educem.cat-config.js.orig
sudo nano /etc/jitsi/meet/meet.educem.cat-config.js
```

```js
    // Idioma de la interfície
    defaultLanguage: 'ca',

    // Obligar a escriure un nom per entrar a la sala
    requireDisplayName: true,

    // Resolució màxima de 360p: estalvia ample de banda a l'aula
    resolution: 360,
    constraints: {
        video: {
            height: { ideal: 360, max: 360, min: 180 },
        },
    },
```

`config.js` és un fitxer que el navegador descarrega: no cal reiniciar cap servei, només recarregar la pàgina. Comproveu a les estadístiques de connexió que la resolució ja no passa de 360p.

**Qui pot crear sales.** Amb la configuració per defecte, qualsevol que arribi a la web pot crear una sala. Per limitar-ho als usuaris registrats, Jitsi s'integra amb un sistema d'autenticació, normalment amb testimonis JWT. No ho configurarem al laboratori.

### Altres Eines Gràfiques

- **Aplicació d'escriptori de Jitsi Meet:** existeix per a Linux, Windows i macOS, i també funciona amb servidors propis.
- **Aplicacions mòbils:** necessiten un certificat vàlid d'una CA, de manera que no funcionen amb el certificat autosignat del laboratori.
- **OBS Studio com a càmera virtual:** amb el botó **Start Virtual Camera**, les escenes d'OBS (apartat 07) es fan servir com a càmera a Jitsi. A Linux cal el mòdul del nucli `v4l2loopback` (paquet `v4l2loopback-dkms`).

### Resolució de Problemes

| Problema                                                     | Causa                                                     | Solució                                                          |
| ------------------------------------------------------------ | --------------------------------------------------------- | ---------------------------------------------------------------- |
| El navegador no troba la càmera o el micròfon                | La màquina no en té, o el navegador no té permís          | Dispositius simulats; revisar els permisos del lloc web          |
| La web no carrega o diu que s'ha perdut la connexió          | Algun servei aturat, o el nom no resol al mateix servidor | `systemctl is-active prosody jicofo jitsi-videobridge2 nginx`; `/etc/hosts` |
| Dos participants es veuen, però amb tres no hi ha vídeo      | El port UDP 10000 està tancat                             | `sudo ufw status`; registre `/var/log/jitsi/jvb.log`             |
| Firefox no es connecta després d'acceptar el certificat      | La connexió WebSocket encara no s'accepta                 | Recarregar la pàgina; tornar a acceptar el certificat            |
| La instal·lació falla en configurar Nginx                    | El port 443 ja està ocupat                                | Jitsi necessita una màquina dedicada: `sudo ss -tlnp \| grep 443` |
| La màquina va molt lenta                                     | Poca memòria per als components Java                     | Assignar-li 4 GB                                                 |

### Punt de Control

- [ ] La màquina `meet.educem.cat` té la IP `192.168.1.3` i surt a Internet a través del 0374.
- [ ] Els serveis `prosody`, `jicofo`, `jitsi-videobridge2` i `nginx` estan actius.
- [ ] Dos navegadors del client fan una videoconferència a `https://meet.educem.cat`.
- [ ] Heu provat la contrasenya de sala, la sala d'espera i compartir la pantalla.
- [ ] `tcpdump` mostra el canvi de P2P al Videobridge quan entra el tercer participant.
- [ ] La interfície surt en català i la resolució màxima és de 360p.
