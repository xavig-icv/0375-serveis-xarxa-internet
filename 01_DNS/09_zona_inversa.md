## 09. Configuració de Zona Inversa (Reverse Zone)

Una Zona Inversa és la part de l'espai de noms que permet obtenir el nom d'host a partir d'una adreça IP (resolució de tipus reverse). Cada zona inversa disposa també d'un fitxer de zona, on es defineixen registres PTR que associen una IP a un nom de domini.

#### Definició en named.conf.local

```
// Zona inversa per 192.168.1.0/24
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
    allow-transfer { 192.168.1.1; };
};
```

#### Fitxer de zona inversa

**Fitxer: `/etc/bind/zones/db.192.168.1`**

```
$TTL 86400
@   IN  SOA ns1.xavig.cat. admin.xavig.cat. (
            2025101001  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400 )     ; Negative Cache TTL

; Name Servers
@       IN  NS      ns1.xavig.cat.
@       IN  NS      ns2.xavig.cat.

; PTR Records (Reverse DNS)
2      IN  PTR     ns1.xavig.cat.
1      IN  PTR     ns2.xavig.cat.
10     IN  PTR     www.xavig.cat.
11     IN  PTR     mail.xavig.cat.
12     IN  PTR     ftp.xavig.cat.
13     IN  PTR     proxy.xavig.cat.
14     IN  PTR     bbdd.xavig.cat.
```

**Notes:**

- El nom de la zona són 3 octets de la IP de xarxa invertida + `.in-addr.arpa`
- Els PTR només necessiten l'últim octet de la IP
