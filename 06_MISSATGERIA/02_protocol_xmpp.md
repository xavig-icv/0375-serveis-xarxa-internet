## 02. El Protocol XMPP

### Què és XMPP?

**XMPP** (*Extensible Messaging and Presence Protocol*) és el protocol obert de missatgeria instantània i presència de l'IETF:

- Va néixer l'any 1999 amb el nom de **Jabber**. Per això encara es parla de "comptes Jabber".
- És un estàndard: RFC 6120 (nucli del protocol), RFC 6121 (missatgeria i presència) i RFC 7622 (format de les adreces).
- Els missatges són fragments d'**XML** que viatgen per una connexió TCP oberta durant tota la sessió.
- És **extensible**: les funcions noves (sales, historial, fitxers, xifratge) es defineixen en documents anomenats **XEP** (*XMPP Extension Protocols*), que publica la XMPP Standards Foundation.

### Adreces XMPP (JID)

Cada usuari, servidor o sala té una adreça anomenada **JID** (*Jabber ID*), amb el format `usuari@domini/recurs`:

| JID                                | Què identifica                                                       |
| ---------------------------------- | -------------------------------------------------------------------- |
| `educem.cat`                       | El servei de missatgeria del domini                                  |
| `usuari1@educem.cat`               | Un compte d'usuari (*bare JID*)                                      |
| `usuari1@educem.cat/portatil`      | Una connexió concreta del compte (*full JID*): el portàtil           |
| `usuari1@educem.cat/mobil`         | Una altra connexió del mateix compte: el mòbil                       |
| `sales.educem.cat`                 | El servei de sales de xat (apartat 05)                               |
| `asix@sales.educem.cat`            | Una sala de xat                                                      |
| `asix@sales.educem.cat/Anna`       | Una participant de la sala, amb l'àlies `Anna`                       |

El **recurs** permet que un mateix usuari estigui connectat des de diversos dispositius alhora. El servidor sap a quina connexió ha d'enviar cada missatge.

### Arquitectura: Clients, Servidors i Federació

```
 usuari1@educem.cat                                          maria@altra.test
     (Gajim)                                                     (Dino)
        │ c2s: port 5222                                            │ c2s: port 5222
        ▼                                                           ▼
 ┌──────────────────┐          s2s: port 5269 (federació)    ┌──────────────────┐
 │ Prosody          │◄──────────────────────────────────────►│ ejabberd         │
 │ educem.cat       │                                         │ altra.test       │
 └──────────────────┘                                         └──────────────────┘
```

- **c2s (*client to server*), port 5222:** cada client es connecta al servidor del seu domini.
- **s2s (*server to server*), port 5269:** els servidors de dominis diferents es connecten entre ells per lliurar els missatges. És la **federació**, igual que fan els servidors de correu.
- Totes dues connexions comencen sense xifrar i passen a TLS amb **STARTTLS**. També hi ha ports amb TLS directe: 5223 per als clients i 5270 per als servidors.

### Com És una Sessió XMPP

```
CLIENT (usuari1)                                         SERVIDOR (educem.cat)
     │  1. Connexió TCP al port 5222                             │
     │  2. <stream:stream to='educem.cat'>   ───────────────────>│
     │  3. STARTTLS: la connexió passa a anar xifrada  <────────>│
     │  4. Autenticació SASL (SCRAM o PLAIN dins de TLS) ───────>│
     │  5. Assignació del recurs (portatil)                     │
     │  6. Presència inicial: "estic en línia"         ─────────>│ ──> avisa els contactes
     │  7. Demana la llista de contactes               ─────────>│
     │  8. Missatges en tots dos sentits               <────────>│
     │  9. </stream:stream>: fi de la sessió                     │
```

Dins de la sessió s'envien tres tipus de paquets XML, anomenats **stanzas**:

```xml
<!-- message: un missatge a un altre usuari -->
<message from='usuari1@educem.cat/portatil' to='usuari2@educem.cat' type='chat'>
  <body>Hola! Has acabat la pràctica?</body>
</message>

<!-- presence: l'usuari canvia el seu estat -->
<presence from='usuari1@educem.cat/portatil'>
  <show>away</show>
  <status>Dinant</status>
</presence>

<!-- iq: una petició amb resposta; aquí, demanar la llista de contactes -->
<iq from='usuari1@educem.cat/portatil' type='get' id='r1'>
  <query xmlns='jabber:iq:roster'/>
</iq>
```

