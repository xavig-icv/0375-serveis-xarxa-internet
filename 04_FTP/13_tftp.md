## 13. TFTP: Transferència Trivial de Fitxers

**TFTP** (Trivial File Transfer Protocol, RFC 1350) és un protocol de transferència de fitxers **extremadament simple**. No té autenticació, no permet llistar directoris i funciona sobre **UDP**. Aquesta simplicitat és justament el que el fa útil: es pot implementar en molt poc codi, i per això el trobem integrat al **firmware de les targetes de xarxa** i dels **dispositius de xarxa**.

### Característiques de TFTP

| Característica       | TFTP                                         | FTP                              |
| -------------------- | -------------------------------------------- | -------------------------------- |
| Transport            | **UDP** (port 69)                            | TCP (port 21 + dades)            |
| Autenticació         | **Cap**                                      | Usuari i contrasenya             |
| Llistar directoris   | **No** (cal saber el nom exacte del fitxer)  | Sí                               |
| Ordres               | Només llegir (`RRQ`) i escriure (`WRQ`)      | Desenes d'ordres                 |
| Fiabilitat           | Pròpia: cada bloc s'ha de confirmar (`ACK`)  | La de TCP                        |
| Mida del codi        | Molt petita (cap en una ROM)                 | Gran                             |

### Funcionament del Protocol

