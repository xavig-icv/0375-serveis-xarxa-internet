## 3. Instal·lació de Servidors DHCP

A continuació es mostra com instal·lar i preparar un servidor DHCP per la vostra màquina d'Ubuntu Server. Es realitza el procés d'instal·lació del programari (isc-dhcp-server), es mostra la ubicació dels fitxers de configuració principals i és realitza una configuració inicial per posar-lo en marxa.

### ISC DHCP Server (Debian/Ubuntu)

L'ISC DHCP Server és el servidor DHCP més utilitzat (o més comú) en sistemes basats en Debian. En un futur proper es comenta que Kea DHCP serà el successor natural de ISC-DHCP-SERVER.

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar servidor DHCP
sudo apt install isc-dhcp-server

# Verificar instal·lació
dhcpd --version
```

### Estructura de Fitxers

```
/etc/dhcp/
├── dhcpd.conf           # Configuració principal del servei DHCPv4
├── dhcpd6.conf          # Configuració DHCPv6
└── dhclient.conf        # Configuració del client (opcions del client)

/var/lib/dhcp/
└── dhcpd.leases         # Base de dades de concessions (IPs ja assignades)

/var/log/
└── syslog               # Logs del servei (rsyslog ha d'estar instal·lat)
```

### Configurar una Interfície d'Escolta

Abans d'iniciar el servei, cal especificar en quina interfície escoltarà.

Abans d'iniciar el servei, cal especificar en quina interfície de xarxa rebrà peticions el servidor DHCP. Això garanteix que el servei no pugui fer servir altres interfícies o targetes de xarxa com la WAN o d'altres xarxes que no volem que disposin de servei DHCP.

```bash
# Editar /etc/default/isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server

# La enp0s3 (és la vostra WAN i el DHCP de l'escola us assigna una IP)
# Especificar interfície d'escolta
INTERFACESv4="enp0s8"
# podem assignar múltiples interfícies: INTERFACESv4="enp0s8 enp0s9"

# Per IPv6 (treballarem amb IPv4)
INTERFACESv6=""
```

### Iniciar i Habilitar el Servei

```bash
# Iniciar o reiniciar el servei
sudo systemctl restart isc-dhcp-server

# Verificar l'estat
sudo systemctl status isc-dhcp-server

# Habilitar l'encesa automàtica del servei durant l'arrencada del servidor
sudo systemctl enable isc-dhcp-server

# Verificar el port d'escola
sudo ss -plutn | grep dhcpd
```
