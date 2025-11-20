## 07. Servei en Múltiples Subxarxes: DHCP Relay

Els missatges DHCP no poden travessar routers (accedir a diferents xarxes), perquè utilitzen missatges de broadcast. Per això, es va implementrar el DHCP Relay que permet reenviar les peticions cap al servidor DHCP ubicat en una altra xarxa.

Els clients al fer un DHCPDISCOVER s'envia un missatge de broadcast (255.255.255.255). Els routers no permeten reenviar broadcast i la petició queda a la LAN. Si el servidor DHCP es troba en una altra subxarxa aquest mai rebrà la petició.

**Possibles solucions:**

1. Implementar un Servidor DHCP a cada subxarxa (és possible però ineficient)
2. Implementar un DHCP Relay Agent (reenvia peticions i és centralitza tot en un únic servei DHCP).

### Què és un DHCP Relay?

El Relay Agent reenvia les peticions DHCP entre subxarxes. Escolta les peticions DHCPDISCOVER (broadcast) per una interfície, i les reenvia via UNICAST al servidor DHCP ubicat en una altra xarxa.

```
SUBXARXA A               ROUTER/RELAY        SUBXARXA B
192.168.1.0/24          (amb relay agent)   192.168.2.0/24
┌─────────┐                                 ┌─────────┐
│ CLIENT  │                                 │ SERVIDOR│
│         │ 1. DHCPDISCOVER (broadcast)     │  DHCP   │
│         │─────────────►│                  │         │
│         │              │ 2. Unicast       │         │
│         │              │─────────────────►│         │
│         │              │ 3. DHCPOFFER     │         │
│         │              │◄─────────────────│         │
│         │ 4. Reenvia   │                  │         │
│         │◄─────────────│                  │         │
└─────────┘              └─────────────────►└─────────┘
```

### Configurar el DHCP Relay (Nou Ubuntu Server)

Al router/gateway (fer servir un Ubuntu Server Nou de Router/Relay)

Aquest nou server només tindrà dues interfícies (enp0s3 i enp0s8) i serà de xarxa interna (intnet i intnet2). Ho configurem tots junts!

**L'escenari tindrà 2 subxarxes diferents:**

- Subxarxa 1 (Client i Server Nou): 192.168.2.0/24
- Subxarxa 2 (Server 0375 amb DHCP Server): 192.168.1.0/24

Així haurien de quedar les interficies i les IPs al Virtual Box.

- Server Nou: 192.168.1.1 (enp0s8 --> xarxa interna --> intnet)
- Server 0375: 192.168.1.2 (enp0s8 --> xarxa interna --> intnet)

- Client IP: 192.168.2.X (enp0s3 --> xarxa interna --> intnet2)
- Server Nou: 192.168.2.1 (enp0s3 --> xarxa interna --> intnet2)

**Instal·lar el DHCP Relay (Server Nou)**

```bash
sudo apt update
sudo apt install isc-dhcp-relay
```

**Durant la instal·lació preguntarà:**

```
Servers: 192.168.1.2        # IP del servidor DHCP
Interfaces: enp0s3          # Interfícies connectades als clients
```

**Canviar la configuració manualment:**

```bash
# Editar /etc/default/isc-dhcp-relay
sudo nano /etc/default/isc-dhcp-relay
```

**Contingut:**

```bash
# Servidor DHCP de destí
SERVERS="192.168.1.2"

# Interfícies on escoltar (cap a clients)
INTERFACES="enp0s3"

# Opcions addicionals
OPTIONS=""
```

**Iniciar i habilitrar el DHCP relay:**

```bash
sudo systemctl restart isc-dhcp-relay
sudo systemctl enable isc-dhcp-relay
sudo systemctl status isc-dhcp-relay
```

### Configurar el Servidor DHCP per Múltiples Subxarxes

**Al servidor 0375 - DHCP (`/etc/dhcp/dhcpd.conf`):**

```
# El servidor pot estar a qualsevol subxarxa (ex: 192.168.2.0/24)

# SUBXARXA 2 (via relay)
subnet 192.168.2.0 netmask 255.255.255.0 {
    range 192.168.2.100 192.168.2.200;
    option routers 192.168.2.1;
    option domain-name-servers 192.168.2.1;
    option domain-name "xarxa2.domini.cat";
}

# SUBXARXA 1 (local)
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.2;
    option domain-name-servers 192.168.1.2;
    option domain-name "xarxa1.domini.cat";
}
```

### Verificar Relay

**Al Server Nou (router/relay):**

```bash
# Veure logs
sudo journalctl -u isc-dhcp-relay -f

# Hauries de veure:
# Relaying DHCP Request from enp0s8 to 192.168.1.2
```

**Al servidor 0375 -DHCP:**

```bash
sudo tail -f /var/log/syslog | grep dhcpd

# Hauries de veure peticions de diferents subxarxes
# DHCPDISCOVER from 00:1a:2b:3c:4d:5e via 192.168.2.1
```

**Des del client (subxarxa 2):**

```bash
sudo dhclient -r
sudo dhclient -v

# Hauria d'obtenir IP del rang 192.168.2.100-200
```
