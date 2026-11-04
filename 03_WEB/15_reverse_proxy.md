## 15. Reverse Proxy: Nginx + Apache

El model híbrid Nginx + Apache aprofita els punts forts de cada servidor i minimitza les limitacions individuals. Aquesta arquitectura és molt utilitzada en entorns de producció amb un gran número de visitants i on es requereix un alt rendiment, escalabilitat i flexibilitat.

En aquest model, Nginx actua com a proxy invers, servint principalment contingut estàtic i redirigint peticions de contingut dinàmic cap a l'Apache que funciona com a servidor de backend i que s'encarrega principalment del contingut dinàmic.

### Avantatges del Model Híbrid

**Limitacions individuals:**

- **Apache**: No és bo per gestionar entorns amb una alta càrrega i múltiples peticions concurrents.
- **Nginx**: No disposa d'un suport incorporat per gestionar i processar el contingut dinàmic PHP (requereix d'un mòdul addicional o d'Apache).

**Solució híbrida:**

**`Nginx:`**

- Gestiona totes les peticions entrants (HTTP/HTTPS)
- Serveix contingut estàtic (HTML, CSS, JS, imatges)
- Implementa memòria cau per reduir peticions repetides
- Actua com a Proxy Invers
- Gestiona SSL/TLS de forma eficient

**`Apache:`**

- Es comunica exclusivament amb Nginx
- Rep únicament les peticions de contingut dinàmic
- Processa PHP (mitjançant mod_php o PHP-FPM)
- Executa aplicacions web complexes

### Configuració del Model Híbrid al Laboratori

Al laboratori, Apache ja escolta als ports 80 i 443, i Nginx als ports 8080 i 8443. Per provar el model híbrid sense tocar aquests llocs:

- Apache escoltarà també en un **port intern** (`127.0.0.1:8081`), només accessible des del mateix servidor. Farà de backend.
- El server block de `web1.educem.cat` al **port 8080** de Nginx passarà a fer de proxy invers cap a aquest port intern.

```
CLIENT ──► Nginx (web1.educem.cat:8080) ──┬──► fitxers estàtics (els serveix Nginx)
                                          └──► 127.0.0.1:8081 Apache + PHP (contingut dinàmic)
```

#### Pas 1: Instal·lar PHP per a Apache

```bash
sudo apt install php libapache2-mod-php
sudo systemctl restart apache2
```

Sense aquest pas, Apache envia els fitxers `.php` com a text en lloc d'executar-los.

#### Pas 2: Port intern d'Apache (backend)

**Afegir el port d'escolta intern:**

```bash
sudo nano /etc/apache2/ports.conf
```

```apache
# Afegir (es manté el Listen 80 i el 443):
Listen 127.0.0.1:8081
```

**Crear el Virtual Host de backend:**

```bash
sudo nano /etc/apache2/sites-available/web1-backend.conf
```

```apache
<VirtualHost 127.0.0.1:8081>
    ServerName web1.educem.cat

    DocumentRoot /var/www/web1.educem.cat/html

    <Directory /var/www/web1.educem.cat/html>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/web1.educem.cat-backend-error.log
    CustomLog ${APACHE_LOG_DIR}/web1.educem.cat-backend-access.log combined
</VirtualHost>
```

**Activar-lo i reiniciar Apache:**

```bash
sudo a2ensite web1-backend.conf
sudo apache2ctl configtest
sudo systemctl restart apache2
```

**Verificar que Apache escolta al port 8081 (només a 127.0.0.1):**

```bash
sudo ss -tlnp | grep apache2
```

#### Pas 3: Configurar Nginx com a Reverse Proxy

Modificar el server block de `web1.educem.cat` de l'apartat 10 (port 8080):

```bash
sudo nano /etc/nginx/sites-available/web1.educem.cat
```

