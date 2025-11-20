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

    option routers 192.168.1.2; #En el nostre cas server 0375 (enp0s8)
    option domain-name-servers 192.168.1.2; #En el nostre cas server 0375 (enp0s8)
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
    option routers 192.168.1.1; #Si disposa d'una altra gateway
}

# PC de l'administrador
host admin-pc {
    hardware ethernet 00:11:22:33:44:55;
    fixed-address 192.168.1.50;
    # Opcions específiques per aquest host
    option domain-name-servers 192.168.1.2, 1.1.1.1;
}
```

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
└── .100-.200 → Pool dinàmic (range)
```

### Obtenir MAC d'un dispositiu des del servidor

```bash
# Veure concessions actives
sudo cat /var/lib/dhcp/dhcpd.leases | grep hardware

# O amb arp (si el client està a la xarxa local)
arp -a
```

### Assignacions per Classe (Class-based)

Es poden crear **classes** per assignar diferents configuracions.

Es poden crear "classes" perquè el servidor pugui separar clients segons:

- El Sistema operatiu
- El Fabricant
- El Vendor Class ID
- Una Substring de la MAC
- Opcions de DHCP específiques

**Exemple: separar Windows i Linux**

```
# Definir classes segons els 4 o 5 primers digits del "vendor" (MSFT o Linux)
class "windows-clients" {
    match if substring (option vendor-class-identifier, 0, 4) = "MSFT";
}

class "linux-clients" {
    match if substring (option vendor-class-identifier, 0, 5) = "Linux";
}

subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.2; # El nostre servidor 0375

    # Pool per sistemes Windows
    pool {
        allow members of "windows-clients";
        range 192.168.1.100 192.168.1.150;
        option domain-name-servers 192.168.1.2; # Limitem Windows a només DNS local
    }

    # Pool per sistemes Linux
    pool {
        allow members of "linux-clients";
        range 192.168.1.151 192.168.1.200;
        option domain-name-servers 192.168.1.2, 8.8.8.8; # El nostre servidor DNS 0375 + Google
    }
}
```
