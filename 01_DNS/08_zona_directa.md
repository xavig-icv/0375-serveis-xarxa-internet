## 08. Configuració de la Zona Directa (Forward Zone)

Una Zona Directa és la part de l’espai de noms d’un domini que un servidor DNS autoritatiu administra i que conté els registres per resoldre noms de host cap a adreces IP. És diu "directa" perquè converteix noms a IPs (resolució de tipus "forward").

Cada zona té un fitxer de zona associat, on es defineixen els registres SOA, NS, A, MX, CNAME, etc. El servidor primari (master) és el responsable de tenir la versió oficial dels registres. Els servidors secundaris (slaves) poden tenir còpies per redundància (per si falla el servidor primari).

**Fitxer: `/etc/bind/named.conf.local`**

```
// Zona primària per xavig.cat
zone "xavig.cat" {
    type master;
    file "/etc/bind/zones/db.xavig.cat";
    allow-transfer { 192.168.1.1; };  // IP del servidor secundari
    also-notify { 192.168.1.1; };
};
```

#### Creació del fitxer de zona

**Fitxer: `/etc/bind/zones/db.xavig.cat`**

```
$TTL 86400
@   IN  SOA ns1.xavig.cat. admin.xavig.cat. (
            2025101001  ; Serial (YYYYMMDDnn)
            3600        ; Refresh (1 hora)
            1800        ; Retry (30 min)
            604800      ; Expire (1 setmana)
            86400 )     ; Negative Cache TTL (1 dia)

; Name Servers
@       IN  NS      ns1.xavig.cat.
@       IN  NS      ns2.xavig.cat.

; Mail Servers
@       IN  MX  10  mail.xavig.cat.

; A Records (IPv4)
ns1         IN  A       192.168.1.2
ns2         IN  A       192.168.1.1
www         IN  A       192.168.1.10
mail        IN  A       192.168.1.11
ftp         IN  A       192.168.1.12
proxy       IN  A       192.168.1.13
bbdd        IN  A       192.168.1.14

; Àlies (CNAME)
web         IN  CNAME   www
```

**Explicació del registre SOA:**

- `ns1.xavig.com.`: Servidor DNS primari
- `admin.xavig.com.`: Email de l'administrador (admin@xavig.com)
- `Serial`: Número de versió (incrementar amb cada canvi)
- `Refresh`: Cada quan el secundari comprova canvis
- `Retry`: Temps d'espera si falla el refresh
- `Expire`: Quan el secundari deixa de respondre si no pot contactar primari
- `Negative Cache TTL`: Temps de cache per respostes negatives