```nginx
server {
    listen 8080;
    listen [::]:8080;
    server_name web1.educem.cat;

    # Logs
    access_log /var/log/nginx/web1.educem.cat-access.log;
    error_log /var/log/nginx/web1.educem.cat-error.log;

    # Servir fitxers estàtics directament amb Nginx
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        root /var/www/web1.educem.cat/html;
        expires 30d;
        add_header X-Servit-Per "nginx";
    }

    # Redirigir la resta de peticions (contingut dinàmic) a Apache
    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header X-Servit-Per "apache (via nginx)";

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

La capçalera `X-Servit-Per` només serveix per veure a les proves quin servidor ha generat cada resposta.

#### Pas 4: Recarregar Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

#### Pas 5: Crear contingut de prova

**Crear fitxer PHP per web1.educem.cat:**

```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/web1.educem.cat/html/info.php
```

**Crear fitxer CSS d'exemple:**

```bash
sudo nano /var/www/web1.educem.cat/html/style.css
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

#### Pas 6: Proves del sistema

**Provar el contingut dinàmic (processat per Apache):**

```bash
curl http://web1.educem.cat:8080/info.php
```

**Provar el contingut estàtic (servit per Nginx):**

```bash
curl http://web1.educem.cat:8080/style.css
```

**Verificar les capçaleres per veure quin servidor respon:**

```bash
curl -I http://web1.educem.cat:8080/info.php    # X-Servit-Per: apache (via nginx)
curl -I http://web1.educem.cat:8080/style.css   # X-Servit-Per: nginx
```

La capçalera `Server` sempre diu `nginx`: Nginx no passa al client la capçalera `Server` d'Apache.

**Comprovar els logs:**

```bash
# Logs de Nginx: totes les peticions, amb la IP del client
sudo tail -f /var/log/nginx/web1.educem.cat-access.log

# Logs d'Apache (backend): només les peticions dinàmiques, que arriben des de 127.0.0.1
sudo tail -f /var/log/apache2/web1.educem.cat-backend-access.log
```

**(Opcional) IP real del client als logs d'Apache**

Apache rep totes les peticions des de Nginx (`127.0.0.1`). Per registrar la IP real del client:

1. Activar `mod_remoteip`: `sudo a2enmod remoteip`.
2. Afegir al VirtualHost de backend:

```apache
    RemoteIPHeader X-Forwarded-For
    RemoteIPInternalProxy 127.0.0.1
    LogFormat "%a %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined_ip_real
    CustomLog ${APACHE_LOG_DIR}/web1.educem.cat-backend-access.log combined_ip_real
```

### Configuració Avançada amb Memòria Cau

Per millorar encara més el rendiment, podem afegir la memòria cau a Nginx.

**1. Definir la zona de memòria cau.** La directiva `proxy_cache_path` només es pot posar al context `http` (no dins d'un bloc `server`). Els fitxers de `conf.d/` s'inclouen dins del context `http`:

```bash
sudo nano /etc/nginx/conf.d/cache.conf
```

```nginx
proxy_cache_path /var/cache/nginx/web levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;
```

**2. Crear el directori de la memòria cau:**

```bash
sudo mkdir -p /var/cache/nginx/web
sudo chown -R www-data:www-data /var/cache/nginx/web
```

**3. Fer-la servir al `location /` del server block:**

```nginx
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 60m;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    add_header X-Cache-Status $upstream_cache_status;

    proxy_pass http://127.0.0.1:8081;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx

# La primera petició és MISS i les següents, HIT
curl -sI http://web1.educem.cat:8080/ | grep X-Cache-Status
```

### Verificació del funcionament

**Comprovar que ambdós serveis estan actius:**

```bash
sudo systemctl status nginx
sudo systemctl status apache2
```

**Comprovar ports d'escolta:**

```bash
# Apache: 80, 443 i 127.0.0.1:8081 | Nginx: 8080 i 8443
sudo ss -tlnp | grep -E 'apache2|nginx'
```
