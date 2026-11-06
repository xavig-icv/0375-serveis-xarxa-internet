## 01. Escenari Únic del Laboratori

Tots els temes del mòdul (DNS, DHCP, web, FTP, correu, missatgeria, àudio i vídeo) es fan sobre el **mateix escenari**. Cada tema afegeix serveis a les mateixes màquines i fa servir el DNS i el DHCP configurats als temes 1 i 2. Per això és important mantenir les adreces i els noms d'aquest document durant tot el curs.

### Topologia

```
                         Xarxa del centre (adaptador pont) → Internet
              ┌───────────────────────────┴───────────────────────────┐
              │ enp0s3: 10.0.1.X (DHCP del centre)                     │ enp0s3: 10.0.1.X (DHCP del centre)
   ┌──────────┴──────────┐                                 ┌──────────┴──────────┐
   │   SERVIDOR 0374     │                                 │   SERVIDOR 0375     │
   │ Gateway (NAT)       │                                 │ Primari de tots     │
   │ Secundari DNS i DHCP│                                 │ els serveis         │
   └──────────┬──────────┘                                 └──────────┬──────────┘
              │ enp0s8: 192.168.1.1/24                                 │ enp0s8: 192.168.1.2/24
              └───────────────────────────┬───────────────────────────┘
                         Xarxa interna "intnet" 192.168.1.0/24
                                          │
                                          │ enp0s3: 192.168.1.50 (reserva DHCP)
                               ┌──────────┴──────────┐
                               │   CLIENT UBUNTU     │
                               │ Gateway: .1         │
                               │ DNS: .2 i .1        │
                               └─────────────────────┘
```

- **Servidor 0375** (Ubuntu Server): és el primari de tots els serveis del mòdul.
- **Servidor 0374** (Ubuntu Server):
  - És la sortida a Internet de la xarxa interna. L'`ip_forward` i el NAT es configuren al mòdul 0374.
  - És el secundari del DNS i del DHCP.
  - Al bloc d'alta disponibilitat (09), és la reserva del web, de l'FTP i de la recepció de correu, amb la IP virtual `192.168.1.10`.
  - També el fem servir com a segon equip de proves, per exemple per provar un accés des d'una IP no autoritzada.
- **Client Ubuntu** (escriptori): només té la xarxa interna i surt a Internet pel 0374.
- **Servidor Jitsi** (Ubuntu Server, només al Tema 8): màquina virtual addicional per a la videoconferència, a la xarxa interna (`192.168.1.3`). Com el client, surt a Internet pel 0374.

### Pla d'Adreces

| IP                              | Equip         | Noms DNS                                    | Funció                                                    |
| ------------------------------- | ------------- | ------------------------------------------- | --------------------------------------------------------- |
| `192.168.1.1`                   | Servidor 0374 | `ns2`                                       | Gateway amb NAT, DNS secundari, DHCP secundari (failover) |
| `192.168.1.2`                   | Servidor 0375 | `ns1`, `www`, `web1`, `web2`, `mail`, `ftp`, `xmpp`, `news`, `radio` | DNS primari, DHCP primari, web, FTP, correu, missatgeria, notícies i àudio |
| `192.168.1.3`                   | Servidor Jitsi | `meet`                                     | Videoconferència (Jitsi Meet, Tema 8). IP fixa            |
| `192.168.1.10`                  | IP virtual    | `serveis` (i, al bloc 09, `www`, `web1`, `web2`, `ftp`) | Bloc 09: la té el servidor actiu (normalment el 0375) |
| `192.168.1.50`                  | Client Ubuntu | `client`                                    | IP fixa al Tema 1; reserva DHCP a partir del Tema 2       |
| `192.168.1.120`-`192.168.1.200` | Altres equips | -                                           | Rang dinàmic del DHCP                                     |
| `192.168.1.254`                 | Relay DHCP    | -                                           | Només a l'activitat de DHCP relay (Tema 2)                |

### Serveis i Ports

