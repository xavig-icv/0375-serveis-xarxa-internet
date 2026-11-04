## 04. Instal·lació del Servidor vsftpd

**vsftpd** (Very Secure FTP Daemon) és el servidor FTP per defecte a Debian i Ubuntu. Està dissenyat amb la seguretat com a prioritat: separa privilegis entre processos, executa les sessions amb l'usuari que s'ha autenticat i té una configuració molt restrictiva per defecte. També és molt lleuger i és el que utilitzen projectes com Debian, Red Hat o el nucli Linux per als seus servidors de descàrregues.

### Requisits Previs

Abans de començar, comproveu que el servidor 0375 té:

- IP estàtica a la interfície interna (`192.168.1.2/24` a `enp0s8`).
- Resolució de noms funcionant (Tema 1). Opcional però recomanat: registre `ftp.domini.cat`.
- Accés a Internet per instal·lar paquets.

```bash
# Comprovar la IP de la interfície interna
ip -4 addr show enp0s8

# Comprovar la resolució del nom (des del servidor o des del client)
dig +short ftp.domini.cat
```

### Instal·lació a Debian/Ubuntu

```bash
# Actualitzar els repositoris
sudo apt update

# Instal·lar el servidor vsftpd
sudo apt install vsftpd

# Verificar la versió instal·lada
apt policy vsftpd
```

**Què fa el paquet en instal·lar-se?**

- Crea el fitxer de configuració `/etc/vsftpd.conf` amb una configuració segura per defecte.
- Crea l'usuari del sistema **`ftp`** amb el directori `/srv/ftp` (es fa servir per a l'accés anònim).
- Crea el servei `vsftpd.service` i l'**engega automàticament**.

### Gestió del Servei

```bash
# Iniciar vsftpd
sudo systemctl start vsftpd

# Aturar vsftpd
sudo systemctl stop vsftpd

# Reiniciar vsftpd (cal fer-ho després de cada canvi de configuració)
sudo systemctl restart vsftpd

# Habilitar a l'arrencada del sistema
sudo systemctl enable vsftpd

# Comprovar l'estat del servei
sudo systemctl status vsftpd
```

`Nota:` `systemctl reload vsftpd` envia el senyal `SIGHUP` i vsftpd rellegeix `/etc/vsftpd.conf` per a les **sessions noves**, però els canvis a les opcions d'escolta (`listen`, `listen_ipv6`, `listen_address`, `listen_port`) només s'apliquen amb `restart`. Per no haver-hi de pensar, en aquest tema farem sempre `restart` després de cada canvi.

### Verificació del Servei

```bash
# Comprovar que el servei està actiu
systemctl is-active vsftpd

# Verificar que escolta al port 21
sudo ss -tlnp | grep ':21 '

# Sortida esperada:
# LISTEN 0  32   *:21   *:*   users:(("vsftpd",pid=1234,fd=3))
```

`*:21` indica que escolta a totes les interfícies, tant en IPv4 com en IPv6 (la configuració per defecte de Debian/Ubuntu és `listen_ipv6=YES`, que accepta els dos protocols).

### Estructura de Fitxers de vsftpd

```
/etc/
├── vsftpd.conf                  # Configuració principal del servei
├── ftpusers                     # Usuaris que MAI poden entrar per FTP (root, daemon...)
├── vsftpd.userlist              # Llista d'usuaris permesos/denegats (la crearem)
├── vsftpd/                      # Directori per a configuracions addicionals (el crearem)
│   └── user_conf/               # Configuració individual per usuari
├── pam.d/
│   └── vsftpd                   # Autenticació PAM dels usuaris locals
└── shells                       # Shells vàlides (PAM les comprova en entrar)

/srv/ftp/                        # Arrel de l'accés anònim (home de l'usuari ftp)

/var/log/
└── vsftpd.log                   # Log de connexions i transferències

/var/run/vsftpd/empty/           # Directori buit que vsftpd fa servir de gàbia interna
```

### El Fitxer /etc/ftpusers i PAM

El fitxer `/etc/pam.d/vsftpd` defineix com s'autentiquen els usuaris locals. Conté (entre d'altres) aquestes dues línies importants:

