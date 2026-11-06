## 06. Alta Disponibilitat: DHCP Failover

El failover DHCP permet disposar de dos servidors DHCP que treballen conjuntament i es sincronitzen per garantir l'alta disponibilitat. Si falla un d'ells, l'altre pot continuar donant servei als clients. També es poden configurar per repartir-se la càrrega entre ells i així millorar el rendiment general del servei.

**Avantatges:**

- Redundància (Alta disponibilitat): Si un servidor cau, l'altre continua donant servei.
- Balanceig de la càrrega (Load Balancing): Cada servidor s'encarrega de gestionar el 50% dels clients.
- Permet realitzar el manteniment d'un dels servidors sense necessitat d'interrompre el servei.

**Modes de funcionament:**

- **Load balancing**: Ambdós servidors assignen IPs (50/50)
- **Hot standby**: Un servidor primari actiu i un de secundari en espera (xarxes petites)

Nota: Els dos servidors mantenen sincronitzada la base de dades de concessions (leases).

**Abans de començar:**

- Els dos servidors han de tenir **IP fixa al netplan**: `192.168.1.2` el 0375 i `192.168.1.1` el 0374. Un servidor DHCP (i el gateway) no pot dependre del DHCP: si el primari caigués, el secundari perdria la seva IP.
- La configuració de la subxarxa i les reserves (`host`) han de ser **iguals als dos servidors**.
- El port de failover (647/tcp) ha d'estar obert entre els dos servidors.

### Configuració Failover - Servidor Primari (Server 0375)

**Fitxer: `/etc/dhcp/dhcpd.conf` (Servidor 0375 - Primari)**

```
# CONFIGURACIÓ FAILOVER - SERVIDOR PRIMARI

failover peer "dhcp-failover" {
    primary;                              # Indica que és el servidor primari
    address 192.168.1.2;                  # IP local del servidor primari (0375)
    port 647;                             # Port de comunicació failover
    peer address 192.168.1.1;             # IP del servidor secundari (0374)
    peer port 647;
    max-response-delay 60;                # Segons a esperar resposta del peer
    max-unacked-updates 10;
    load balance max seconds 3;

    # Paràmetres que només es posen al primari
    mclt 3600;    # Maximum Client Lead Time: temps màxim que un servidor pot allargar una concessió sense l'acord de l'altre
    split 128;    # Load balancing 50/50 (128 de 256)
}

# SUBNET associada al failover
subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.2, 192.168.1.1;
    option domain-name "educem.cat";

    # Pool associat al failover
    pool {
        failover peer "dhcp-failover";
        range 192.168.1.120 192.168.1.200;
    }
}
```

### Configuració Failover - Servidor Secundari (Server 0374)

**Fitxer: `/etc/dhcp/dhcpd.conf` (Servidor 0374 - Secundari)**

```
# CONFIGURACIÓ FAILOVER - SERVIDOR SECUNDARI

failover peer "dhcp-failover" {
    secondary;                            # Indica que és el servidor secundari
    address 192.168.1.1;                  # IP del servidor secundari (0374)
    port 647;
    peer address 192.168.1.2;             # IP del servidor primari (0375)
    peer port 647;
    max-response-delay 60;
    max-unacked-updates 10;
    load balance max seconds 3;
}

# SUBNET associada al failover (LA MATEIXA configuració que al primari)
subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.2, 192.168.1.1;
    option domain-name "educem.cat";

    pool {
        failover peer "dhcp-failover";
        range 192.168.1.120 192.168.1.200;
    }
}
```

Al secundari cal instal·lar també `isc-dhcp-server` i posar `INTERFACESv4="enp0s8"` a `/etc/default/isc-dhcp-server` (apartat 03). A més, el 0374 té el tallafoc `ufw` actiu: obriu-hi el DHCP de la xarxa interna i el port del failover.

```bash
sudo ufw allow in on enp0s8 to any port 67 proto udp           # peticions dels clients
sudo ufw allow from 192.168.1.2 to any port 647 proto tcp      # failover amb el 0375
```

