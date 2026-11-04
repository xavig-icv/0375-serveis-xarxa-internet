## 02. El Protocol FTP

### Arquitectura de l'FTP

L'FTP (File Transfer Protocol) és un protocol **client-servidor** de la capa d'aplicació que funciona sobre **TCP**. La seva característica més important, i la que el fa diferent de la majoria de protocols, és que utilitza **dues connexions TCP independents**:

**1. Connexió de control (port 21)**

- La inicia sempre el **client** cap al port **21** del servidor.
- Per aquesta connexió viatgen les **ordres** del client (`USER`, `LIST`, `RETR`...) i les **respostes** del servidor (codis numèrics `220`, `331`, `226`...).
- Es manté oberta durant **tota la sessió**.
- És text pla i llegible (com SMTP o HTTP/1.1).

**2. Connexió de dades (port 20 o port dinàmic)**

- S'obre **cada vegada** que cal transferir informació: un fitxer o un llistat de directori.
- Es tanca quan acaba la transferència.
- Qui l'obre (client o servidor) depèn del **mode de connexió**: **actiu** o **passiu** (ho veurem al punt següent).

```
        CLIENT                                         SERVIDOR FTP
   ┌──────────────┐                                ┌──────────────┐
   │              │  CONNEXIÓ DE CONTROL (TCP 21)  │              │
   │  Intèrpret   │<══════════════════════════════>│  Intèrpret   │
   │  de protocol │   USER, PASS, LIST, RETR...    │  de protocol │
   │     (PI)     │   220, 331, 230, 150, 226...   │     (PI)     │
   │              │                                │              │
   │  Procés de   │  CONNEXIÓ DE DADES (TCP 20/xx) │  Procés de   │
   │  transfer.   │<──────────────────────────────>│  transfer.   │
   │     (DTP)    │   contingut dels fitxers i     │     (DTP)    │
   │              │   llistats de directoris       │              │
   └──────────────┘                                └──────────────┘
```

L'RFC 959 anomena aquests components **PI** (Protocol Interpreter, gestiona les ordres) i **DTP** (Data Transfer Process, mou les dades).

**Per què dues connexions?** L'FTP es va dissenyar als anys 70 perquè el client pogués continuar enviant ordres (per exemple, avortar una transferència amb `ABOR`) mentre les dades es transferien. Aquest disseny és el que avui provoca problemes amb tallafocs i NAT.

### Seqüència d'una Sessió FTP

```
CLIENT                                                SERVIDOR (port 21)
  │                                                          │
  │──────────────── Connexió TCP al port 21 ────────────────>│
  │<──────────── 220 (vsFTPd 3.0.5) ─────────────────────────│  Benvinguda
  │                                                          │
  │──────────── USER usuari1 ───────────────────────────────>│
  │<──────────── 331 Please specify the password. ───────────│
  │──────────── PASS Contrasenya ───────────────────────────>│  (en text pla!)
  │<──────────── 230 Login successful. ──────────────────────│
  │                                                          │
  │──────────── PWD ────────────────────────────────────────>│
  │<──────────── 257 "/" is the current directory ───────────│
  │                                                          │
  │──────────── TYPE I ─────────────────────────────────────>│  Mode binari
  │<──────────── 200 Switching to Binary mode. ──────────────│
  │                                                          │
  │──────────── PASV ───────────────────────────────────────>│  Mode passiu
  │<──────────── 227 Entering Passive Mode (192,168,1,2,156,64)
  │                                                          │
  │═══════ Connexió de dades al port 40000 ═════════════════>│
  │──────────── RETR informe.pdf ───────────────────────────>│
  │<──────────── 150 Opening BINARY mode data connection ────│
  │<═══════ contingut del fitxer ════════════════════════════│
  │<──────────── 226 Transfer complete. ─────────────────────│
  │                                                          │
  │──────────── QUIT ───────────────────────────────────────>│
  │<──────────── 221 Goodbye. ───────────────────────────────│
```

### Ordres del Protocol FTP

