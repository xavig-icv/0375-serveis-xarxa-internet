## 10. Verificació de les Zones

```bash
# Verificar configuració general (errors de sintaxi)
sudo named-checkconf

# Verificar la sintaxi del fitxer de zona directa
sudo named-checkzone educem.cat /etc/bind/zones/db.educem.cat

# Verificar la sintaxi de la zona inversa
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

# Recarregar zones (sense reiniciar servei)
sudo rndc reload

# Reiniciar el servei (alternativament)
sudo systemctl restart named
```

**Sortida correcta per directa i inversa:**

```
zone educem.cat/IN: loaded serial 2026110401
OK
```

### Verificació al servidor

```bash
dig @localhost www.educem.cat
dig @localhost MX educem.cat
dig @localhost -x 192.168.1.2
```

### Verificació al client

El client ha de tenir `192.168.1.2` com a servidor DNS i `educem.cat` com a domini de cerca. La configuració de netplan és a l'apartat 07.

```bash
resolvectl dns
resolvectl domain
```

**Resolució directa**

```bash
nslookup ns1.educem.cat
nslookup ns2.educem.cat
nslookup www.educem.cat
dig web.educem.cat      # CNAME cap a www
dig MX educem.cat
```

**Resolució inversa**

```bash
nslookup 192.168.1.2
nslookup 192.168.1.1
dig -x 192.168.1.50
```

`Nota:` Si un nom que acabeu d'afegir no resol al client, potser el client té guardada la resposta negativa a la memòria cau:

1. Buideu-la amb `resolvectl flush-caches`.
2. Comproveu que heu incrementat el serial.
3. Comproveu que heu recarregat la zona (`sudo rndc reload`).