TFTP només té **5 tipus de missatge**: `RRQ` (petició de lectura), `WRQ` (petició d'escriptura), `DATA`, `ACK` i `ERROR`. Les dades s'envien en **blocs de 512 bytes** (ampliable amb opcions modernes) i cada bloc s'ha de confirmar abans d'enviar el següent.

```
CLIENT (port 50123)                                 SERVIDOR
     │                                                  │
     │── RRQ "pxelinux.0" mode octet ──────────► :69    │
     │                                                  │  El servidor respon des
     │◄──────────── DATA bloc 1 (512 bytes) ───── :41000│  d'un PORT NOU (aleatori)
     │── ACK bloc 1 ───────────────────────────► :41000 │
     │◄──────────── DATA bloc 2 (512 bytes) ───── :41000│
     │── ACK bloc 2 ───────────────────────────► :41000 │
     │                    ...                           │
     │◄──────────── DATA bloc N (< 512 bytes) ─── :41000│  Un bloc més petit de 512
     │── ACK bloc N ───────────────────────────► :41000 │  indica el final del fitxer
```

### Usos Actuals de TFTP

**1. Arrencada per xarxa (PXE)**

Els ordinadors poden arrencar sense disc (o instal·lar un sistema operatiu) des de la xarxa. El procés combina **DHCP** (Tema 2) i **TFTP**:

```
  EQUIP NOU                      SERVIDOR DHCP                 SERVIDOR TFTP
(arrencada PXE)                                               
     │── DHCPDISCOVER (sóc PXE) ──►│                                │
     │◄── DHCPOFFER ───────────────│  IP + next-server 192.168.1.2  │
     │                             │  + filename "pxelinux.0"       │
     │── RRQ pxelinux.0 ───────────────────────────────────────────►│
     │◄── carregador d'arrencada ───────────────────────────────────│
     │── RRQ kernel + initrd ──────────────────────────────────────►│  (o per HTTP amb iPXE)
     │◄── ... ──────────────────────────────────────────────────────│
     │
  Arrenca l'instal·lador o el sistema operatiu
```

A l'ISC DHCP Server del Tema 2, això es configuraria afegint dins de la `subnet`:

```
# Servidor TFTP i fitxer d'arrencada per als clients PXE
next-server 192.168.1.2;
filename "pxelinux.0";
```

Actualment, molts entorns només fan servir TFTP per descarregar un carregador petit (com **iPXE**) i la resta de fitxers (nucli, imatges) es descarreguen per **HTTP**, que és molt més ràpid.

**2. Dispositius de xarxa**

- **Còpies de seguretat** de la configuració de routers i switches: `copy running-config tftp:` (Cisco).
- **Actualització del firmware** de routers, switches, punts d'accés i telèfons IP.
- **Aprovisionament automàtic** de telèfons VoIP i equips que descarreguen la seva configuració en arrencar.

### Pas 1: Instal·lació del Servidor TFTP

Utilitzarem **tftpd-hpa**, el servidor TFTP més habitual a Debian i Ubuntu.

```bash
sudo apt update
sudo apt install tftpd-hpa

# Comprovar el servei i el port (UDP 69)
sudo systemctl status tftpd-hpa --no-pager
sudo ss -ulnp | grep ':69 '
```

**Fitxer de configuració `/etc/default/tftpd-hpa`:**

```bash
cat /etc/default/tftpd-hpa
```

```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

| Variable          | Descripció                                                                 |
| ----------------- | -------------------------------------------------------------------------- |
| `TFTP_USERNAME`   | Usuari amb què s'executa el servei (sense privilegis)                      |
| `TFTP_DIRECTORY`  | Carpeta arrel dels fitxers que es serveixen                                |
| `TFTP_ADDRESS`    | Adreça i port d'escolta (`:69` = totes les interfícies)                    |
| `TFTP_OPTIONS`    | `--secure`: engabia el servei a `TFTP_DIRECTORY` (chroot)                  |

### Pas 2: Servir Fitxers (Només Lectura)

```bash
# Crear fitxers de prova a la carpeta del servei
echo "Fitxer de prova TFTP de domini.cat" | sudo tee /srv/tftp/prova.txt
sudo mkdir -p /srv/tftp/firmware
echo "firmware v2.1" | sudo tee /srv/tftp/firmware/switch-v2.1.bin

# Els fitxers han de ser llegibles per l'usuari tftp
sudo chmod -R a+rX /srv/tftp
```

**Prova des del client:**

```bash
# Instal·lar el client TFTP
sudo apt install tftp-hpa

# Mode interactiu
tftp 192.168.1.2
tftp> verbose
tftp> binary
tftp> get prova.txt
Received 35 bytes in 0.0 seconds
tftp> get firmware/switch-v2.1.bin
tftp> quit

# Mode no interactiu (-c ordre)
tftp 192.168.1.2 -c get prova.txt
cat prova.txt
```

Fixeu-vos que **no s'ha demanat cap usuari ni contrasenya** i que no hi ha cap ordre per llistar fitxers.

### Pas 3: Permetre Pujar Fitxers (Còpies de Configuració)

Per defecte, tftpd-hpa **no permet crear fitxers nous**. Per rebre còpies de seguretat de configuracions de routers i switches, crearem una carpeta específica i activarem l'opció `--create`.

```bash
# Carpeta per a les còpies de configuració, propietat de l'usuari tftp
sudo mkdir /srv/tftp/configs
sudo chown tftp:tftp /srv/tftp/configs

sudo nano /etc/default/tftpd-hpa
```

```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="192.168.1.2:69"
TFTP_OPTIONS="--secure --create --umask 077"
```

- `TFTP_ADDRESS="192.168.1.2:69"`: escolta **només a la interfície interna**.
- `--create`: permet crear fitxers nous (només on l'usuari `tftp` té permís d'escriptura, és a dir, a `configs`).
- `--umask 077`: els fitxers rebuts només els pot llegir l'usuari `tftp` (les configuracions solen contenir contrasenyes).

```bash
sudo systemctl restart tftpd-hpa
```

**Prova des del client** (simulant la còpia de seguretat d'un switch):

```bash
echo "hostname SW-PLANTA1" > SW-PLANTA1-confg
tftp 192.168.1.2 -c put SW-PLANTA1-confg configs/SW-PLANTA1-confg
```

```bash
# Al servidor
sudo ls -l /srv/tftp/configs/
# -rw------- 1 tftp tftp 20 ... SW-PLANTA1-confg
```

En un switch o router Cisco real, l'ordre equivalent seria:

```
SW-PLANTA1# copy running-config tftp:
Address or name of remote host []? 192.168.1.2
Destination filename [sw-planta1-confg]? configs/SW-PLANTA1-confg
```

### Seguretat de TFTP

TFTP **no té cap mecanisme de seguretat**: qualsevol equip que pugui arribar al port 69 pot descarregar (o pujar) fitxers si en sap el nom. Per això:

- **Mai** s'ha d'exposar a Internet.
- S'ha de limitar a la **xarxa o VLAN de gestió** amb el tallafoc (punt 14).
- La carpeta de pujades ha d'estar separada de la de descàrregues.
- Els fitxers rebuts s'han de protegir (`--umask 077`) i moure periòdicament a una ubicació segura.
- Si no s'utilitza, el servei s'ha de **desactivar**: `sudo systemctl disable --now tftpd-hpa`.

`Nota:` TFTP respon des d'un port UDP aleatori (no des del 69). Els tallafocs amb estat del servidor ho gestionen bé, però un tallafoc al **client** pot bloquejar les respostes. Si cal restringir els ports del servidor, tftpd-hpa té l'opció `--port-range 50000:50100`.

### Resolució de Problemes

**`Transfer timed out.`**

- El servei no està actiu o el tallafoc bloqueja el port 69/UDP (o les respostes cap al client).
- Comproveu-ho amb `sudo ss -ulnp | grep ':69 '` i `sudo ufw status`.

**`Error code 1: File not found`**

- El nom no és exacte (TFTP distingeix majúscules i minúscules) o el fitxer no és dins de `/srv/tftp`.

**`Error code 2: Access violation` / `Permission denied`**

- El fitxer no és llegible per l'usuari `tftp`, o s'intenta pujar un fitxer sense `--create` o a una carpeta on `tftp` no pot escriure.

### Punt de Control

- [ ] `tftpd-hpa` escolta al port 69/UDP de la interfície interna.
- [ ] Podeu descarregar `prova.txt` i `firmware/switch-v2.1.bin` sense autenticació.
- [ ] Podeu pujar una configuració a `configs/` i es desa amb permisos `600`.
- [ ] Sabeu quines opcions del DHCP (`next-server` i `filename`) indiquen el servidor TFTP als clients PXE.
