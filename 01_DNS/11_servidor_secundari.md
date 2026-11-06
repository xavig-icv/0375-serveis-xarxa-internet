## 11. Configuració del Servidor Secundari

Un servidor DNS secundari (slave) és un servidor que manté còpies dels registres del servidor primari (master) per garantir redundància i alta disponibilitat. No és responsable de modificar els registres originals; només rep actualitzacions del primari. En cas que el servidor primari falli, els clients poden seguir consultant el secundari.

Al laboratori, el secundari és el servidor **0374** (`192.168.1.1`).

**Pas 1: Instal·lar BIND9 al servidor secundari (0374)**

La instal·lació és la mateixa de l'apartat 06. A més, el 0374 té el tallafoc `ufw` actiu ([escenari](../00_RECURSOS/01_escenari_laboratori.md)): obriu-hi el DNS per a la xarxa interna, que inclou les consultes del client i els avisos del primari.

```bash
sudo ufw allow from 192.168.1.0/24 to any port 53
```

**Pas 2: Opcions del secundari (`/etc/bind/named.conf.options`)**

Les opcions són les mateixes que al primari (apartat 07). Només canvia la IP d'escolta:

```
    listen-on { 127.0.0.1; 192.168.1.1; };
```

Al netplan del 0374, poseu `addresses: [127.0.0.1]` a `nameservers` i `search: [educem.cat]` a l'enp0s8, igual que al 0375.

**Pas 3: ![IMPORTANT] Zones secundàries al 0374 (`/etc/bind/named.conf.local`)**

```
// Zona secundària (FORWARD)
zone "educem.cat" {
    type slave;
    file "/var/cache/bind/db.educem.cat";
    masters { 192.168.1.2; };  // IP del servidor primari (0375)
};

// Zona secundària (REVERSE)
zone "1.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192.168.1";
    masters { 192.168.1.2; };
};
```

`Nota:` A BIND 9.18 també es pot escriure `type secondary;` i `primaries { ... };`. Les zones secundàries es desen a `/var/cache/bind/` perquè BIND hi pot escriure.

**Pas 4: ![IMPORTANT] Al servidor primari 0375 (`192.168.1.2`)**

Cal assegurar que el primari permet les transferències i avisa el secundari. Ja ho hem configurat a les zones dels apartats 08 i 09:

```
zone "educem.cat" {
    type master;
    file "/etc/bind/zones/db.educem.cat";
    allow-transfer { 192.168.1.1; };
    also-notify { 192.168.1.1; };
};
```

### Verificacions

```bash
# Als dos servidors
sudo named-checkconf
sudo systemctl restart named

# Al secundari: s'han creat els fitxers transferits
ls -l /var/cache/bind/

# Al secundari: logs de la transferència de zona
sudo journalctl -u named | grep -i transfer
# o bé
sudo grep named /var/log/syslog | grep -i transfer

# Comparar el serial del primari i del secundari (ha de ser el mateix)
dig @192.168.1.2 educem.cat SOA +short
dig @192.168.1.1 educem.cat SOA +short
```

El secundari desa les zones que rep en format binari (`raw`). Si les obriu amb `cat`, no es poden llegir. Per veure'n el contingut:

```bash
sudo named-compilezone -f raw -F text -o - educem.cat /var/cache/bind/db.educem.cat
```

**Prova de redundància**

1. Comproveu que el client té els dos servidors DNS: `192.168.1.2` i `192.168.1.1`.
2. Desconnecteu la xarxa interna del primari. Ho podeu fer al 0375 amb `sudo ip link set enp0s8 down`, o bé desmarcant "Cable connectat" a la configuració de xarxa de VirtualBox.
3. Comproveu que el client continua resolent:

```bash
dig www.educem.cat
dig @192.168.1.1 www.educem.cat
```

4. Quan acabeu, torneu a activar la interfície del primari: `sudo ip link set enp0s8 up`.
