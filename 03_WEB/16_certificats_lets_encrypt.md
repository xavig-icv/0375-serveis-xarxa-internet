## 16. Certificats Digitals i SSL/TLS

`NOTA`: Només fer aquest procés amb dominis reals i que en sigueu propietaris.

**Aquest apartat no es pot fer al laboratori**, per dues raons:

- `educem.cat` no és un domini registrat.
- Els servidors no són accessibles des d'Internet. Let's Encrypt ha de poder arribar al servidor pel port 80 (validació HTTP-01) o ha de poder consultar un registre al DNS públic del domini (validació DNS-01).

Als exemples, `example.com` (domini reservat per a documentació) representa el vostre domini real.

Els certificats digitals emesos per una Autoritat Certificadora (CA) permeten establir connexions segures mitjançant SSL/TLS, garantint:

- Confidencialitat (xifrat de dades)
- Integritat (no permet la manipulació de la informació)
- Autenticació (validen la identitat del servidor)

### Obtenció de Certificats

Hi ha diverses opcions per obtenir certificats SSL/TLS:

1. **Let's Encrypt:** Gratuït, automatitzat i vàlid durant 90 dies. El recomano.
2. **Certificats comercials:** Els generen CA com DigiCert, Sectigo, etc. Els demanen la majoria d'empreses.
   - Des del 15 de març de 2026, la validesa màxima d'un certificat públic és de **200 dies**.
   - Baixarà a 100 dies el 2027 i a 47 dies el 2029 (acord del CA/Browser Forum).
   - Per això la renovació automàtica és cada vegada més important.
3. **Certificats autosignats:** Només s'ha d'utilitzar per desenvolupament i per realitzar proves amb aplicacions internes (apartat 12).

### Let's Encrypt amb Certbot

Certbot és l'eina oficial per gestionar certificats de Let's Encrypt de forma automàtica.

**Instal·lació de Certbot:**

```bash
# Instal·lar el programari
sudo apt update
sudo apt install certbot

# Integració amb Apache
sudo apt install python3-certbot-apache

# Integració amb Nginx
sudo apt install python3-certbot-nginx
```

**Obtenir certificat (Apache):**

```bash
# Automàtic (configura Apache automàticament i les redireccions)
sudo certbot --apache -d example.com -d www.example.com

# Només s'obté el certificat (s'ha de fer la configuració manual)
sudo certbot certonly --apache -d example.com -d www.example.com
```

**Obtenir certificat (Nginx):**

```bash
# Automàtic
sudo certbot --nginx -d example.com -d www.example.com

# Manual
sudo certbot certonly --nginx -d example.com -d www.example.com
```

**Obtenir certificat (mode standalone amb el servidor aturat):**

```bash
sudo systemctl stop apache2  # o nginx
sudo certbot certonly --standalone -d example.com -d www.example.com
sudo systemctl start apache2
```

**Renovació automàtica:**

```bash
# Certbot instal·la un temporitzador de systemd que renova els certificats automàticament
# Verificar el temporitzador:
sudo systemctl status certbot.timer

# Simular la renovació
sudo certbot renew --dry-run

# Renovar-ho manualment
sudo certbot renew
```

**Ubicació dels certificats:**

```
/etc/letsencrypt/live/example.com/
├── cert.pem          # Certificat del lloc
├── chain.pem         # Cadena de certificació
├── fullchain.pem     # cert.pem + chain.pem
└── privkey.pem       # Clau privada
```

### Configurar SSL/TLS amb Apache

**Activar els mòduls SSL i headers:**

```bash
sudo a2enmod ssl headers
sudo systemctl restart apache2
```

La capçalera HSTS de l'exemple fa servir la directiva `Header`, del mòdul `mod_headers`. Si el mòdul no està activat, Apache no arrenca (`Invalid command 'Header'`).

**Virtual Host amb HTTPS:**

```apache
<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com

    DocumentRoot /var/www/example.com/html

    # SSL
    SSLEngine on

    # Certificats Let's Encrypt
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    # Protocols segurs (només TLS 1.2 i 1.3)
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

    # Xifratge modern i segur
    SSLCipherSuite HIGH:!aNULL:!MD5:!3DES
    SSLHonorCipherOrder on

    # HSTS (per forçar l'ús de HTTPS)
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    <Directory /var/www/example.com/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/example.com-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-ssl-access.log combined
</VirtualHost>

# Redirigir HTTP a HTTPS
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com

    Redirect permanent / https://example.com/
</VirtualHost>
```

### Configurar SSL/TLS amb Nginx

**Virtual Host amb HTTPS:**

```nginx
# Redirigir HTTP a HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    return 301 https://$server_name$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name example.com www.example.com;
    root /var/www/example.com/html;
    index index.html index.php;

    # Certificats Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Protocols segurs
    ssl_protocols TLSv1.2 TLSv1.3;

    # Xifratge modern
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    # Memòria cau de les sessions
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/example.com-ssl-access.log;
    error_log /var/log/nginx/example.com-ssl-error.log;
}
```

`Nota:` Let's Encrypt va deixar d'oferir el servei OCSP l'any 2025. Per això ja no cal configurar l'OCSP stapling (`ssl_stapling`) amb els seus certificats.

### Test de Seguretat SSL

**SSL Labs (Test online):**

```
https://www.ssllabs.com/ssltest/analyze.html?d=example.com
```

**Amb openssl (En local):**

```bash
# Comprovar certificat i negociació TLS
openssl s_client -connect example.com:443 -servername example.com

# Enumerar (Verificar protocols i algorismes de xifrat)
nmap --script ssl-enum-ciphers -p 443 example.com

# Provar connexió TLS 1.2
openssl s_client -connect example.com:443 -tls1_2

# Provar connexió TLS 1.3
openssl s_client -connect example.com:443 -tls1_3
```

Les ordres amb `openssl` i `nmap` també es poden provar al laboratori amb els certificats autosignats, per exemple:

```bash
openssl s_client -connect web1.educem.cat:443 -servername web1.educem.cat
```