```bash
# Consultar la configuració PAM de vsftpd
cat /etc/pam.d/vsftpd
```

```
# Denega l'accés als usuaris que apareixen a /etc/ftpusers
auth  required  pam_listfile.so item=user sense=deny file=/etc/ftpusers onerr=succeed
...
# Només deixa entrar usuaris amb una shell que aparegui a /etc/shells
auth  required  pam_shells.so
```

- **`/etc/ftpusers`**: llista negra d'usuaris del sistema (`root`, `daemon`, `bin`...) que mai poden entrar per FTP. **No hi afegiu usuaris que vulgueu deixar entrar.**
- **`pam_shells.so`**: si un usuari té una shell que no és a `/etc/shells` (per exemple `/usr/sbin/nologin`), **no podrà entrar per FTP**. Ho resoldrem al punt 6.

```bash
# Veure els usuaris bloquejats per FTP
cat /etc/ftpusers
```

### Còpia de Seguretat de la Configuració Original

Abans de fer cap canvi, deseu una còpia del fitxer original. Us servirà per consultar les opcions comentades i per tornar enrere si alguna cosa falla.

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.orig

# Veure només les línies actives (sense comentaris ni línies buides)
grep -Ev '^\s*(#|$)' /etc/vsftpd.conf
```

**Configuració per defecte (Ubuntu 24.04 / Debian 12):**

```
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
```

Per defecte: sense accés anònim, els usuaris locals poden entrar però **no poden escriure** (`write_enable` està comentat), i **no estan engabiats** (`chroot_local_user` també està comentat).

### Instal·lació dels Clients de Proves

Al **client** (Ubuntu Desktop) i també al mateix **servidor** (per fer proves locals) instal·lem els clients de línia d'ordres:

```bash
# Client FTP clàssic, lftp (avançat), telnet i eines de diagnòstic de xarxa
sudo apt install ftp lftp telnet curl tcpdump netcat-openbsd
```

### Primera Prova de Connexió

Creem un usuari de proves i comprovem que el servei respon. De moment ho fem des del mateix servidor.

```bash
# Crear un usuari de proves (us demanarà la contrasenya)
sudo adduser usuari1

# Connectar al servidor FTP local
ftp localhost
```

**Sessió esperada:**

```
Connected to localhost.
220 (vsFTPd 3.0.5)
Name (localhost:admin): usuari1
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> pwd
257 "/home/usuari1" is the current directory
ftp> ls
229 Entering Extended Passive Mode (|||xxxxx|)
150 Here comes the directory listing.
226 Directory send OK.
ftp> bye
221 Goodbye.
```

Fixeu-vos que el directori actual és `/home/usuari1`: l'usuari **no està engabiat** i podria fer `cd /etc` i navegar per tot el servidor. Ho corregirem als punts següents.

### Resolució de Problemes Inicials

**El servei no arrenca:**

```bash
# Veure el motiu de l'error
sudo systemctl status vsftpd
sudo journalctl -u vsftpd -n 30 --no-pager

# Error típic: "500 OOPS: could not bind listening IPv4 socket"
# → Un altre programa ja fa servir el port 21
sudo ss -tlnp | grep ':21 '
```

**`530 Login incorrect.` amb un usuari correcte:**

```bash
# 1. L'usuari és a /etc/ftpusers?
grep -w usuari1 /etc/ftpusers

# 2. La shell de l'usuari és a /etc/shells?
getent passwd usuari1 | cut -d: -f7
cat /etc/shells

# 3. Veure el detall de l'error d'autenticació
sudo journalctl -t vsftpd -n 20 --no-pager
sudo grep vsftpd /var/log/auth.log | tail
```

**`ftp: connect: Connection refused` des del client:**

```bash
# El servei està actiu i escolta?
sudo ss -tlnp | grep ':21 '

# Hi ha tallafoc actiu al servidor?
sudo ufw status
```

### Punt de Control

- [ ] `systemctl is-active vsftpd` retorna `active`.
- [ ] `ss -tlnp` mostra vsftpd escoltant al port 21.
- [ ] Teniu una còpia de `/etc/vsftpd.conf.orig`.
- [ ] L'usuari `usuari1` pot iniciar sessió amb `ftp localhost`.
