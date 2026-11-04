## 06. Usuaris Locals, Grups i Gàbia Chroot

Al punt anterior hem vist que un usuari FTP pot navegar per **tot el sistema de fitxers** del servidor (`cd /etc`, `get /etc/passwd`...). En aquest punt aprendrem a:

- Crear usuaris i grups **exclusius per a FTP**, sense accés a la consola.
- **Engabiar** (chroot) cada usuari dins la seva carpeta.
- Controlar **quins usuaris** poden entrar al servei.
- Crear carpetes **compartides** entre usuaris d'un mateix grup.

### Què és una Gàbia Chroot?

**chroot** (*change root*) canvia el directori arrel (`/`) que veu un procés. Un usuari engabiat a `/home/usuari1/ftp` veu aquesta carpeta com si fos `/` i **no pot sortir-ne**.

```
SISTEMA DE FITXERS REAL                  EL QUE VEU usuari1 (engabiat)
/                                        
├── etc/                                 
├── var/                                 
└── home/                                
    └── usuari1/                         
        └── ftp/         ◄─── chroot ───  /
            ├── fitxers/                  ├── fitxers/
            └── LLEGEIX-ME.txt            └── LLEGEIX-ME.txt
```

### Usuaris Només per a FTP

Els usuaris del servei FTP **no haurien de poder iniciar sessió per SSH** ni a la consola. Els crearem amb la shell `/usr/sbin/nologin`.

**Pas 1: Crear el grup i els usuaris**

```bash
# Grup per als usuaris del servei FTP
sudo groupadd ftpgrup

# Crear usuari2 sense accés a la consola i dins del grup ftpgrup
sudo adduser --shell /usr/sbin/nologin --ingroup ftpgrup usuari2

# Modificar usuari1 (creat al punt 4): sense consola i dins del grup
sudo usermod -s /usr/sbin/nologin -aG ftpgrup usuari1

# Comprovar-ho
getent passwd usuari1 usuari2
getent group ftpgrup
```

**Pas 2: Permetre la shell nologin a PAM**

Recordeu que el mòdul `pam_shells.so` de `/etc/pam.d/vsftpd` només deixa entrar usuaris amb una shell que aparegui a `/etc/shells`. Si ara proveu d'entrar amb `usuari1`, obtindreu `530 Login incorrect.`

```bash
# Afegir nologin a la llista de shells vàlides (add-shell, del paquet debianutils, evita duplicats)
sudo add-shell /usr/sbin/nologin

# Comprovar-ho
tail -n 3 /etc/shells
```

`Nota:` afegir `nologin` a `/etc/shells` no permet iniciar sessió per SSH: en entrar, la shell `nologin` mostra un missatge i tanca la sessió. Alternativament, es pot comentar la línia `auth required pam_shells.so` de `/etc/pam.d/vsftpd`.

**Pas 3: Comprovar que no poden entrar per SSH**

```bash
ssh usuari1@192.168.1.2
# This account is currently not available.
# Connection to 192.168.1.2 closed.
```

### Activar la Gàbia Chroot

Afegiu les directives següents a `/etc/vsftpd.conf`:

```
# --- GÀBIA CHROOT ---
# Engabiar tots els usuaris locals a la seva carpeta
chroot_local_user=YES
# Arrel FTP de cada usuari: /home/<usuari>/ftp
user_sub_token=$USER
local_root=/home/$USER/ftp
```

- `chroot_local_user=YES`: tots els usuaris locals queden engabiats.
- `local_root`: carpeta on entra l'usuari i que serà la seva arrel. `$USER` es substitueix pel nom de l'usuari gràcies a `user_sub_token`.

**La regla de l'arrel no modificable**

Per seguretat, vsftpd **es nega a engabiar un usuari en una carpeta on pot escriure**. Si l'arrel és modificable, en iniciar sessió obtindreu:

```
500 OOPS: vsftpd: refusing to run with writable root inside chroot()
```

Hi ha dues solucions:

**Opció A (recomanada): arrel sense permisos d'escriptura i subcarpeta per treballar**

```
/home/usuari1/ftp          → propietari nobody:nogroup, permisos 555 (només lectura)
/home/usuari1/ftp/fitxers  → propietari usuari1:usuari1, permisos 755 (escriptura)
```

**Opció B: permetre una arrel modificable**

```
allow_writeable_chroot=YES
```

És més còmode però menys segur: en algunes situacions un usuari podria manipular fitxers del sistema dins la gàbia. **Farem servir l'opció A.**

### Preparar les Carpetes dels Usuaris

