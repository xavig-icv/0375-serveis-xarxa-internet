## 12. SFTP en Entorns Reals: Claus, Automatització i Integració

Al punt anterior hem configurat SFTP per a usuaris que es connecten manualment amb contrasenya. En entorns professionals, però, la majoria de transferències SFTP **les fan programes, no persones**: còpies de seguretat nocturnes, desplegaments de webs, intercanvi de fitxers entre empreses... En aquest punt veurem:

- Escenaris reals on s'utilitza SFTP i com es combina amb altres protocols i serveis.
- Autenticació amb **claus SSH** restringides (sense contrasenyes).
- **Automatització** de transferències amb scripts i `cron`.
- **Desplegament** d'un lloc web per SFTP (integració amb el Tema 3).

### Escenaris Típics d'Ús de SFTP

**1. Administració de servidors i desplegament web**

- Els proveïdors d'allotjament (hosting) ofereixen SFTP per pujar els fitxers del web, en lloc de l'antic FTP.
- Els sistemes d'integració i desplegament continu (CI/CD), com GitHub Actions o GitLab CI, publiquen la web al servidor per SFTP amb una clau desada com a secret.
- Els administradors copien configuracions, logs i scripts entre servidors amb `scp`/`sftp`.

**2. Còpies de seguretat fora del servidor (*offsite*)**

- Cada nit, un script empaqueta les dades (web, bases de dades, configuracions) i les envia a un servidor de còpies remot.
- S'utilitza un usuari dedicat, engabiat, amb clau i sense possibilitat d'obrir una shell.

**3. Intercanvi de fitxers entre empreses (B2B)**

- **Comerç i logística**: comandes, albarans, inventaris i factures electròniques en formats EDI, XML o CSV entre proveïdors, distribuïdors i transportistes.
- **Banca**: moltes entitats ofereixen canals SFTP perquè les empreses enviïn **remeses de pagaments i cobraments SEPA** (fitxers XML ISO 20022) i rebin els **extractes bancaris** (Norma 43) de manera automàtica.
- **Recursos humans i assegurances**: nòmines, altes i baixes, pòlisses.
- **Administracions públiques**: tramesa massiva de fitxers entre organismes.

**4. Recollida de dades**

- Botigues o sucursals que envien les vendes del dia a la central.
- Equips industrials, estacions meteorològiques o càmeres que pugen dades i imatges periòdicament.
- Escàners i impressores multifunció amb la funció "escanejar a SFTP".

**5. Serveis gestionats al núvol**

- **AWS Transfer Family** i **Azure Blob Storage (SFTP)** ofereixen un servidor SFTP que desa els fitxers directament a l'emmagatzematge del núvol (S3 o Blob), sense administrar cap servidor.
- Així, els socis comercials continuen fent servir SFTP mentre l'empresa processa els fitxers amb serveis al núvol.

### SFTP Combinat amb Altres Protocols i Serveis

SFTP rarament treballa sol. En una infraestructura real es combina amb la resta de serveis del mòdul:

```
                         ┌──────────────────────────┐
     DNS (Tema 1)  ─────►│ sftp.educem.cat → IP     │  Registres A/CNAME i SSHFP
                         └──────────────────────────┘
                                      │
  ┌──────────┐   SFTP (22) + claus    ▼                ┌────────────────────┐
  │ Client / │ ════════════════► SERVIDOR SFTP ───────►│ /var/www/...       │──► HTTP/HTTPS (Tema 3)
  │ CI/CD /  │                   (OpenSSH)             │ (web publicada)    │    Apache / Nginx
  │ soci B2B │                        │                └────────────────────┘
  └──────────┘                        │
                                      ├──► cron / systemd: processar fitxers rebuts
                                      ├──► SMTP (Tema 5): notificar "fitxer rebut" o "error"
                                      ├──► GPG: fitxers xifrats també en repòs
                                      └──► LDAP / Active Directory: usuaris centralitzats
```

