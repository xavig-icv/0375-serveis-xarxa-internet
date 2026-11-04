## 01. Introducció als Serveis de Transferència de Fitxers

### Què és un Servei de Transferència de Fitxers?

Un **servei de transferència de fitxers** permet copiar fitxers entre dos equips a través d'una xarxa (LAN o Internet): **pujar** fitxers (upload) del client cap al servidor i **descarregar** fitxers (download) del servidor cap al client. A més, permet gestionar remotament el sistema de fitxers del servidor: crear i esborrar carpetes, canviar noms, consultar mides, canviar permisos, etc.

Tot i que avui dia molts usuaris comparteixen fitxers amb serveis al núvol (Google Drive, OneDrive, Dropbox), a nivell d'administració de sistemes els protocols de transferència de fitxers continuen sent imprescindibles:

- Publicar i actualitzar llocs web en un servidor (hosting).
- Fer còpies de seguretat automàtiques cap a un servidor remot.
- Intercanviar fitxers entre empreses (factures, comandes, fitxers bancaris).
- Distribuir programari, imatges ISO i actualitzacions.
- Arrencar equips per xarxa (PXE) i desar configuracions de routers i switches.

**Funcionament bàsic:**

```
CLIENT (FileZilla / ftp / sftp)               SERVIDOR (vsftpd / OpenSSH)
      │                                              │
      │  1. Connexió + autenticació                  │
      │     (usuari i contrasenya, clau o anònim)    │
      │─────────────────────────────────────────────>│
      │                                              │ 2. Comprova credencials
      │                                              │    i permisos de l'usuari
      │  3. Ordres: llistar, canviar de carpeta...   │
      │─────────────────────────────────────────────>│
      │                                              │
      │  4. Transferència de dades                   │
      │     put fitxer.zip  ──────────────────────>  │  (pujada)
      │     get informe.pdf <──────────────────────  │  (descàrrega)
      │                                              │
      │  5. Tancament de la sessió                   │
```

### Protocols de Transferència de Fitxers

No hi ha un únic protocol. Cada un neix en un context diferent i té avantatges i inconvenients:

**FTP (File Transfer Protocol)**

