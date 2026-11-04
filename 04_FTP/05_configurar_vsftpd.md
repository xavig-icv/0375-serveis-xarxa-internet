## 05. Configuració Bàsica de vsftpd

Un cop instal·lat el servei, configurarem el fitxer principal `/etc/vsftpd.conf` per permetre que els usuaris locals puguin pujar fitxers, fixar el rang de ports del mode passiu i activar els logs. Al final del punt veurem el protocol FTP "en directe" fent una sessió manual i capturant el trànsit.

### Sintaxi del Fitxer vsftpd.conf

El fitxer de configuració de vsftpd és molt senzill però també molt estricte:

```
opcio=valor
```

- Una opció per línia, amb el format `opcio=valor`.
- **Sense espais** abans ni després del signe `=`.
- Els valors booleans són `YES` o `NO`.
- Els comentaris comencen amb `#` i **han d'anar en una línia pròpia**.

```
# CORRECTE
# Permet escriure als usuaris
write_enable=YES

# INCORRECTE (el servei no arrencarà)
write_enable = YES
write_enable=YES   # Permet escriure
```

Si la sintaxi és incorrecta, en reiniciar el servei apareixerà un error com `500 OOPS: bad bool value in config file for: write_enable`.

### Directives Principals

**Mode d'escolta**

| Directiva     | Descripció                                                                     |
| ------------- | ------------------------------------------------------------------------------ |
| `listen`      | `YES`: escolta només en IPv4. Incompatible amb `listen_ipv6=YES`.             |
| `listen_ipv6` | `YES`: escolta en IPv6 **i també en IPv4** (configuració per defecte).        |
| `listen_port` | Port de control (per defecte 21).                                              |

**Usuaris**

| Directiva          | Descripció                                                            |
| ------------------ | --------------------------------------------------------------------- |
| `anonymous_enable` | Permet l'accés anònim (`anonymous` / `ftp`).                          |
| `local_enable`     | Permet l'accés als usuaris locals del sistema.                        |
| `write_enable`     | Permet ordres d'escriptura: `STOR`, `DELE`, `MKD`, `RMD`, `RNFR`...   |
| `local_umask`      | Màscara dels fitxers creats. `022` → fitxers `644` i carpetes `755`.  |

**Missatges i informació**

| Directiva           | Descripció                                                          |
| ------------------- | ------------------------------------------------------------------- |
| `ftpd_banner`       | Text que es mostra en connectar (en lloc de `vsFTPd 3.0.5`).        |
| `dirmessage_enable` | Mostra el contingut del fitxer `.message` en entrar a una carpeta.  |
| `use_localtime`     | Mostra les dates en l'hora local en lloc d'UTC.                     |

**Logs**

| Directiva            | Descripció                                                           |
| -------------------- | -------------------------------------------------------------------- |
| `xferlog_enable`     | Registra les pujades i descàrregues.                                 |
| `xferlog_std_format` | `YES`: format estàndard `xferlog`. `NO`: format propi de vsftpd (més detallat). |
| `vsftpd_log_file`    | Fitxer de log en format vsftpd (per defecte `/var/log/vsftpd.log`).  |
| `log_ftp_protocol`   | Registra totes les ordres i respostes FTP (molt útil per aprendre i depurar). |

**Modes actiu i passiu**

| Directiva              | Descripció                                                      |
| ---------------------- | --------------------------------------------------------------- |
| `connect_from_port_20` | En mode actiu, el servidor connecta des del port 20.           |
| `pasv_enable`          | Permet el mode passiu (per defecte `YES`).                      |
| `pasv_min_port`        | Primer port del rang passiu.                                    |
| `pasv_max_port`        | Últim port del rang passiu.                                     |
| `pasv_address`         | IP que s'anuncia a la resposta `227` (només si hi ha NAT).      |

### Configuració Base del Servidor

Partim de la còpia que vam desar al punt anterior i escrivim una configuració neta i comentada:

```bash
sudo nano /etc/vsftpd.conf
```

**Contingut:**

```
# =====================================================
# /etc/vsftpd.conf - Servidor FTP de domini.cat
# =====================================================

# --- MODE D'ESCOLTA ---
# Escolta en IPv6 i IPv4 (mode standalone)
listen=NO
listen_ipv6=YES

# --- USUARIS ---
# Sense accés anònim (l'activarem al punt 7)
anonymous_enable=NO
# Accés per als usuaris locals del sistema
local_enable=YES
# Permetre pujar, esborrar i crear carpetes
write_enable=YES
# Fitxers nous amb permisos 644 i carpetes 755
local_umask=022

# --- MISSATGES ---
ftpd_banner=Benvinguts al servidor FTP de domini.cat
dirmessage_enable=YES
use_localtime=YES

# --- LOGS ---
xferlog_enable=YES
xferlog_std_format=NO
vsftpd_log_file=/var/log/vsftpd.log
log_ftp_protocol=YES

# --- MODE ACTIU ---
connect_from_port_20=YES

# --- MODE PASSIU ---
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

# --- SEGURETAT I SISTEMA ---
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
utf8_filesystem=YES

# --- SSL/TLS (el configurarem al punt 10) ---
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
```