| Combinació             | Com s'utilitza                                                                          |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **SFTP + DNS**         | Nom estable (`sftp.educem.cat`) per als clients. Els registres **SSHFP** publiquen l'empremta de la clau del servidor (amb DNSSEC). |
| **SFTP + HTTP(S)**     | Els desenvolupadors pugen la web per SFTP i Apache/Nginx la serveixen. Els fitxers pujats es poden publicar per HTTPS. |
| **SFTP + SMTP**        | Els scripts envien un correu quan arriba un fitxer o quan una còpia falla.              |
| **SFTP + GPG/PGP**     | SFTP xifra el **transport**; GPG xifra el **fitxer**, que continua protegit quan ja és al servidor. Molt habitual en banca. |
| **SFTP + cron/systemd**| Programació de les transferències i del processament posterior dels fitxers.           |
| **SFTP + rsync/rclone**| Sincronització incremental. `rsync` necessita una shell al servidor; `rclone` i `lftp` funcionen amb usuaris només SFTP. |
| **SFTP + LDAP/AD**     | Els usuaris SFTP s'autentiquen amb el directori corporatiu (SSSD).                      |
| **SFTP vs AS2/HTTPS**  | En entorns B2B grans es fan servir plataformes **MFT** (*Managed File Transfer*) que combinen SFTP, FTPS, AS2 i HTTPS amb traçabilitat. |

### Pràctica 1: Autenticació amb Claus SSH Restringides

Els scripts no poden escriure contrasenyes. La solució és l'**autenticació per clau pública**:

- El **client** té una **clau privada** (secreta, no surt mai de la màquina).
- El **servidor** té la **clau pública** corresponent a la llista de claus autoritzades.
- En connectar, el client demostra que té la clau privada sense enviar-la.

```
CLIENT                                                SERVIDOR
~/.ssh/id_backup      (privada)                       /etc/ssh/authorized_keys/backup
~/.ssh/id_backup.pub  (pública) ── es copia una vegada ──►  (clau pública + restriccions)
```

#### Pas 1: Crear l'usuari de còpies al servidor

```bash
# Usuari només SFTP, engabiat i sense contrasenya (només podrà entrar amb clau)
sudo useradd -M -d /srv/sftp/backup -g sftpusers -s /usr/sbin/nologin backup

sudo mkdir -p /srv/sftp/backup/pujades
sudo chown root:root /srv/sftp/backup
sudo chmod 755 /srv/sftp/backup
sudo chown backup:sftpusers /srv/sftp/backup/pujades
sudo chmod 750 /srv/sftp/backup/pujades
```

`Nota:` com que forma part del grup `sftpusers`, li aplica automàticament la gàbia i el `ForceCommand internal-sftp` del punt 11.

#### Pas 2: Generar el parell de claus al client

```bash
# Al CLIENT, amb el vostre usuari
# -t ed25519: algorisme recomanat actualment
# -N "": sense frase de pas (necessari per a scripts desatesos)
ssh-keygen -t ed25519 -f ~/.ssh/id_backup -C "backup@$(hostname)" -N ""

ls -l ~/.ssh/id_backup*
# -rw------- ... id_backup        ← clau privada (només el propietari)
# -rw-r--r-- ... id_backup.pub    ← clau pública
```

Una clau sense frase de pas és com una contrasenya escrita en un fitxer: si algú la roba, pot entrar. Per això **la limitarem al màxim** al servidor.

#### Pas 3: Autoritzar la clau al servidor (gestionada per l'administrador)

Com que `backup` no té shell ni contrasenya, `ssh-copy-id` no funcionarà. A més, en entorns professionals és millor que les claus autoritzades les gestioni **root** en una ubicació central, i no l'usuari.

```bash
# Al CLIENT: copiar la clau pública al servidor amb l'usuari administrador
scp ~/.ssh/id_backup.pub admin@192.168.1.2:/tmp/
```

```bash
# Al SERVIDOR: directori central de claus autoritzades
sudo mkdir -p /etc/ssh/authorized_keys
sudo chmod 755 /etc/ssh/authorized_keys

# Clau pública amb restriccions:
#   restrict              → sense túnels, sense terminal, sense reenviaments
#   from="192.168.1.0/24" → només es pot fer servir des de la xarxa interna
echo "restrict,from=\"192.168.1.0/24\" $(cat /tmp/id_backup.pub)" | sudo tee /etc/ssh/authorized_keys/backup
sudo chmod 644 /etc/ssh/authorized_keys/backup
rm /tmp/id_backup.pub
```

Afegiu al **final** de `/etc/ssh/sshd_config.d/10-sftp.conf`:

```
# =====================================================
# Usuari de còpies de seguretat: només clau, des de la LAN
# =====================================================
Match User backup
    AuthorizedKeysFile /etc/ssh/authorized_keys/%u
    PasswordAuthentication no
```

