## 07. Configuració Inicial de BIND

El fitxer `/etc/bind/named.conf.options` defineix la configuració global del servidor BIND, que afecta com respon a les consultes DNS, quines IPs o subxarxes poden utilitzar-lo i si pot resoldre noms externs (amb recursivitat).

**Fitxer: `/etc/bind/named.conf.options`** (servidor primari 0375, `192.168.1.2`)

```
acl "xarxa-interna" {
    192.168.1.0/24;
    localhost;
};

options {
    directory "/var/cache/bind";

    // Escoltar per localhost i per la interfície interna. No escoltar per IPv6.
    listen-on { 127.0.0.1; 192.168.1.2; };
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

- `directory`: Directori de BIND on s'emmagatzemen fitxers de treball i la memòria cau.
- `listen-on`: Indica per quines adreces escolta el servidor per rebre consultes:
  - la IP de la xarxa interna (`192.168.1.2`), per als clients;
  - `127.0.0.1`, perquè el mateix servidor pugui fer servir el seu DNS.
- `allow-query`: Indica quines IPs poden fer consultes al servidor. Limitar-ho evita que qualsevol host extern faci servir el DNS.
- `recursion yes`: Permet al servidor resoldre noms externs que no coneix directament.
- `allow-recursion`: Només les IPs de la ACL podran fer consultes recursives, millorant la seguretat.
- `forwarders`: Són els servidors externs als que es reenvien consultes que el servidor no pot resoldre.
- `dnssec-validation`: Activa la validació de signatures DNSSEC per verificar l'autenticitat de les respostes.
- `auth-nxdomain no`: Evita que BIND es consideri autoritat per dominis inexistents.

`Nota:` Si la xarxa del centre no permet consultes DNS cap a servidors externs, feu servir com a forwarder el DNS que rep l'adaptador pont. El podeu consultar amb `resolvectl dns enp0s3`.

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
sudo journalctl -u named -f

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

#### Verificar el funcionament de la memòria cau

```bash
# Primera consulta (fora de la memòria cau)
dig @localhost steampowered.com
# Segona consulta (resposta des de la memòria cau)
dig @localhost steampowered.com
```

### Configuració de Xarxa amb Netplan

A Ubuntu, el fitxer `/etc/resolv.conf` el gestiona **systemd-resolved**. El fitxer apunta a `127.0.0.53` (el resolver local) i es regenera sol, de manera que, si l'editem a mà, els canvis es perden. Els servidors DNS de cada equip es configuren amb **netplan**.

L'escenari complet és a [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md).

#### Servidor 0375: fer servir el seu propi BIND

```bash
# El nom del fitxer pot variar (a Ubuntu Server sol ser 50-cloud-init.yaml)
ls /etc/netplan/
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                       # adaptador pont (xarxa del centre)
      dhcp4: true
    enp0s8:                       # xarxa interna
      addresses: [192.168.1.2/24]
      nameservers:
        addresses: [127.0.0.1]    # el BIND del mateix servidor
        search: [educem.cat]
```

```bash
sudo netplan apply

# Comprovar el DNS i el domini de cerca de cada interfície
resolvectl dns          # enp0s8: 127.0.0.1
resolvectl domain       # enp0s8: educem.cat
```

Amb `search: [educem.cat]`, systemd-resolved envia les consultes dels noms d'`educem.cat` només al BIND local. La resta de consultes poden anar també al DNS del centre (adaptador pont).

#### Client Ubuntu (IP fixa durant el Tema 1)

El client d'escriptori fa servir NetworkManager com a gestor de xarxa:

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
```

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:                       # xarxa interna
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1        # servidor 0374 (gateway amb NAT)
      nameservers:
        addresses: [192.168.1.2, 192.168.1.1]
        search: [educem.cat]
```

```bash
# netplan avisa si el fitxer és llegible per tothom
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml
sudo netplan apply

# Comprovar els DNS que fa servir el client
resolvectl dns
```

`Nota:` A partir del Tema 2, el client rebrà aquesta configuració per DHCP (`dhcp4: true`).

#### Verificació amb el client Ubuntu

Comprovar que les màquines de la xarxa interna poden fer consultes:

```bash
# Consultant directament el servidor
dig @192.168.1.2 google.com

# Amb el DNS configurat per defecte
dig google.com
nslookup facebook.com
```