| Stanza       | Funció                                                                  |
| ------------ | ----------------------------------------------------------------------- |
| `<message>`  | Enviar un missatge (a una persona o a una sala)                         |
| `<presence>` | Informar de l'estat (en línia, absent, no molestar) i gestionar subscripcions |
| `<iq>`       | Peticions i respostes (*info/query*): llista de contactes, configuració, etc. |

### Llista de Contactes i Subscripció de Presència

La llista de contactes (*roster*) es guarda al servidor. Per veure l'estat d'un contacte, cal que el contacte ho autoritzi:

1. `usuari1` afegeix `usuari2@educem.cat` a la seva llista i li demana la **subscripció** a la seva presència.
2. `usuari2` rep la petició i l'accepta (o la rebutja).
3. A partir d'aquí, `usuari1` veu quan `usuari2` està connectat. Si `usuari2` també ho demana i `usuari1` ho accepta, la subscripció és mútua.

Ningú no pot veure el teu estat sense el teu permís.

### Extensions (XEP) Més Importants

| XEP          | Nom                            | Per a què serveix                                                   |
| ------------ | ------------------------------ | ------------------------------------------------------------------- |
| **XEP-0030** | Service Discovery              | Saber quins serveis i funcions ofereix un servidor o un client      |
| **XEP-0045** | Multi-User Chat (MUC)          | Sales de xat (apartat 05)                                           |
| **XEP-0198** | Stream Management              | Recuperar la sessió sense perdre missatges quan falla la connexió (mòbils) |
| **XEP-0280** | Message Carbons                | Rebre una còpia dels missatges a tots els dispositius connectats    |
| **XEP-0313** | Message Archive Management     | Historial de converses al servidor                                  |
| **XEP-0363** | HTTP File Upload               | Enviar fitxers a contactes i sales (apartat 05)                     |
| **XEP-0384** | OMEMO                          | Xifratge extrem a extrem                                            |
| **XEP-0166** | Jingle                         | Trucades d'àudio i vídeo                                            |

Els clients i els servidors no implementen tots els XEP. El client pregunta al servidor quins suporta (XEP-0030) i fa servir només aquests.

### Registres DNS SRV

Les adreces de missatgeria fan servir el domini (`usuari1@educem.cat`), però el servidor és un altre equip (`xmpp.educem.cat`). Els registres **SRV** indiquen quin equip i quin port donen cada servei:

```
; Servei             Classe Tipus Prioritat Pes Port Destí
_xmpp-client._tcp    IN     SRV   0         5   5222 xmpp.educem.cat.   ; per als clients
_xmpp-server._tcp    IN     SRV   0         5   5269 xmpp.educem.cat.   ; per als servidors (federació)
```

- **Prioritat:** es fa servir primer el destí amb el número més baix, com al registre MX.
- **Pes:** reparteix la càrrega entre destins amb la mateixa prioritat.
- Si el domini no té registres SRV, el client prova de connectar-se a l'adreça del mateix domini (`educem.cat`) pel port 5222.
- Al laboratori només creem el registre `_xmpp-client`, perquè no farem federació.

**El certificat ha de ser del domini, no de l'equip.** El client es vol connectar al servei `educem.cat`. Per això, el certificat TLS del servidor ha d'incloure `educem.cat`, encara que el servidor es digui `xmpp.educem.cat`.

### Seguretat en XMPP

| Capa                          | Què protegeix                                       | Com                                         |
| ----------------------------- | --------------------------------------------------- | ------------------------------------------- |
| **TLS** (c2s i s2s)           | La connexió entre el client i el servidor, i entre servidors | STARTTLS al port 5222/5269          |
| **SASL SCRAM**                | La contrasenya: el servidor no la guarda en clar    | `authentication = "internal_hashed"` a Prosody |
| **OMEMO** (extrem a extrem)   | El contingut dels missatges, fins i tot davant del servidor | Claus als dispositius dels usuaris  |

Encara que els missatges vagin xifrats extrem a extrem, el servidor sap **qui parla amb qui i quan** (metadades). L'administrador del servei ha de protegir aquestes dades igual que els registres del correu.
