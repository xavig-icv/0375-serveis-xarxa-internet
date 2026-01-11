## 15. Certificats Digitals i SSL/TLS

`NOTA`: Només fer aquest procés amb dominis reals i que en sigueu propietaris.

Els certificats digitals emesos per una Autoritat Certificadora (CA) permeten establir connexions segures mitjançant SSL/TLS, garantint:

- Confidencialitat (xifrat de dades)
- Integritat (no permet la manipulació de la informació)
- Autenticació (validen la identitat del servidor)

### Obtenció de Certificats

Hi ha diverses opcions per obtenir certificats SSL/TLS:

1. **Let's Encrypt:** Gratuït, automatitzat i vàlid durant 90 dies. El recomano.
2. **Certificats comercials:** El generen CA's com Comodo, DigiCert, etc. Els demanen la majoria d'empreses i disposen normalment d'una valdiesa d'1 any.
3. **Certificats autosignats:** Només s'ha d'utilitzar per desenvolupament i per realitzar proves amb aplicacions internes.

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
sudo certbot --apache -d exemple1.cat -d www.exemple1.cat

# Només s'obté el certificat (s'ha de fer la configuració manual)
sudo certbot certonly --apache -d exemple1.cat -d www.exemple1.cat
```

**Obtenir certificat (Nginx):**

```bash
# Automàtic
sudo certbot --nginx -d exemple1.cat -d www.exemple1.cat

# Manual
sudo certbot certonly --nginx -d exemple1.cat -d www.exemple1.cat
```

**Obtenir certificat (mode standalone amb el servidor aturat):**

```bash
sudo systemctl stop apache2  # o nginx
sudo certbot certonly --standalone -d exemple1.cat -d www.exemple1.cat
sudo systemctl start apache2
```

**Renovació automàtica:**

```bash
# Certbot crea un cron automàticament
# Verificar el temporitzador:
sudo systemctl status certbot.timer

# Simular la renovació
sudo certbot renew --dry-run

# Renovar-ho manualment
sudo certbot renew
```

**Ubicació dels certificats:**

```
/etc/letsencrypt/live/exemple1.cat/
├── cert.pem          # Certificat del lloc
├── chain.pem         # Cadena de certificació
├── fullchain.pem     # cert.pem + chain.pem
└── privy.pem         # Clau privada
```

### Configurar SSL/TLS amb Apache

**Activar mòdul SSL:**

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

**Virtual Host amb HTTPS:**

```apache
<VirtualHost *:443>
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat

    DocumentRoot /var/www/exemple1.cat/html

    # SSL
    SSLEngine on

    # Certificats Let's Encrypt
    SSLCertificateFile /etc/letsencrypt/live/exemple1.cat/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/exemple1.cat/privkey.pem

    # Protocols segurs (només TLS 1.2 i 1.3)
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

    # Xifratge modern i segur
    SSLCipherSuite HIGH:!aNULL:!MD5:!3DES
    SSLHonorCipherOrder on

    # HSTS (per forçar l'ús de HTTPS)
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    <Directory /var/www/exemple1.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/exemple1.cat-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/exemple1.cat-ssl-access.log combined
</VirtualHost>

# Redirigir HTTP a HTTPS
<VirtualHost *:80>
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat

    Redirect permanent / https://exemple1.cat/
</VirtualHost>
```

### Configurar SSL/TLS amb Nginx

**Virtual Host amb HTTPS:**

```nginx
# Redirigir HTTP a HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name exemple1.cat www.exemple1.cat;

    return 301 https://$server_name$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name exemple1.cat www.exemple1.cat;
    root /var/www/exemple1.cat/html;
    index index.html index.php;

    # Certificats Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/exemple1.cat/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/exemple1.cat/privkey.pem;

    # Protocols segurs
    ssl_protocols TLSv1.2 TLSv1.3;

    # Xifratge modern
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    # Cache de les sessions
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/exemple1.cat/chain.pem;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/exemple1.cat-ssl-access.log;
    error_log /var/log/nginx/exemple1.cat-ssl-error.log;
}
```

### Test de Seguretat SSL

**SSL Labs (Test online):**

```
https://www.ssllabs.com/ssltest/analyze.html?d=exemple1.cat
```

**Amb openssl (En local):**

```bash
# Comprovar certificat i negociació TLS
openssl s_client -connect exemple1.cat:443 -servername exemple1.cat

# Enumerar (Verificar protocols i algorismes de xifrat)
nmap --script ssl-enum-ciphers -p 443 exemple1.cat

# Provar connexió TLS 1.2
openssl s_client -connect exemple1.cat:443 -tls1_2

# Provar connexió TLS 1.3
openssl s_client -connect exemple1.cat:443 -tls1_3
```
