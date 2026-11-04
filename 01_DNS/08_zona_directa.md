## 08. Configuració de la Zona Directa (Forward Zone)

Una Zona Directa és la part de l'espai de noms d'un domini que un servidor DNS autoritatiu administra i que conté els registres per resoldre noms de host cap a adreces IP. Es diu "directa" perquè converteix noms a IPs (resolució de tipus "forward").

Cada zona té un fitxer de zona associat, on es defineixen els registres SOA, NS, A, MX, CNAME, etc. El servidor primari (master) és el responsable de tenir la versió oficial dels registres. Els servidors secundaris (slaves) poden tenir còpies per redundància (per si falla el servidor primari).

Al laboratori, el domini és **`educem.cat`** (vegeu l'[escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md)):

- el servidor 0375 (`192.168.1.2`) és el primari;
- el servidor 0374 (`192.168.1.1`) és el secundari.

**Fitxer: `/etc/bind/named.conf.local`**

```
// Zona primària per educem.cat
zone "educem.cat" {
    type master;
    file "/etc/bind/zones/db.educem.cat";
    allow-transfer { 192.168.1.1; };  // IP del servidor secundari (0374)
    also-notify { 192.168.1.1; };
};
```

#### Creació del fitxer de zona

La carpeta `zones/` no existeix per defecte; la creem:

```bash
sudo mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/db.educem.cat
```

**Fitxer: `/etc/bind/zones/db.educem.cat`**

```
$TTL 86400
@   IN  SOA ns1.educem.cat. admin.educem.cat. (
            2026110401  ; Serial (AAAAMMDDnn)
            3600        ; Refresh (1 hora)
            1800        ; Retry (30 min)
            604800      ; Expire (1 setmana)
            300 )       ; Negative Cache TTL (5 min)

; Name Servers
@       IN  NS      ns1.educem.cat.
@       IN  NS      ns2.educem.cat.

; Mail Servers
@       IN  MX  10  mail.educem.cat.

; A Records (IPv4)
ns1         IN  A       192.168.1.2     ; servidor 0375 (primari)
ns2         IN  A       192.168.1.1     ; servidor 0374 (secundari i gateway)
www         IN  A       192.168.1.2
mail        IN  A       192.168.1.2
ftp         IN  A       192.168.1.2
client      IN  A       192.168.1.50    ; client Ubuntu

; Àlies (CNAME)
web         IN  CNAME   www
```

**Explicació del registre SOA:**

- `ns1.educem.cat.`: Servidor DNS primari.
- `admin.educem.cat.`: Correu electrònic de l'administrador (`admin@educem.cat`). El primer punt substitueix l'`@`.
- `Serial`: Número de versió de la zona. **Cal incrementar-lo a cada canvi**, perquè el secundari només copia la zona si el serial és més gran que el que ja té.
- `Refresh`: Cada quan el secundari comprova si hi ha canvis.
- `Retry`: Temps d'espera si falla el refresh.
- `Expire`: Temps a partir del qual el secundari deixa de respondre si no pot contactar amb el primari.
- `Negative Cache TTL`: Temps que els altres servidors DNS guarden les respostes negatives (quan el nom no existeix).
  - Al laboratori fem servir 5 minuts.
  - Així, si un client pregunta per un nom abans de crear-lo, no haurà d'esperar hores per veure'l un cop creat.

**Notes:**

- Tots els serveis del mòdul (web, correu i FTP) són al servidor 0375. Per això `www`, `mail` i `ftp` apunten a `192.168.1.2`: un mateix equip pot tenir tants noms com calgui.
- Els noms acabats en punt (`ns1.educem.cat.`) són FQDN. Si no hi ha el punt final, BIND hi afegeix el nom de la zona: `ns1` es converteix en `ns1.educem.cat.`.
- Abans d'afegir un registre, comproveu que no existeix. Si hi ha dos registres A amb el mateix nom, el DNS retorna les dues IP i les connexions van alternativament a l'una o a l'altra.

### Diversos Dominis al Mateix Servidor

Un mateix servidor BIND pot ser autoritatiu de tants dominis com calgui. Cada domini és una zona més a `named.conf.local`, amb el seu fitxer de zona i, si cal, la seva zona inversa:

```
zone "altredomini.test" {
    type master;
    file "/etc/bind/zones/db.altredomini.test";
};
```

- Si el servidor té adreces en una altra xarxa, cal afegir-les a `listen-on` i a les xarxes permeses: `allow-query` i l'ACL de `named.conf.options`.
- Feu servir dominis que no existeixin a Internet, per exemple acabats en `.test`, que l'RFC 6761 reserva per a proves.
