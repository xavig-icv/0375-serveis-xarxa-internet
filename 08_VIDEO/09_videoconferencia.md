## 09. Videoconferència: Característiques i Protocols

### Què és una Videoconferència?

Una **videoconferència** és una comunicació d'àudio i vídeo **en temps real i en tots dos sentits** entre dos o més participants. A més, sol permetre:

- compartir la pantalla;
- xatejar;
- aixecar la mà;
- fer sales de treball en grup;
- gravar la sessió;
- generar subtítols.

Té requisits molt diferents dels de la resta de serveis de vídeo:

| Requisit                     | Per què                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------- |
| **Latència baixa**           | Per conversar amb naturalitat, el retard en un sol sentit ha de ser menor de 150 ms aproximadament. Amb més de 400 ms, els participants es trepitgen |
| **Variació del retard** (*jitter*) i paquets perduts | Es toleren petites pèrdues: és millor perdre una part de la imatge que esperar-la |
| **Ample de banda de pujada** | Cada participant també envia vídeo. Moltes connexions domèstiques tenen poca pujada |
| **Cancel·lació de l'eco**    | Si el so dels altaveus torna a entrar pel micròfon, els altres participants se senten a si mateixos |

### Protocols

| Protocol o tecnologia        | Funció                                                                  | Ús actual                                                |
| ---------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------- |
| **H.323** (ITU-T, 1996)      | Família de protocols de videoconferència                                | Equips de sala antics                                    |
| **SIP** (IETF, RFC 3261)     | Senyalització: establir, modificar i acabar sessions                     | Telefonia IP, centraletes (Asterisk), equips de sala     |
| **RTP / RTCP** (RFC 3550)    | Transport de l'àudio i el vídeo en temps real sobre UDP / estadístiques de qualitat | Tots els sistemes                            |
| **SRTP**                     | RTP xifrat                                                              | WebRTC, telefonia segura                                 |
| **WebRTC** (W3C i IETF)      | Àudio, vídeo i dades en temps real des del navegador, sense instal·lar res | Jitsi Meet, Google Meet, la versió web de Teams      |

### Com Funciona WebRTC

```
             Senyalització (HTTPS / WebSocket): qui es connecta, quins còdecs fa servir...
 Navegador A ◄──────────────────────► Servidor de l'aplicació ◄──────────────────────► Navegador B
     │                                                                                     │
     │   STUN: "quina és la meva adreça pública?"       TURN: retransmet si no hi ha camí   │
     ├─────────────────────────────► Servidor STUN/TURN ◄──────────────────────────────────┤
     │                                                                                     │
     └════════════ Àudio i vídeo (SRTP sobre UDP): directament, o a través d'un servidor ═══┘
```

- **Accés a la càmera i al micròfon** (`getUserMedia`): el navegador demana permís a l'usuari, i **només funciona en pàgines HTTPS**. Per això Jitsi Meet necessita un certificat.
- **Senyalització:** WebRTC no la defineix; cada aplicació fa servir la seva. **Jitsi Meet fa servir XMPP**, el protocol del Tema 6, amb un servidor Prosody.
- **ICE** (*Interactive Connectivity Establishment*): prova tots els camins possibles entre els participants i tria el millor. Fa servir:
  - **STUN:** un servidor que diu a cada equip quina és la seva adreça pública quan és darrere d'un NAT.
  - **TURN:** un servidor que fa de pont quan no hi ha cap camí directe, per exemple per culpa d'un tallafoc molt restrictiu. Gasta ample de banda del servidor.
- **Xifratge obligatori:** l'àudio i el vídeo sempre viatgen xifrats (DTLS-SRTP).
- **Còdecs:** Opus per a l'àudio; VP8 i H.264 per al vídeo (obligatoris), i també VP9 i AV1.

### Arquitectures