```bash
# --- usuari1 ---
# Arrel de la gàbia (no modificable per l'usuari)
sudo mkdir -p /home/usuari1/ftp
sudo chown nobody:nogroup /home/usuari1/ftp
sudo chmod a-w /home/usuari1/ftp

# Carpeta de treball (modificable per l'usuari)
sudo mkdir /home/usuari1/ftp/fitxers
sudo chown usuari1:usuari1 /home/usuari1/ftp/fitxers

# Fitxer de benvinguda a l'arrel
echo "Benvingut/da. Puja els teus fitxers a la carpeta 'fitxers'." | sudo tee /home/usuari1/ftp/LLEGEIX-ME.txt

# --- usuari2 --- (adduser amb --ingroup ha fet que el seu grup principal sigui ftpgrup)
sudo mkdir -p /home/usuari2/ftp/fitxers
sudo chown nobody:nogroup /home/usuari2/ftp
sudo chmod a-w /home/usuari2/ftp
sudo chown usuari2:ftpgrup /home/usuari2/ftp/fitxers

# Comprovar els permisos
sudo ls -ld /home/usuari1/ftp /home/usuari1/ftp/fitxers
# dr-xr-xr-x 3 nobody  nogroup 4096 ... /home/usuari1/ftp
# drwxr-xr-x 2 usuari1 usuari1 4096 ... /home/usuari1/ftp/fitxers
```

```bash
# Moure el fitxer de proves del punt anterior a la carpeta de treball
sudo mv /home/usuari1/prova.txt /home/usuari1/ftp/fitxers/ 2>/dev/null

# Reiniciar el servei
sudo systemctl restart vsftpd
```

**Prova des del client:**

```
ftp> pwd
257 "/" is the current directory
ftp> ls
-rw-r--r--    1 0        0              58 Oct 09 11:02 LLEGEIX-ME.txt
drwxr-xr-x    2 1001     1001         4096 Oct 09 11:02 fitxers
ftp> cd /etc
550 Failed to change directory.
ftp> put prova2.txt
553 Could not create file.
ftp> cd fitxers
250 Directory successfully changed.
ftp> put prova2.txt
226 Transfer complete.
```

L'usuari veu `/` com a arrel, **no pot sortir de la gàbia** i només pot escriure a la carpeta `fitxers`.

### Excepcions: Usuaris sense Gàbia

De vegades l'administrador necessita un usuari FTP que **no** estigui engabiat. Amb `chroot_list_enable` es crea una llista d'excepcions:

```
# Llista d'usuaris que NO s'engabien (amb chroot_local_user=YES)
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list
```

```bash
# Afegir l'usuari administrador a la llista d'excepcions
echo "admin" | sudo tee /etc/vsftpd.chroot_list
```

**Atenció, el significat de la llista s'inverteix segons `chroot_local_user`:**

| `chroot_local_user` | `chroot_list_enable` | Resultat                                           |
| ------------------- | -------------------- | -------------------------------------------------- |
| `NO`                | `NO`                 | Ningú engabiat                                     |
| `YES`               | `NO`                 | **Tothom engabiat**                                |
| `NO`                | `YES`                | Només s'engabien els usuaris de la llista         |
| `YES`               | `YES`                | Tothom engabiat **excepte** els usuaris de la llista |

`Nota:` per a aquest tema **no cal** crear excepcions. Si ho proveu, recordeu que l'usuari exceptuat també ha de tenir la carpeta `local_root` (`/home/admin/ftp`) o no podrà entrar.

### Llista d'Usuaris Permesos (userlist)

Per defecte, **qualsevol** usuari del sistema que no sigui a `/etc/ftpusers` pot entrar per FTP (també el vostre usuari administrador). En un servidor real és millor tenir una **llista blanca**: només poden entrar els usuaris que hi apareixen.

```
# --- CONTROL D'USUARIS ---
# Només poden entrar els usuaris de /etc/vsftpd.userlist
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```

```bash
# Crear la llista blanca
printf "usuari1\nusuari2\n" | sudo tee /etc/vsftpd.userlist

sudo systemctl restart vsftpd
```

**Funcionament segons `userlist_deny`:**

| `userlist_deny` | Significat de la llista | Un usuari que NO hi és... |
| --------------- | ----------------------- | ------------------------- |
| `YES` (defecte) | Llista **negra**        | Pot entrar                |
| `NO`            | Llista **blanca**       | **No pot entrar**         |

Si un usuari no permès prova d'entrar, el servidor el rebutja **abans de demanar-li la contrasenya** (així la contrasenya ni tan sols viatja per la xarxa):

```
Name (192.168.1.2:alumne): admin
530 Permission denied.
Login failed.
```

**Resum dels tres mecanismes de control d'accés:**

| Mecanisme               | Qui el gestiona | Funció                                              |
| ----------------------- | --------------- | --------------------------------------------------- |
| `/etc/ftpusers`         | PAM             | Usuaris del sistema que mai poden entrar            |
| `/etc/shells`           | PAM             | Només entren usuaris amb una shell vàlida           |
| `/etc/vsftpd.userlist`  | vsftpd          | Llista blanca o negra pròpia del servei             |

### Carpeta Compartida per a un Grup

Volem que `usuari1` i `usuari2` (grup `ftpgrup`) comparteixin una carpeta `projectes` on tots dos puguin llegir i escriure.