- És el protocol clàssic (RFC 959, de l'any 1985; la primera versió és de 1971).
- Utilitza **dues connexions**: una de **control** (port 21) i una de **dades**.
- **No xifra res**: usuari, contrasenya i fitxers viatgen en text pla.
- Permet accés **anònim** (servidors públics de descàrregues).
- Encara és molt present en hostings, dispositius antics, impressores i càmeres IP.

**FTPS (FTP over SSL/TLS)**

- És FTP amb una capa de xifrat **TLS** (RFC 4217), igual que HTTPS respecte HTTP.
- Mode **explícit** (port 21 + ordre `AUTH TLS`) o **implícit** (port 990, obsolet).
- Manté les dues connexions de l'FTP: continua sent complicat amb tallafocs i NAT.

**SFTP (SSH File Transfer Protocol)**

- **No és FTP**: és un subsistema del protocol **SSH** (port 22).
- Tot (autenticació, ordres i dades) viatja **xifrat** en **una única connexió**.
- Permet autenticació per **contrasenya** o per **claus públiques** (ideal per automatitzar).
- És l'estàndard actual per a l'administració de servidors i l'intercanvi segur de fitxers.

**SCP (Secure Copy)**

- Còpia de fitxers sobre SSH amb una sintaxi semblant a `cp`.
- A partir d'OpenSSH 9.0 (2022), l'ordre `scp` utilitza internament el protocol SFTP.

**TFTP (Trivial File Transfer Protocol)**

- Versió mínima de transferència de fitxers sobre **UDP** (port 69).
- **Sense autenticació** ni llistat de directoris.
- S'utilitza per arrencar equips per xarxa (PXE) i per desar o carregar configuracions i firmware de dispositius de xarxa.

**Altres alternatives actuals**

- **HTTP/HTTPS**: descàrregues públiques (mirrors de distribucions Linux, repositoris `apt`).
- **WebDAV**: extensió d'HTTP per pujar i gestionar fitxers (Nextcloud, SharePoint).
- **rsync**: sincronització incremental (només copia les diferències), normalment sobre SSH.
- **Emmagatzematge d'objectes (S3)**: API HTTP per desar fitxers al núvol (AWS S3, MinIO).

### Comparativa de Protocols

| Protocol | Port(s)                  | Transport | Xifrat | Connexions | Autenticació             | Ús típic actual                      |
| -------- | ------------------------ | --------- | ------ | ---------- | ------------------------ | ------------------------------------ |
| **FTP**  | 21 (control) + dades     | TCP       | No     | 2          | Usuari/contrasenya, anònim | Xarxes internes, dispositius antics  |
| **FTPS** | 21 (explícit) / 990      | TCP       | TLS    | 2          | Usuari/contrasenya, certificats | Hostings, entorns que exigeixen FTP |
| **SFTP** | 22                       | TCP (SSH) | SSH    | 1          | Contrasenya, claus SSH   | Administració, B2B, còpies de seguretat |
| **SCP**  | 22                       | TCP (SSH) | SSH    | 1          | Contrasenya, claus SSH   | Còpies puntuals entre servidors      |
| **TFTP** | 69                       | UDP       | No     | 1          | Cap                      | PXE, firmware de routers i switches  |

**Idea clau:** FTP i SFTP tenen noms semblants però són protocols **completament diferents**. FTPS és FTP xifrat; SFTP és SSH.

### Tipus d'Usuaris en un Servidor FTP

**Usuaris anònims**

- Entren amb l'usuari `anonymous` (o `ftp`) i sense contrasenya real.
- Normalment només poden **descarregar** fitxers públics.
- Es fan servir en servidors públics de distribució de programari.

**Usuaris locals (del sistema)**

- Són comptes reals del sistema Linux (`/etc/passwd`).
- Accedeixen amb la seva contrasenya del sistema i treballen dins la seva carpeta.
- S'han de **tancar en una gàbia (chroot)** perquè no puguin navegar per tot el servidor.

**Usuaris virtuals**

- No existeixen al sistema: es guarden en una base de dades pròpia del servei (fitxer, MySQL, LDAP).
- Tots s'executen amb un únic usuari del sistema (per exemple `ftpvirtual`).
- És el model que fan servir els proveïdors d'allotjament amb milers de clients.

### Servidors de Transferència de Fitxers Populars

**Servidors FTP/FTPS**

- **vsftpd** (Very Secure FTP Daemon): lleuger, segur i el més utilitzat a Debian/Ubuntu. **És el que farem servir.**
- **ProFTPD**: molt configurable, amb sintaxi semblant a la d'Apache.
- **Pure-FTPd**: senzill, amb bon suport per a usuaris virtuals.
- **FileZilla Server** i **IIS FTP**: opcions habituals en entorns Windows.

**Servidors SFTP**

- **OpenSSH**: inclòs a pràcticament tots els sistemes Linux. **És el que farem servir.**
- **SFTPGo**: servidor modern amb panell web, usuaris virtuals i suport per a SFTP, FTP/S, WebDAV i emmagatzematge S3.
- **Serveis gestionats al núvol**: AWS Transfer Family, Azure Blob Storage (SFTP).

### Escenari de Pràctiques

Durant tot el tema treballarem amb el mateix escenari dels temes anteriors. El servidor 0375 tindrà els serveis de transferència de fitxers i un client de la xarxa interna farà les proves.

```
                        Xarxa interna 192.168.1.0/24 (enp0s8)
                                       │
     ┌─────────────────────────────────┼───────────────────────────────┐
     │                                 │                               │
SERVIDOR 0375 (192.168.1.2)      SERVIDOR 0374 (192.168.1.1)     CLIENT (DHCP)
Ubuntu Server / Debian           DNS secundari (Tema 1)          192.168.1.1xx
  ├─ vsftpd   → FTP/FTPS  21 + 40000-40100/tcp                   Ubuntu Desktop / Windows
  ├─ OpenSSH  → SFTP      22/tcp                                   ├─ ftp, lftp, curl
  ├─ tftpd-hpa→ TFTP      69/udp                                   ├─ sftp, scp
  ├─ BIND9    → DNS       (Tema 1)                                 └─ FileZilla / WinSCP
  └─ Apache/Nginx → Web   (Tema 3)
```

**Registres DNS recomanats** (afegiu-los a la vostra zona del Tema 1 i incrementeu el serial):

```
; Serveis de transferència de fitxers
ftp         IN  A       192.168.1.2
sftp        IN  CNAME   ftp
tftp        IN  CNAME   ftp
```

`Nota:` Substituïu `domini.cat` pel vostre domini i `192.168.1.2` per la IP del vostre servidor. Les ordres del tema estan escrites per a **Ubuntu Server 24.04 LTS** i **Debian 12/13**; quan hi ha diferències entre distribucions o versions, s'indiquen al mateix punt.

### Ports Utilitzats en Transferència de Fitxers

| Port            | Protocol | Descripció                                         |
| --------------- | -------- | -------------------------------------------------- |
| **20/tcp**      | FTP      | Dades en mode actiu (el servidor connecta al client) |
| **21/tcp**      | FTP/FTPS | Connexió de control (ordres i respostes)           |
| **40000-40100** | FTP/FTPS | Rang de ports de dades en mode passiu (configurable) |
| **990/tcp**     | FTPS     | FTPS implícit (obsolet)                            |
| **22/tcp**      | SFTP/SCP | SSH (control i dades en la mateixa connexió)      |
| **69/udp**      | TFTP     | Peticions TFTP                                     |
