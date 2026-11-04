## 14. Seguretat, Tallafoc i Documentació del Servei

En aquest darrer punt revisarem la seguretat de tot el servei de transferència de fitxers:

- **Tallafoc** (`ufw`) amb només els ports necessaris.
- **fail2ban** per bloquejar atacs de força bruta contra FTP i SFTP.
- **Configuració final** de vsftpd, amb totes les directives del tema.
- **Criteris per triar** el protocol adequat a cada situació.
- **Documentació** del servei instal·lat.

### Escoltar Només on Cal

Fins ara vsftpd escolta a **totes** les interfícies (també a la WAN `enp0s3`). Si el servei només s'ha d'oferir a la xarxa interna, és millor limitar-lo a la seva IP:

```
# --- MODE D'ESCOLTA (només IPv4 a la interfície interna) ---
listen=YES
listen_ipv6=NO
listen_address=192.168.1.2
```

- `listen_address` només funciona amb `listen=YES` (IPv4).
- A partir d'ara `ftp localhost` **no funcionarà**: cal connectar a `192.168.1.2`.
- Com a efecte secundari, els logs mostraran les IPs en format IPv4 normal (sense el prefix `::ffff:`), cosa que facilita el treball de fail2ban.

```bash
sudo systemctl restart vsftpd
sudo ss -tlnp | grep vsftpd
# LISTEN 0 32 192.168.1.2:21 0.0.0.0:* users:(("vsftpd",...))
```

### Tallafoc amb UFW

**Ports necessaris per als serveis d'aquest tema:**

| Servei      | Port/s                | Protocol | Origen permès             |
| ----------- | --------------------- | -------- | ------------------------- |
| SSH / SFTP  | 22                    | TCP      | Qualsevol (o xarxa d'administració) |
| FTP / FTPS  | 21                    | TCP      | Xarxa interna             |
| FTP passiu  | 40000-40100           | TCP      | Xarxa interna             |
| TFTP        | 69                    | UDP      | Xarxa interna             |

**Configuració:**

```bash
# 1. PRIMER de tot, permetre SSH (si no, perdreu l'accés remot en activar el tallafoc)
sudo ufw allow OpenSSH

# 2. FTP/FTPS: control i rang passiu, només des de la xarxa interna
sudo ufw allow from 192.168.1.0/24 to any port 21 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 40000:40100 proto tcp

# 3. TFTP només des de la xarxa interna
sudo ufw allow from 192.168.1.0/24 to any port 69 proto udp
```

**Atenció:** en activar `ufw` es bloquejaran **tots** els serveis que no tinguin una regla. El servidor 0375 també té els serveis dels altres temes: afegiu-ne les regles **abans** d'activar el tallafoc.

```bash
# 4. Serveis dels altres temes
sudo ufw allow 53                                       # DNS (Tema 1), TCP i UDP
sudo ufw allow 67/udp                                   # DHCP (Tema 2)
sudo ufw allow from 192.168.1.1 to any port 647 proto tcp  # Failover DHCP amb el 0374 (Tema 2)
sudo ufw allow 80,443,8080,8443/tcp                     # Web: Apache i Nginx (Tema 3)
sudo ufw allow 25,587,993,995/tcp                       # Correu (Tema 5); afegiu 110 i 143 si els feu servir

# 5. Activar el tallafoc i comprovar les regles
sudo ufw enable
sudo ufw status numbered
```

Després d'activar-lo, comproveu des del client que els serveis dels altres temes continuen funcionant (vegeu el "Pas 0" de l'[escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md)).

**Mode actiu:** el servidor obre la connexió de dades **cap enfora** des del port 20. Com que `ufw` permet per defecte tot el trànsit sortint, no cal cap regla al servidor. El problema del mode actiu està en el tallafoc del **client**.

**Comprovació des del client:**

```bash
# El port de control ha de respondre "succeeded"
nc -zv 192.168.1.2 21

# Una sessió FTPS completa amb llistat (fa servir el rang passiu) ha de funcionar
curl --ssl-reqd -k -u usuari1 ftp://192.168.1.2/fitxers/
```

`Nota:` no proveu el rang passiu amb `nc -zv 192.168.1.2 40000`: cap port del rang està en escolta fins que una sessió envia `PASV`, i obtindreu `Connection refused` encara que el tallafoc el deixi passar. Si el tallafoc el bloquegés, el símptoma seria un *timeout*, no un *refused*. Aquesta diferència (refusat = arriba al servidor però no hi ha ningú escoltant; *timeout* = el tallafoc descarta els paquets) és molt útil per diagnosticar problemes de xarxa.

### Protecció contra Força Bruta amb fail2ban

Els servidors FTP i SSH exposats reben contínuament intents automàtics d'endevinar contrasenyes. **fail2ban** llegeix els logs, detecta les IP amb massa intents fallits i les **bloqueja al tallafoc** durant un temps.

```
 Intents fallits             fail2ban                        Tallafoc
/var/log/vsftpd.log ──► "FAIL LOGIN" 5 vegades ──► BAN 192.168.1.50 durant 10 min
journal (sshd)      ──► "Failed password" ...  ──►
```

