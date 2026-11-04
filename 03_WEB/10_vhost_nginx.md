## 10. Virtual Hosts en Nginx

Nginx gestiona els Virtual Hosts mitjançant blocs de configuració anomenats server blocks, que defineixen com s'han de servir els diferents llocs web. Aquests blocs permeten associar dominis, IPs o ports a directoris concrets del servidor.

De manera similar a Apache, Nginx utilitza fitxers de configuració per organitzar els llocs web.

Al laboratori, Nginx escolta al **port 8080** (apartat 04) i Apache continua funcionant al port 80. No cal aturar cap dels dos serveis.

**Pas 1-2: Directoris, pàgines i registres DNS** (igual que hem fet a l'Apache)

Per l'exemple farem servir les mateixes webs de l'Apache: tenen els mateixos noms DNS i les mateixes carpetes.

- web1.educem.cat
- web2.educem.cat

**Pas 3: Crear fitxer de configuració del Virtual Host**

```bash
sudo nano /etc/nginx/sites-available/web1.educem.cat
```

```nginx
server {
    listen 8080;
    listen [::]:8080;

    server_name web1.educem.cat;

    root /var/www/web1.educem.cat/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/web1.educem.cat-access.log;
    error_log /var/log/nginx/web1.educem.cat-error.log;
}
```

**Repeteix el procés per web2.educem.cat:**

```bash
sudo nano /etc/nginx/sites-available/web2.educem.cat
```

```nginx
server {
    listen 8080;
    listen [::]:8080;

    server_name web2.educem.cat;

    root /var/www/web2.educem.cat/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/web2.educem.cat-access.log;
    error_log /var/log/nginx/web2.educem.cat-error.log;
}
```

**Pas 4: Activar els Virtual Hosts**

```bash
# Crear l'enllaç simbolic (symlink) apache ho fa automàticament amb a2ensite
sudo ln -s /etc/nginx/sites-available/web1.educem.cat /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/web2.educem.cat /etc/nginx/sites-enabled/

# Verificar configuració
sudo nginx -t

# Recarregar Nginx
sudo systemctl reload nginx
```

`Nota:` Creeu l'enllaç només quan el fitxer de `sites-available` ja existeixi. Si un enllaç apunta a un fitxer que no existeix, `nginx -t` falla amb `open() ... failed (2: No such file or directory)`.

**Pas 5: Proves al servidor i al client**

```bash
# Nginx (port 8080)
curl http://web1.educem.cat:8080
curl http://web2.educem.cat:8080

# Comparar quin servidor respon a cada port
curl -I http://web1.educem.cat        # Server: Apache/...
curl -I http://web1.educem.cat:8080   # Server: nginx/...
```

És un exemple de Virtual Host **basat en port** (apartat 08): el mateix nom i el mateix contingut, servits per un programari diferent segons el port.