Cal distingir entre les **ordres del protocol** (les que viatgen per la connexió de control, en majúscules i definides a l'RFC) i les **ordres del client** (les que escriu l'usuari al programa `ftp`, com `get`, `put` o `ls`). El client tradueix les seves ordres a ordres del protocol.

| Ordre del protocol | Ordre del client `ftp` | Descripció                                          |
| ------------------ | ---------------------- | --------------------------------------------------- |
| `USER nom`         | (automàtic en connectar) | Envia el nom d'usuari                             |
| `PASS contrasenya` | (automàtic en connectar) | Envia la contrasenya                              |
| `PWD`              | `pwd`                  | Mostra el directori actual del servidor             |
| `CWD carpeta`      | `cd carpeta`           | Canvia de directori al servidor                     |
| `CDUP`             | `cd ..`                | Puja al directori pare                              |
| `LIST` / `NLST`    | `ls` / `dir`           | Llistat del directori (per la connexió de dades)    |
| `MLSD`             | `ls` (clients moderns) | Llistat en format normalitzat (RFC 3659)            |
| `RETR fitxer`      | `get fitxer`           | Descarrega un fitxer                                |
| `STOR fitxer`      | `put fitxer`           | Puja un fitxer                                      |
| `APPE fitxer`      | `append fitxer`        | Afegeix contingut al final d'un fitxer              |
| `DELE fitxer`      | `delete fitxer`        | Esborra un fitxer                                   |
| `MKD carpeta`      | `mkdir carpeta`        | Crea un directori                                   |
| `RMD carpeta`      | `rmdir carpeta`        | Esborra un directori buit                           |
| `RNFR` + `RNTO`    | `rename origen destí`  | Canvia el nom d'un fitxer (dues ordres seguides)    |
| `TYPE A` / `TYPE I`| `ascii` / `binary`     | Tipus de transferència: text o binari               |
| `PASV` / `EPSV`    | `passive`              | Demana el mode passiu                               |
| `PORT` / `EPRT`    | (mode actiu)           | Indica al servidor on connectar en mode actiu       |
| `SIZE fitxer`      | `size fitxer`          | Mida del fitxer en bytes                            |
| `REST posició`     | `reget fitxer`         | Reprèn una transferència interrompuda               |
| `SITE CHMOD`       | `chmod 644 fitxer`     | Canvia permisos (ordre específica del servidor)     |
| `SYST`             | `system`               | Tipus de sistema operatiu del servidor              |
| `FEAT`             | `features`             | Llista les extensions que suporta el servidor       |
| `AUTH TLS`         | (clients FTPS)         | Inicia el xifrat TLS (FTPS explícit, RFC 4217)      |
| `QUIT`             | `bye` / `quit`         | Tanca la sessió                                     |

### Codis de Resposta FTP

Igual que SMTP i HTTP, el servidor respon amb un **codi de 3 xifres** seguit d'un text. **El primer dígit** indica el tipus de resposta:

| Primer dígit | Significat                        | Exemple                                        |
| ------------ | --------------------------------- | ---------------------------------------------- |
| **1xx**      | Resposta preliminar (en curs)     | `150 Here comes the directory listing.`        |
| **2xx**      | Acció completada correctament     | `226 Transfer complete.`                       |
| **3xx**      | Cal més informació del client     | `331 Please specify the password.`             |
| **4xx**      | Error temporal (es pot reintentar) | `425 Failed to establish connection.`         |
| **5xx**      | Error permanent                   | `530 Login incorrect.`                         |

**Codis més habituals:**

