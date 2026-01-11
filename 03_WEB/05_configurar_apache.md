## 05. Configuració Bàsica d'Apache

### Fitxer de Configuració Principal

El fitxer `/etc/apache2/apache2.conf` disposa de la configuració global que carrega Apache (paràmetres generals que afecten a tot el servidor).

**Paràmetres importants:**

```apache
# Directori principal/base del servidor (ruta on es troben els fitxers d'apache)
ServerRoot "/etc/apache2"

# Temps d'espera per tancar connexions amb els clients (expressat en segons)
Timeout 300

# Mantenir les connexions vives
KeepAlive On
MaxKeepAliveRequests 100 #Màxim número de peticions per connexió
KeepAliveTimeout 5 #Segons d'espera de noves peticions abans de tancar connexió

# Usuari i grup que executa Apache (amb permisos limitats)
User www-data
Group www-data

# Ports d'escolta (normalment disposat a ports.conf)
Listen 80

# Document root (normalment es creen Virtual Hosts que tenen preferència)
DocumentRoot "/var/www/html"

# Directives de directori (configuració del comportament del directori arrel)
<Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

# Arxius d'índex per defecte
DirectoryIndex index.html index.php

# Nivell de log
LogLevel warn

# Format dels logs
LogFormat "%h %l %u %t \"%r\" %>s %b" common
LogFormat "%{Referer}i -> %U" referer
LogFormat "%{User-agent}i" agent
```

### Directiva Directory

Controla el comportament d'un directori específic i permet establir uns controls detallats per cada pàgina web/directori:

```apache
<Directory "/var/www/html">
    # Options: Funcionalitats permeses en el directori
    Options Indexes FollowSymLinks

    # AllowOverride: Indica si un fitxer .htaccess sobrescriu la configuració
    AllowOverride All

    # Require: Controla qui pot accedir al directori
    Require all granted
</Directory>
```

**Options (Oopcions disponibles):**

| Option           | Descripció                                                        |
| ---------------- | ----------------------------------------------------------------- |
| `Indexes`        | Mostra el llistat de fitxers si no hi ha un index.html...         |
| `FollowSymLinks` | Segueix enllaços simbòlics apuntant a altres fitxers o directoris |
| `ExecCGI`        | Permet executar scripts CGI                                       |
| `Includes`       | Permet SSI (Server Side Includes) dintre del HTML                 |
| `MultiViews`     | Negociació de contingut segons l'idioma o format del client       |
| `None`           | Desactiva totes les opcions                                       |
| `All`            | Activa totes (excepte MultiViews)                                 |

**Directiva AllowOverride:**

| Valor        | Descripció                                       |
| ------------ | ------------------------------------------------ |
| `None`       | No permet configuració mitjançant .htaccess      |
| `All`        | Permet totes les directives dins de .htaccess    |
| `AuthConfig` | Només permet directives d'autenticació           |
| `FileInfo`   | Només permet directives relacionades amb fitxers |
| `Indexes`    | Només permet controlar llistat d'índexs          |
| `Limit`      | Només permet controlar l'accés a usuaris i IPs   |

**Directiva Require:**

| Exemple                    | Significat                                  |
| -------------------------- | ------------------------------------------- |
| `Require all granted`      | Permet accés a tots els clients.            |
| `Require all denied`       | Denega l'accés a tothom.                    |
| `Require ip 192.168.1`     | Només permet accés a IPs de la xarxa local. |
| `Require host exemple.cat` | Només permet accés als hostnames indicats.  |

### Fitxer .htaccess

Fitxer de configuració per directori que Apache llegeix cada vegada que es fa una petició.

Permet fer configuracions específiques, sense modificar el fitxer global (apache2.conf).

Funciona només si AllowOverride està activat pel directori en qüestió.

**Exemple `/var/www/html/.htaccess` o valors d'un VirtualHost:**

```apache
# Desactivar llistat de directoris si no hi ha un index.html, index.php, etc.
Options -Indexes

# Protecció contra hotlinking (evita que altres webs enllacin directament imatges del teu servidor)
RewriteEngine On
RewriteCond %{HTTP_REFERER} !^$
RewriteCond %{HTTP_REFERER} !^http(s)?://(www\.)?exemple\.cat [NC]
RewriteRule \.(jpg|jpeg|png|gif)$ - [F]

# Redirecció HTTPS (força que totes les peticions siguin per HTTPS)
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Headers de seguretat (molt importants!)
Header set X-Frame-Options "SAMEORIGIN"
Header set X-Content-Type-Options "nosniff"
Header set X-XSS-Protection "1; mode=block"

# Compressió GZIP (comprimeix el contingut de text abans d'enviar-lo al client)
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript application/javascript
</IfModule>

# Cache per recursos estàtics (indica al navegador que guardi en caché els recursos durant un cert temps)
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresByType image/jpg "access plus 1 year"
    ExpiresByType image/jpeg "access plus 1 year"
    ExpiresByType image/gif "access plus 1 year"
    ExpiresByType image/png "access plus 1 year"
    ExpiresByType text/css "access plus 1 month"
    ExpiresByType application/javascript "access plus 1 month"
</IfModule>
```

**Impacte en el Rendiment:**

- `.htaccess` es llegeix en **cada petició** i pot fer el servidor més lent si hi ha molts fitxers.
- Millor configurar tot a Virtual Host si és possible

### Comandes Útils d'Apache

```bash
# Verificar sintaxi de configuració
sudo apache2ctl configtest

# Veure configuració carregada
sudo apache2ctl -S

# Veure mòduls carregats
sudo apache2ctl -M

# Graceful restart (espera a tancar connexions actives)
sudo apache2ctl graceful

# Veure versió i mòduls compilats
sudo apache2 -V
```
