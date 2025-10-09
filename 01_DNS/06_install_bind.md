## 06. Instal·lació i Informació Inicial de BIND

### Què és BIND?

BIND (Berkeley Internet Name Domain) és el **programari DNS més utilitzat** en sistemes Unix/Linux. Permet:

- Resoldre noms de domini a adreces IP (i viceversa).
- Gestionar zones autoritatives i subdominis.
- Actuar com a servidor recursiu, autoritatiu o forwarder.
- Configurar rèpliques de zones (primàries i secundàries).

És programari lliure i molt flexible, i és la base de molts serveis DNS a Internet.

### Instal·lació de BIND9

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar BIND9
sudo apt install bind9 bind9utils bind9-doc dnsutils

# Verificar de la instal·lació
named -v

# Iniciar i habilitar servei
sudo systemctl start named
sudo systemctl enable named
sudo systemctl status named

# Verificar el port d'escolta
sudo ss -plutn
```

### Estructura de Fitxers de BIND

L'arxiu principal de BIND és `/etc/bind/named.conf`, que inclou referències a altres fitxers i defineix com s'ha de comportar el servidor DNS. La carpeta `/etc/bind()` inclou:

```
/etc/bind/
├── named.conf                 # Configuració principal
├── named.conf.options         # Opcions globals (forwarders, listen-on, etc.)
├── named.conf.local           # Zones locals (dominis que volem gestionar)
├── named.conf.default-zones   # Zones per defecte (localhost, reverse lookup)
└── zones/                     # Fitxers de zona amb registres DNS
    ├── db.example.com         # Exemple de fitxer de zona per example.com (zona directa)
    └── db.192.168.1           # Exemple de fitxer de zona inversa per xarxa 192.168.1.0/24
```

#### Detalls de cada fitxer

- **named.conf**  
  Arxiu principal que inclou la resta de fitxers i defineix configuracions generals. És l’entrada de BIND.

- **named.conf.options**  
  Conté opcions globals del servidor DNS, com:

  - `forwarders`: servidors recursius als quals enviar consultes que no podem resoldre.
  - `listen-on`: quines interfícies escolten per consultes DNS.
  - `allow-query`: quins clients poden consultar el DNS.

- **named.conf.local**  
  Defineix **zones locals** que gestionem. Aquí indiquem:

  - Quins dominis són autoritatius per aquest servidor.
  - On es troben els fitxers de zona corresponents.

- **named.conf.default-zones**  
  Zones que venen per defecte amb BIND, normalment:

  - `localhost` (127.0.0.1)
  - Zona inversa de 127.0.0.0/8
  - Zona inversa per IPv6 `::1`

- **zones/**  
  Carpeta on es guarden els fitxers de zona:
  - `db.example.com`: fitxer de zona directa per al domini `example.com`. Conté registres **A, AAAA, MX, CNAME, NS**, etc.
  - `db.192.168.1`: fitxer de zona inversa per a la xarxa 192.168.1.0/24. Permet resoldre IP → nom (PTR).
