## 06. Configuració Bàsica de Nginx

### Fitxer de Configuració Principal

El fitxer `/etc/nginx/nginx.conf` disposa de la configuració global que carrega Nginx (paràmetres generals que afecten a tot el servidor).

```nginx
# Usuari que executa Nginx
user www-data;

# Nombre de processos workers (normalment 1 Worker per 1 CPUs disponible)
worker_processes auto;

# PID del procés principal (útil per controlar el servei de nginx)
pid /run/nginx.pid;

# Esdeveniments
events {
    # Connexions simultànies per cada worker
    worker_connections 768;

    # Mètode de gestió d'esdeveniments (Linux: epoll) és el més eficient
    use epoll;

    # Acceptar múltiples connexions simultànies per cada worker
    multi_accept on;
}

http {
    # Configuració Bàsica

    # Incloure tipus MIME (defineix tipus de fitxer i el seu content-type)
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging (peticions i errors del servidor)
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Optimitzacions de connexió (agrupa paquets i els envia conjuntament)
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # Timeouts (temps d'espera)
    keepalive_timeout 65;
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;

    # Límits de mida de fitxers i informació que s'envia al servidor
    client_max_body_size 100M;
    client_body_buffer_size 128k;

    # Compressió GZIP
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript
               application/json application/javascript application/xml+rss;

    # Virtual Hosts (equivalents als d'Apache)
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Bloc Server (Virtual Host)

El fitxer `/etc/nginx/sites-available/default` equival al fitxer de Virtual Host d'Apache.

```nginx
server {
    # Port d'escolta
    listen 80 default_server;
    listen [::]:80 default_server;

    # Nom del servidor (domini)
    server_name exemple.cat www.exemple.cat;

    # Document root (directori arrel amb els documents web)
    root /var/www/html;

    # Fitxers d'índex (llista de fitxers que busca nginx com a índex)
    index index.html index.htm index.php;

    # Bloc location principal
    location / {
        # Intenta fitxer demanat "$uri", sinó directori "$uri/", sinó 404
        try_files $uri $uri/ =404;
    }

    # Logs específics del virtual host (accessos i errors)
    access_log /var/log/nginx/exemple.cat-access.log;
    error_log /var/log/nginx/exemple.cat-error.log;
}
```

### Bloc Location

Defineix com processar URIs específiques:

```nginx
# Coincidencia Exacta (=)
location = /exacte {
    # Només s'executa si la ruta és igual a /exacte
}

# Prefix normal
location /imatges/ {
    # Vàlida qualsevol ruta que comenci amb /imatges/*
}

# Expressió regular (case-sensitive)
location ~ \.php$ {
    # Qualsevol fitxer que finalitzi amb *.php
    # Diferencia entre majúscules i minúscules
}

# Expressió regular (case-insensitive)
location ~* \.(jpg|jpeg|png|gif)$ {
    # Coincideix amb imatges (.jpg, .JPG, etc.)
    # No diferencia entre majúscules i minúscules
}

# Prefix prioritari
location ^~ /admin/ {
    # Coincideix amb /admin/* amb prioritat sobre regex
    # Si denega accés evita executar un /admin/index.php per exemple
}
```

**Prioritat de coincidència:**

1. `= ` (exacte)
2. `^~` (prefix prioritari)
3. `~` i `~*` (regex, en ordre d'aparició)
4. Prefix normal (més llarg primer)

### Variables de Nginx

Les variables de Nginx són valors dinàmics que representen informació de la petició, del client, del servidor o de l’estat intern de Nginx.

**Exemple:**

```
http://www.exemple.cat/producte.php?id=5
```

- $request_uri → /producte.php?id=5
- $uri → /producte.php
- $args → id=5

```nginx
server {
    listen 80;
    server_name exemple.cat;

    location / {
        # Variables disponibles:
        # $uri - URI actual (fitxer)
        # $request_uri - URI amb query string (amb arguments)
        # $args - Arguments query string (els paràmetres)
        # $host - Nom host de la petició
        # $remote_addr - IP del client
        # $scheme - http o https
        # $server_name - Nom del server block
        # $request_method - GET, POST, etc.

        # Exemple: Afegir header personalitzat
        add_header X-Client-IP $remote_addr;
        add_header X-Request-URI $request_uri;

        try_files $uri $uri/ =404;
    }
}
```

### Comandes Útils de Nginx

```bash
# Verificar sintaxi
sudo nginx -t

# Recarregar configuració
sudo nginx -s reload

# Aturar (gracefully)
sudo nginx -s quit

# Aturar (immediately)
sudo nginx -s stop

# Reopenir logs
sudo nginx -s reopen

# Veure configuració compilada
sudo nginx -V

# Veure processos
ps aux | grep nginx
```