**Aplicar els canvis:**

```bash
# Reiniciar el servei (els canvis d'escolta només s'apliquen amb restart)
sudo systemctl restart vsftpd

# Comprovar que ha arrencat correctament
sudo systemctl status vsftpd --no-pager
```

`Nota:` vsftpd no té una ordre de comprovació de sintaxi (com `apache2ctl configtest` o `named-checkconf`). Si la configuració és incorrecta, el servei no arrencarà i l'error apareixerà a `systemctl status` o a `journalctl -u vsftpd`.

### Prova des del Client

Des del **client** de la xarxa interna, connecteu-vos amb l'usuari creat al punt anterior:

```bash
ftp 192.168.1.2
# o bé pel nom DNS
ftp ftp.domini.cat
```

```
Connected to 192.168.1.2.
220 Benvinguts al servidor FTP de domini.cat
Name (192.168.1.2:alumne): usuari1
331 Please specify the password.
Password:
230 Login successful.
ftp> put /etc/hostname prova.txt
local: /etc/hostname remote: prova.txt
229 Entering Extended Passive Mode (|||40037|)
150 Ok to send data.
226 Transfer complete.
ftp> ls
229 Entering Extended Passive Mode (|||40012|)
150 Here comes the directory listing.
-rw-r--r--    1 1001     1001           10 Oct 09 10:15 prova.txt
226 Directory send OK.
ftp> bye
221 Goodbye.
```

Ara podem pujar fitxers (`write_enable=YES`) i les connexions de dades fan servir ports del rang **40000-40100**.

### Consultar els Logs

```bash
# Veure el log en temps real
sudo tail -f /var/log/vsftpd.log
```

**Exemple de sortida** (amb `log_ftp_protocol=YES`):

```
Fri Oct  9 10:15:02 2026 [pid 2210] CONNECT: Client "::ffff:192.168.1.100"
Fri Oct  9 10:15:02 2026 [pid 2210] FTP response: Client "::ffff:192.168.1.100", "220 Benvinguts al servidor FTP de domini.cat"
Fri Oct  9 10:15:05 2026 [pid 2210] FTP command: Client "::ffff:192.168.1.100", "USER usuari1"
Fri Oct  9 10:15:07 2026 [pid 2210] FTP command: Client "::ffff:192.168.1.100", "PASS <password>"
Fri Oct  9 10:15:07 2026 [pid 2209] [usuari1] OK LOGIN: Client "::ffff:192.168.1.100"
Fri Oct  9 10:15:12 2026 [pid 2211] [usuari1] OK UPLOAD: Client "::ffff:192.168.1.100", "/home/usuari1/prova.txt", 10 bytes, 4.12Kbyte/sec
```

**Missatges importants:**

- `OK LOGIN` / `FAIL LOGIN`: inici de sessió correcte o fallit.
- `OK UPLOAD` / `FAIL UPLOAD`: pujada de fitxers.
- `OK DOWNLOAD` / `FAIL DOWNLOAD`: descàrrega de fitxers.
- `OK DELETE`, `OK MKDIR`, `OK RENAME`: altres operacions.

`Nota:` les IPs apareixen amb el prefix `::ffff:` perquè el servidor escolta en IPv6 i les connexions IPv4 es representen com a adreces IPv4 mapejades en IPv6.

```bash
# Comptar inicis de sessió fallits
sudo grep -c "FAIL LOGIN" /var/log/vsftpd.log

# Veure totes les pujades
sudo grep "OK UPLOAD" /var/log/vsftpd.log
```

### Pràctica Guiada: el Protocol FTP en Directe

Ara que el servidor funciona, comprovarem a la pràctica tot el que vam veure als punts 2 i 3.

#### Pas 1: Sessió FTP manual amb telnet i nc

Farem el paper del client FTP escrivint les ordres del protocol a mà. Necessitem **dos terminals** al client.

**Terminal 1 (connexió de control):**

```bash
telnet 192.168.1.2 21
```

