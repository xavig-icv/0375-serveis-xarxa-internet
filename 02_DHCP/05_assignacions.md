## 05. Assignacions Estàtiques i Dinàmiques

El servidor DHCP pot assignar adreces IP de dues maneres:

- Assignació dinàmica `(Dynamic Allocation)`: S'estableix un rang d'IPs determinat (pool) i s'assignen les IPs als clients per ordre d'arribada.

- Assignació estàtica `(Reservations o Fixed Allocation)`: S'associa una IP fixa a una MAC concreta, així un dispositiu tindrà sempre la mateixa IP.

Cada mètode té usos específics i avantatges, i és molt comú combinar-los en entorns de producció.

### Assignació Dinàmica (Rang o Pool d'IPs)

És el tipus d'assignació habitual. El servidor DHCP assigna IPs automàticament a partir d'un pool o rang predefinit.

**Exemple de configuració:**

```
subnet 192.168.1.0 netmask 255.255.255.0 {
    # Pool d'IPs per clients dinàmics
    range 192.168.1.120 192.168.1.200;

    option routers 192.168.1.1; #En el nostre cas server 0374 (gateway amb NAT)
    option domain-name-servers 192.168.1.2, 192.168.1.1; #DNS primari (0375) i secundari (0374)
}
```

**Característiques principals:**

- Les IPs assignades són temporals (basades en el lease time o temps de concessió).
- Un client pot obtenir una IP diferent en cada connexió nova que realitzi.
- És habitual per dispositius com: PCs portàtils, mòbils, Wi-Fi convidats, IoT o dispositius que utilitzen temporalment la xarxa.
- Simplifica i millora la gestió i escalabilitat en xarxes mitjanes i grans.

### Assignació Estàtica (Reserva per MAC)

El servidor DHCP assigna sempre la mateixa IP a una adreça MAC concreta.
Evita així que els clients configurin les seves IPs manualment, evitant errors i centralitzant tota la gestió.

El seu ús principal és per assignar IPs a servidors, impressores, workstations, etc.

**Exemple de configuració:**

Normalment s'assignen IPs de la subxarxa però fora del rang (pool).

```
# Servidor web
host webserver {
    hardware ethernet 00:1a:2b:3c:4d:5e;
    fixed-address 192.168.1.20;
}

# Servidor de correu
host mailserver {
    hardware ethernet 00:5e:4d:3c:2b:1a;
    fixed-address 192.168.1.30;
}

# Impressora de xarxa
host printer01 {
    hardware ethernet 00:aa:bb:cc:dd:ee;
    fixed-address 192.168.1.40;
    # Opcions específiques per aquest host
    option host-name "impressora01";
}

# Client Ubuntu del laboratori
host client {
    hardware ethernet 08:00:27:aa:bb:cc;   # MAC de l'enp0s3 del client
    fixed-address 192.168.1.50;
}
```

`Nota:` Al laboratori, els servidors 0375 i 0374 tenen la IP fixa al netplan, no per DHCP. El gateway i els servidors DHCP no poden dependre del DHCP: si el servidor DHCP no hi fos, es quedarien sense IP.

**Paràmetres:**

- `hardware ethernet`: MAC del dispositiu (obligatori)
- `fixed-address`: IP fixa assignada
- Opcions addicionals (DNS, routers, etc.) - opcional

**Esquema recomanat per una subxarxa:**

```
192.168.1.0/24
├── .1        → Gateway (router)
├── .2-.20    → Servidors crítics (IP estàtica manual)
├── .21-.99   → Reserves DHCP (host declarations)
└── .120-.200 → Pool dinàmic (range)
```

### Obtenir MAC d'un dispositiu

```bash
# Al servidor: veure concessions actives
sudo cat /var/lib/dhcp/dhcpd.leases | grep hardware

# Al servidor: equips de la xarxa local amb qui s'ha comunicat recentment
ip neigh show dev enp0s8

# Al client: MAC de la seva interfície (línia link/ether)
ip link show enp0s3
```

### Assignacions per Classe (Class-based)

Es poden crear **classes** per assignar diferents configuracions.

Es poden crear "classes" perquè el servidor pugui separar clients segons:

- El Sistema operatiu
- El Fabricant
- El Vendor Class ID
- Una Substring de la MAC
- Opcions de DHCP específiques

**Exemple: separar els clients Windows de la resta**

```
# Definir una classe segons els 4 primers caràcters del "vendor" (els Windows envien "MSFT")
class "windows-clients" {
    match if substring (option vendor-class-identifier, 0, 4) = "MSFT";
}

subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.1; # El servidor 0374 (gateway)

    # Pool per sistemes Windows
    pool {
        allow members of "windows-clients";
        range 192.168.1.120 192.168.1.160;
        option domain-name-servers 192.168.1.2; # Windows: només el DNS primari
    }

    # Pool per a la resta d'equips (Linux, mòbils...)
    pool {
        deny members of "windows-clients";
        range 192.168.1.161 192.168.1.200;
        option domain-name-servers 192.168.1.2, 192.168.1.1; # DNS primari i secundari
    }
}
```

`Nota:` Els clients Linux habituals (NetworkManager, systemd-networkd) normalment no envien l'opció `vendor-class-identifier`. Per això el segon pool no fa servir `allow members of` una classe "Linux", sinó que accepta tots els equips que no són Windows. Si el pool només admetés membres d'una classe que no coincideix amb cap client, aquests clients no rebrien cap IP.
