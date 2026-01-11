## 14. Certificats Autosignats (Només per fer Proves)

Els certificats autosignats s'utilitzen exclusivament en entorns de proves o desenvolupament. No estan signats per cap Autoritat de Certificació (CA) reconeguda, per tant els navegadors no els consideren de confiança i apareixerà un missatge d'advertència cada cop que volem accedir al lloc web.

### Generar un certificat SSL autosignat:

```bash
# Crear el directori si no existeix
sudo mkdir -p /etc/ssl/private
# Restringir els permisos (només root)
sudo chmod 700 /etc/ssl/private

# Generar el certificat i la clau privada (exemple vàlid per 365 dies)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/certificat-autosignat.key -out /etc/ssl/certs/certificat-autosignat.crt

# Respondre preguntes:
# Country Name: ES
# State: Catalunya
# Locality: Barcelona
# Organization: Empresa Exemple 1
# Common Name: exemple1.cat  ← IMPORTANT QUE COINCIDEIXI AMB EL DOMINI
# Email: admin@exemple1.cat
```

### Configurar SSL/TLS a Apache

**Activar el mòdul SSL:**

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
```

**Habilitar SSL al Virtual Host:**

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

**Avís del navegador:** Els certificats autosignats **sempre** mostraran avís de seguretat al navegador (ja que no són de confiança).
