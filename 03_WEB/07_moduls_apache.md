## 07. Mòduls d'Apache

Apache és un servidor modular (com un transformer): Els mòduls es poden afegir a la funcionalitat principal i es poden activar o desactivar segons les necessitats.

### Gestió de Mòduls

```bash
# Llistar els mòduls disponibles
ls /etc/apache2/mods-available/

# Llistar els mòduls actius
ls /etc/apache2/mods-enabled/

# Mostrar mòduls carregats (recomanat)
sudo apache2ctl -M

# Activar un mòdul
sudo a2enmod nom_modul

# Desactivar un mòdul
sudo a2dismod nom_modul

# Reiniciar Apache després de fer els canvis
sudo systemctl restart apache2
```

### Mòduls Essencials

#### mod_rewrite - Reescriptura d'URLs

```bash
# Activar
sudo a2enmod rewrite
sudo systemctl restart apache2
```

**Exemple d'ús (`.htaccess` o VirtualHost):**

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Redirigir http://www a http://
    RewriteCond %{HTTP_HOST} ^www\.(.*)$ [NC]
    RewriteRule ^(.*)$ http://%1/$1 [R=301,L]

    # Forçar HTTPS
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

    # URLs amigables
    RewriteRule ^producte/([0-9]+)$ producte.php?id=$1 [L]
    RewriteRule ^categoria/([a-z]+)$ categoria.php?nom=$1 [L]
</IfModule>
```

#### mod_ssl - Suport SSL/TLS

```bash
# Activar
sudo a2enmod ssl
sudo systemctl restart apache2
```

**Configuració:**

```apache
<VirtualHost *:443>
    ServerName exemple.cat

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/exemple.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/exemple.cat.key
    SSLCertificateChainFile /etc/ssl/certs/ca-bundle.crt

    # Protocols i cifratge segurs
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5:!3DES
    SSLHonorCipherOrder on
</VirtualHost>
```

#### mod_headers - Modificació de headers HTTP

```bash
# Activar
sudo a2enmod headers
sudo systemctl restart apache2
```

**Configuració:**

```apache
<IfModule mod_headers.c>
    # Headers de seguretat
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # HSTS (Strict-Transport-Security)
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    # CSP (Content-Security-Policy)
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'"

    # Eliminar header de versió
    Header unset Server
    Header unset X-Powered-By

    # Cache per recursos estàtics
    <FilesMatch "\.(jpg|jpeg|png|gif|css|js)$">
        Header set Cache-Control "max-age=31536000, public"
    </FilesMatch>
</IfModule>
```

#### mod_deflate - Compressió GZIP

```bash
# Activar
sudo a2enmod deflate
sudo systemctl restart apache2
```

**Configuració:**

```apache
<IfModule mod_deflate.c>
    # Comprimir tipus de contingut
    AddOutputFilterByType DEFLATE text/html text/plain text/xml
    AddOutputFilterByType DEFLATE text/css
    AddOutputFilterByType DEFLATE application/javascript
    AddOutputFilterByType DEFLATE application/json
    AddOutputFilterByType DEFLATE application/xml

    # No comprimir imatges ja comprimides
    SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png)$ no-gzip

    # Navegadors antics sense suport GZIP
    BrowserMatch ^Mozilla/4 gzip-only-text/html
    BrowserMatch ^Mozilla/4\.0[678] no-gzip
    BrowserMatch \bMSIE !no-gzip !gzip-only-text/html
</IfModule>
```

#### mod_expires - Control de cache

```bash
# Activar
sudo a2enmod expires
sudo systemctl restart apache2
```

**Configuració:**

```apache
<IfModule mod_expires.c>
    ExpiresActive On

    # Per defecte: 1 mes
    ExpiresDefault "access plus 1 month"

    # HTML: 1 hora
    ExpiresByType text/html "access plus 1 hour"

    # CSS i JavaScript: 1 mes
    ExpiresByType text/css "access plus 1 month"
    ExpiresByType application/javascript "access plus 1 month"

    # Imatges: 1 any
    ExpiresByType image/jpg "access plus 1 year"
    ExpiresByType image/jpeg "access plus 1 year"
    ExpiresByType image/gif "access plus 1 year"
    ExpiresByType image/png "access plus 1 year"
    ExpiresByType image/svg+xml "access plus 1 year"

    # Fonts: 1 any
    ExpiresByType font/woff2 "access plus 1 year"
    ExpiresByType application/font-woff "access plus 1 year"
</IfModule>
```

#### mod_php - Integració de PHP

```bash
# Instal·lar PHP i el mòdul php d'Apache
sudo apt install php libapache2-mod-php

# El mòdul s'activa automàticament
# Reiniciar l'Apache
sudo systemctl restart apache2
```

**Configuració:**

```apache
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>

# Seguretat: No mostrar fitxers .phps
<FilesMatch "\.ph(p[3-5]?|tml)$">
    SetHandler application/x-httpd-php
</FilesMatch>
<FilesMatch "\.phps$">
    Require all denied
</FilesMatch>

# Index PHP per defecte
DirectoryIndex index.php index.html
```

#### mod_security - WAF (Web Application Firewall)

```bash
# Instal·lar
sudo apt install libapache2-mod-security2

# Activar
sudo a2enmod security2
sudo systemctl restart apache2

# Configurar
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
sudo nano /etc/modsecurity/modsecurity.conf
```

**Configuració bàsica:**

```apache
# Canviar de DetectionOnly a On
SecRuleEngine On

# Activar OWASP Core Rule Set
sudo git clone https://github.com/coreruleset/coreruleset /etc/modsecurity/crs
sudo cp /etc/modsecurity/crs/crs-setup.conf.example /etc/modsecurity/crs/crs-setup.conf
```

**Incloure a Apache:**

```apache
<IfModule security2_module>
    Include /etc/modsecurity/crs/crs-setup.conf
    Include /etc/modsecurity/crs/rules/*.conf
</IfModule>
```