```bash
sudo sshd -t && sudo systemctl restart ssh
```

#### Pas 4: Provar la connexió amb clau

```bash
# Al CLIENT
sftp -i ~/.ssh/id_backup backup@192.168.1.2
# Connected to 192.168.1.2.   ← sense demanar contrasenya
sftp> ls
pujades
sftp> bye
```

Per no haver d'escriure les opcions cada vegada, creeu un **àlies** a `~/.ssh/config` del client:

```bash
nano ~/.ssh/config
```

```
Host backup-srv
    HostName 192.168.1.2
    User backup
    IdentityFile ~/.ssh/id_backup
    IdentitiesOnly yes
```

```bash
chmod 600 ~/.ssh/config

# Ara n'hi ha prou amb
sftp backup-srv
```

### Pràctica 2: Còpia de Seguretat Automàtica per SFTP

Crearem un script que cada nit empaqueta una carpeta del client, en calcula la suma de verificació i la puja al servidor SFTP.

`Nota:` si no teniu una segona màquina client, podeu fer tota la pràctica des del mateix servidor (canvieu `HostName` per `127.0.0.1` i la xarxa de `from=` per `127.0.0.1`).

#### Pas 1: Dades de prova

```bash
mkdir -p ~/dades ~/bin
echo "Factura 001" > ~/dades/factura_001.txt
echo "Inventari"   > ~/dades/inventari.csv
```

#### Pas 2: L'script

```bash
nano ~/bin/backup_sftp.sh
```

```bash
#!/usr/bin/env bash
# =====================================================
# backup_sftp.sh - Còpia de seguretat diària per SFTP
# =====================================================
# Atura l'script si qualsevol ordre falla
set -euo pipefail

ORIGEN="$HOME/dades"
DESTI="backup-srv"                 # Àlies definit a ~/.ssh/config
CARPETA_REMOTA="pujades"
DATA="$(date +%Y%m%d_%H%M%S)"
NOM="backup_$(hostname)_${DATA}.tar.gz"

# Directori temporal que s'esborra en acabar (tant si va bé com si falla)
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

echo "[$(date '+%F %T')] Inici de la còpia de $ORIGEN"

# 1. Empaquetar i comprimir
tar -czf "$TMP/$NOM" -C "$(dirname "$ORIGEN")" "$(basename "$ORIGEN")"

# 2. Calcular la suma de verificació per comprovar la integritat
( cd "$TMP" && sha256sum "$NOM" > "$NOM.sha256" )

# 3. Pujar per SFTP en mode batch (-b -): llegeix les ordres de l'entrada estàndard
#    Es puja amb extensió .part i després es canvia el nom: qui processi els
#    fitxers al servidor mai veurà un fitxer a mig pujar.
sftp -b - "$DESTI" <<EOF
cd $CARPETA_REMOTA
put $TMP/$NOM $NOM.part
rename $NOM.part $NOM
put $TMP/$NOM.sha256
EOF

echo "[$(date '+%F %T')] Còpia pujada correctament: $NOM ($(du -h "$TMP/$NOM" | cut -f1))"
```

```bash
chmod +x ~/bin/backup_sftp.sh

# Executar-lo manualment
~/bin/backup_sftp.sh
```

**Per què `sftp -b`?**

- El mode *batch* no demana res de manera interactiva: només funciona amb **claus**.
- Si **qualsevol ordre falla**, `sftp` s'atura i retorna un codi d'error, i l'script també s'atura (`set -e`).

#### Pas 3: Verificar la còpia al servidor

```bash
# Al SERVIDOR
sudo ls -l /srv/sftp/backup/pujades/

# Comprovar la integritat del fitxer rebut
# (la carpeta només és accessible per backup i root, per això tot s'executa amb sudo)
sudo sh -c 'cd /srv/sftp/backup/pujades && sha256sum -c backup_*.sha256'
# backup_client_20261009_131500.tar.gz: OK
```

#### Pas 4: Programar l'script amb cron

```bash
# Al CLIENT
crontab -e
```

```
# Còpia de seguretat cada dia a les 02:30
30 2 * * * $HOME/bin/backup_sftp.sh >> $HOME/backup_sftp.log 2>&1
```

Per provar-ho sense esperar a la nit, poseu temporalment una execució cada minut (`* * * * *`), comproveu el fitxer `~/backup_sftp.log` i torneu a deixar l'horari correcte.

