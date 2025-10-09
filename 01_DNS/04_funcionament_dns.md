## 04. Funcionament i Nomenclatura DNS

### FQDN (Fully Qualified Domain Name)

Un FQDN és un nom de domini complet que indica la ubicació exacta d’un host dins de la jerarquia DNS.
Sempre acaba amb un punt final (.) que representa l’arrel del sistema de noms, tot i que s'omet.

```
srv1.www.exemple.cat.
│   │   │       │   │
│   │   │       │   └─ Arrel (Root)
│   │   │       └───── TLD (.cat)
│   │   └───────────── Domini (exemple)
│   └───────────────── Subdomini (www)
└───────────────────── Host (srv1)
```

**Exemples de FQDN:**

- `mail.google.com`
- `www.gencat.cat`
- `servidor.departament.escola.cat`

### Zones d'Autoritat

Una **zona DNS** és la part de l’espai de noms que un servidor DNS administra directament.
Un domini pot dividir-se en diverses zones i així poder delegar-ne la gestió.

```
Zona: exemple.cat
├── SOA    → exemple.cat. SOA ns1.exemple.cat. admin.exemple.cat.
├── NS     → exemple.cat. NS ns1.exemple.cat.
├── A      → www.exemple.cat. A 192.168.1.10
├── AAAA   → srv1.exemple.cat. AAAA 2001:db8:85a3::8a2e:370:7334
├── MX     → exemple.cat. MX 10 mail.exemple.cat.
├── CNAME  → ftp.exemple.cat. CNAME srv1.exemple.cat.
├── TXT    → exemple.cat. TXT "v=spf1 mx ~all"
├── SRV    → _sip._tcp.exemple.cat. SRV 10 60 5060 sipserver.exemple.cat.
└── PTR    → 10.1.168.192.in-addr.arpa. PTR www.exemple.cat.
```

**Conceptes clau:**

- **Zona primària**: Conté la informació oficial i modificable dels registres DNS.
- **Zona secundària**: Còpia de la informació (només lectura) de la zona primària. (redundància per si falla)
- **Delegació**: Transfereix la autoritat (gestió) d'un subdomini a un altre servidor DNS.

### Delegació de Dominis

Permet que un subdomini sigui gestionat per un altre servidor.
Exemple de delegació del subdomini `dept.exemple.cat`:

**Fitxer de zona de example.cat:**

```
; Delegació del subdomini dept
dept.exemple.cat.    IN    NS    ns1.dept.exemple.cat.
dept.exemple.cat.    IN    NS    ns2.dept.exemple.cat.
ns1.dept.exemple.cat. IN    A     192.168.10.1
ns2.dept.exemple.cat. IN    A     192.168.10.2
```

### Servidors Arrel i Cadena de Resolució

#### Els 13 Servidors Arrel

Els root servers són el punt inicial de tota resolució DNS.
Encara que hi ha 13 noms lògics (a.root-servers.net fins m.root-servers.net), existeixen més de 1000 rèpliques distribuïdes arreu del món (anycast) per millorar la disponibilitat.Els servidors arrel (de `a.root-servers.net` a `m.root-servers.net`) són el punt de partida de tota resolució DNS.

```
Servidors Arrel (13 lletres):
a.root-servers.net → 198.41.0.4
b.root-servers.net → 199.9.14.201
c.root-servers.net → 192.33.4.12
...
m.root-servers.net → 202.12.27.33
```

#### Cadena de Resolució

```
1. Client demana: www.exemple.cat
         ↓
2. Resolver consulta Root Server
         ↓
3. Root responde: "Pregunta al servidor .cat"
         ↓
4. Resolver consulta TLD server (.cat)
         ↓
5. TLD respon: "Pregunta al servidor de exemple.cat"
         ↓
6. Resolver consulta Authoritative server
         ↓
7. Authoritative respon: "www.example.com = 174.18.120.12"
         ↓
8. Client rep la IP i connecta
```