### Mode Hot Standby (Primari Actiu - Secundari Reserva)

Si volem establir un servidor com actiu i un secundari com a reserva (xarxes petites):

**Servidor Primari:**

```
failover peer "dhcp-failover" {
    primary;
    address 192.168.1.2; #primari (0375)
    port 647;
    peer address 192.168.1.1; #secundari (0374)
    peer port 647;
    max-response-delay 60;
    max-unacked-updates 10;

    # mclt és obligatori al primari (sense mclt, dhcpd no arrenca)
    mclt 3600;
    # Hot standby: primari gestiona tot
    split 255;  # Primari: 100%, Secundari: 0%
}
```

**Servidor Secundari:**

```
failover peer "dhcp-failover" {
    secondary;
    address 192.168.1.1; #secundari (0374)
    port 647;
    peer address 192.168.1.2; #primari (0375)
    peer port 647;
    max-response-delay 60;
    max-unacked-updates 10;
}
```

La subxarxa i el `pool` amb `failover peer` són els mateixos que al mode load balancing.

### Iniciar el Failover

Les següents comandes s'han de realitzar `ALS DOS SERVIDORS`.

```bash
# FER ALS DOS SERVIDORS:

# 1. Verificar configuració
sudo dhcpd -t

# 2. Reiniciar servei
sudo systemctl restart isc-dhcp-server

# 3. Verificar logs
sudo journalctl -u isc-dhcp-server -f
# o bé
sudo tail -f /var/log/syslog | grep dhcpd

# Hauria de mostrar el missatge:
# dhcpd: failover peer dhcp-failover: I move from ... to normal
```

### Monitorar el Failover

```bash
# Veure l'estat del failover
sudo journalctl -u isc-dhcp-server | grep failover
```

| Estat                          | Significat                            |
| ------------------------------ | ------------------------------------- |
| **startup**                    | Iniciant i sincronitzant              |
| **normal**                     | Tot funciona (funcionament normal)    |
| **communications-interrupted** | No pot contactar amb el peer          |
| **partner-down**               | L'altre servidor està caigut          |
| **potential-conflict**         | Possible conflicte d'assignació d'IPs |
| **recover**                    | Recuperant la base de dades de leases |

**Comprovar el balanceig (mode load balancing)**

- Cada servidor registra al log els `DHCPACK` de les IPs que ha donat.
- El servidor que deixa una petició a l'altre ho registra amb un missatge semblant a `load balance to peer dhcp-failover`.
- El repartiment es fa segons la **MAC** del client: un mateix client sempre va al mateix servidor. Per veure el repartiment calen diversos clients o interfícies (MAC diferents).
- Amb només dues interfícies, hi ha un 50 % de probabilitats que totes dues vagin al mateix servidor. En aquest cas, afegiu una altra interfície al client o canvieu-ne la MAC a VirtualBox.

**Recrear un Failover Real: Aturar el servidor Primari**

```bash
# Al servidor primari (0375), aturar el servei
sudo systemctl stop isc-dhcp-server

# Al servidor secundari (0374), verificar els logs
sudo journalctl -u isc-dhcp-server -f
# Hauria de dir: "failover peer dhcp-failover: I move from normal to communications-interrupted"
```

**Què fa el secundari en estat `communications-interrupted`:**

- Continua renovant les concessions dels clients.
- Dona IPs noves de la part del rang que té reservada.
- En aquest estat, les concessions duren com a màxim l'`mclt` (1 hora).

L'estat `partner-down` (el secundari assumeix tot el rang) no s'activa sol. Cal forçar-lo manualment o configurar `auto-partner-down`. Aquesta opció s'ha de fer servir amb compte: si l'altre servidor encara funciona, es poden assignar IPs duplicades.

Iniciar sessió a l'ubuntu client i verificar que el servei DHCP segueix operatiu.

```bash
# Tornar a demanar la configuració al client d'escriptori (NetworkManager)
sudo nmcli device disconnect enp0s3
sudo nmcli device connect enp0s3
ip -4 addr show enp0s3
```