| Tema | Servei           | Programari                 | Servidor 0375               | Servidor 0374        | Ports                         |
| ---- | ---------------- | -------------------------- | --------------------------- | -------------------- | ----------------------------- |
| 1    | DNS              | BIND9                      | Zones primàries             | Zones secundàries    | 53/udp, 53/tcp                |
| 2    | DHCP             | isc-dhcp-server            | Failover (primari)          | Failover (secundari) | 67/udp, 647/tcp               |
| 3    | Web              | Apache                     | Llocs `web1` i `web2`       | -                    | 80, 443                       |
| 3    | Web              | Nginx                      | Els mateixos llocs          | -                    | 8080, 8443                    |
| 4    | FTP, SFTP i TFTP | vsftpd, OpenSSH, tftpd-hpa | Sí                          | -                    | 21 i 40000-40100, 22, 69/udp  |
| 5    | Correu           | Postfix i Dovecot          | Sí                          | -                    | 25, 587, 110, 143, 993, 995   |
| 6    | Missatgeria      | Prosody (XMPP)             | Sí                          | -                    | 5222, 5281                    |
| 6    | Llistes          | Mailman 3 (amb Postfix i Apache) | Sí                    | -                    | 25 i 443 (web `llistes`)      |
| 6    | Notícies         | INN2                       | Sí                          | -                    | 119                           |
| 7    | Àudio            | Icecast i ffmpeg; web i podcast a Apache | Sí            | -                    | 8000 i 443 (web `radio`)      |
| 8    | Vídeo            | Apache (vídeo i HLS), Nginx RTMP, ffmpeg | Sí            | -                    | 443 (web `video`) i 1935      |
| 8    | Videoconferència | Jitsi Meet                 | -                           | -                    | Servidor Jitsi (`.3`): 443 i 10000/udp |
| 09   | Alta disponibilitat | keepalived (VRRP), rsync | Principal (IP virtual)     | Reserva: web, FTP i MX secundari | VRRP (protocol IP 112); els dels serveis |

Apache i Nginx funcionen **alhora** al servidor 0375. Apache fa servir els ports estàndard (80 i 443) i Nginx, els ports 8080 i 8443.

### El Domini `educem.cat`

- El domini del laboratori és **`educem.cat`**, i els servidors DNS del laboratori en són els servidors autoritatius.
- `educem.cat` no està registrat a Internet (comprovat el 4/11/2026).
- **No feu servir mai dominis que existeixen a Internet**, com `domini.cat` o `exemple.cat`. Si un equip preguntés a un DNS extern, arribaria als servidors reals d'una altra organització.
- Si algun dia algú registrés `educem.cat`, el laboratori continuaria funcionant, sempre que tots els equips resolguin amb el DNS del laboratori (vegeu "Configuració de Xarxa").
- Per a dominis addicionals de proves, feu servir el sufix `.test`, que l'RFC 6761 reserva per a proves.

### Zona DNS de Referència

Contingut de la zona `educem.cat` al final del mòdul. Els comentaris indiquen quin tema afegeix cada registre:

```
$TTL 86400
@   IN  SOA ns1.educem.cat. admin.educem.cat. (
            2026110401  ; Serial (AAAAMMDDnn): s'incrementa a cada canvi
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            300 )       ; Negative Cache TTL

@        IN  NS      ns1.educem.cat.
@        IN  NS      ns2.educem.cat.
@        IN  MX  10  mail.educem.cat.     ; Tema 1 (el fa servir el Tema 5)

ns1      IN  A       192.168.1.2          ; Tema 1 - servidor 0375
ns2      IN  A       192.168.1.1          ; Tema 1 - servidor 0374
www      IN  A       192.168.1.2          ; Tema 1
mail     IN  A       192.168.1.2          ; Tema 1 (el fa servir el Tema 5)
ftp      IN  A       192.168.1.2          ; Tema 1 (el fa servir el Tema 4)
client   IN  A       192.168.1.50         ; Tema 1 - client Ubuntu
web      IN  CNAME   www                  ; Tema 1
web1     IN  A       192.168.1.2          ; Tema 3
web2     IN  A       192.168.1.2          ; Tema 3
sftp     IN  CNAME   ftp                  ; Tema 4
tftp     IN  CNAME   ftp                  ; Tema 4
xmpp     IN  A       192.168.1.2          ; Tema 6 - missatgeria (Prosody)
fitxers  IN  CNAME   xmpp                 ; Tema 6 - fitxers del xat
llistes  IN  CNAME   www                  ; Tema 6 - web de les llistes (Mailman)
news     IN  A       192.168.1.2          ; Tema 6 - notícies (INN2)
_xmpp-client._tcp  IN  SRV  0 5 5222 xmpp.educem.cat.   ; Tema 6
radio    IN  A       192.168.1.2          ; Tema 7 - ràdio (Icecast) i podcast
video    IN  A       192.168.1.2          ; Tema 8 - vídeo sota demanda, HLS i directes
meet     IN  A       192.168.1.3          ; Tema 8 - servidor Jitsi (videoconferència)

; Bloc 09 (alta disponibilitat). A més d'afegir aquests registres, en canvia d'altres:
;   www, web1, web2 i ftp passen a la IP virtual 192.168.1.10
;   llistes, sftp i tftp passen a ser registres A cap a 192.168.1.2
serveis  IN  A       192.168.1.10         ; Bloc 09 - IP virtual
mail2    IN  A       192.168.1.1          ; Bloc 09 - MX secundari (0374)
@        IN  MX  20  mail2.educem.cat.    ; Bloc 09
```

Zona inversa `1.168.192.in-addr.arpa`:

```
2        IN  PTR     ns1.educem.cat.
1        IN  PTR     ns2.educem.cat.
3        IN  PTR     meet.educem.cat.     ; Tema 8
10       IN  PTR     serveis.educem.cat.  ; Bloc 09
50       IN  PTR     client.educem.cat.
```

