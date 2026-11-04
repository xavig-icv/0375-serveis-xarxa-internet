## 11. SFTP amb OpenSSH

**SFTP** (SSH File Transfer Protocol) és avui el protocol de referència per transferir fitxers de manera segura en l'administració de sistemes. **No té res a veure amb l'FTP**: és un subsistema del protocol **SSH**, el mateix que fem servir per administrar els servidors remotament.

### SFTP vs FTP/FTPS

```
FTP / FTPS                                      SFTP
                                                
CLIENT          SERVIDOR                        CLIENT              SERVIDOR
  │══ control :21 ══>│                            │                    │
  │══ dades :40xxx ══>│  (una per transferència)   │═══ SSH :22 ═══════>│
  │══ dades :40xxx ══>│                            │  ┌──────────────┐  │
  │       ...        │                            │  │ autenticació │  │
                                                   │  │ ordres       │  │  Tot dins d'una
2+ connexions, ports dinàmics                      │  │ dades        │  │  única connexió
Xifrat opcional (TLS)                              │  └──────────────┘  │  xifrada
                                                   │                    │
```

| Característica             | FTP / FTPS                         | SFTP                                     |
| -------------------------- | ---------------------------------- | ---------------------------------------- |
| Protocol base              | FTP (RFC 959) + TLS                | SSH (RFC 4251-4254)                      |
| Ports                      | 21 + rang passiu (o 990)           | **Només el 22**                          |
| Connexions                 | Control + una per a cada transferència | **Una sola connexió**                |
| Xifrat                     | Opcional (FTPS)                    | **Sempre**                               |
| Autenticació               | Contrasenya, certificat de client  | Contrasenya, **claus SSH**, certificats SSH |
| Verificació del servidor   | Certificat X.509 (CA)              | Empremta de la clau de host (`known_hosts`) |
| Tallafocs i NAT            | Complicat                          | **Trivial**                              |
| Operacions sobre fitxers   | Bàsiques                           | Completes (permisos, enllaços, reprendre...) |
| Accés anònim               | Sí                                 | No (no està pensat per a això)           |

`Nota:` l'SFTP mai ha arribat a ser un RFC. OpenSSH implementa la **versió 3** de l'esborrany de l'IETF (*draft-ietf-secsh-filexfer*), que és la que suporten pràcticament tots els clients i servidors.

### Components d'OpenSSH per a SFTP

- **`sshd`**: el servidor SSH, escolta al port 22.
- **`sftp-server`**: programa extern que implementa el protocol SFTP (`/usr/lib/openssh/sftp-server`).
- **`internal-sftp`**: el mateix servidor SFTP, però **integrat dins de `sshd`**. No necessita cap programa ni biblioteca dins d'una gàbia chroot, i per això és el que s'utilitza per engabiar usuaris.
- **`sftp`** i **`scp`**: clients de línia d'ordres.

`Nota:` des d'**OpenSSH 9.0** (2022), l'ordre `scp` utilitza internament el protocol **SFTP** en lloc de l'antic protocol SCP/RCP, que tenia problemes de seguretat.

### Pas 1: Comprovar el Servidor SSH

Al servidor 0375 ja hauríeu de tenir OpenSSH instal·lat (és el que feu servir per administrar-lo).

```bash
# Instal·lar-lo si no hi és
sudo apt install openssh-server

# Comprovar l'estat del servei (a Debian/Ubuntu el servei es diu "ssh")
sudo systemctl status ssh --no-pager

# Comprovar el port
sudo ss -tlnp | grep ':22 '

# Veure la línia del subsistema SFTP
grep -i subsystem /etc/ssh/sshd_config
# Subsystem sftp /usr/lib/openssh/sftp-server
```

### Pas 2: Primer Ús de SFTP

**Qualsevol usuari amb accés SSH ja pot fer servir SFTP**, sense configurar res. Proveu-ho des del client amb el vostre usuari administrador:

```bash
sftp admin@192.168.1.2
```

La primera vegada, el client mostra l'**empremta de la clau del servidor**:

