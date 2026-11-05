## 01. Introducció als Serveis d'Àudio

### Què és un Servei d'Àudio?

Un **servei d'àudio** distribueix so a través de la xarxa: música, programes de ràdio, classes o converses. Avui és una part molt important del trànsit d'Internet:

| Servei                         | Exemples                                                    | Com s'escolta                         |
| ------------------------------ | ----------------------------------------------------------- | ------------------------------------- |
| **Ràdio en línia**             | Emissores en directe per Internet, ràdios de centres educatius | En directe, tots alhora             |
| **Podcast**                    | Programes a la carta, cursos, entrevistes                   | Quan l'oient vol, episodi per episodi |
| **Música sota demanda**        | Spotify, Apple Music, Bandcamp                              | Quan l'oient vol, cançó per cançó     |
| **Veu en temps real**          | Telefonia IP, videoconferència (Tema 8)                     | Conversa, amb molt poc retard         |
| **Àudio a les aplicacions web** | Cursos en línia, avisos, audiollibres                      | Des del navegador                     |

### Models de Distribució

| Model                        | Com funciona                                                          | Exemple                                    | Protocol habitual |
| ---------------------------- | --------------------------------------------------------------------- | ------------------------------------------ | ----------------- |
| **Descàrrega**               | El fitxer es baixa sencer i després s'escolta                         | Episodis de podcast desats al mòbil        | HTTP(S)           |
| **Descàrrega progressiva**   | Es comença a escoltar mentre el fitxer encara es baixa                | Un MP3 dins d'una pàgina web               | HTTP(S)           |
| **Streaming sota demanda**   | El servidor envia el contingut a mesura que s'escolta; es pot avançar i tirar enrere | Plataformes de música      | HLS, DASH         |
| **Streaming en directe**     | Es transmet en temps real: tots els oients senten el mateix moment    | Ràdio en línia                             | HTTP (Icecast), HLS |

Al **streaming en directe** intervenen tres elements:

```
 FONT D'EMISSIÓ                  SERVIDOR DE STREAMING                 OIENTS
 (micròfon, llista de            (Icecast)                             (VLC, navegador, mòbil)
  reproducció)
      │   1 connexió: envia               │   1 connexió per oient:
      │   el so codificat                 │   una còpia per a cadascun
      └──────────────────────────────►    ├──────────────────────────────► oient 1
                                          ├──────────────────────────────► oient 2
                                          └──────────────────────────────► oient N
```

- **Font d'emissió** (*source client*): codifica el so (per exemple, en MP3) i l'envia al servidor. Pot ser un programa de locució amb micròfon o un reproductor automàtic que emet una llista de reproducció.
- **Servidor de streaming:** rep l'emissió i n'envia una còpia a cada oient. No modifica el so.
- **Oients:** reproductors que es connecten al servidor.

**Idea clau:** la font envia una sola còpia, però el servidor n'envia una per oient. Per això, l'ample de banda que necessita el servidor creix amb el nombre d'oients (apartat 04).

### Què Farem en Aquest Tema

Tot es fa al servidor 0375, amb el mateix escenari dels temes anteriors ([escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md)):

| Funció                                 | Programari                     | On             | Adreça                                      |
| -------------------------------------- | ------------------------------ | -------------- | ------------------------------------------- |
| Preparar i convertir l'àudio           | ffmpeg, espeak-ng              | 0375 i client  | -                                           |
| Ràdio en directe                       | Icecast                        | 0375           | `http://radio.educem.cat:8000/radio.mp3`    |
| Emissió automàtica (llista de reproducció) | ffmpeg com a servei        | 0375           | -                                           |
| Emissió en directe amb micròfon        | BUTT                           | Client         | -                                           |
| Web de la ràdio i podcast              | Apache (HTTPS, Tema 3)         | 0375           | `https://radio.educem.cat`                  |
| Escoltar i subscriure's                | VLC, mpv, navegador, gPodder   | Client         | -                                           |

```
                        Xarxa interna 192.168.1.0/24 (enp0s8)
                                       │
     ┌─────────────────────────────────┼───────────────────────────────┐
     │                                 │                               │
SERVIDOR 0375 (192.168.1.2)      SERVIDOR 0374 (192.168.1.1)     CLIENT (DHCP)
Primari dels serveis 0375        Gateway (NAT) + secundari       192.168.1.50 (reserva)
  ├─ Icecast → ràdio      8000/tcp                               Ubuntu Desktop
  ├─ ffmpeg  → emissió automàtica                                  ├─ VLC, mpv, navegador
  ├─ Apache  → web de la ràdio i podcast  443/tcp                  ├─ BUTT (emissió en directe)
  └─ BIND9   → DNS (Tema 1)                                        └─ gPodder (podcast)
```

`Nota:` Les ordres del tema estan escrites per a **Ubuntu Server 24.04 LTS** (Icecast 2.4 i ffmpeg 6.1). Ubuntu 26.04 porta Icecast 2.5, que canvia alguns aspectes de la configuració.

### Registre DNS del Tema

Al servidor DNS primari (0375), afegiu el registre a la zona `educem.cat` (`/etc/bind/zones/db.educem.cat`), incrementeu el serial i feu `sudo rndc reload`:

```
; Tema 7 - Àudio
radio    IN  A    192.168.1.2
```

```bash
# Comprovació des del client
dig +short radio.educem.cat        # 192.168.1.2
```

### Ports Utilitzats en Aquest Tema

| Port         | Servei                                                                      |
| ------------ | --------------------------------------------------------------------------- |
| **8000/tcp** | Icecast: hi connecten les fonts d'emissió i els oients (HTTP)               |
| **443/tcp**  | Apache: web de la ràdio, podcast i emissions per HTTPS (ja obert al Tema 3) |

### Drets d'Autor

Emetre o publicar música té les mateixes obligacions legals que una ràdio convencional. Cal tenir els drets de tot el que s'emet. Al laboratori farem servir àudio generat per nosaltres (tons i veu sintètica), gravacions pròpies o música amb llicència **Creative Commons**, respectant-ne les condicions. En parlem a l'apartat 09.
