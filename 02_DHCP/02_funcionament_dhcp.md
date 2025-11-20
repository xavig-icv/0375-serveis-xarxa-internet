## 02. Funcionament del Protocol DHCP

El protocol DHCP (Dynamic Host Configuration Protocol) utilitza un mecanisme d'intercanvi de missatges (entre el client i el servidor) per assignar i gestionar adreces IP i altres paràmetres de xarxa. Aquest procés "DORA" garanteix que cada dispositiu rebi la configuració de manera automàtica, segura i sense conflictes.

### Procés DORA (Discover, Offer, Request, Acknowledge)

Quan un dispositiu s'uneix a una xarxa, el primer que fa és iniciar el procés DORA, un intercanvi de quatre missatges entre el client i el servidor DHCP.

```
CLIENT                                      SERVIDOR DHCP
  │                                              │
  │ 1. DHCPDISCOVER (broadcast)                  │
  │─────────────────────────────────────────────>│
  │   "Necessito configuració de xarxa"          │
  │   Src: 0.0.0.0:68  →  Dst: 255.255.255.255:67│
  │                                              │
  │ 2. DHCPOFFER (unicast o broadcast)           │
  │<─────────────────────────────────────────────│
  │   "T'ofereixo la IP 192.168.1.100"           │
  │   + màscara, gateway, DNS, leas time, etc.   │
  │                                              │
  │ 3. DHCPREQUEST (broadcast)                   │
  │─────────────────────────────────────────────>│
  │   "Accepto l'oferta de la IP 192.168.1.100"  │
  │   (Informa als altres servidors DHCP)        │
  │                                              │
  │ 4. DHCPACK (unicast)                         │
  │<─────────────────────────────────────────────│
  │   "IP confirmada! Aquí tens la configuració" │
  │                                              │
```

**Detall dels missatges:**

1. **DHCPDISCOVER**

- El client envia un missatge de broadcast buscant servidors DHCP disponibles.
- Al missatge inclou la seva MAC i el hostname (opcional).
- Port d'origen: 68 (client), port de destí: 67 (servidor).

2. **DHCPOFFER**

   - Servidor respon amb una oferta d'IP disponible per assignar.
   - Inclou tots els paràmetres de xarxa (màscara, gateway, DNS i altres opcions).
   - Reserva temporalment la IP (mentre espera la confirmació del client).

3. **DHCPREQUEST**

   - El client accepta una oferta d'assignació d'IP (pot rebre diverses).
   - Envia un missatge de broadcast per informar a tots els servidors quina oferta ha acceptat.
   - Sol·licita formalment la IP (request).

4. **DHCPACK**
   - Servidor confirma l'assignació (es guarda un registre).
   - La IP la pot fer servir el client (és vàlida pel seu ús).
   - Inclou el lease time (durada de la concessió) i altres paràmetres de configuració avançats.

### Renovació de la Concessió (Lease Renewal)

Les adreces IP assignades pel DHCP disposen d'un temps de concessió limitat. Quan aquest temps s'apropa al límit, el client intenta renovar la seva IP sense haver de passar de nou pel procés complet (DORA).

Exemple d'un lease time (temps de concessió) de 24 hores.

```
0h                  12h                  18h                 24h
├────────────────────┼────────────────────┼────────────────────┤
│    IP Assignada    │  Fase T1 (50%)     │ Fase T2 (87.5%)    │ Expiració
│                    │  Renovació unicast │ Broadcast general  │
└────────────────────┴────────────────────┴────────────────────┘
```

**Fases del procés de renovació**

- T1 (50% del lease time)
  El client envia un DHCPREQUEST per unicast al mateix servidor que li va concedir l'IP.

- T2 (87,5% del lease time)
  Si el servidor no respon, el client envia un missatge de broadcast per intentar contactar amb qualsevol altre servidor DHCP.

- Expiració
  Si no aconsegueix renovar, el client ha de deixar d'utilitzar la IP i tornar a fer el procés de "DORA".

### Alliberar, Rebutjar i Errors d'Assignació de IPs

**El client Allibera la IP (DHCPRELEASE):**

- El client allibera la IP abans de que expiri el lease time
- Es produeix quan s'apaga un equip, es desconnecta una interfície (Ethernet o Wi-Fi) o es realtizar un "dhclient -r".

**El client Rebutja la IP (DHCPDECLINE):**

- El client detecta un conflicte d'IP mitjançant el protocol ARP.
- Un altra dispositiu respon a la IP que se li vol assignar.

**El servidor Rebutja el request (DHCPNAK):**

- El client intenta renovar una IP que ja ha expirat.
- La IP és invàlida (o pertany a una altra subxarxa).
- La configuració actual del client és incompatible amb la del servidor.

### Ports i Protocol

- **Protocol**: UDP
- **Ports**:
  - Servidor DHCP: 67 (bootps)
  - Client DHCP: 68 (bootpc)
- **Broadcast**: 255.255.255.255 (és el tipus de missatge utilitzat pel descobriment)

**Connexions establertes (DHCP de l'escola)**

```bash
sudo ss -plutn | grep -E ':(67|68)'
```

**Captura amb tcpdump del procés de DORA:**

Veure el procés amb el servidor DHCP de l'escola.

```bash
#Terminal 1
sudo tcpdump -i any port 67 or port 68 -vvv
```

```bash
#Terminal 2
sudo dhclient -r
sudo dhclient
```
