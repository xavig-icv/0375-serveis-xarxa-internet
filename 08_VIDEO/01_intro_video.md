## 01. Introducció als Serveis de Vídeo

### Què és un Servei de Vídeo?

Un **servei de vídeo** distribueix imatges en moviment, normalment amb so, a través de la xarxa. El vídeo és més de la meitat de tot el trànsit d'Internet:

| Servei                       | Exemples                                                       |
| ---------------------------- | -------------------------------------------------------------- |
| **Vídeo sota demanda**       | YouTube, plataformes de sèries, vídeos d'un curs a Moodle      |
| **Directe** (*streaming*)    | Retransmissió d'un acte, d'una classe o d'un esdeveniment esportiu |
| **Videoconferència**         | Classes i reunions en línia: Jitsi Meet, Teams, Google Meet    |
| **Videovigilància**          | Càmeres IP que envien el vídeo a un enregistrador              |
| **Televisió IP**             | Canals de televisió dels operadors de fibra                    |

Comparat amb l'àudio (Tema 7), el vídeo necessita molts més recursos:

- **Taxa de bits:** entre 10 i 50 vegades més. Un vídeo HD fa servir uns 2,5-5 Mbps, mentre que una ràdio en fa servir 128 kbps.
- **CPU:** codificar vídeo és una de les tasques més pesades que fa un servidor.
- **Disc:** una hora de vídeo HD ocupa uns quants GB.

### Models de Distribució

| Model                    | Comunicació     | Latència        | Protocols                         | Exemple                      | En aquest tema |
| ------------------------ | --------------- | --------------- | --------------------------------- | ---------------------------- | -------------- |
| **Vídeo sota demanda**   | Un a un         | No n'hi ha      | HTTP (descàrrega progressiva), HLS, DASH | Vídeos d'un curs      | Apartats 05 i 06 |
| **Directe**              | Un a molts      | 10-30 segons    | RTMP o SRT (enviar); HLS (veure)  | Retransmissió d'un acte      | Apartat 07     |
| **Videoconferència**     | Molts a molts   | Menys de 0,5 s  | WebRTC                            | Classe en línia              | Apartats 09 i 10 |

La latència importa molt en una videoconferència, perquè els participants han de poder parlar com en una conversa normal. En un directe, en canvi, un retard de 20 segons no és cap problema.

### Què Farem en Aquest Tema

| Funció                              | Programari                     | On                     | Adreça                                      |
| ----------------------------------- | ------------------------------ | ---------------------- | ------------------------------------------- |
| Convertir i preparar vídeo          | ffmpeg                         | 0375                   | -                                           |
| Vídeo sota demanda i HLS            | Apache (HTTPS, Tema 3)         | 0375                   | `https://video.educem.cat`                  |
| Rebre el directe                    | Nginx amb el mòdul RTMP        | 0375                   | `rtmp://video.educem.cat/live`              |
| Emetre el directe                   | OBS Studio                     | Client                 | -                                           |
| Videopodcast                        | Apache i un *feed* RSS         | 0375                   | `https://video.educem.cat/videopodcast/feed.xml` |
| Videoconferència                    | Jitsi Meet                     | Màquina virtual nova (`192.168.1.3`) | `https://meet.educem.cat`     |
| Veure, escoltar i participar        | Navegador, VLC, mpv, gPodder   | Client                 | -                                           |

```
                            Xarxa interna 192.168.1.0/24
                                         │
   ┌──────────────────────┬──────────────┴───────────┬───────────────────────────┐
   │                      │                          │                           │
SERVIDOR 0375          SERVIDOR 0374             SERVIDOR JITSI              CLIENT (DHCP)
192.168.1.2            192.168.1.1               192.168.1.3                 192.168.1.50
 ├─ Apache: vídeo       Gateway (NAT)            meet.educem.cat              ├─ Navegador, VLC, mpv
 │  sota demanda, HLS   + secundari              (màquina nova)               ├─ OBS Studio (directe)
 ├─ Nginx RTMP: directe                           Jitsi Meet                  └─ gPodder
 └─ ffmpeg
```

`Nota:` Les ordres del tema estan escrites per a **Ubuntu Server 24.04 LTS** (ffmpeg 6.1, Nginx 1.24 i Apache 2.4).

### La Màquina Virtual de Videoconferència

Jitsi Meet necessita un servidor propi:

- Instal·la el seu servidor web (Nginx al port 443) i el seu servidor XMPP (Prosody). Al servidor 0375 xocarien amb l'Apache del Tema 3 i el Prosody del Tema 6.
- Els seus components (Jicofo i el Videobridge) són programes Java i necessiten memòria.
- Als serveis reals, la videoconferència sempre té servidors dedicats, perquè és un servei en temps real.

| Paràmetre       | Valor                                                                   |
| --------------- | ----------------------------------------------------------------------- |
| Sistema         | Ubuntu Server 24.04 LTS                                                 |
| Memòria         | 4 GB (amb 2 GB es pot provar amb molt pocs participants)                |
| CPU             | 2 nuclis o més                                                          |
| Disc            | 20 GB                                                                   |
| Xarxa           | Una interfície a la xarxa interna `intnet`: `192.168.1.3/24`, porta d'enllaç `192.168.1.1` |
| Nom             | `meet.educem.cat`                                                       |

La instal·lació es fa a l'apartat 10. Com el client, aquesta màquina surt a Internet a través del NAT del 0374.

### Registres DNS del Tema

Al servidor DNS primari (0375), afegiu els registres a la zona `educem.cat` i a la zona inversa. Incrementeu el serial de totes dues i feu `sudo rndc reload`:

```
; Zona educem.cat - Tema 8: vídeo i videoconferència
video    IN  A    192.168.1.2
meet     IN  A    192.168.1.3
```

```
; Zona inversa 1.168.192.in-addr.arpa
3        IN  PTR  meet.educem.cat.
```

```bash
# Comprovació des del client
dig +short video.educem.cat       # 192.168.1.2
dig +short meet.educem.cat        # 192.168.1.3
dig +short -x 192.168.1.3         # meet.educem.cat.
```

### Ports Utilitzats en Aquest Tema

| Port          | Servidor           | Servei                                                              |
| ------------- | ------------------ | ------------------------------------------------------------------- |
| **443/tcp**   | 0375 (Apache)      | Vídeo sota demanda, HLS i videopodcast (ja obert al Tema 3)         |
| **1935/tcp**  | 0375 (Nginx)       | RTMP: rebre el directe que envia OBS                                |
| **443/tcp**   | Jitsi (Nginx)      | Web de Jitsi Meet i connexió de senyalització                       |
| **10000/udp** | Jitsi (Videobridge) | Àudio i vídeo de les videoconferències                             |

### Contingut de Proves i Drets

Al laboratori farem servir dos tipus de vídeo, sense problemes de drets:

- **Vídeos generats per ffmpeg:** imatges de prova i un to (apartat 04).
- **Pel·lícules lliures de la Blender Foundation**, com *Big Buck Bunny*. Tenen la llicència Creative Commons BY 3.0: es poden fer servir i modificar citant-ne l'autoria («© Blender Foundation | peach.blender.org»).

Per publicar imatges de persones, cal la seva autorització. En un centre educatiu, cal la de l'alumnat i, si són menors, la de les famílies (apartat 11).