```
220  Servei preparat (missatge de benvinguda)
221  Tancament de la connexió de control (adéu)
226  Transferència completada; es tanca la connexió de dades
227  Entrant en mode passiu (h1,h2,h3,h4,p1,p2)
229  Entrant en mode passiu estès (|||port|)
230  Usuari autenticat correctament
234  AUTH TLS acceptat: comença la negociació TLS
250  Acció sobre el fitxer completada (CWD, DELE...)
257  Directori actual o directori creat
331  Usuari correcte, cal la contrasenya
350  Pendent d'informació addicional (després de RNFR)
421  Servei no disponible (massa connexions, timeout)
425  No s'ha pogut obrir la connexió de dades
426  Connexió de dades tancada; transferència avortada
451  Error local en processar (p. ex. quota de disc superada)
500  Ordre desconeguda o error de sintaxi
530  No autenticat (usuari o contrasenya incorrectes)
550  Acció no realitzada: fitxer no trobat o sense permisos
553  Nom de fitxer no permès
```

### Tipus de Transferència: ASCII i Binari

L'FTP pot transferir els fitxers de dues maneres:

**Mode ASCII (`TYPE A`)**

- Pensat per a **fitxers de text**.
- Converteix els salts de línia entre sistemes: Windows fa servir `CR+LF` (`\r\n`) i Linux només `LF` (`\n`).
- Si s'utilitza amb fitxers binaris, **els corromp** (una imatge o un ZIP deixen de funcionar).

**Mode binari (`TYPE I`, Image)**

- Copia els bytes **exactament** tal com són, sense cap conversió.
- S'ha d'utilitzar per a imatges, executables, ZIP, PDF, ISO, documents ofimàtics, etc.
- És el **mode per defecte** a tots els clients moderns i el més segur d'utilitzar sempre.

`Nota:` vsftpd, per seguretat, **no fa la conversió ASCII** encara que el client la demani (opcions `ascii_upload_enable` i `ascii_download_enable` desactivades per defecte). A la pràctica, sempre transferirem en binari.

### Modes de Transferència

L'RFC 959 defineix també tres **modes de transferència** de dades (no confondre amb actiu/passiu):

- **Stream (`MODE S`)**: les dades s'envien com un flux continu i el final del fitxer s'indica tancant la connexió. **És l'únic que s'utilitza a la pràctica.**
- **Block (`MODE B`)**: les dades s'envien en blocs amb capçalera.
- **Compressed (`MODE C`)**: les dades s'envien comprimides. Pràcticament no s'utilitza.

### Extensions Modernes de l'FTP

L'RFC 959 s'ha anat ampliant amb altres RFC per adaptar-lo a les necessitats actuals:

| RFC      | Any  | Extensió                                                         |
| -------- | ---- | ---------------------------------------------------------------- |
| RFC 2228 | 1997 | Extensions de seguretat (`AUTH`, `PROT`, `PBSZ`)                 |
| RFC 2389 | 1998 | Ordre `FEAT` per descobrir les capacitats del servidor           |
| RFC 2428 | 1998 | `EPRT` i `EPSV`: suport per a **IPv6** i NAT                     |
| RFC 2640 | 1999 | Internacionalització: noms de fitxer en **UTF-8**                |
| RFC 3659 | 2007 | `MLSD`, `MLST`, `SIZE`, `MDTM`, `REST STREAM`                    |
| RFC 4217 | 2005 | **FTPS**: FTP sobre TLS                                          |

### Problemes de Seguretat de l'FTP Clàssic

L'FTP es va dissenyar quan Internet era una xarxa acadèmica de confiança. Per això té problemes greus de seguretat:

- **Credencials en text pla**: qualsevol que capturi el trànsit veu l'usuari i la contrasenya (ho comprovarem amb `tcpdump` al punt 5).
- **Dades sense xifrar**: el contingut dels fitxers també es pot llegir o modificar pel camí.
- **Sense verificació del servidor**: el client no pot saber si es connecta al servidor real (atacs *man-in-the-middle*).
- **Atac FTP bounce**: amb l'ordre `PORT` un atacant podia fer que el servidor connectés a tercers (els servidors actuals ho bloquegen).
- **Problemes amb tallafocs**: obrir rangs de ports augmenta la superfície d'atac.

**Conclusió:** l'FTP sense xifrar només s'hauria d'utilitzar en xarxes internes controlades o per a descàrregues públiques anònimes. Per a la resta de casos s'utilitza **FTPS** o, preferiblement, **SFTP**.