**Problema:** cada usuari està engabiat a la seva carpeta i **els enllaços simbòlics no funcionen** per sortir de la gàbia (apuntarien a una ruta que no existeix dins el chroot).

**Solució:** fer un **muntatge enllaçat** (*bind mount*): la mateixa carpeta real apareix dins de cada gàbia.

```bash
# 1. Carpeta real compartida, propietat del grup amb el bit setgid
sudo mkdir -p /srv/compartit/projectes
sudo chown root:ftpgrup /srv/compartit/projectes
sudo chmod 2775 /srv/compartit/projectes

# 2. Punts de muntatge dins de cada gàbia
sudo mkdir /home/usuari1/ftp/projectes /home/usuari2/ftp/projectes

# 3. Muntar la carpeta real dins de cada gàbia
sudo mount --bind /srv/compartit/projectes /home/usuari1/ftp/projectes
sudo mount --bind /srv/compartit/projectes /home/usuari2/ftp/projectes
```

- **`2775`**: el `2` és el bit **setgid**. Els fitxers creats a dins hereten el grup `ftpgrup` (en lloc del grup principal de l'usuari), i així els altres membres del grup hi poden accedir.
- Perquè els membres del grup puguin **modificar** els fitxers dels altres, els fitxers s'han de crear amb permisos `664`. Afegiu a `vsftpd.conf` la directiva `local_umask=002` i reinicieu el servei.

**Fer el muntatge permanent** (afegir a `/etc/fstab`):

```bash
sudo nano /etc/fstab
```

```
# Carpeta compartida FTP del grup ftpgrup
/srv/compartit/projectes  /home/usuari1/ftp/projectes  none  bind  0  0
/srv/compartit/projectes  /home/usuari2/ftp/projectes  none  bind  0  0
```

```bash
# Comprovar que fstab és correcte abans de reiniciar (molt important!)
sudo findmnt --verify

# Comprovar els muntatges actius
findmnt | grep projectes
```

**Prova:** puja un fitxer a `projectes` amb `usuari1` i comprova que `usuari2` el veu i el pot modificar.

### Usuaris Virtuals (Concepte)

En un proveïdor d'allotjament amb milers de clients no es crea un usuari del sistema per a cada client. S'utilitzen **usuaris virtuals**:

- Els usuaris i contrasenyes es guarden en un **fitxer o base de dades** (MySQL, LDAP).
- vsftpd els autentica amb un mòdul PAM específic (per exemple `pam_pwdfile` o `pam_mysql`).
- Totes les sessions s'executen amb **un únic usuari del sistema** (`guest_enable=YES`, `guest_username=ftpvirtual`).
- Cada usuari virtual té la seva carpeta i la seva configuració (`user_config_dir`, punt 8).

```
                      ┌──────────────────────┐
client1 ──┐           │  Base de dades       │
client2 ──┼── vsftpd ─│  client1:hash        │──► s'executen com a "ftpvirtual"
client3 ──┘    (PAM)  │  client2:hash ...    │    /srv/ftp/virtual/client1 ...
                      └──────────────────────┘
```

Avantatges: els clients no tenen cap compte al sistema, és fàcil donar d'alta i de baixa usuaris i s'integra amb panells d'administració. Avui dia, solucions com **SFTPGo** ofereixen usuaris virtuals amb panell web per a FTP, FTPS i SFTP alhora.

### Resolució de Problemes

**`500 OOPS: vsftpd: refusing to run with writable root inside chroot()`**

```bash
# L'arrel de la gàbia no ha de ser modificable per l'usuari
sudo chmod a-w /home/usuari1/ftp
sudo ls -ld /home/usuari1/ftp
```

**`500 OOPS: cannot change directory:/home/usuari1/ftp`**

- La carpeta indicada a `local_root` no existeix per a aquest usuari. Creeu-la.

**`530 Login incorrect.` amb usuaris `nologin`**

- Falta afegir `/usr/sbin/nologin` a `/etc/shells`.

**`530 Permission denied.`**

- L'usuari no és a `/etc/vsftpd.userlist` (amb `userlist_deny=NO`).

**`500 OOPS: child died` en llistar directoris**

- Alguns nuclis i versions de vsftpd tenen problemes amb el filtre de crides al sistema. Afegiu `seccomp_sandbox=NO` a `vsftpd.conf` i reinicieu.

### Punt de Control

- [ ] `usuari1` i `usuari2` tenen la shell `/usr/sbin/nologin` i no poden entrar per SSH.
- [ ] En connectar per FTP, `pwd` retorna `/` i no es pot accedir a `/etc`.
- [ ] Els usuaris només poden escriure a la carpeta `fitxers`.
- [ ] L'usuari administrador rep `530 Permission denied.` (no és a la llista blanca).
- [ ] La carpeta `projectes` és compartida entre `usuari1` i `usuari2`.