`Nota:` als sistemes actuals també es poden fer servir els **temporitzadors de systemd** (`.timer`), que registren les execucions al journal i permeten recuperar execucions perdudes si l'equip estava apagat.

#### Pas 5: Retenció de còpies al servidor

Les còpies s'acumulen cada dia. Al **servidor**, root esborrarà automàticament les de més de 7 dies:

```bash
sudo nano /etc/cron.d/retencio-backups
```

```
# Esborrar les còpies de més de 7 dies cada dia a les 04:00
0 4 * * * root find /srv/sftp/backup/pujades -type f -name 'backup_*' -mtime +7 -delete
```

#### Pas 6: Restaurar una còpia

Una còpia de seguretat que no s'ha restaurat mai **no és una còpia fiable**. Comproveu que es pot recuperar:

```bash
# Al CLIENT: carpeta de proves per a la restauració
mkdir -p ~/restaurat && cd ~/restaurat
sftp backup-srv
```

```
sftp> cd pujades
sftp> ls -lt
sftp> get backup_client_20261009_131500.tar.gz
sftp> bye
```

```bash
# Substituïu el nom pel de la còpia més recent que heu descarregat
tar -xzvf backup_client_20261009_131500.tar.gz
diff -r ~/dades ~/restaurat/dades && echo "Restauració correcta"
```

**Ampliacions possibles:**

- Enviar un correu si la còpia falla (Tema 5): `trap 'echo "Error a la còpia" | mail -s "BACKUP FALLIT $(hostname)" admin@educem.cat' ERR`.
- Xifrar el fitxer amb GPG abans de pujar-lo (`gpg --symmetric --cipher-algo AES256 fitxer.tar.gz`), perquè el servidor de còpies no pugui llegir-ne el contingut.

### Pràctica 3: Desplegament d'un Lloc Web per SFTP (SFTP + HTTP)

Al Tema 3 vam crear el lloc `web1.educem.cat` amb el contingut a `/var/www/web1.educem.cat/html`. Ara crearem un usuari **`webdev`** que només pugui actualitzar aquest lloc web per SFTP.

```
/var/                        root:root          755
└── www/                     root:root          755
    └── web1.educem.cat/        root:root          755   ← arrel de la gàbia de webdev
        └── html/            webdev:www-data   2755   ← DocumentRoot d'Apache/Nginx
```

#### Pas 1: Usuari i permisos

```bash
# Usuari només SFTP amb la gàbia a la carpeta del lloc web
sudo useradd -M -d /var/www/web1.educem.cat -s /usr/sbin/nologin webdev
sudo passwd webdev

# La gàbia i les carpetes superiors han de ser de root
sudo chown root:root /var/www /var/www/web1.educem.cat
sudo chmod 755 /var/www /var/www/web1.educem.cat

# El contingut web és de webdev i del grup www-data (el servidor web el pot llegir)
sudo chown -R webdev:www-data /var/www/web1.educem.cat/html
sudo find /var/www/web1.educem.cat/html -type d -exec chmod 2755 {} \;
sudo find /var/www/web1.educem.cat/html -type f -exec chmod 644 {} \;
```

El bit **setgid** (`2755`) fa que els fitxers nous que pugi `webdev` pertanyin al grup `www-data`.

#### Pas 2: Configuració de sshd

Afegiu al final de `/etc/ssh/sshd_config.d/10-sftp.conf`:

```
# =====================================================
# Desenvolupador web: només pot actualitzar web1.educem.cat
# =====================================================
Match User webdev
    ChrootDirectory /var/www/web1.educem.cat
    ForceCommand internal-sftp -u 0022 -l INFO
    AllowTcpForwarding no
    AllowAgentForwarding no
    X11Forwarding no
    PermitTunnel no
```

- `-u 0022`: màscara dels fitxers pujats (fitxers `644`, carpetes `755`) perquè el servidor web els pugui llegir.
- Per registrar les operacions d'aquest usuari cal el socket de logs dins de la seva gàbia (`/var/www/web1.educem.cat/dev/log`), com hem fet al punt 11.

```bash
sudo sshd -t && sudo systemctl restart ssh
```

#### Pas 3: Desplegar la web des del client