```
220 Benvinguts al servidor FTP de domini.cat
USER usuari1
331 Please specify the password.
PASS LaVostraContrasenya
230 Login successful.
PWD
257 "/home/usuari1" is the current directory
TYPE I
200 Switching to Binary mode.
PASV
227 Entering Passive Mode (192,168,1,2,156,73).
```

Calculeu el port: `156 × 256 + 73 = 40009`.

**Terminal 2 (connexió de dades):** obriu la connexió de dades al port calculat.

```bash
nc 192.168.1.2 40009
```

**Terminal 1:** demaneu el llistat.

```
LIST
150 Here comes the directory listing.
226 Directory send OK.
```

El llistat **apareix al terminal 2**, no al terminal 1: les dades viatgen per una connexió diferent de les ordres. Per descarregar un fitxer, repetiu el procés amb `PASV` i `RETR prova.txt` (el contingut sortirà al terminal 2). Per acabar:

```
QUIT
221 Goodbye.
```

`Nota:` si tardeu massa entre `PASV` i la connexió de dades, el servidor tancarà el port (`425 Failed to establish connection`). Torneu a enviar `PASV`.

#### Pas 2: Capturar la contrasenya amb tcpdump

Comprovarem que l'FTP envia les credencials **en text pla**. Al **servidor**, inicieu una captura de la connexió de control:

```bash
# -l: sortida línia a línia (sense -l, en passar per una canonada, tcpdump acumula la sortida i grep no mostra res fins al final)
sudo tcpdump -l -i enp0s8 -nn -A 'tcp port 21' | grep -E 'USER|PASS|227|230'
```

Des del **client**, inicieu sessió amb `ftp 192.168.1.2`. Al servidor veureu:

```
USER usuari1
PASS LaVostraContrasenya
230 Login successful.
227 Entering Passive Mode (192,168,1,2,156,90).
```

**Conclusió:** qualsevol persona que pugui capturar el trànsit de la xarxa veu l'usuari i la contrasenya. Ho solucionarem amb **FTPS** (punt 10) i **SFTP** (punt 11).

#### Pas 3: Observar els modes actiu i passiu

Al **servidor**, captureu ara totes les connexions TCP amb el client (només les capçaleres):

```bash
sudo tcpdump -i enp0s8 -nn 'host 192.168.1.100 and tcp[tcpflags] & tcp-syn != 0'
```

Al **client**, connecteu-vos i alterneu els modes amb l'ordre `passive` del client `ftp`:

```
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls
200 EPRT command successful. Consider using EPSV.
...
ftp> passive
Passive mode: on; fallback to active mode: on.
ftp> ls
229 Entering Extended Passive Mode (|||40051|)
```

**Al tcpdump observareu:**

```
# Mode actiu: el SERVIDOR (port 20) obre la connexió cap al client
192.168.1.2.20 > 192.168.1.100.41235: Flags [S] ...

# Mode passiu: el CLIENT obre la connexió cap al port 40051 del servidor
192.168.1.100.53120 > 192.168.1.2.40051: Flags [S] ...
```

`Nota:` si el client té el tallafoc activat, el mode actiu fallarà (`425 Failed to establish connection`) perquè el servidor no pot obrir la connexió cap al client. És justament el problema que vam explicar al punt 3.

### Resolució de Problemes

**El servei no arrenca després de modificar la configuració:**

```bash
sudo journalctl -u vsftpd -n 20 --no-pager

# Errors típics:
# 500 OOPS: bad bool value in config file for: XXX   → valor diferent de YES/NO o espais
# 500 OOPS: unrecognised variable in config file: XXX → directiva mal escrita
# 500 OOPS: run two copies of vsftpd for IPv4 and IPv6 → listen=YES i listen_ipv6=YES alhora
```

**`550 Permission denied.` en pujar un fitxer:**

- Comproveu que `write_enable=YES` està activat i que heu reiniciat el servei.
- Comproveu que l'usuari té permís d'escriptura a la carpeta (`ls -ld /home/usuari1`).

**El llistat es queda penjat o dona `425 Failed to establish connection`:**

- Hi ha un tallafoc que bloqueja la connexió de dades. Comproveu que el rang `40000-40100/tcp` està obert al servidor (punt 14) i que el client fa servir el mode passiu.

### Punt de Control

- [ ] El banner personalitzat apareix en connectar des del client.
- [ ] L'usuari pot pujar fitxers (`put`) i el log mostra `OK UPLOAD`.
- [ ] Les connexions passives fan servir ports entre 40000 i 40100.
- [ ] Heu fet la sessió manual amb telnet i nc i heu vist el llistat pel canal de dades.
- [ ] Heu capturat la contrasenya en text pla amb tcpdump.
