## 08. Límits, Restriccions i Quotes de Disc

Un servidor de fitxers compartit ha de repartir els recursos de manera justa i evitar abusos: un sol usuari no pot ocupar tot el disc ni consumir tota l'amplada de banda. En aquest punt configurarem:

- **Límits de connexions** (quants clients i quantes connexions per IP).
- **Límits de velocitat** (amplada de banda per tipus d'usuari).
- **Temps d'espera** (desconnectar sessions inactives).
- **Configuració individual per usuari** (permisos i límits diferents per a cada compte).
- **Restriccions de fitxers i ordres**.
- **Quotes de disc** (espai màxim per usuari).

### Límits de Connexions

```
# --- LÍMITS DE CONNEXIONS ---
# Nombre màxim de clients connectats alhora
max_clients=50
# Nombre màxim de connexions des d'una mateixa IP
max_per_ip=3
```

- `max_clients`: quan s'arriba al màxim, els nous clients reben `421 There are too many connected users, please try later.`
- `max_per_ip`: evita que un sol client (o un programa de descàrregues amb moltes connexions paral·leles) monopolitzi el servei. Si se supera: `421 There are too many connections from your internet address.`

### Límits d'Amplada de Banda

Els valors s'indiquen en **bytes per segon** (0 = sense límit).

```
# --- LÍMITS DE VELOCITAT ---
# Usuaris anònims: 100 KB/s
anon_max_rate=102400
# Usuaris locals: 1 MB/s
local_max_rate=1048576
```

| Valor     | Velocitat aproximada |
| --------- | -------------------- |
| `51200`   | 50 KB/s              |
| `102400`  | 100 KB/s             |
| `1048576` | 1 MB/s               |
| `5242880` | 5 MB/s               |

**Prova:** descarregueu el fitxer de 10 MB del punt anterior com a anònim i observeu la velocitat que mostra el client (uns 100 KB/s, uns 100 segons):

```bash
curl -o /dev/null ftp://192.168.1.2/pub/fitxer_10MB.bin
```

### Temps d'Espera (Timeouts)

```
# --- TEMPS D'ESPERA ---
# Desconnectar si no s'envia cap ordre durant 5 minuts
idle_session_timeout=300
# Desconnectar si una transferència s'atura durant 2 minuts
data_connection_timeout=120
# Temps màxim per connectar la connexió de dades (segons)
accept_timeout=60
connect_timeout=60
```

Quan una sessió supera el temps d'inactivitat, el servidor envia `421 Timeout.` i tanca la connexió. Així s'alliberen recursos de clients que s'han oblidat la sessió oberta.

### Protecció contra Atacs de Força Bruta

```
# --- INICIS DE SESSIÓ FALLITS ---
# Desconnectar després de 3 contrasenyes incorrectes
max_login_fails=3
# Esperar 3 segons després de cada intent fallit
delay_failed_login=3
```

Això fa més lents els atacs de diccionari. Al punt 14 ho complementarem amb **fail2ban**, que bloqueja la IP de l'atacant.

### Configuració Individual per Usuari

Fins ara totes les directives afecten tots els usuaris. Amb `user_config_dir` podem crear un fitxer per a cada usuari amb directives que **sobreescriuen** la configuració general.

```
# --- CONFIGURACIÓ PER USUARI ---
user_config_dir=/etc/vsftpd/user_conf
```

```bash
sudo mkdir -p /etc/vsftpd/user_conf
```

El nom de cada fitxer ha de ser **exactament el nom de l'usuari**.

**Exemple 1: `usuari2` és un usuari de "només lectura" amb velocitat limitada**

```bash
sudo nano /etc/vsftpd/user_conf/usuari2
```

```
# usuari2: només pot descarregar, a 50 KB/s
write_enable=NO
local_max_rate=51200
```

**Exemple 2: `usuari1` pot pujar però no pot esborrar res**

```bash
sudo nano /etc/vsftpd/user_conf/usuari1
```

```
# usuari1: no pot esborrar fitxers ni carpetes
cmds_denied=DELE,RMD
```

```bash
sudo systemctl restart vsftpd
```

**Prova:**

```
# Amb usuari2
ftp> cd fitxers
ftp> put prova.txt
550 Permission denied.

# Amb usuari1
ftp> cd fitxers
ftp> delete prova.txt
550 Permission denied.
```

**Directives útils per a fitxers d'usuari:**

| Directiva          | Ús                                                                 |
| ------------------ | ------------------------------------------------------------------ |
| `write_enable`     | Permetre o no escriure                                             |
| `download_enable`  | Permetre o no descarregar (`NO` → només pot pujar)                 |
| `local_max_rate`   | Velocitat màxima de l'usuari                                       |
| `local_root`       | Carpeta arrel diferent per a l'usuari                              |
| `cmds_allowed`     | Llista blanca d'ordres FTP permeses                                |
| `cmds_denied`      | Llista negra d'ordres FTP (`DELE`, `RMD`, `RNFR`, `MKD`, `STOR`...) |
| `idle_session_timeout` | Temps d'inactivitat propi                                      |

### Restricció de Tipus de Fitxers

```
# --- RESTRICCIÓ DE FITXERS ---
# Prohibir pujar, descarregar o veure aquests fitxers
deny_file={*.exe,*.bat,*.sh,*.mp3,*.mp4}
# Amagar fitxers del llistat (però es poden descarregar si se'n sap el nom)
hide_file={.*}
```

- `deny_file`: qualsevol operació sobre aquests noms retorna `550 Permission denied.`
- `hide_file`: el fitxer no apareix al llistat (útil per amagar fitxers ocults com `.bashrc`).

`Nota:` `deny_file` compara **noms**, no el contingut. Un usuari pot canviar el nom de l'extensió. És una mesura bàsica, no un antivirus.

### Quotes de Disc

vsftpd **no té quotes pròpies**. L'espai en disc es limita amb el sistema de **quotes del sistema de fitxers** de Linux, que funciona per a **qualsevol** servei (FTP, SFTP, Samba, correu...), perquè es controla a nivell de nucli.

**Conceptes:**

- **Límit tou (*soft*)**: es pot superar temporalment durant un **període de gràcia** (per defecte 7 dies). Passat aquest temps es comporta com el límit dur.
- **Límit dur (*hard*)**: no es pot superar mai. L'escriptura falla.
- **Blocs**: espai ocupat (en blocs d'1 KB).
- **Inodes**: nombre de fitxers.

#### Pas 1: Instal·lar les eines i comprovar el mòdul del nucli

```bash
sudo apt install quota

# Comprovar que existeixen els mòduls de quotes del nucli
find /lib/modules/$(uname -r) -type f -name '*quota_v*.ko*'

# Si no apareix cap resultat (nuclis "virtual" o "cloud" d'Ubuntu):
sudo apt install linux-modules-extra-$(uname -r)
```

#### Pas 2: Activar les quotes al sistema de fitxers

Les carpetes dels usuaris són a `/home`, que normalment forma part del sistema de fitxers arrel (`/`). Cal afegir les opcions `usrquota` i `grpquota` a la línia de `/` del fitxer `/etc/fstab`.

```bash
# Fer una còpia de seguretat de fstab (un error pot impedir que el sistema arrenqui)
sudo cp /etc/fstab /etc/fstab.bak

sudo nano /etc/fstab
```

Busqueu la línia que munta `/` i afegiu les opcions al quart camp:

```
# ABANS
/dev/disk/by-id/dm-uuid-LVM-xxxxxxxx  /  ext4  defaults                    0 1
# DESPRÉS
/dev/disk/by-id/dm-uuid-LVM-xxxxxxxx  /  ext4  defaults,usrquota,grpquota  0 1
```

```bash
# Verificar que fstab és correcte (no ha de mostrar errors)
sudo findmnt --verify

# Tornar a muntar / amb les noves opcions
sudo mount -o remount /

# Comprovar que les opcions estan actives
findmnt -no OPTIONS /
# rw,relatime,quota,usrquota,grpquota
```

#### Pas 3: Crear els fitxers de quotes i activar-les

```bash
# Analitzar el sistema de fitxers i crear aquota.user i aquota.group
sudo quotacheck -cugm /

# Activar les quotes
sudo quotaon -v /

# Comprovar l'estat
sudo quotaon -pa
```

#### Pas 4: Assignar quotes als usuaris

```bash
# usuari1: 100 MB (tou) i 120 MB (dur), sense límit de fitxers
#                   blocs-tou blocs-dur inodes-tou inodes-dur
sudo setquota -u usuari1 102400 122880 0 0 /

# usuari2: 50 MB (tou) i 60 MB (dur), màxim 1000 fitxers
sudo setquota -u usuari2 51200 61440 1000 1100 /

# Quota de grup: tot el grup ftpgrup junt no pot superar 500 MB
sudo setquota -g ftpgrup 512000 524288 0 0 /

# Alternativa interactiva (obre un editor)
sudo edquota -u usuari1
```

#### Pas 5: Consultar l'ús i provar el límit

```bash
# Quota d'un usuari (-s mostra les mides en format llegible)
sudo quota -vs usuari1

# Informe de tots els usuaris
sudo repquota -s /
```

```
*** Report for user quotas on device /dev/mapper/ubuntu--vg-ubuntu--lv
                        Space limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
usuari1   --     24K    100M    120M              8     0     0
usuari2   --     20K     50M     60M              6  1000  1100
```

**Prova de superar la quota:** des del client, creeu un fitxer de 130 MB i proveu de pujar-lo amb `usuari1`:

```bash
# Al client
dd if=/dev/zero of=gran.bin bs=1M count=130

ftp 192.168.1.2
ftp> cd fitxers
ftp> put gran.bin
451 Failure writing to local file.
```

La transferència s'atura quan s'arriba al límit dur de 120 MB. Al servidor, `sudo repquota -s /` mostrarà l'usuari amb un `+` indicant que ha superat el límit.

```bash
# Eliminar el fitxer parcial que ha quedat al servidor
sudo rm /home/usuari1/ftp/fitxers/gran.bin
```

**Període de gràcia:**

```bash
# Veure i modificar el període de gràcia (per defecte 7 dies)
sudo edquota -t
```

### Configuració Completa dels Límits

Resum de totes les directives d'aquest punt per afegir a `/etc/vsftpd.conf`:

```
# --- LÍMITS DE CONNEXIONS ---
max_clients=50
max_per_ip=3

# --- LÍMITS DE VELOCITAT ---
anon_max_rate=102400
local_max_rate=1048576

# --- TEMPS D'ESPERA ---
idle_session_timeout=300
data_connection_timeout=120

# --- INICIS DE SESSIÓ FALLITS ---
max_login_fails=3
delay_failed_login=3

# --- CONFIGURACIÓ PER USUARI ---
user_config_dir=/etc/vsftpd/user_conf

# --- RESTRICCIÓ DE FITXERS ---
deny_file={*.exe,*.bat,*.sh}
hide_file={.*}
```

### Resolució de Problemes

**`quotaon: cannot find //aquota.user on /dev/... [/]`**

- No s'ha executat `quotacheck -cugm /` o el sistema de fitxers no té les opcions de quota. Comproveu `findmnt -no OPTIONS /`.

**`quotaon: Quota format not supported in kernel.`**

- Falta el mòdul `quota_v2` del nucli. Instal·leu `linux-modules-extra-$(uname -r)` i reinicieu.

**La configuració per usuari no s'aplica:**

- El nom del fitxer de `/etc/vsftpd/user_conf/` ha de coincidir exactament amb el nom de l'usuari.
- Els fitxers d'usuari segueixen la mateixa sintaxi estricta: sense espais al voltant de `=` i comentaris en línies pròpies.

### Punt de Control

- [ ] Una 4a connexió des de la mateixa IP és rebutjada amb `421`.
- [ ] La descàrrega anònima va a uns 100 KB/s.
- [ ] `usuari2` no pot pujar fitxers i `usuari1` no pot esborrar-ne.
- [ ] `repquota -s /` mostra les quotes de `usuari1` i `usuari2`.
- [ ] La pujada d'un fitxer de 130 MB amb `usuari1` falla per quota.
