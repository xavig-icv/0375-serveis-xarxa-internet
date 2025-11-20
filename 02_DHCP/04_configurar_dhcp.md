## 04. Configuració Bàsica del Servidor DHCP

Un cop instal·lat el servidor, hem de configurar el fitxer principal del servei DHCP. Veurem l'estructura general del fitxer de configuració, realitzarem un exemple complet i com validar i aplicar la configuració realitzada.

### Fitxer de Configuració Principal

El fitxer `/etc/dhcp/dhcpd.conf` permet definir tota la configuració del servidor: els paràmetres globals, les subxarxes, els rang d'IPs per assignar (pools), les opcions de xarxa i també permet realitzar reserves estàtiques d'adreces IP.

**Resum de l'estructura del fitxer:**

```
┌──────────────────────────────────────────────┐
│ PARÀMETRES GLOBALS                           │
│ (Afecten a tota la configuració)             |
|  - Lease time per defecte                    │
│  - Servidors DNS, nom de domini, logs, etc.  │
├──────────────────────────────────────────────┤
│ SUBNET 1                                     │
│  - Rang d'IPs                                │
│  - Opcions específiques (router, DNS, etc.)  │
│  - Reserves (MAC + IP fixa) (opcional)       │
├──────────────────────────────────────────────┤
│ SUBNET 2 (opcional)                          │
│  - Rang i configuració específica            │
└──────────────────────────────────────────────┘
```

### Exemple de Configuració Bàsica

```bash
# Editar el fitxer de configuració
sudo nano /etc/dhcp/dhcpd.conf
```

**Contingut específic:**

```bash
# /etc/dhcp/dhcpd.conf
# Configuració bàsica del servidor DHCP

###################################################
# PARÀMETRES GLOBALS
###################################################

# Aquest servidor és autoritatiu per la xarxa (el principal)
authoritative;

# Temps de concessió per defecte (segons)
default-lease-time 86400;      # 24 hores
max-lease-time 172800;         # 48 hores

# Opcions globals (si no s'especifiquen a la subnet)
option domain-name "domini.cat";
option domain-name-servers 1.1.1.1, 1.0.0.1;

# Logging (local7 és una "categoria" per identificar els logs al syslog)
# Permet separar els logs de DHCP dels altres (podem indicar un fitxer /var/log/dhcpd.log)
log-facility local7;

###################################################
# DEFINICIÓ DE LA SUBNET (SUBXARXA PRINCIPAL)
###################################################

subnet 192.168.1.0 netmask 255.255.255.0 {
    # Rang d'IPs disponibles per assignar (pool)
    range 192.168.1.120 192.168.1.200;

    # Gateway per defecte (router) - Per nosaltres el propi servidor 0375
    option routers 192.168.1.2;

    # Servidors DNS (sobreescriu global) - Per nosaltres el propi servidor 0375
    option domain-name-servers 192.168.1.2, 1.1.1.1;

    # Domini de cerca (ping www = ping www.domini.cat)
    option domain-name "domini.cat";

    # Broadcast (disposem d'una màscara /24)
    option broadcast-address 192.168.1.255;

    # Temps de concessió específic per aquesta subnet
    default-lease-time 43200;     # 10 hores (jornada laboral + descansos)
    max-lease-time 86400;         # 24 hores
}
```

### Paràmetres Globals Importants

**authoritative**

- Indica que aquest servidor és l'autoritat per la xarxa (servidor principal).
- Enviarà un DHCPNAK si un client sol·licita o disposa d'una IP incorrecta.
- Essencial tenir-ho habilitat per xarxes de producció per evitar errors.

**Temps de concessió**

```
default-lease-time 86400;    # Temps normal (24h)
max-lease-time 172800;       # Màxim permès (48h)
min-lease-time 3600;         # Mínim (1h) - opcional
```

- default-lease-time: Temps per defecte d'assignació d'una IP.
- max-lease-time: Màxim de temps permès per disposar d'una IP vàlida.
- min-lease-time: Evita que els clients sol·licitin un lease time molt baix.

**Logging**

```
log-facility local7;
```

- Especifica on es registren els logs (syslog facility)
- Per defecte: `local7`
- Permet separar els logs de DHCP al fitxer de syslog facilitant el diagnosi d'errors.

### Verificar la configuració i aplicar els canvis

```bash
# Després de modificar el fitxer dhcpd.conf

# 1. Verificar la sintaxi
sudo dhcpd -t

# 2. Reiniciar el servei
sudo systemctl restart isc-dhcp-server    # Debian/Ubuntu

# 3. Verificar l'estat
sudo systemctl status isc-dhcp-server
```

### Base de Dades de Concessions

El fitxer `/var/lib/dhcp/dhcpd.leases` emmagatzema totes les concessions actives (IPs assignades i l'estat de cada concessió):

```bash
# Veure les concessions actives
sudo cat /var/lib/dhcp/dhcpd.leases
```

**Exemple de concessió:**

```
lease 192.168.1.120 {
  starts 4 2026/05/10 09:00:00;
  ends 5 2026/05/11 09:00:00;
  cltt 4 2026/05/10 09:00:00;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 08:00:27:5d:33:01;
  client-hostname "PC-xavi";
}
```

**Camps importants:**

- `starts` i `ends`: Quan comença i s'acaba l'assignació.
- `hardware ethernet`: MAC del dispositiu client
- `binding state`: Estat (active, free, expired, etc.)
- `client-hostname`: Nom de l'equip que ha sol·licitat la IP
