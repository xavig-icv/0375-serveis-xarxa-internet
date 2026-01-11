## 12. Virtual Host amb SSL/TLS

El protocol SSL/TLS permet xifrar la comunicació entre el client i el servidor web, garantint confidencialitat i integritat de la informació.

En aquesta secció mostrem una de les opcions de configuració d'un Virtual Host amb redirecció d'HTTP a HTTPS tant en Apache com en Nginx.

**Configuració amb Apache:**

- Cal tenir el mòdul SSL activat (mod_ssl)
- El port 443 s'utilitza exclusivament per HTTPS
- És necessari de disposar d'un certificat d'una CA.
  - A la pràctica haurem de crear un certificat autosignat i un de Let's Encrypt

```bash
sudo nano /etc/apache2/sites-available/exemple1.cat.conf
```

```apache
<VirtualHost *:80>
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat
    # Redirigir HTTP a HTTPS
    Redirect permanent / https://exemple1.cat/
</VirtualHost>
<VirtualHost *:443>
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat
    DocumentRoot /var/www/exemple1.cat/html

    # Activar SSL
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/exemple1.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/exemple1.cat.key
    SSLCertificateChainFile /etc/ssl/certs/ca-bundle.crt

    <Directory /var/www/exemple1.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/exemple1.cat-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/exemple1.cat-ssl-access.log combined
</VirtualHost>
```

**Configuració amb Nginx:**

- Nginx no utilitza .htaccess
- Els server block son l'equivalent als Virtual Hosts d’Apache
- HTTP/2 millora el rendiment amb HTTPS

```bash
sudo nano /etc/nginx/sites-available/exemple1.cat
```

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

    # Certificats SSL
    ssl_certificate /etc/ssl/certs/exemple1.cat.crt;
    ssl_certificate_key /etc/ssl/private/exemple1.cat.key;

    # Configuració SSL moderna (protocols TLS i algorismes de xifrat)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/exemple1.cat-ssl-access.log;
    error_log /var/log/nginx/exemple1.cat-ssl-error.log;
}
```
