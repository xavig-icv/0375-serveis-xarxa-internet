## 01. Introducció als Serveis de Comunicació

### Comunicació Síncrona i Asíncrona

Al Tema 5 vam configurar el correu electrònic. En aquest tema afegim tres serveis de comunicació més. Es diferencien per **quan** es llegeixen els missatges i per **a quantes persones** van dirigits:

| Servei                      | Tipus      | Destinataris                         | Exemples                                  | Protocol      |
| --------------------------- | ---------- | ------------------------------------ | ----------------------------------------- | ------------- |
| **Missatgeria instantània** | Síncrona   | Una persona o una sala               | Gajim, WhatsApp, Teams, Slack             | XMPP, Matrix  |
| **Correu electrònic**       | Asíncrona  | Una persona o unes quantes           | Thunderbird, Gmail (Tema 5)               | SMTP, IMAP    |
| **Llistes de distribució**  | Asíncrona  | Tots els membres de la llista        | Butlletins, llistes de projectes          | SMTP (correu) |
| **Notícies i fòrums**       | Asíncrona  | Qualsevol lector del grup o del fòrum | Usenet, fòrums web, Discourse             | NNTP, HTTP    |

- **Síncrona:** emissor i receptor estan connectats alhora i la resposta és immediata, com en una conversa.
- **Asíncrona:** el missatge es guarda i el receptor el llegeix quan vol, com una carta.

### Característiques de la Missatgeria Instantània

- **Temps real:** el missatge arriba en una fracció de segon.
- **Presència:** cada usuari mostra el seu estat (en línia, absent, no molestar) i veu el dels seus contactes.
- **Llista de contactes:** es guarda al servidor, de manera que l'usuari la té a tots els dispositius.
- **Sales de xat:** converses de grup.
- **Missatges fora de línia:** si el destinatari no està connectat, el servidor guarda el missatge i l'hi lliura quan es connecta.
- **Historial i diversos dispositius:** el servidor pot guardar les converses per sincronitzar el mòbil i l'ordinador.
- **Altres funcions:** enviament de fitxers, trucades d'àudio i vídeo i xifratge extrem a extrem.

### Models: Centralitzat i Federat

**Centralitzat.** Una sola empresa gestiona tots els servidors. Només es poden comunicar els usuaris del mateix servei: un usuari de WhatsApp no pot escriure a un de Teams.

**Federat.** Cada organització té el seu servidor, i els servidors es comuniquen entre ells. Les adreces tenen la forma `usuari@domini`. El correu electrònic funciona igual: `usuari1@educem.cat` pot escriure a qualsevol adreça de Gmail.

```
         CENTRALITZAT                                   FEDERAT

   usuari A        usuari B             usuari1@educem.cat         maria@altra.test
       │               │                         │                         │
       └──► Servidor ◄─┘                  Servidor educem.cat ◄──────► Servidor altra.test
          de l'empresa                         (Prosody)                 (un altre)
```

| Model         | Avantatges                                                      | Inconvenients                                                   |
| ------------- | --------------------------------------------------------------- | --------------------------------------------------------------- |
| Centralitzat  | Fàcil d'usar; l'empresa s'encarrega de tot                       | Les dades són a l'empresa; no es pot parlar amb altres serveis |
| Federat       | Les dades són al servidor de l'organització; protocols oberts   | Cal administrar el servidor                                    |

Per a una empresa, tenir el servei de missatgeria als seus servidors vol dir controlar on són les converses i qui hi té accés. És important per complir el Reglament general de protecció de dades (RGPD).

### Protocols i Programari

| Protocol / plataforma | Model         | Servidors                     | Clients                                   | Ús actual                                           |
| --------------------- | ------------- | ----------------------------- | ----------------------------------------- | --------------------------------------------------- |
| **XMPP**              | Federat, obert | Prosody, ejabberd, Openfire  | Gajim, Dino, Profanity, Conversations     | Empreses, administracions; Jitsi Meet el fa servir internament |
| **Matrix**            | Federat, obert | Synapse, Conduit             | Element                                   | Administracions i comunitats de programari lliure   |
| **IRC**               | Xarxes de servidors | InspIRCd, UnrealIRCd    | HexChat, irssi                            | Comunitats tècniques (canals de projectes)          |
| **Mattermost, Rocket.Chat, Zulip** | Autoallotjat | Servidor propi (web) | Navegador i aplicacions                  | Alternatives lliures a Slack o Teams                |
| **Teams, Slack, Google Chat** | Centralitzat | Al núvol del proveïdor  | Aplicacions del proveïdor                 | Plataformes col·laboratives de les empreses         |

