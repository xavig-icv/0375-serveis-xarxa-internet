## 13. Autenticació i Control d'Accés

L'autenticació HTTP bàsica permet restringir l'accés a determinats recursos mitjançant un login (un usuari i una contrasenya). Les credencials s'emmagatzemen xifrades (un hash) en un fitxer i el navegador les sol·licita quan l'usuari intenta accedir a una zona protegida.

L'autenticació bàsica ha d'utilitzar-se sempre amb HTTPS, ja que les credencials viatgen codificades però no xifrades. Per això la configurarem als Virtual Hosts HTTPS de l'apartat 11.

### Autenticació Bàsica HTTP (Apache)

Apache utilitza el mòdul `mod_auth_basic` juntament amb fitxers de contrasenyes creats amb l'eina htpasswd.

**Pas 1: Crear fitxer de contrasenyes**

```bash
# Instal·lar l'eina htpasswd (si no s'ha instal·lat prèviament)
sudo apt install apache2-utils

# Crear un usuari (la primera vegada s'ha d'utilitzar el paràmetre -c)
sudo htpasswd -c /etc/apache2/.htpasswd usuari1

# Afegir més usuaris (no utilitzar el paràmetre -c)
sudo htpasswd /etc/apache2/.htpasswd usuari2

# Establir permisos segurs pel fitxer
sudo chmod 640 /etc/apache2/.htpasswd
sudo chown root:www-data /etc/apache2/.htpasswd
```

**Pas 2: Crear la carpeta protegida**

```bash
sudo mkdir -p /var/www/web1.educem.cat/html/admin
echo "<h1>Zona d'administració de web1</h1>" | sudo tee /var/www/web1.educem.cat/html/admin/index.html
```

**Pas 3: Configurar el Virtual Host**

Afegir el bloc dins del `<VirtualHost *:443>` de `web1.educem.cat.conf` (apartat 11). Si el poseu al VirtualHost del port 80, que ara només redirigeix, la protecció no s'aplicarà a les peticions HTTPS.

La ruta del bloc `<Directory>` ha de ser **exactament** la de la carpeta dins del `DocumentRoot`. Si no coincideix, Apache no aplica la protecció i la carpeta queda oberta sense donar cap error.

```apache
<Directory "/var/www/web1.educem.cat/html/admin">
    AuthType Basic
    AuthName "Àrea Restringida :)"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Directory>
```

**Pas 4: Recarregar Apache**

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

**Pas 5: Comprovar l'accés**

```bash
# Sense credencials: 401 Unauthorized
curl -k -I https://web1.educem.cat/admin/

# Amb un usuari vàlid: 200 OK (curl demana la contrasenya)
curl -k -I -u usuari1 https://web1.educem.cat/admin/
```

Al navegador ha d'aparèixer la finestra de login. Els intents fallits queden registrats al log d'errors del lloc (apartat 17).

**Opcions de la directiva Require:**

```apache
# Qualsevol usuari vàlid
Require valid-user

# Només permet l'accés a uns usuaris específicats
Require user usuari1 usuari2

# Només permet l'accés a uns grups d'usuaris (fitxer de grups separat)
AuthGroupFile /etc/apache2/.htgroup
Require group admin
```

### Autenticació Bàsica HTTP (Nginx)

Nginx també suporta l'autenticació bàsica HTTP mitjançant fitxers .htpasswd, però no utilitza .htaccess. Tota la configuració es fa dins dels server blocks.

**Pas 1: Crear fitxer de contrasenyes**

```bash
# Instal·lar eina (si no està)
sudo apt install apache2-utils

# Crear el fitxer i el primer usuari
sudo htpasswd -c /etc/nginx/.htpasswd usuari1

# Afegir més usuaris
sudo htpasswd /etc/nginx/.htpasswd usuari2

# Permisos segurs
sudo chmod 640 /etc/nginx/.htpasswd
sudo chown root:www-data /etc/nginx/.htpasswd
```

**Pas 2: Crear la carpeta protegida**

```bash
sudo mkdir -p /var/www/web2.educem.cat/html/admin
echo "<h1>Zona d'administració de web2</h1>" | sudo tee /var/www/web2.educem.cat/html/admin/index.html
```

**Pas 3: Configurar Nginx**

Afegir el bloc `location` **dins del server block HTTPS que ja existeix** (port 8443) de `/etc/nginx/sites-available/web2.educem.cat`.

No creeu un altre bloc `server` amb el mateix `server_name` i el mateix port. Nginx només en faria servir un, mostraria l'avís `conflicting server name` i la protecció es podria quedar sense aplicar.

```nginx
    # Només per una ubicació concreta
    location /admin {
        auth_basic "Zona de l'Administrador";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    # O bé per tot el lloc: les dues directives a nivell de server
    # auth_basic "Àrea Restringida Nginx :)";
    # auth_basic_user_file /etc/nginx/.htpasswd;
```

**Pas 4: Recarregar Nginx**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Pas 5: Comprovar l'accés**

```bash
curl -k -I https://web2.educem.cat:8443/admin/              # 401
curl -k -I -u usuari1 https://web2.educem.cat:8443/admin/   # 200
```
