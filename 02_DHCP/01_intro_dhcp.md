## 01. Introducció als Serveis de Configuració Automàtica

La configuració automàtica de xarxa, és un element imprescindible que ha de disposar una infraestructura empresarial. Aquest servei permet que els dispositius obtinguin de manera automàtica tots els paràmetres necessaris per comunicar-se amb altres dispositius dins d'una xarxa local i puguin sortir a Internet, eliminant la necessitat de configuracions manuals i reduint el risc d'errors humans. Aquest servei es realitza amb el protocol DHCP (Dynamic Host Configuration Protocol).

Quan un dispositiu (un ordinador, un mòbil, una impressora, un servidor, etc.) es connecta a una xarxa, necessita obtenir un conjunt de paràmetres essencials per comunicar-se amb altres dispositius de la xarxa i també poder connectar-se a Internet.

Quins són aquests paràmetres?

**Paràmetres essencials per a la comunicació de xarxa:**

- **Adreça IP**: Identificador lògic i únic assignat al dispositiu dins de la xarxa.
- **Màscara de xarxa**: Defineix quins dispositius pertanyen a la mateixa xarxa local.
- **Gateway per defecte**: Dispositiu (normalment Router) que habilita la sortida a Internet o la comunicació amb altres xarxes.
- **Servidors DNS**: S'utilitza un DNS intern per traduir noms de domini en adreces IP, facilitant l'accés a recursos com pàgines web o serveis corporatius. També és poden indicar servidors DNS externs com els de Google, Cloudflare, etc.

**Limitacions alhora de realitzar una configuració manual:**

```
Farragós i lent (especialment en xarxes amb molt dispositius).
Fàcil de cometre errors humans (IPs mal escrites, màscares incorrectes, DNS equivocats, etc).
Es fa dificil de gestionar (quan hi ha canvis o la xarxa creix o es modifica).
Poden haver conflictes d'IP duplicades per un error de la persona que ho gestiona.
Requereix documentar-ho manualment i realitzar el manteniment constant d'aquesta documentació.
```

**Avantatges de la configuració automàtica (DHCP):**

```
Obtenció al moment de tota la configuració i assignació automàtica de la IP als dispositius.
Gestió centralitzada de tota la configuració de xarxa i de les subxarxes.
Evita els conflictes d'IP i elimina els conflictes d'IPs duplicades.
Permet realitzar una reconfiguració de xarxa massiva de manera automàtica.
Permet moure els dispositius (mòbils, PCs, IoT, etc.) i que es connectin a diferents punts de xarxa.
Es pot realtizar un control dels dispositius connectats.
És fàcilment escalable, podent afegir centenars o milers de dispositius.
```

El servei de DHCP també permet utilitzar polítiques d'assignació flexibles, com ara:

- **Assignació dinàmica**: Establir als dispositius IPs durant un cert temps repartides segons la demanda (quan es desconnecta un dispositiu deixa lliure aquella IP).

- **Assignació estàtica**: Establir una IP fixa un dispositiu, és molt utilitzar per a servidors, impressores o equips de xarxa estàtics que interessa que mantinguin sempre aquella IP.

**Opcions avançades**: Per cada subxarxa es poden establir els servidors DNS, servidors NTP, rutes estàtiques, VLANs, PXE Boot (arrancada de discos en xarxa), etc.

```
Infraestructura sense DHCP:
tècnic → configura manualment IP, màscara, gateway i DNS en cada dispositiu = risc d'errors + molt temps invertit

Infraestructura Amb DHCP:
dispositiu → sol·licita configuració → servidor DHCP respon amb els paràmetres = connexió instantània i sense errors
```

**Exemple d'una xarxa corporativa**

```
SERVIDOR DHCP (192.168.1.2) - Ubuntu Server 0375
    │
    ├── PC-01 (Client DHCP)
    │     → IP: 192.168.1.100
    │     → Mask: 255.255.255.0
    │     → Gateway: 192.168.1.1 (Ubuntu Server 0374)
    │     → DNS: 192.168.1.2 (Ubuntu Server 0375)
    │
    ├── PC-02 (Client DHCP)
    │     → IP: 192.168.1.101
    │     → Mask: 255.255.255.0
    │     → Gateway: 192.168.1.1 (Ubuntu Server 0374)
    │     → DNS: 192.168.1.2 (Ubuntu Server 0375)
    │
    └── PC-03 (Client DHCP)
          → IP: 192.168.1.102
          → Mask: 255.255.255.0
          → Gateway: 192.168.1.1 (Ubuntu Server 0374)
          → DNS: 192.168.1.2 (Ubuntu Server 0375)
```

**Exemples d'us del Servei de DHCP**

**WiFi públic (hotspot):**

En espais públics (restaurants, biblioteques, aeroports, etc.), s'utilitza un servei de DHCP perquè:

- Els usuaris es connecten i desconnecten constantment.
- S'assigna una IP temporal (amb temps de concessió molt petits).
- Una configuració automàtica permet disposar de molts usuaris i gestionar-los amb facilitat.
- Establir un lloguer del Wi-Fi (limitant la connexió establint un temps per la concessió).

**ISP (Proveïdor d'Internet):**

Els proveïdors d'Internet com Movistar, Digi, Emagina, etc., fan servir un DHCP a gran escala per assignar una configuració automàtica de xarxa a:

- Assigna IP als routers dels clients (ONTs).
- Configuració i gestió de milers de dispositius simultàniament
- Simulen una gran xarxa local i peretem assignar IP pública dinàmica o estàtica als clients.