```bash
# Al CLIENT: preparar el lloc web en local
mkdir -p ~/web
cat > ~/web/index.html <<EOF
<!DOCTYPE html>
<html lang="ca">
<head><meta charset="utf-8"><title>web1.educem.cat</title></head>
<body><h1>Web desplegada per SFTP</h1><p>Versió: $(date '+%F %T')</p></body>
</html>
EOF

# Primera connexió manual per acceptar l'empremta del servidor
sftp webdev@192.168.1.2
sftp> ls html
sftp> bye

# Sincronitzar tota la carpeta local amb el DocumentRoot remot
lftp -u webdev sftp://192.168.1.2 -e "mirror -R --delete --verbose $HOME/web /html; bye"
```

- `mirror -R`: puja (*reverse mirror*) la carpeta local al servidor.
- `--delete`: esborra del servidor els fitxers que ja no existeixen en local. **Compte:** deixarà el DocumentRoot exactament igual que la carpeta local. S'esborraran els fitxers que hi havia, com el contingut de proves del Tema 3 i les carpetes `admin` i `adminpanel`. Si les voleu conservar, copieu-les primer a `~/web`.

#### Pas 4: Comprovar la web publicada

```bash
# El lloc redirigeix l'HTTP a HTTPS (Tema 3, apartat 11); -k accepta el certificat autosignat
curl -k https://web1.educem.cat
```

Cada vegada que modifiqueu `~/web` i torneu a executar `lftp mirror`, la web es publicarà amb els canvis. És la base de molts processos de desplegament automàtic.

**Per què no `rsync`?** `rsync` necessita executar el programa `rsync` al servidor a través d'una shell SSH. Amb `ForceCommand internal-sftp` no és possible. Les alternatives que funcionen amb usuaris només SFTP són `lftp mirror`, `rclone` (backend SFTP) o la sincronització de WinSCP.

### Bones Pràctiques de SFTP en Producció

- **Un usuari per a cada client, aplicació o procés**, mai comptes compartits.
- **Gàbia (`ChrootDirectory`) + `ForceCommand internal-sftp`** per a tots els usuaris que només intercanvien fitxers.
- **Claus en lloc de contrasenyes** per als processos automàtics, amb `restrict` i `from=`.
- **Claus gestionades per root** en una ubicació central (`AuthorizedKeysFile /etc/ssh/authorized_keys/%u`).
- **Algorismes moderns**: claus `ed25519` (o RSA de 3072 bits o més).
- **Rotació de claus** periòdica i eliminació immediata de les claus de clients que ja no treballen amb l'empresa.
- **Pujades atòmiques** (`.part` + `rename`) quan un altre procés llegeix els fitxers rebuts.
- **Logs activats** (`-l INFO`, amb el socket `/dev/log` muntat dins de cada gàbia) per tenir traçabilitat de qui ha pujat o descarregat cada fitxer.
- **Verificar l'empremta** del servidor la primera vegada i no desactivar mai `StrictHostKeyChecking` als scripts.
- **Protecció contra força bruta** amb fail2ban (punt 14).

### Resolució de Problemes

**L'script demana contrasenya o falla amb `Permission denied (publickey)`:**

```bash
# Veure el detall de l'autenticació al client
sftp -v backup-srv 2>&1 | grep -Ei 'offering|authenticat|denied'

# Al servidor: el fitxer de claus ha de ser de root i no modificable per altres
ls -l /etc/ssh/authorized_keys/backup
sudo journalctl -n 20 --no-pager | grep -i sshd
```

**`Host key verification failed.` a l'script:**

- El client no té l'empremta del servidor a `~/.ssh/known_hosts`. Connecteu-vos una vegada manualment i verifiqueu-la.

**La clau funciona des d'un equip però no des d'un altre:**

- Comproveu la restricció `from="192.168.1.0/24"` del fitxer de claus.

**`webdev` rep `Permission denied` en pujar fitxers:**

- Ha d'escriure dins de `/html`, no a l'arrel de la gàbia. Comproveu `ls -ld /var/www/web1.educem.cat/html`.

### Punt de Control

- [ ] L'usuari `backup` només pot entrar amb clau i des de la xarxa interna.
- [ ] L'script `backup_sftp.sh` puja la còpia i la suma de verificació, i `sha256sum -c` retorna `OK`.
- [ ] `cron` executa la còpia automàticament i el log mostra les execucions.
- [ ] Heu restaurat una còpia i heu comprovat que és idèntica a l'original.
- [ ] `webdev` pot actualitzar `web1.educem.cat` amb `lftp mirror` i el canvi es veu amb `curl`.
