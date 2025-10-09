## 11. Configuració del Servidor Secundari

Un servidor DNS secundari (slave) és un servidor que manté còpies dels registres del servidor primari (master) per garantir redundància i alta disponibilitat. No és responsable de modificar els registres originals; només rep actualitzacions del primari. En cas que el servidor primari falli, els clients poden seguir consultant el secundari.

**![IMPORTANT] Al servidor secundari 192.168.1.1 (`/etc/bind/named.conf.local`):**

Al servidor secundari instal·lar BIND9 i editar named.conf.local

```
// Zona secundària (FORWARD)
zone "xavig.cat" {
    type slave;
    file "/var/cache/bind/db.xavig.cat";
    masters { 192.168.1.2; };  // IP del servidor primari
};

// Zona secundària (REVERSE)
zone "1.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192.168.1";
    masters { 192.168.1.2; };
};
```

**![IMPORTANT] Al servidor primari 192.168.1.2:** Assegurar que permet transferències:

```
zone "xavig.cat" {
    type master;
    file "/etc/bind/zones/db.xavig.cat";
    allow-transfer { 192.168.1.1; };
    also-notify { 192.168.1.1; };
};
```

### Verificacions

- sudo systemctl restart bind9 (Fer al servidor Primari i Secundari)
- Logs del Servidor Secondary /var/log/syslog (verificar transfer zone)
- Proveu-ho la resolució amb el Client Ubuntu i el Servidor Primari “caigut”.
