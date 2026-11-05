## 04. Streaming d'Àudio: Protocols i Ample de Banda

### Com Funciona el Streaming

En el streaming, el reproductor comença a reproduir abans de tenir tot el contingut:

- El reproductor guarda uns segons de so en una **memòria intermèdia** (*buffer*). Si la xarxa s'alenteix uns instants, continua sonant amb el que té guardat.
- Si la xarxa és més lenta que la taxa de bits del flux durant massa estona, la memòria intermèdia es buida i el so es talla.
- Com més gran és la memòria intermèdia, menys talls hi ha, però més **latència** (retard respecte del directe).

**Condició bàsica:** la velocitat de la connexió de l'oient ha de ser més alta que la taxa de bits del flux. Per escoltar un flux de 128 kbps cal una connexió de més de 128 kbps sostinguts.

### Protocols de Streaming d'Àudio

| Protocol                       | Com funciona                                                            | Latència     | Ús                                               |
| ------------------------------ | ----------------------------------------------------------------------- | ------------ | ------------------------------------------------ |
| **HTTP continu (Icecast, SHOUTcast)** | Una connexió HTTP que no s'acaba mai; el servidor hi va enviant el so | 2-10 s       | Ràdio en línia. **És el que farem servir**       |
| **HLS i DASH**                 | El so es divideix en fragments de pocs segons, que el reproductor descarrega per HTTP amb una llista de reproducció | 10-30 s | Grans plataformes, també per a vídeo (Tema 8) |
| **RTP / RTSP**                 | Paquets en temps real sobre UDP; RTSP controla la sessió (reproduir, pausa) | < 1 s     | Telefonia IP, càmeres de vigilància              |
| **WebRTC**                     | Comunicació en temps real entre navegadors, xifrada                     | < 0,5 s      | Videoconferència (Tema 8), trucades des del web  |

L'HTTP continu i HLS funcionen sobre **TCP** i HTTP: passen per tots els tallafocs i servidors intermediaris (*proxies*), però tenen més latència. RTP i WebRTC fan servir **UDP**: si es perd un paquet, no es torna a enviar, perquè arribaria massa tard. Per a una conversa és millor perdre un fragment de so que esperar.

### Unicast, Multicast i Difusió per Relleus

- **Unicast:** el servidor envia una còpia per a cada oient. És el que es fa a Internet.
- **Multicast:** el servidor envia una sola còpia i la xarxa la replica cap als equips subscrits. És molt eficient, però només funciona en xarxes que ho permeten, com la televisió IP dels operadors. A Internet no funciona.
- **Relleus (*relays*) i CDN:** per arribar a molts oients, el servidor principal envia el flux a altres servidors repartits per la xarxa, i cada oient es connecta al més proper. Icecast ho permet amb la configuració de *relay*.

### Càlcul de l'Ample de Banda

Amb unicast, l'ample de banda de sortida del servidor és:

```
ample de banda = nombre d'oients × taxa de bits del flux
```

| Oients | Flux de 128 kbps (MP3) | Flux de 64 kbps (Opus) |
| ------ | ---------------------- | ---------------------- |
| 10     | 1,28 Mbps              | 0,64 Mbps              |
| 100    | 12,8 Mbps              | 6,4 Mbps               |
| 1.000  | 128 Mbps               | 64 Mbps                |

Cal afegir-hi un 5-10 % per les capçaleres dels protocols (TCP/IP i HTTP).

**Exemple:** un centre educatiu vol emetre una jornada de portes obertes per a 300 oients amb un flux MP3 de 128 kbps.

- Calen 300 × 128 kbps = 38,4 Mbps de sortida, més un 10 %: uns 42 Mbps.
- Si la connexió del centre té 100 Mbps de pujada, la ràdio n'ocuparia gairebé la meitat i tota la resta del centre funcionaria més lenta.
- Solucions: oferir també un flux de 64 kbps, limitar el nombre d'oients (apartat 05) o contractar un servidor o un relleu fora del centre.

**Podcast:** cada descàrrega és el fitxer sencer. Un episodi de 30 minuts a 128 kbps ocupa 28,8 MB; si es descarrega 1.000 vegades, són 28,8 GB de trànsit.

### Adaptar el Format a l'Oient

Un mateix programa es pot emetre en diversos formats alhora, cadascun en un **punt de muntatge** (*mountpoint*) diferent del servidor:

| Punt de muntatge | Format                       | Per a qui                                               |
| ---------------- | ---------------------------- | ------------------------------------------------------- |
| `/radio.mp3`     | MP3, 128 kbps, estèreo       | Compatibilitat màxima: qualsevol navegador o reproductor |
| `/radio.opus`    | Opus, 64 kbps, estèreo       | Connexions mòbils o lentes; navegadors actuals           |

L'oient (o la pàgina web, apartat 07) tria el que li convé. Les grans plataformes fan aquesta adaptació automàticament amb HLS i DASH (**streaming adaptatiu**, Tema 8).

### Servidors de Streaming d'Àudio

| Programari                | Tipus                                            | Característiques                                                  |
| ------------------------- | ------------------------------------------------ | ----------------------------------------------------------------- |
| **Icecast**               | Servidor de streaming lliure (Xiph.Org)          | Lleuger, punts de muntatge, relleus, estadístiques. **El farem servir** |
| **SHOUTcast**             | Servidor de streaming propietari                 | Històric; compatible amb molts reproductors                       |
| **AzuraCast, LibreTime**  | Plataformes completes de ràdio                   | Web de gestió, programació horària, llistes, fan servir Icecast internament |
| **Liquidsoap**            | Automatització d'emissions                       | Llistes de reproducció, transicions, emissió de reserva           |
| **Serveis al núvol**      | Allotjament de ràdios i podcasts                 | El proveïdor posa l'ample de banda                                |

### Prestacions del Servidor

Un servidor de streaming **no transforma el so**: rep el flux de la font i en reparteix còpies. Per això:

- **CPU:** Icecast en fa servir molt poca. On cal CPU és a la font, si ha de codificar diversos formats alhora (apartat 06).
- **Memòria:** poca. Cada flux té una cua de mig MB, a la qual s'afegeix una memòria intermèdia per oient.
- **Ample de banda:** és el recurs que limita el servei. Es calcula com hem vist.
- **Disc:** només per als registres i, si n'hi ha, per a les gravacions i els podcasts. Una hora de MP3 a 128 kbps ocupa 57,6 MB.