```
The authenticity of host '192.168.1.2 (192.168.1.2)' can't be established.
ED25519 key fingerprint is SHA256:Qm3k...x8Y.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Per verificar que l'empremta és correcta (i que no hi ha cap atac *man-in-the-middle*), consulteu-la **al servidor**:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Un cop acceptada, es desa a `~/.ssh/known_hosts` del client i no es tornarà a demanar.

**Ordres del client `sftp`:**

| Ordre                     | Descripció                                                  |
| ------------------------- | ----------------------------------------------------------- |
| `pwd` / `lpwd`            | Directori remot / local                                     |
| `ls` / `lls`              | Llistar el directori remot / local                          |
| `cd` / `lcd`              | Canviar de directori remot / local                          |
| `get fitxer`              | Descarregar                                                 |
| `get -r carpeta`          | Descarregar una carpeta recursivament                       |
| `put fitxer`              | Pujar                                                       |
| `put -r carpeta`          | Pujar una carpeta recursivament                             |
| `reget` / `reput`         | Reprendre una transferència interrompuda                    |
| `mkdir` / `rmdir`         | Crear / esborrar carpetes remotes                           |
| `rm fitxer`               | Esborrar un fitxer remot                                    |
| `rename origen destí`     | Canviar el nom                                              |
| `chmod 640 fitxer`        | Canviar permisos                                            |
| `ln -s origen enllaç`     | Crear un enllaç simbòlic                                    |
| `df -h`                   | Espai lliure al servidor                                    |
| `!ordre`                  | Executar una ordre local                                    |
| `bye` / `exit`            | Sortir                                                      |

```
sftp> lcd /tmp
sftp> put /etc/hostname prova_sftp.txt
Uploading /etc/hostname to /home/admin/prova_sftp.txt
sftp> ls -l
-rw-r--r--    1 admin    admin          10 Oct  9 13:10 prova_sftp.txt
sftp> rm prova_sftp.txt
sftp> bye
```

**Problema:** aquest usuari té accés **complet** al servidor: pot obrir una shell SSH, navegar per tot el sistema de fitxers i fer túnels. Per a clients o aplicacions que només han d'intercanviar fitxers, crearem usuaris **només SFTP i engabiats**.

### Pas 3: Usuaris Només SFTP Engabiats

**Requisits de la gàbia d'OpenSSH (`ChrootDirectory`):**

- La carpeta arrel de la gàbia **i totes les carpetes superiors** han de ser propietat de **root**.
- **Ningú més que root** hi pot escriure (permisos màxims `755`).
- L'usuari escriu en una **subcarpeta** que sí que és seva.

És la mateixa idea que la gàbia de vsftpd, però OpenSSH és encara més estricte (no hi ha cap opció per saltar-se-la).

```
/srv/sftp/                       root:root        755
├── client1/                     root:root        755   ← arrel de la gàbia (ChrootDirectory)
│   └── pujades/                 client1:sftpusers 750  ← l'usuari treballa aquí
└── client2/                     root:root        755
    └── pujades/                 client2:sftpusers 750
```

**Crear el grup, els usuaris i les carpetes:**

```bash
# Grup per als usuaris només SFTP
sudo groupadd sftpusers

# Carpeta base
sudo mkdir -p /srv/sftp
sudo chown root:root /srv/sftp
sudo chmod 755 /srv/sftp

# Usuari client1: home = arrel de la gàbia, sense shell, sense crear el home (-M)
sudo useradd -M -d /srv/sftp/client1 -g sftpusers -s /usr/sbin/nologin client1
sudo passwd client1

# Arrel de la gàbia (propietat de root) i carpeta de treball (propietat de l'usuari)
sudo mkdir -p /srv/sftp/client1/pujades
sudo chown root:root /srv/sftp/client1
sudo chmod 755 /srv/sftp/client1
sudo chown client1:sftpusers /srv/sftp/client1/pujades
sudo chmod 750 /srv/sftp/client1/pujades