**Instal·lació:**

```bash
# python3-systemd permet a fail2ban llegir els logs de sshd del journal
# (imprescindible a Debian 12/13, que no tenen /var/log/auth.log per defecte)
sudo apt install fail2ban python3-systemd
```

**Configuració** (no modifiqueu mai `jail.conf`, que es sobreescriu en actualitzar el paquet):

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Temps de bloqueig, finestra d'observació i nombre d'intents permesos
bantime  = 10m
findtime = 10m
maxretry = 5
# IP que mai es bloquejaran: afegiu-hi l'equip des d'on administreu el servidor,
# però NO el client amb què fareu la prova de bloqueig de més avall
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
# Llegir els logs del journal de systemd (necessari a Debian; també funciona a Ubuntu)
backend = systemd

[vsftpd]
enabled = true
# Bloqueja el port 21 (sense connexió de control no hi pot haver connexió de dades)
logpath = /var/log/vsftpd.log
```

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban

# Comprovar les gàbies (jails) actives
sudo fail2ban-client status
```

**Prova des del client** (intents d'inici de sessió fallits per FTPS):

```bash
for i in $(seq 1 6); do
  curl -s --ssl-reqd -k -u usuari1:contrasenyaincorrecta ftp://192.168.1.2/ -o /dev/null
done

# El següent intent ja no podrà connectar (la IP està bloquejada)
curl --ssl-reqd -k -u usuari1 ftp://192.168.1.2/
```

**Al servidor:**

```bash
sudo fail2ban-client status vsftpd
# Status for the jail: vsftpd
# |- Filter
# |  |- Currently failed: 0
# |  `- Total failed:     6
# `- Actions
#    |- Currently banned: 1
#    `- Banned IP list:   192.168.1.50

# Desbloquejar una IP manualment
sudo fail2ban-client set vsftpd unbanip 192.168.1.50

# Veure l'activitat de fail2ban
sudo tail -f /var/log/fail2ban.log
```

Repetiu la prova amb SFTP (`sftp client1@192.168.1.2` amb contrasenya incorrecta) i comproveu la gàbia `sshd`.

### Configuració Final de vsftpd

Resum de totes les directives configurades durant el tema (`/etc/vsftpd.conf`):

```
# =====================================================
# /etc/vsftpd.conf - Servidor FTP/FTPS d'educem.cat
# =====================================================

# --- MODE D'ESCOLTA (punt 14) ---
listen=YES
listen_ipv6=NO
listen_address=192.168.1.2

# --- USUARIS (punt 5) ---
anonymous_enable=YES
local_enable=YES
write_enable=YES
# 002 si feu servir la carpeta compartida del grup (punt 6); 022 si no
local_umask=002

# --- MISSATGES (punt 5) ---
ftpd_banner=Benvinguts al servidor FTP d'educem.cat
dirmessage_enable=YES
use_localtime=YES

# --- LOGS (punt 5) ---
xferlog_enable=YES
xferlog_std_format=NO
vsftpd_log_file=/var/log/vsftpd.log
log_ftp_protocol=YES

# --- MODES ACTIU I PASSIU (punt 5) ---
connect_from_port_20=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

# --- GÀBIA CHROOT (punt 6) ---
chroot_local_user=YES
user_sub_token=$USER
local_root=/home/$USER/ftp

# --- CONTROL D'USUARIS (punt 6) ---
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO

# --- ACCÉS ANÒNIM (punt 7) ---
anon_root=/srv/ftp
no_anon_password=YES
hide_ids=YES
anon_world_readable_only=YES
anon_upload_enable=YES
anon_mkdir_write_enable=NO
anon_other_write_enable=NO
anon_umask=077
chown_uploads=YES
chown_username=usuari1

# --- LÍMITS (punt 8) ---
max_clients=50
max_per_ip=3
anon_max_rate=102400
local_max_rate=1048576
idle_session_timeout=300
data_connection_timeout=120
max_login_fails=3
delay_failed_login=3
user_config_dir=/etc/vsftpd/user_conf
deny_file={*.exe,*.bat,*.sh}
hide_file={.*}

# --- SSL/TLS - FTPS EXPLÍCIT (punt 10) ---
ssl_enable=YES
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key
allow_anon_ssl=NO
force_local_logins_ssl=YES
force_local_data_ssl=YES
ssl_ciphers=HIGH
require_ssl_reuse=NO

# --- SISTEMA ---
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
utf8_filesystem=YES
```

**Fitxers relacionats:**

```
/etc/vsftpd.userlist            → usuari1, usuari2, anonymous, ftp
/etc/vsftpd/user_conf/usuari1   → cmds_denied=DELE,RMD
/etc/vsftpd/user_conf/usuari2   → write_enable=NO, local_max_rate=51200
/etc/ssh/sshd_config.d/10-sftp.conf → Match Group sftpusers, Match User backup, Match User webdev
/etc/ssh/authorized_keys/backup → clau pública restringida
/etc/default/tftpd-hpa          → TFTP amb --secure --create
/etc/fail2ban/jail.local        → jails sshd i vsftpd
```

### Quin Protocol Triar?

| Situació                                                        | Protocol recomanat             |
| --------------------------------------------------------------- | ------------------------------ |
| Administrar servidors, copiar fitxers entre màquines            | **SFTP / SCP**                 |
| Còpies de seguretat automàtiques i scripts                      | **SFTP amb claus**             |
| Intercanvi de fitxers amb clients o altres empreses             | **SFTP** (o FTPS si el soci ho exigeix) |
| Pujar una web a un hosting                                      | **SFTP** (o desplegament amb Git/CI) |
| Aplicació o dispositiu antic que només suporta FTP              | **FTPS explícit** (o FTP en xarxa aïllada) |
| Descàrregues públiques per a qualsevol persona                  | **HTTPS** (o FTP anònim només lectura) |
| Compartir carpetes de treball entre usuaris de l'empresa        | **Samba/NFS** o núvol privat (Nextcloud) |
| Arrencada per xarxa (PXE), firmware i configuracions de dispositius | **TFTP** en xarxa de gestió aïllada |
| Fitxers grans al núvol                                          | **Emmagatzematge d'objectes (S3)** |

### Llista de Comprovació de Seguretat

- [ ] L'FTP sense xifrar no s'utilitza per a usuaris amb contrasenya (FTPS obligatori o només SFTP).
- [ ] Tots els usuaris estan engabiats (vsftpd `chroot_local_user`, OpenSSH `ChrootDirectory`).
- [ ] Els usuaris de transferència de fitxers no tenen shell (`/usr/sbin/nologin`).
- [ ] Hi ha una llista blanca d'usuaris (`userlist_deny=NO`) i `root` no pot entrar.
- [ ] L'accés anònim, si existeix, és només de lectura o amb bústia sense llistat.
- [ ] Hi ha límits de connexions, velocitat i quotes de disc.
- [ ] El tallafoc només obre els ports necessaris i el rang passiu és fix.
- [ ] fail2ban protegeix FTP i SSH.
- [ ] Els logs estan activats i es revisen.
- [ ] Els processos automàtics fan servir claus SSH restringides (`restrict`, `from=`).
- [ ] TFTP només és accessible des de la xarxa de gestió (o està desactivat).

### Documentació del Servei

Tot servei que es posa en producció s'ha de **documentar**, perquè qualsevol altre administrador el pugui mantenir. Com a entrega final del tema, completeu una fitxa com aquesta amb les dades del vostre servidor:

| Apartat                | Contingut                                                                      |
| ---------------------- | ------------------------------------------------------------------------------ |
| **Servidor**           | Nom, IP, sistema operatiu i versió                                             |
| **Serveis i versions** | vsftpd x.y.z, OpenSSH x.y, tftpd-hpa x.y                                       |
| **Noms DNS**           | `ftp.educem.cat`, `sftp.educem.cat`...                                          |
| **Ports**              | Taula de ports oberts i origen permès                                          |
| **Usuaris**            | Usuari, tipus (local, anònim, SFTP, automàtic), carpeta, permisos, quota       |
| **Carpetes**           | Ruta, propietari, permisos i finalitat                                         |
| **Seguretat**          | Certificat (emissor, caducitat), claus autoritzades, fail2ban, tallafoc        |
| **Fitxers de configuració** | Ruta de cada fitxer modificat i còpia de seguretat                       |
| **Logs**               | On es troben i què registren                                                   |
| **Proves realitzades** | Prova, client utilitzat, resultat esperat i resultat obtingut (amb captures)   |
| **Procediments**       | Com donar d'alta/baixa un usuari, renovar el certificat, restaurar una còpia   |

**Ordres útils per recollir la informació:**

```bash
# Versions dels serveis
apt policy vsftpd openssh-server tftpd-hpa | grep -E '^[a-z]|Installed'

# Ports en escolta
sudo ss -tulnp | grep -E 'vsftpd|sshd|in.tftpd'

# Configuració activa de vsftpd (sense comentaris)
grep -Ev '^\s*(#|$)' /etc/vsftpd.conf

# Configuració efectiva de sshd per a un usuari SFTP
sudo sshd -T -C user=client1,host=client,addr=192.168.1.50 | grep -Ei 'chroot|forcecommand'

# Quotes de disc
sudo repquota -s /

# Caducitat del certificat FTPS
openssl x509 -in /etc/ssl/certs/vsftpd.crt -noout -enddate

# Regles del tallafoc i estat de fail2ban
sudo ufw status numbered
sudo fail2ban-client status
```

### Punt de Control Final

- [ ] vsftpd escolta només a `192.168.1.2:21`.
- [ ] El tallafoc està actiu i només hi ha oberts els ports necessaris.
- [ ] fail2ban bloqueja una IP després de 5 intents fallits per FTPS i per SFTP.
- [ ] Teniu la configuració final de vsftpd revisada i comentada.
- [ ] Heu completat la fitxa de documentació del servei.