**XMPP** és un estàndard de l'IETF des de l'any 2004 i és el protocol obert de missatgeria més estès. **És el que farem servir**, amb el servidor **Prosody**:

- És lleuger i té una configuració senzilla.
- És als repositoris de Debian i Ubuntu.

### Què Farem en Aquest Tema

Tots els serveis s'instal·len al servidor 0375, amb el mateix escenari dels temes anteriors ([escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md)):

| Servei                             | Programari                     | Nom DNS                | Ports                                       |
| ---------------------------------- | ------------------------------ | ---------------------- | ------------------------------------------- |
| Missatgeria instantània (XMPP)     | Prosody                        | `xmpp.educem.cat`      | 5222/tcp (clients), 5281/tcp (fitxers)      |
| Llistes de distribució             | Mailman 3, amb Postfix i Apache | `llistes.educem.cat`  | 25 (Postfix, Tema 5), 80 i 443 (Apache, Tema 3) |
| Notícies (NNTP)                    | INN2                           | `news.educem.cat`      | 119/tcp                                     |

```
                        Xarxa interna 192.168.1.0/24 (enp0s8)
                                       │
     ┌─────────────────────────────────┼───────────────────────────────┐
     │                                 │                               │
SERVIDOR 0375 (192.168.1.2)      SERVIDOR 0374 (192.168.1.1)     CLIENT (DHCP)
Primari dels serveis 0375        Gateway (NAT) + secundari       192.168.1.50 (reserva)
  ├─ Prosody  → XMPP     5222, 5281                              Ubuntu Desktop
  ├─ Mailman 3 → llistes (Postfix + Apache)                        ├─ Gajim, Profanity
  ├─ INN2     → notícies 119                                       ├─ Thunderbird (correu, llistes
  ├─ Postfix/Dovecot → correu (Tema 5)                             │   i notícies)
  ├─ Apache/Nginx    → web (Tema 3)                                └─ navegador (Postorius)
  └─ BIND9    → DNS (Tema 1)
```

`Nota:` Les ordres del tema estan escrites per a **Ubuntu Server 24.04 LTS** (Prosody 0.12, Mailman 3.3 i INN 2.7). Ubuntu 26.04 i Debian 13 porten Prosody 13; quan una opció canvia, s'indica al mateix punt.

### Registres DNS del Tema

Al servidor DNS primari (0375), afegiu aquests registres a la zona `educem.cat` (`/etc/bind/zones/db.educem.cat`). Després, incrementeu el serial i feu `sudo rndc reload`:

```
; Tema 6 - Missatgeria, llistes i notícies
xmpp                IN  A      192.168.1.2
fitxers             IN  CNAME  xmpp
llistes             IN  CNAME  www
news                IN  A      192.168.1.2
_xmpp-client._tcp   IN  SRV    0 5 5222 xmpp.educem.cat.
```

- **`xmpp`:** el servidor de missatgeria.
- **`fitxers`:** el servei per compartir fitxers del xat (apartat 05).
- **`llistes`:** la interfície web de les llistes de distribució (apartat 08).
- **`news`:** el servidor de notícies (apartat 10).
- **Registre SRV:** les adreces de missatgeria són `usuari@educem.cat`, però el servidor és `xmpp.educem.cat`. El registre SRV indica als clients quin equip i quin port donen el servei del domini. Fa la mateixa funció que el registre MX per al correu. El veurem en detall a l'apartat 02.

**Comprovació (des del client):**

```bash
dig +short xmpp.educem.cat                       # 192.168.1.2
dig +short SRV _xmpp-client._tcp.educem.cat      # 0 5 5222 xmpp.educem.cat.
dig +short llistes.educem.cat                    # www.educem.cat. i 192.168.1.2
dig +short news.educem.cat                       # 192.168.1.2
```

Si un nom no resol al client però sí al servidor, feu `resolvectl flush-caches` al client.

### Ports Utilitzats en Aquest Tema

| Port       | Protocol | Servei                                                        |
| ---------- | -------- | ------------------------------------------------------------- |
| **5222**   | TCP      | XMPP: connexions dels clients (STARTTLS)                      |
| **5269**   | TCP      | XMPP: connexions entre servidors (federació; no l'obrirem)    |
| **5281**   | TCP      | Prosody: HTTPS per enviar fitxers                             |
| **25**     | TCP      | SMTP: el correu de les llistes entra i surt per Postfix       |
| **8024**   | TCP      | LMTP: Postfix lliura el correu de les llistes a Mailman (només local) |
| **8001**   | TCP      | API REST de Mailman (només local)                             |
| **80/443** | TCP      | Interfície web de les llistes, a través d'Apache              |
| **119**    | TCP      | NNTP: servidor de notícies                                    |
