## 03. Sistemes de Noms Jeràrquics

El DNS (Domain Name System) és un sistema jeràrquic i distribuït que permet traduir noms de domini per adreces IP. Aquesta jerarquia s’organitza en nivells d’autoritat que treballen conjuntament per respondre les consultes.

### Estructura d'Arbre del DNS

El DNS s’organitza com un arbre invertit, on el nivell més alt és l’arrel (.) i cada branca representa un domini d'un nivell inferior. Cada nivell o node es separa dels altres mitjançant un punt (.)

- L’arrel (.) és el punt de partida de tot el sistema DNS.
- Els dominis .com, .org, .net, .cat són dominis de primer nivell (Top-Level Domains o TLD).
- Els dominis de segon nivell serien (google.com, mozilla.org, gencat.cat)
- Els dominis de tercer nivell, "els subdominis" serien (www.google.com, mail.xtec.cat, etc.).

> Nota: El punt final d’un domini (.) representa l’arrel, però es pot ometre: www.google.com. → www.google.com

```
                    . (root)
                    |
    +---------------+---------------+---------------+
    |               |               |               |
   com             org             net             cat
    |               |               |               |
+---+---+       +---+---+       +---+---+       +---+---+
|       |       |       |       |       |       |       |
google amazon wikipedia mozilla        ...    gencat    xtec
|                                               |       |
+---+---+                                   +---+---+   +---+---+
|       |                                   |       |       |
www    mail                                web     ovt   odissea
```

### Els Dominis de Primer Nivell (TLD)

Els TLD (Top-Level Domains) són el primer nivell per sota de l’arrel. Estan dividits segons el seu tipus o finalitat d'ús.

#### gTLD (Generic Top-Level Domains)

Són dominis genèrics utilitzats a escala mundial:

| Extensió        | Significat      | Exemple d’ús                   |
| --------------- | --------------- | ------------------------------ |
| `.com`          | Comercial       | `google.com`, `amazon.com`     |
| `.org`          | Organitzacions  | `wikipedia.org`, `mozilla.org` |
| `.net`          | Xarxes / ISP    | `cloudflare.net`               |
| `.edu`          | Educació        | `harvard.edu`                  |
| `.gov`          | Govern dels EUA | `nasa.gov`                     |
| `.info`, `.biz` | Altres genèrics | `travel.info`, `shop.biz`      |

#### ccTLD (Country Code Top-Level Domains)

Són dominis associats a un país o territori (segons el codi ISO-3166).

| Extensió | País o regió | Exemple      |
| -------- | ------------ | ------------ |
| `.es`    | Espanya      | `rtve.es`    |
| `.cat`   | Catalunya    | `gencat.cat` |
| `.uk`    | Regne Unit   | `bbc.co.uk`  |
| `.fr`    | França       | `lemonde.fr` |
| `.de`    | Alemanya     | `dw.de`      |

#### Nous gTLD acceptats

Des de 2014, s’han creat nous dominis genèrics per ampliar les possibilitats d’identitat digital.

Exemples: `.tech`, `.app`, `.blog`, `.shop`, `.barcelona`, `.xyz`, `.online`, etc.

### Els Subdominis i Delegació

Un domini pot dividir-se en subdominis per organitzar serveis o departaments dins d’una mateixa organització.

**Exemple d'estructura:**

```
example.com (zona pare)
├── www.example.com
├── mail.example.com
├── ftp.example.com
└── dept.example.com (subdomini delegat)
    ├── web.dept.example.com
    └── db.dept.example.com
```

**Concepte de Delegació:**: El domini example.com pot delegar l’administració del subdomini dept.example.com a un altre servidor DNS. Això permet distribuir la gestió i reduir la càrrega.

Exemple: `xtec.cat` pot delegar `odissea.xtec.cat` a un altre servidor responsable del portal educatiu Odissea.

### Distribució de l'Autoritat

El sistema DNS és distribuït i jeràrquic, de manera que cada nivell és responsable d’una part del nom i pot delegar l'autoritat al nivell inferior.

- **1er: Root servers**: El sistema de DNS mundial està controlat per un total de 13 servidors, coneguts com a servidors arrel. Com la seguretat d'aquests servidors és importantíssima (sense ells, no funcionaria Internet), aquests estan replicats i poden arribar a sumar un total de 1000 servidors físics arreu del món.

> Es troben a dalt de tot de la jerarquia i es contacta amb ells quan un servidor DNS no pot resoldre un nom de domini. Per exemple: Si busco el domini `google.cat`, es pregunta al servidor arrel on puc trobar informació sobre dominis `.cat`, i aquest em redirigirà a un servidor TLD del 2n nivell de la jerarquia que coneix els servidors responsables de dominis `.cat`.

- **2n: TLD servers**: Els servidors TLD (Dominis de Nivell Superior) són responsables de resoldre direccions IP que tenen una terminació `.com`, `.org`, `.es`, `.cat`, etc. El ICANN (l'autoritat sobre els TLD), delega la responsabilitat de la seva gestió a organitzacions dels diferents països, i aquestes ho deriven a "Registradors de Noms de Domini" que permeten als usuaris (persones o entitats), `registrar un nou nom de domini`.

  - L’ICANN delega el domini `.cat` a la **Fundació puntCAT**.
  - El domini `.es` està gestionat per **Red.es**.
  - El domini `.com` està gestionat per **Verisign**.

- **3er: Authoritative servers**: Són els servidors DNS dels Registradors de Domini que contenen les diferents "bases de dades" amb la informació dels noms de domini i les IPs associades. També contenen tots els registres associats a un nom de domini i que el seu propietari ha decidit posar, com per exemple:

| Tipus de registre | Exemple                                  | Funció                                  |
| ----------------- | ---------------------------------------- | --------------------------------------- |
| **A**             | `www.exemple.cat → 174.15.80.3`          | Associa un nom a una IP IPv4            |
| **AAAA**          | `www.exemple.cat → 2001:4860:4860::8888` | Associa un nom a una IP IPv6            |
| **MX**            | `correu.exemple.cat`                     | Indica el servidor de correu            |
| **CNAME**         | `botiga.exemple.cat → www.exemple.cat`   | Àlies o redirecció                      |
| **NS**            | `ns1.exemple.cat`, `ns2.exemple.cat`     | Indica quins servidors són autoritatius |

- **4t: Recursive servers**: Són els servidors DNS de grans companyies com Cloudflare, Google, Movistar, Quad9, etc. Els ordinadors de la majoria d'usuaris d'Internet tenen configurats algun d'aquests DNS per què realitzin la resolució de noms de domini. Quan consultem per un nom de domini concret `www.viquipedia.cat`, aquests servidors busquen la resposta realitzant el procés:
  - Pregunta a un root server → qui sap sobre `.cat`
  - Pregunta al TLD server de `.cat` → qui sap sobre `viquipedia.cat`
  - Pregunta al authoritative server de `viquipedia.cat` → quina és la IP de `www.viquipedia.cat`
  - Guarda el resultat de la resolució a la memòria caché per donar una resposta immediata el proper cop que es consulti.
