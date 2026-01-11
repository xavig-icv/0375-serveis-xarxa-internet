## 15. Reverse Proxy: Nginx + Apache

El model híbrid Nginx + Apache aprofitar els punts forts de cada servidor i minimitzar les limitacions individuals. Aquesta arquitectura és molt utilitzada en entorns de producció amb un gran número de visitants i on es requereix un alt rendiment, escalabilitat i flexibilitat.

En aquest model, Nginx actua com a proxy invers, servint principalment contingut estàtic i redirigint peticions de contingut dinàmic cap a l'Apache que funciona com a servidor de backend i que s'encarrega principalment del contingut dinàmic.

### Avantatges del Model Híbrid

**Limitacions individuals:**

- **Apache**: No és bo per gestionar entorns amb una alta càrrega i múltiples peticions concurrents.
- **Nginx**: No disposa d'un suport incorporat per gestionar i processar el contingut dinàmic PHP (requereix d'un mòdul addicional o d'Apache).

**Solució híbrida:**

**`Nginx:`**

- Gestiona totes les peticions entrants (HTTP/HTTPS)
- Serveix contingut estàtic (HTML, CSS, JS, imatges)
- Implementa caché per reduir peticions repetides
- Actua com a Proxy Invers
- Gestiona SSL/TLS de forma eficient

**`Apache:`**

- Es comunica exclusivament amb Nginx
- Rep únicament les peticions de contingut dinàmic
- Processa PHP (mitjançant mod_php o PHP-FPM)
- Executa aplicacions web complexes

### Configuració del Model Híbrid

#### Pas 1: Configurar Apache per escoltar al port 8080

**Modificar el port d'escolta d'Apache:**

```bash
sudo nano /etc/apache2/ports.conf
```

```apache
# Canviar de Listen 80 a:
Listen 8080
```

**Modificar els Virtual Hosts d'Apache:**

```bash
sudo nano /etc/apache2/sites-available/exemple1.cat.conf
```

```apache
<VirtualHost *:8080>
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat
    ServerAdmin admin@exemple1.cat

    DocumentRoot /var/www/exemple1.cat/html

    <Directory /var/www/exemple1.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/exemple1.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/exemple1.cat-access.log combined
</VirtualHost>
```

**Repetir per exemple2.cat:**

```bash
sudo nano /etc/apache2/sites-available/exemple2.cat.conf
```

```apache
<VirtualHost *:8080>
    ServerName exemple2.cat
    ServerAlias www.exemple2.cat
    ServerAdmin admin@exemple2.cat

    DocumentRoot /var/www/exemple2.cat/html

    <Directory /var/www/exemple2.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/exemple2.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/exemple2.cat-access.log combined
</VirtualHost>
```

**Reiniciar Apache:**

```bash
sudo systemctl restart apache2
```

**Verificar que Apache escolta al port 8080:**

```bash
sudo ss -plutn | grep apache2
```

#### Pas 2: Configurar Nginx com a Reverse Proxy

**Crear la configuració per exemple1.cat:**

```bash
sudo nano /etc/nginx/sites-available/exemple1.cat
```

```nginx
server {
    listen 80;
    server_name exemple1.cat www.exemple1.cat;

    # Logs
    access_log /var/log/nginx/exemple1.cat-access.log;
    error_log /var/log/nginx/exemple1.cat-error.log;

    # Servir fitxers estàtics directament amb Nginx
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        root /var/www/exemple1.cat/html;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Redirigir peticions dinàmiques a Apache
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Crear la configuració per exemple2.cat:**

```bash
sudo nano /etc/nginx/sites-available/exemple2.cat
```

```nginx
server {
    listen 80;
    server_name exemple2.cat www.exemple2.cat;

    # Logs
    access_log /var/log/nginx/exemple2.cat-access.log;
    error_log /var/log/nginx/exemple2.cat-error.log;

    # Servir fitxers estàtics directament amb Nginx
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        root /var/www/exemple2.cat/html;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Redirigir peticions dinàmiques a Apache
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

#### Pas 3: Activar els llocs i reiniciar Nginx

```bash
# Activar els llocs web
sudo ln -s /etc/nginx/sites-available/exemple1.cat /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/exemple2.cat /etc/nginx/sites-enabled/

# Desactivar lloc per defecte (opcional)
sudo rm /etc/nginx/sites-enabled/default

# Verificar configuració
sudo nginx -t

# Reiniciar Nginx
sudo systemctl restart nginx
```

#### Pas 4: Configurar DNS/hosts

**Al servidor i al client:**

```bash
sudo nano /etc/hosts
```

```
# Afegir (utilitzar IP del servidor si es fa des del client):
127.0.0.1    exemple1.cat www.exemple1.cat
127.0.0.1    exemple2.cat www.exemple2.cat
```

#### Pas 5: Crear contingut de prova

**Crear fitxer PHP per exemple1.cat:**

```bash
sudo nano /var/www/exemple1.cat/html/info.php
```

```php
<?php
phpinfo();
?>
```

**Crear fitxer CSS d'exemple:**

```bash
sudo nano /var/www/exemple1.cat/html/style.css
```

```css
body {
  background-color: dodgerblue;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

**Repetir-ho també per exemple2.cat.**

#### Pas 6: Proves del sistema

**Provar el contingut dinàmic (processat per Apache):**

```bash
curl http://exemple1.cat/info.php
curl http://exemple2.cat/
```

**Provar el contingut estàtic (servit per Nginx):**

```bash
curl http://exemple1.cat/style.css
```

**Verificar les capçaleres per veure quin servidor respon:**

```bash
curl -I http://exemple1.cat/
curl -I http://exemple1.cat/style.css
```

**Comprovar els logs:**

```bash
# Logs de Nginx
sudo tail -f /var/log/nginx/exemple1.cat-access.log

# Logs d'Apache
sudo tail -f /var/log/apache2/exemple1.cat-access.log
```

### Configuració Avançada amb Caché

Per millorar encara més el rendiment, podem afegir la caché a Nginx:

```nginx
# Afegir dins del bloc server {} abans de les directives location
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 60m;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    add_header X-Cache-Status $upstream_cache_status;

    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**Crear el directori de caché:**

```bash
sudo mkdir -p /var/cache/nginx
sudo chown -R www-data:www-data /var/cache/nginx
```

### Verificació del funcionament

**Comprovar que ambdós serveis estan actius:**

```bash
sudo systemctl status nginx
sudo systemctl status apache2
```

**Comprovar ports d'escolta:**

```bash
#Nginx port 80 i Apache 8080
sudo netstat -tlnp | grep -E '(:80|:8080)'
```
