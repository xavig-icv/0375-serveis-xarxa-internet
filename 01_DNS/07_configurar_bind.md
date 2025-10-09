## 07. Configuració Inicial de BIND

El fitxer `/etc/bind/named.conf.options` defineix la configuració global del servidor BIND, que afecta com respon a les consultes DNS, quines IPs o subxarxes poden utilitzar-lo i si pot resoldre noms externs (amb recursivitat).

**Fitxer: `/etc/bind/named.conf.options`**

```
acl "xarxa-interna" {
    192.168.1.0/24;
    localhost;
};

options {
    directory "/var/cache/bind";

    // Escoltar per la interfície interna i no IPv6.
    listen-on { 192.168.1.2; };
    listen-on-v6 { none; };

    // Permetre consultes només des de IPs de la LAN.
    allow-query { localhost; 192.168.1.0/24; };

    // Recursió habilitada: permet al servidor respondre consultes per noms que no coneix directament
    recursion yes;
    allow-recursion { xarxa-interna; };


    // Forwarders per consultes externes: si el servidor no té la resposta, la reenviarà a aquests DNS
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // Altres opcions
    dnssec-validation auto;
    auth-nxdomain no;
};
```

**Explicació dels paràmetres:**

- `directory`: Directori de BIND on s'emmagatzemen fitxers de treball i cache.
- `listen-on`: Indica per quines interfícies escolta el servidor per rebre consultes. En el nostre cas només la LAN interna.
- `allow-query`: Indica quines IPs poden fer consultes al servidor. Limitar-ho evita que qualsevol host extern faci servir el DNS.
- `recursion yes`: Permet al servidor resoldre noms externs que no coneix directament.
- `allow-recursion`: Només les IPs de la ACL podran fer consultes recursives, millorant la seguretat.
- `forwarders`: Són els servidors externs als que es reenvien consultes que el servidor no pot resoldre.
- `dnssec-validation`: Activa la validació de signatures DNSSEC per verificar l’autenticitat de les respostes.
- `auth-nxdomain no`: Evita que BIND es consideri autoritat per dominis inexistents.

### Verificació del Funcionament

#### Comprovar sintaxis de configuració

```bash
# Verificar named.conf
sudo named-checkconf

# Si no retorna cap error, la sintaxis és correcta
```

#### Verificar logs

```bash
# Veure logs del servei
sudo journalctl -u bind -f

# O directament:
sudo tail -f /var/log/syslog | grep named
```

#### Provar resolució DNS des del servidor

```bash
# Consultar el propi servidor DNS
dig @localhost google.com

# Verificar que respon el DNS
nslookup google.com localhost
```

#### Verificar el funcionament de la caché

```bash
# Primera consulta (fora de cache)
dig @localhost steampowered.com
# Segona consulta (després de cache)
dig @localhost steampowered.com
```

#### Verificació amb el client Ubuntu

Comprovar que les màquines de la teva xarxa interna poden fer consultes:

```
dig @192.168.1.2 google.com
```

**Configurar el resolver del client per l'ús per defecte d'aquest servidor DNS:**

```bash
# Editar /etc/resolv.conf
nameserver 192.168.1.2
```

**Provar el funcionament:**

```bash
# Des del client
dig google.com
nslookup facebook.com
```