| Arquitectura                        | Com funciona                                                                 | El que envia cada participant | Servidor                        | Ús                          |
| ----------------------------------- | ---------------------------------------------------------------------------- | ----------------------------- | ------------------------------- | --------------------------- |
| **P2P en malla** (*mesh*)           | Cada participant envia el seu vídeo directament a cadascun dels altres      | Una còpia per a cada altre participant | Només per a la senyalització | 2 o 3 participants |
| **MCU** (*Multipoint Control Unit*) | El servidor rep tots els vídeos, en compon un mosaic i envia una sola imatge a cada participant | Una còpia          | Molta CPU: descodifica i torna a codificar | Equips de sala antics |
| **SFU** (*Selective Forwarding Unit*) | El servidor rep el vídeo de cada participant i el reenvia als altres, sense descodificar-lo | Una còpia (o diverses qualitats) | Poca CPU, molt ample de banda | **Jitsi Meet**, Google Meet, Zoom, Teams |

**Exemple: 6 participants, cadascun amb un vídeo de 1,5 Mbps**

- **En malla:** cada participant envia 5 × 1,5 = **7,5 Mbps** i en rep 7,5 Mbps. Moltes connexions domèstiques no ho suporten.
- **Amb un SFU:** cada participant envia només **1,5 Mbps**. El servidor rep 6 × 1,5 = 9 Mbps i en pot enviar fins a 6 × 5 × 1,5 = 45 Mbps.

**Simulcast:** amb un SFU, cada participant envia el seu vídeo en dues o tres qualitats (per exemple, 180p, 360p i 720p). El servidor envia l'alta qualitat de qui parla, que es veu en gran, i la baixa de les miniatures. Així estalvia ample de banda.

**Jitsi Meet** fa servir el mode P2P quan només hi ha dos participants i passa a l'SFU (**Jitsi Videobridge**) quan entra el tercer.

### Ample de Banda per Participant

Valors orientatius de Jitsi Meet per a cada vídeo:

| Resolució | Taxa aproximada |
| --------- | --------------- |
| 180p      | 200 kbps        |
| 360p      | 500 kbps        |
| 720p (HD) | 2,5 Mbps        |
| 4K        | 10 Mbps         |

### Eines de Videoconferència

| Eina                         | Llicència      | Característiques                                                                 |
| ---------------------------- | -------------- | -------------------------------------------------------------------------------- |
| **Jitsi Meet**               | Lliure         | No calen comptes; funciona al navegador; es pot instal·lar en un servidor propi. **La instal·larem (apartat 10)** |
| **BigBlueButton**            | Lliure         | Pensat per a l'educació: pissarra, enquestes, sales de grup; s'integra amb Moodle |
| **Nextcloud Talk**           | Lliure         | Integrat amb Nextcloud (fitxers, calendari)                                      |
| **Element Call** (Matrix)    | Lliure         | Integrat amb la missatgeria Matrix                                               |
| **Teams, Google Meet, Zoom, Webex** | Propietàries | Serveis al núvol de les grans empreses                                       |

### Requisits dels Participants

- **Navegador** amb WebRTC: Firefox, Chrome, Edge o Safari actuals.
- **Càmera, micròfon i, millor, auriculars**, perquè no hi hagi eco.
- **Connexió:** com a mínim 1 Mbps de pujada i de baixada per a una qualitat mitjana. Amb cable funciona millor que amb wifi, perquè hi ha menys variació del retard.
- **Permisos:** el navegador demana permís per fer servir la càmera i el micròfon a cada lloc web.

### Seguretat i Privacitat

- **Xifratge:** les connexions sempre van xifrades, però amb un SFU el xifratge és entre cada participant i el servidor, i el servidor pot accedir al contingut. Jitsi Meet ofereix com a opció el xifratge extrem a extrem, que impedeix que el servidor vegi el contingut.
- **Accés a les sales:** noms de sala difícils d'endevinar, contrasenyes, sala d'espera (*lobby*) i moderadors.
- **Gravacions:** cal informar els participants i tenir-ne el consentiment (apartat 11).
