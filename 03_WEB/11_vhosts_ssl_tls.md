## 11. Virtual Host amb SSL/TLS

El protocol SSL/TLS permet xifrar la comunicació entre el client i el servidor web, garantint confidencialitat i integritat de la informació.

En aquesta secció mostrem una de les opcions de configuració d'un Virtual Host amb redirecció d'HTTP a HTTPS tant en Apache com en Nginx:

- **Apache:** `web1.educem.cat` als ports 80 (redirecció) i 443 (HTTPS).
- **Nginx:** `web2.educem.cat` als ports 8080 (redirecció) i 8443 (HTTPS).

`IMPORTANT:` Abans de configurar els Virtual Hosts, genereu els certificats autosignats (apartat 12). Si els fitxers del certificat no existeixen, Apache i Nginx no arrenquen. Els fitxers que farem servir són:

| Lloc              | Certificat                             | Clau privada                             |
| ----------------- | -------------------------------------- | ---------------------------------------- |
| `web1.educem.cat` | `/etc/ssl/certs/web1.educem.cat.crt`   | `/etc/ssl/private/web1.educem.cat.key`   |
| `web2.educem.cat` | `/etc/ssl/certs/web2.educem.cat.crt`   | `/etc/ssl/private/web2.educem.cat.key`   |

**Configuració amb Apache:**

- Cal tenir el mòdul SSL activat (mod_ssl)
- El port 443 s'utilitza exclusivament per HTTPS
- És necessari disposar d'un certificat: al laboratori, un d'autosignat (apartat 12). Amb un domini real, un d'una CA (apartat 16).

```bash
# Activar el mòdul SSL (habilita també el port 443 a ports.conf)
sudo a2enmod ssl

# Editar el Virtual Host de web1 (el de l'apartat 09)
sudo nano /etc/apache2/sites-available/web1.educem.cat.conf
```

```apache
<VirtualHost *:80>
    ServerName web1.educem.cat
    # Redirigir HTTP a HTTPS
    Redirect permanent / https://web1.educem.cat/
</VirtualHost>
<VirtualHost *:443>
    ServerName web1.educem.cat
    DocumentRoot /var/www/web1.educem.cat/html

    # Activar SSL
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/web1.educem.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/web1.educem.cat.key

    <Directory /var/www/web1.educem.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/web1.educem.cat-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/web1.educem.cat-ssl-access.log combined
</VirtualHost>
```

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

`Nota:` Amb un certificat d'una CA real, `SSLCertificateFile` ha de contenir el certificat i la cadena de la CA (`fullchain.pem`). La directiva antiga `SSLCertificateChainFile` ja no cal.

**Configuració amb Nginx:**

- Nginx no utilitza .htaccess
- Els server block son l'equivalent als Virtual Hosts d'Apache
- HTTP/2 millora el rendiment amb HTTPS
- Al laboratori, Nginx fa servir el port 8443 per HTTPS (el 443 és d'Apache)

```bash
# Editar el server block de web2 (el de l'apartat 10)
sudo nano /etc/nginx/sites-available/web2.educem.cat
```

```nginx
# Redirigir HTTP (8080) a HTTPS (8443)
server {
    listen 8080;
    listen [::]:8080;
    server_name web2.educem.cat;
    return 301 https://$host:8443$request_uri;
}

# HTTPS
server {
    listen 8443 ssl http2;
    listen [::]:8443 ssl http2;

    server_name web2.educem.cat;
    root /var/www/web2.educem.cat/html;
    index index.html index.htm;

    # Certificats SSL
    ssl_certificate /etc/ssl/certs/web2.educem.cat.crt;
    ssl_certificate_key /etc/ssl/private/web2.educem.cat.key;

    # Configuració SSL moderna (protocols TLS i algorismes de xifrat)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/web2.educem.cat-ssl-access.log;
    error_log /var/log/nginx/web2.educem.cat-ssl-error.log;
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

`Nota:` A partir de Nginx 1.25.1, `http2` a la línia `listen` mostra un avís d'obsolescència i es recomana la directiva `http2 on;`. Ubuntu 24.04 porta Nginx 1.24, on la forma de l'exemple és la correcta.

**Proves (des del client):**

```bash
# Redireccions
curl -I http://web1.educem.cat          # 301 → https://web1.educem.cat/
curl -I http://web2.educem.cat:8080     # 301 → https://web2.educem.cat:8443/

# HTTPS (-k accepta el certificat autosignat)
curl -k https://web1.educem.cat
curl -k https://web2.educem.cat:8443
```

Amb el navegador: `https://web1.educem.cat` i `https://web2.educem.cat:8443`. Apareixerà l'avís del certificat autosignat (apartat 12).
