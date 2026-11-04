## 10. FTPS: FTP Segur amb TLS

Al punt 5 hem comprovat amb `tcpdump` que l'FTP envia l'usuari, la contrasenya i els fitxers en text pla. **FTPS** (FTP over SSL/TLS, RFC 4217) afegeix una capa de xifrat **TLS** al protocol FTP, de la mateixa manera que HTTPS ho fa amb HTTP (Tema 3).

### FTPS Explícit i FTPS Implícit

**FTPS explícit (FTPES) - Recomanat**

- El client es connecta al port **21** (com en FTP normal).
- Abans d'autenticar-se, el client envia l'ordre **`AUTH TLS`** i es negocia el xifrat.
- El mateix port serveix per a clients amb i sense TLS (el servidor decideix si l'exigeix).

**FTPS implícit - Obsolet**

- El client es connecta al port **990** i la negociació TLS comença **immediatament**, abans de qualsevol ordre FTP.
- No està estandarditzat a l'RFC 4217. Només s'utilitza per compatibilitat amb clients antics.

```
FTPS EXPLÍCIT (port 21)                         FTPS IMPLÍCIT (port 990)

CLIENT                    SERVIDOR              CLIENT                    SERVIDOR
  │── TCP :21 ────────────────>│                  │── TCP :990 ───────────────>│
  │<─ 220 Benvinguts ──────────│                  │<══ Negociació TLS ════════>│
  │── AUTH TLS ───────────────>│                  │<─ 220 Benvinguts ──────────│  (xifrat)
  │<─ 234 Proceed with negotiation.               │── USER usuari1 ───────────>│  (xifrat)
  │<══ Negociació TLS ════════>│                  │   ...                      │
  │── USER usuari1 ───────────>│  (xifrat)
  │── PASS ******** ──────────>│  (xifrat)
  │── PBSZ 0 / PROT P ────────>│  (dades xifrades)
```

**Ordres FTP de seguretat (RFC 2228 / RFC 4217):**

| Ordre      | Funció                                                                 |
| ---------- | ---------------------------------------------------------------------- |
| `AUTH TLS` | Inicia la negociació TLS a la connexió de control                     |
| `PBSZ 0`   | Mida del buffer de protecció (amb TLS sempre és 0)                    |
| `PROT P`   | Connexió de **dades xifrada** (*Private*)                              |
| `PROT C`   | Connexió de dades **en clar** (*Clear*): només es xifra el control   |
| `CCC`      | Torna la connexió de control a text pla (per a tallafocs; desaconsellat) |

### FTPS i Tallafocs

Amb FTPS, la connexió de control va xifrada. Els tallafocs i routers **no poden llegir** les respostes `227` per obrir automàticament el port de dades (*conntrack helpers*, punt 3). Per tant:

- Cal un **rang fix de ports passius** (`pasv_min_port`/`pasv_max_port`) obert al tallafoc. Ja el tenim configurat des del punt 5.
- Si el servidor és darrere d'un NAT, cal indicar `pasv_address`.

### Pas 1: Crear el Certificat

Com al Tema 3, per a un entorn de proves farem servir un certificat **autosignat**. En un servidor públic es pot utilitzar el mateix certificat de Let's Encrypt del servidor web si el nom (`ftp.domini.cat`) hi està inclòs.

```bash
# Generar la clau privada i el certificat autosignat (vàlid 365 dies)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vsftpd.key \
  -out /etc/ssl/certs/vsftpd.crt \
  -subj "/C=ES/ST=Catalunya/L=Barcelona/O=Domini/CN=ftp.domini.cat" \
  -addext "subjectAltName=DNS:ftp.domini.cat,IP:192.168.1.2"

# Protegir la clau privada (només root)
sudo chmod 600 /etc/ssl/private/vsftpd.key

# Consultar el certificat creat
openssl x509 -in /etc/ssl/certs/vsftpd.crt -noout -subject -dates -ext subjectAltName
```

- `-subj`: evita les preguntes interactives. Canvieu `ftp.domini.cat` pel vostre nom.
- `-addext "subjectAltName=..."`: els clients moderns comproven el nom del servidor a l'extensió **SAN**, no al `CN`.

### Pas 2: Configurar vsftpd

Modifiqueu la secció SSL/TLS de `/etc/vsftpd.conf`:

```
# --- SSL/TLS (FTPS EXPLÍCIT) ---
ssl_enable=YES
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key

# Els usuaris locals HAN d'utilitzar TLS (login i dades)
force_local_logins_ssl=YES
force_local_data_ssl=YES

# Els usuaris anònims continuen en FTP normal (contingut públic)
allow_anon_ssl=NO

# Només xifrats forts
ssl_ciphers=HIGH

# Compatibilitat amb clients que no reutilitzen la sessió TLS a la connexió de dades
require_ssl_reuse=NO
```

| Directiva                | Descripció                                                                 |
| ------------------------ | -------------------------------------------------------------------------- |
| `ssl_enable`             | Activa el suport TLS (`AUTH TLS`)                                          |
| `rsa_cert_file`          | Certificat del servidor                                                    |
| `rsa_private_key_file`   | Clau privada                                                               |
| `force_local_logins_ssl` | Rebutja el login d'usuaris locals sense TLS                                |
| `force_local_data_ssl`   | Obliga a xifrar les dades (`PROT P`) dels usuaris locals                   |
| `allow_anon_ssl`         | Permet (o no) que els anònims facin servir TLS                             |
| `ssl_ciphers`            | Llista de xifrats d'OpenSSL acceptats                                      |
| `require_ssl_reuse`      | Exigeix que la connexió de dades reutilitzi la sessió TLS del control (més segur, però alguns clients fallen) |
| `implicit_ssl`           | `YES` per a FTPS implícit (amb `listen_port=990`)                          |

`Nota:` vsftpd no permet SSLv2 ni SSLv3 per defecte, i les versions antigues de TLS (1.0 i 1.1) queden desactivades per la política d'OpenSSL de les distribucions actuals. Normalment es negociarà **TLS 1.3**.

```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd --no-pager
```

### Pas 3: Comprovar que el Servidor Exigeix TLS

**Amb el client `ftp` clàssic (sense TLS):**

```
$ ftp 192.168.1.2
220 Benvinguts al servidor FTP de domini.cat
Name (192.168.1.2:alumne): usuari1
530 Non-anonymous sessions must use encryption.
Login failed.
```

Els usuaris locals **ja no poden entrar sense xifrat**. En canvi, `anonymous` continua funcionant en FTP normal.

**Amb openssl (veure la negociació TLS):**

```bash
openssl s_client -connect 192.168.1.2:21 -starttls ftp
```

A la sortida comproveu:

```
subject=C=ES, ST=Catalunya, L=Barcelona, O=Domini, CN=ftp.domini.cat
...
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
...
Verify return code: 18 (self-signed certificate)
```

El codi 18 és normal: el certificat és autosignat.

### Pas 4: Connectar amb Clients FTPS

**lftp** (prova primer `AUTH TLS` automàticament):

```bash
# Desactivar la verificació del certificat NOMÉS perquè és autosignat
lftp -u usuari1 -e "set ssl:verify-certificate no; set ftp:ssl-force true; ls; bye" 192.168.1.2
```

Per no haver de desactivar la verificació, es pot confiar en el certificat del servidor:

```bash
# Al client: copiar el certificat del servidor (amb el vostre usuari administrador)
scp admin@192.168.1.2:/etc/ssl/certs/vsftpd.crt ~/vsftpd.crt
# Connectar pel nom DNS (ha de coincidir amb el subjectAltName del certificat)
lftp -u usuari1 -e "set ssl:ca-file $HOME/vsftpd.crt; set ftp:ssl-force true; ls; bye" ftp.domini.cat
```

**curl:**

```bash
# --ssl-reqd: exigeix TLS; -k: accepta el certificat autosignat
curl --ssl-reqd -k -u usuari1 ftp://192.168.1.2/fitxers/

# Pujar un fitxer per FTPS
curl --ssl-reqd -k -u usuari1 -T informe.pdf ftp://192.168.1.2/fitxers/
```

**FileZilla:** al gestor de llocs, canvieu el xifratge:

```
Protocol:   FTP - Protocol de transferència de fitxers
Xifratge:   Requereix FTP explícit sobre TLS
```

En connectar, FileZilla mostrarà el certificat (emissor, empremta digital, dates). Com que és autosignat, cal **acceptar-lo manualment** (marqueu "Confia sempre en aquest certificat"). Al registre de missatges veureu:

```
Ordre:     AUTH TLS
Resposta:  234 Proceed with negotiation.
Estat:     Inicialitzant TLS...
Estat:     Connexió TLS establerta.
Ordre:     USER usuari1
...
Ordre:     PBSZ 0
Ordre:     PROT P
Resposta:  200 PROT now Private.
```

### Pas 5: Comprovar el Xifrat amb tcpdump

Repetiu la captura del punt 5 al servidor:

```bash
sudo tcpdump -l -i enp0s8 -nn -A 'tcp port 21' | grep -E 'USER|PASS|AUTH'
```

Connecteu-vos amb `lftp` o FileZilla. Ara **només veureu `AUTH TLS`**: la resta de la conversa (usuari, contrasenya, ordres i dades) és xifrada i il·legible.

### FTPS Implícit (Opcional)

Si un client antic només suporta FTPS implícit, es pot configurar una **segona instància** de vsftpd al port 990. No ho farem a la pràctica, però la configuració seria:

```
listen_port=990
implicit_ssl=YES
ssl_enable=YES
```

### Resolució de Problemes

**`500 OOPS: SSL: cannot load RSA certificate`**

- La ruta de `rsa_cert_file` o `rsa_private_key_file` és incorrecta o el fitxer no existeix.

**El login funciona però el llistat es queda penjat o falla (`425`):**

- El rang passiu no està obert al tallafoc. Amb FTPS els *conntrack helpers* no funcionen.

**`522 SSL connection failed; session reuse required: see require_ssl_reuse option in vsftpd.conf man page`**

- El client no reutilitza la sessió TLS a la connexió de dades. Afegiu `require_ssl_reuse=NO`.

**`530 Non-anonymous sessions must use encryption.`**

- El client no fa servir TLS. Configureu-lo en mode "FTP explícit sobre TLS".

**`lftp: Fatal error: Certificate verification: Not trusted`**

- El certificat és autosignat. Feu servir `set ssl:ca-file` o, només per a proves, `set ssl:verify-certificate no`.

### Punt de Control

- [ ] El client `ftp` sense TLS rep `530 Non-anonymous sessions must use encryption.`
- [ ] `openssl s_client -starttls ftp` mostra el vostre certificat i TLS 1.3.
- [ ] Podeu transferir fitxers amb `lftp`, `curl --ssl-reqd` i FileZilla en mode FTPS explícit.
- [ ] La captura amb `tcpdump` ja no mostra ni l'usuari ni la contrasenya.
- [ ] L'accés anònim continua funcionant sense TLS.