# Repetir per a client2
sudo useradd -M -d /srv/sftp/client2 -g sftpusers -s /usr/sbin/nologin client2
sudo passwd client2
sudo mkdir -p /srv/sftp/client2/pujades
sudo chown root:root /srv/sftp/client2
sudo chmod 755 /srv/sftp/client2
sudo chown client2:sftpusers /srv/sftp/client2/pujades
sudo chmod 750 /srv/sftp/client2/pujades
```

### Pas 4: Configurar sshd

A Debian 12+ i Ubuntu 22.04+, el fitxer `/etc/ssh/sshd_config` inclou automàticament els fitxers de `/etc/ssh/sshd_config.d/*.conf`. Crearem la nostra configuració en un fitxer separat (així no es perd en actualitzar el paquet):

```bash
sudo nano /etc/ssh/sshd_config.d/10-sftp.conf
```

```
# =====================================================
# Usuaris només SFTP (grup sftpusers)
# =====================================================
Match Group sftpusers
    # Engabiar l'usuari al seu directori personal (/srv/sftp/<usuari>)
    ChrootDirectory %h
    # Forçar SFTP: no es pot obrir una shell ni executar ordres
    ForceCommand internal-sftp -l INFO
    # Desactivar túnels i reenviaments
    AllowTcpForwarding no
    AllowAgentForwarding no
    X11Forwarding no
    PermitTunnel no
```

| Directiva            | Funció                                                                  |
| -------------------- | ----------------------------------------------------------------------- |
| `Match Group`        | Les directives següents només s'apliquen als membres del grup           |
| `ChrootDirectory %h` | Engabia l'usuari al seu home (`%h`). També es pot usar `/srv/sftp/%u` |
| `ForceCommand internal-sftp` | Qualsevol connexió executa el servidor SFTP intern, mai una shell |
| `-l INFO`            | Registra les operacions de fitxers (obrir, pujar, esborrar...) als logs |
| `AllowTcpForwarding no` | Impedeix fer servir el servidor com a túnel cap a altres màquines    |

`Nota:` un bloc `Match` s'acaba quan comença un altre `Match` o quan s'acaba el fitxer. Poseu-lo sempre **al final** del fitxer de configuració.

**Comprovar la configuració i reiniciar el servei:**

```bash
# Verificar la sintaxi (si no mostra res, és correcta)
sudo sshd -t

# Veure la configuració efectiva que s'aplicarà a client1
sudo sshd -T -C user=client1,host=client,addr=192.168.1.100 | grep -Ei 'chroot|forcecommand|forwarding'

# Reiniciar el servei SSH
sudo systemctl restart ssh
```

**Atenció:** si treballeu per SSH, **no tanqueu la sessió actual** fins que hàgiu comprovat que podeu obrir-ne una de nova. Si hi ha un error de configuració, us podríeu quedar sense accés al servidor.

**Autenticació per contrasenya:** a Ubuntu, si el servidor s'ha instal·lat amb *cloud-init*, pot existir el fitxer `/etc/ssh/sshd_config.d/50-cloud-init.conf` amb `PasswordAuthentication no`. Comproveu-ho:

```bash
sudo sshd -T | grep -i passwordauthentication
```

### Pas 5: Proves des del Client

```bash
sftp client1@192.168.1.2
```

```
client1@192.168.1.2's password:
Connected to 192.168.1.2.
sftp> pwd
Remote working directory: /
sftp> ls
pujades
sftp> cd /etc
Couldn't canonicalize: No such file or directory
sftp> put informe.pdf
dest open "/informe.pdf": Permission denied
sftp> cd pujades
sftp> put informe.pdf
Uploading informe.pdf to /pujades/informe.pdf
informe.pdf                         100%   47KB   9.8MB/s   00:00
sftp> bye
```

L'usuari està engabiat, només pot escriure a `pujades` i no veu res més del servidor.

**Comprovar que no pot obrir una shell:**

```bash
ssh client1@192.168.1.2
# This service allows sftp connections only.
# Connection to 192.168.1.2 closed.
```

**Comprovar que `client1` no veu els fitxers de `client2`:**

```bash
sftp client1@192.168.1.2
sftp> cd /srv/sftp/client2
Couldn't canonicalize: No such file or directory
```

### Pas 6: SCP i Altres Clients

**scp** (còpia directa, sense sessió interactiva):

```bash
# Pujar un fitxer
scp informe.pdf client1@192.168.1.2:/pujades/

# Descarregar un fitxer
scp client1@192.168.1.2:/pujades/informe.pdf ./descarregat.pdf

# Copiar una carpeta sencera
scp -r ./documents client1@192.168.1.2:/pujades/
```

`Nota:` `scp` utilitza el protocol SFTP des d'OpenSSH 9.0. Amb versions anteriors (Ubuntu 22.04 porta la 8.9), `scp` necessita executar una ordre al servidor i falla amb `ForceCommand internal-sftp`: afegiu-hi l'opció `-s` (`scp -s fitxer client1@...`) perquè faci servir SFTP.

**sftp en mode no interactiu:**

```bash
# Descarregar directament un fitxer
sftp client1@192.168.1.2:/pujades/informe.pdf

# Executar ordres llegides de l'entrada estàndard (demana la contrasenya igualment)
echo "ls -l pujades" | sftp client1@192.168.1.2
```

`Nota:` per a scripts existeix el mode *batch* (`sftp -b fitxer_d_ordres`), que s'atura al primer error. Però el mode batch **desactiva la pregunta de la contrasenya**: només funciona amb autenticació per claus. Ho farem servir al punt 12.

**Clients gràfics:**

- **FileZilla**: `Protocol: SFTP - SSH File Transfer Protocol`, port `22`.
- **WinSCP** (Windows): protocol de fitxers `SFTP`.
- **Nautilus**: `Altres ubicacions → Connecta al servidor` → `sftp://client1@192.168.1.2`.
- **Windows 10/11**: inclou els clients `ssh`, `sftp` i `scp` de sèrie (PowerShell o CMD).

### Logs de SFTP

Amb `-l INFO`, `internal-sftp` registra cada operació de fitxers (obrir, pujar, esborrar...) al sistema de logs, amb la *facility* `AUTH`. Però hi ha un detall important: **els programes envien els logs a través del socket `/dev/log`, i dins d'una gàbia aquest socket no existeix**. Per tant, les operacions dels usuaris engabiats es perden silenciosament: només veuríem les línies de `sshd` (connexió i autenticació), que no està engabiat.

La solució és la mateixa que hem fet servir per a la carpeta compartida del punt 6: un **muntatge enllaçat** del socket real dins de cada gàbia. Als sistemes actuals, `/dev/log` és un enllaç al socket de `systemd-journald`:

```bash
ls -l /dev/log
# lrwxrwxrwx 1 root root 28 ... /dev/log -> /run/systemd/journal/dev-log
```

**Crear el socket dins de les gàbies:**

```bash
for u in client1 client2; do
  # Carpeta dev/ (propietat de root, com exigeix la gàbia) i fitxer buit com a punt de muntatge
  sudo mkdir -p /srv/sftp/$u/dev
  sudo touch /srv/sftp/$u/dev/log
  # Muntar el socket real a sobre
  sudo mount --bind /run/systemd/journal/dev-log /srv/sftp/$u/dev/log
done

# Comprovar-ho: ha d'aparèixer com a socket (una "s" al principi dels permisos)
ls -l /srv/sftp/client1/dev/log
# srw-rw-rw- 1 root root 0 ... /srv/sftp/client1/dev/log
```

L'usuari veurà una carpeta `dev` buida a l'arrel de la seva gàbia; és normal.

**Fer-ho permanent** (afegir a `/etc/fstab`):

```
# Socket de logs dins de les gàbies SFTP
/run/systemd/journal/dev-log  /srv/sftp/client1/dev/log  none  bind,nofail,x-systemd.requires=systemd-journald-dev-log.socket  0  0
/run/systemd/journal/dev-log  /srv/sftp/client2/dev/log  none  bind,nofail,x-systemd.requires=systemd-journald-dev-log.socket  0  0
```

- `nofail`: si el muntatge falla, el sistema arrenca igualment.
- `x-systemd.requires=...`: garanteix que el socket de journald ja existeix quan es fa el muntatge.

```bash
# Provar les línies de fstab de debò: desmuntar els sockets i tornar-los a muntar des de fstab
sudo umount /srv/sftp/client1/dev/log /srv/sftp/client2/dev/log
sudo mount -a
findmnt | grep dev/log
```

`Nota:` l'alternativa clàssica és fer que `rsyslog` creï un socket addicional dins de cada gàbia (`input(type="imuxsock" Socket="/srv/sftp/client1/dev/log" CreatePath="on")` en un fitxer de `/etc/rsyslog.d/`). A Debian 12/13 cal instal·lar `rsyslog` abans i, a Ubuntu, el perfil d'AppArmor de `rsyslogd` pot impedir crear sockets fora de `/dev` i `/var/log`.

**Consultar els logs:**

```bash
# Línies de sshd (autenticació) i d'internal-sftp (operacions de fitxers)
sudo journalctl -f | grep -Ei 'sshd|sftp'

# A Ubuntu (amb rsyslog) també a /var/log/auth.log
sudo tail -f /var/log/auth.log | grep -Ei 'sshd|sftp'
```

**Exemple de sortida** (el format exacte pot variar segons la versió i la distribució):

```
sshd[3120]: Accepted password for client1 from 192.168.1.100 port 51544 ssh2
internal-sftp[3125]: session opened for local user client1 from [192.168.1.100]
internal-sftp[3125]: opendir "/pujades"
internal-sftp[3125]: open "/pujades/informe.pdf" flags WRITE,CREATE,TRUNCATE mode 0644
internal-sftp[3125]: close "/pujades/informe.pdf" bytes read 0 written 48213
internal-sftp[3125]: session closed for local user client1 from [192.168.1.100]
```

Si només veieu les línies de `sshd` i cap d'`internal-sftp`, el socket no està muntat dins de la gàbia: comproveu-ho amb `findmnt | grep dev/log`.

### Quotes de Disc amb SFTP

Les quotes del punt 8 són del **sistema de fitxers**, de manera que també funcionen amb SFTP sense cap configuració addicional:

```bash
sudo setquota -u client1 204800 225280 0 0 /
sudo quota -vs client1
```

### Resolució de Problemes

**La connexió es tanca just després d'introduir la contrasenya** (`Connection closed` o `client_loop: send disconnect: Broken pipe`):

```bash
# Consultar el motiu al servidor
sudo journalctl -n 30 --no-pager | grep -i sshd

# Error típic:
# fatal: bad ownership or modes for chroot directory "/srv/sftp/client1"
# → La gàbia o alguna carpeta superior no és de root o té permisos d'escriptura per a altres
sudo chown root:root /srv/sftp /srv/sftp/client1
sudo chmod 755 /srv/sftp /srv/sftp/client1
```

**`Permission denied` en pujar fitxers:**

- L'usuari està escrivint a l'arrel de la gàbia (`/`), que és de root. Ha d'escriure a `pujades`.
- Comproveu el propietari de la carpeta: `ls -ld /srv/sftp/client1/pujades`.

**`Permission denied (publickey)` sense demanar contrasenya:**

- L'autenticació per contrasenya està desactivada (`PasswordAuthentication no`). Reviseu `sudo sshd -T | grep -i passwordauth`.

**El servei SSH no arrenca després del canvi:**

- Executeu `sudo sshd -t` per veure la línia exacta de l'error.
- Si `sshd -t` mostra `Missing privilege separation directory: /run/sshd` (habitual a Ubuntu 24.04 quan el servei s'activa per socket), creeu-lo amb `sudo mkdir -p /run/sshd` i torneu-ho a provar.

### Punt de Control

- [ ] Podeu connectar amb `sftp` i `scp` amb el vostre usuari administrador.
- [ ] Heu verificat l'empremta de la clau del servidor amb `ssh-keygen -lf`.
- [ ] `client1` i `client2` estan engabiats: `pwd` retorna `/` i només poden escriure a `pujades`.
- [ ] `ssh client1@servidor` retorna `This service allows sftp connections only.`
- [ ] Els logs mostren les operacions de fitxers dels usuaris SFTP.
- [ ] Heu connectat per SFTP des de FileZilla o WinSCP.
