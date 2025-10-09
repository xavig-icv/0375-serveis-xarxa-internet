## 10. Verificació de les Zones

```bash
# Verificar configuració general (errors de sintaxi)
sudo named-checkconf

# Verificar la sintaxi del fitxer de zona directa
sudo named-checkzone xavig.cat /etc/bind/zones/db.xavig.cat

# Verificar la sintaxi de la zona inversa
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

# Recarregar zones (sense reiniciar servei)
sudo rndc reload

# Reiniciar el servei (alternativament)
sudo systemctl restart bind9
```

**Sortida correcta per directa i inversa:**

```
zone xavig.cat/IN: loaded serial 2025101001
OK
```

### Verificació al client

```bash
# Editar /etc/resolv.conf
nameserver 192.168.1.2
search xavig.cat
```

**Resolució directa**

```bash
nslookup ns1.xavig.cat
nslookup ns2.xavig.cat
nslookup www.xavig.cat
```

**Resolució inversa**

```bash
nslookup 192.168.1.2
nslookup 192.168.1.1
nslookup 192.168.1.10
```
