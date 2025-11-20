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

### Configuració Failover - Servidor Primari (Server 0375)

**Fitxer: `/etc/dhcp/dhcpd.conf` (Servidor 1 - Primari)**

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

    # Clau compartida per l'autenticació (opcional però recomanat)
    mclt 3600;    # Maximum Client Lead Time (només al primari)
    split 128;    # Load balancing 50/50 (128 de 256)
}

# SUBNET associada al failover
subnet 192.168.1.0 netmask 255.255.255.0 {
    option routers 192.168.1.2;
    option domain-name-servers 192.168.1.2;
    option domain-name "domini.cat";

    # Pool associat al failover
    pool {
        failover peer "dhcp-failover";
        range 192.168.1.120 192.168.1.200;
    }
}
```

### Configuració Failover - Servidor Secundari (Server 0374)

**Fitxer: `/etc/dhcp/dhcpd.conf` (Servidor 2 - Secundari)**

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
    option routers 192.168.1.2;
    option domain-name-servers 192.168.1.2;
    option domain-name "domini.cat";

    pool {
        failover peer "dhcp-failover";
        range 192.168.1.120 192.168.1.200;
    }
}
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

# Hauria de mostrar el missatge:
# dhcpd: failover peer dhcp-failover: I move from startup to normal
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

**Recrear un Failover Real: Aturar el servidor Primari**

```bash
# Al servidor primari (0375), aturar el servei
sudo systemctl stop isc-dhcp-server

# Al servidor secundari (0374), verificar els logs
sudo journalctl -u isc-dhcp-server -f
# Hauria de dir: "failover peer ... I move from normal to partner-down"
```

Iniciar sessió a l'ubuntu client i verificar que el servei DHCP segueix operatiu.

```bash
#Release de la IP actual i requerir una nova concessió
sudo dhclient -r
sudo dhclient
ip a
```
