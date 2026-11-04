## 12. Certificats Autosignats (Només per fer Proves)

Els certificats autosignats s'utilitzen exclusivament en entorns de proves o desenvolupament. No estan signats per cap Autoritat de Certificació (CA) reconeguda, per tant els navegadors no els consideren de confiança i apareixerà un missatge d'advertència cada cop que volem accedir al lloc web.

És el tipus de certificat que farem servir al laboratori: el domini `educem.cat` no és un domini registrat, i una CA pública (com Let's Encrypt) no ens en podria emetre cap.

### Generar un certificat SSL autosignat:

La carpeta `/etc/ssl/private` ja existeix a Ubuntu i només hi té accés root (i el grup `ssl-cert`).

```bash
# Certificat i clau privada per web1.educem.cat (vàlid per 365 dies)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/web1.educem.cat.key \
    -out /etc/ssl/certs/web1.educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=web1.educem.cat" \
    -addext "subjectAltName=DNS:web1.educem.cat"

# Certificat i clau privada per web2.educem.cat
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/web2.educem.cat.key \
    -out /etc/ssl/certs/web2.educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=web2.educem.cat" \
    -addext "subjectAltName=DNS:web2.educem.cat"
```

**Paràmetres:**

- `-x509`: genera directament un certificat (autosignat), no una sol·licitud per a una CA.
- `-nodes`: la clau privada no es xifra amb contrasenya. Si es xifrés, el servei la demanaria cada vegada que arrenca.
- `-days 365`: validesa del certificat.
- `-newkey rsa:2048`: genera una clau RSA nova de 2048 bits.
- `-subj`: les dades del certificat. Si no s'indica, `openssl` les pregunta una per una:

```
# Country Name: ES
# State: Catalunya
# Locality: Barcelona
# Organization: EDUCEM
# Common Name: web1.educem.cat  ← IMPORTANT QUE COINCIDEIXI AMB EL DOMINI
# Email: admin@educem.cat
```

- `-addext "subjectAltName=..."`: el **SAN** (Subject Alternative Name). Els navegadors actuals comproven el nom del lloc al SAN, no al Common Name. Si el certificat no té SAN, el navegador afegeix un error de nom a l'avís de certificat no fiable.

### Comprovar el certificat

```bash
# Nom, SAN i dates de validesa
openssl x509 -in /etc/ssl/certs/web1.educem.cat.crt -noout -subject -ext subjectAltName -dates

# Permisos de la clau privada (només root)
sudo ls -l /etc/ssl/private/web1.educem.cat.key
```

### Configurar SSL/TLS a Apache i Nginx

```bash
# Activar el mòdul SSL d'Apache
sudo a2enmod ssl
sudo systemctl restart apache2
```

La configuració dels Virtual Hosts amb aquests certificats és a l'apartat 11:

- Apache: `web1.educem.cat`, port 443.
- Nginx: `web2.educem.cat`, port 8443.

### Avís del navegador

Els certificats autosignats **sempre** mostraran avís de seguretat al navegador (ja que no són de confiança):

- **Firefox:** "Avançat" → "Accepta el risc i continua".
- **Chrome:** "Configuració avançada" → "Continua a ... (no segur)".

Amb `curl`, cal l'opció `-k` (no verificar el certificat), o bé indicar-li que confiï en aquest certificat concret:

```bash
curl -k https://web1.educem.cat
curl --cacert web1.educem.cat.crt https://web1.educem.cat   # amb una còpia del certificat al client
```