### Configuració de Xarxa (netplan)

A Ubuntu, el fitxer `/etc/resolv.conf` el gestiona systemd-resolved i no s'ha d'editar a mà. Els servidors DNS de cada equip es configuren amb netplan.

**Servidor 0375.** El fitxer és a `/etc/netplan/`; a Ubuntu Server sol ser `50-cloud-init.yaml`.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                       # adaptador pont (xarxa del centre)
      dhcp4: true
    enp0s8:                       # xarxa interna
      addresses: [192.168.1.2/24]
      nameservers:
        addresses: [127.0.0.1]    # el BIND del mateix servidor
        search: [educem.cat]
```

Amb `search: [educem.cat]`, systemd-resolved envia les consultes dels noms d'`educem.cat` només al BIND local. La resta de consultes també poden anar al DNS del centre.

**Servidor 0374.** El fitxer és igual, amb `192.168.1.1/24` a l'enp0s8.

**Client Ubuntu, Tema 1 (IP fixa).** Fitxer `/etc/netplan/01-network-manager-all.yaml`:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:                       # xarxa interna
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1        # servidor 0374 (gateway amb NAT)
      nameservers:
        addresses: [192.168.1.2, 192.168.1.1]
        search: [educem.cat]
```

**Client Ubuntu, a partir del Tema 2 (DHCP).** Es posa `dhcp4: true` en lloc de les adreces, la ruta i els DNS. El servidor DHCP li dona:

- la IP `192.168.1.50` (reserva per MAC);
- el gateway `192.168.1.1`;
- els DNS `192.168.1.2` i `192.168.1.1`;
- el domini `educem.cat`.

Per aplicar els canvis, feu `sudo netplan apply`. Si netplan avisa que els permisos del fitxer són massa oberts, feu `sudo chmod 600 /etc/netplan/*.yaml`.

### DHCP de Referència

Paràmetres que reparteix el servidor DHCP (Tema 2):

```
option domain-name "educem.cat";
option domain-name-servers 192.168.1.2, 192.168.1.1;

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.120 192.168.1.200;
    option routers 192.168.1.1;
    option broadcast-address 192.168.1.255;
}

host client {
    hardware ethernet 08:00:27:XX:XX:XX;   # MAC de l'enp0s3 del client
    fixed-address 192.168.1.50;
}
```

**El DHCP només ha de repartir els DNS del laboratori** (`.2` i `.1`), mai DNS públics com `1.1.1.1` o `8.8.8.8`:

- El domini `educem.cat` no existeix a Internet.
- systemd-resolved fa servir un sol DNS cada vegada. Si el client passa a fer servir un DNS públic (per exemple, quan el primari no respon), deixa de resoldre els noms del laboratori.
- Els noms d'Internet ja els resol el BIND del laboratori a través dels forwarders.

### Pas 0: Comprovació de l'Escenari

Abans de començar cada tema, a partir del Tema 2, comproveu que l'escenari funciona.

**Al client:**

```bash
ip -4 addr show enp0s3              # 192.168.1.50/24
ip route | grep default             # default via 192.168.1.1
resolvectl dns                      # enp0s3: 192.168.1.2 192.168.1.1 (cap DNS públic)
dig +short ns1.educem.cat           # 192.168.1.2
dig +short MX educem.cat            # 10 mail.educem.cat.
ping -c1 www.google.com             # resolució externa i sortida a Internet pel 0374
```

**Als servidors 0375 i 0374:**

```bash
resolvectl dns                      # enp0s8: 127.0.0.1
dig +short www.educem.cat           # 192.168.1.2
```

### Problemes Habituals

| Símptoma | Causa probable | Solució |
| -------- | -------------- | ------- |
| El client no resol els noms d'`educem.cat`, o només a vegades | El client té un DNS públic a la llista | Repartir només `192.168.1.2` i `192.168.1.1` (DHCP o netplan) |
| Un nom que acabem de crear no resol al client | El client guarda la resposta negativa a la memòria cau | `resolvectl flush-caches` al client |
| El secundari respon dades antigues | No s'ha incrementat el serial | Incrementar el serial i fer `sudo rndc reload` al primari |
| Un nom respon dues IP diferents | Hi ha un registre A duplicat | Modificar el registre existent en lloc d'afegir-ne un altre |
| `dig @localhost` falla al servidor | BIND no escolta a `127.0.0.1` | Afegir `127.0.0.1` a `listen-on` (Tema 1, apartat 07) |
| El servidor no resol `educem.cat` | Al netplan falten els `nameservers` de l'enp0s8 | Configurar netplan com s'indica més amunt |
| El client no surt a Internet | Gateway incorrecte o NAT del 0374 aturat | Gateway `192.168.1.1` i comprovar el 0374 |
