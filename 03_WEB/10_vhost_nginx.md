## 10. Virtual Hosts en Nginx

Nginx gestiona els Virtual Hosts mitjançant blocs de configuració anomenats server blocks, que defineixen com s'han de servir els diferents llocs web. Aquests blocs permeten associar dominis, IPs o ports a directoris concrets del servidor.

De manera similar a Apache, Nginx utilitza fitxers de configuració per organitzar els llocs web.

**Pas 0: Aturar l'Apache per no tenir conflicte de serveis al mateix port**

```bash
sudo systemctl stop apache2
sudo systemctl start nginx
```

**Pas 1-2: Crear directoris i pàgines** (igual que hem fet a l'Apache)

Per l'exemple farem servir les mateixes web de l'Apache.

- exemple1.cat
- exemple2.cat

**Pas 3: Crear fitxer de configuració del Virtual Host**

```bash
sudo nano /etc/nginx/sites-available/exemple1.cat
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name exemple1.cat www.exemple1.cat;

    root /var/www/exemple1.cat/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/exemple1.cat-access.log;
    error_log /var/log/nginx/exemple1.cat-error.log;
}
```

**Pas 4: Activar els Virtual Hosts**

```bash
# Crear l'enllaç simbolic (symlink) apache ho fa automàticament amb a2ensite
sudo ln -s /etc/nginx/sites-available/exemple1.cat /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/exemple2.cat /etc/nginx/sites-enabled/

# Verificar configuració
sudo nginx -t

# Recarregar Nginx
sudo systemctl reload nginx
```

**Pas 5: Proves al servidor i al client**

```bash
curl http://exemple1.cat
curl http://exemple2.cat
```
