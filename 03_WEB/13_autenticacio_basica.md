## 12. Autenticació i Control d'Accés

L'autenticació HTTP bàsica permet restringir l'accés a determinats recursos mitjançant un login (un usuari i una contrasenya). Les credencials s'emmagatzemen xifrades (un hash) en un fitxer i el navegador les sol·licita quan l'usuari intenta accedir a una zona protegida.

L'autenticació bàsica ha d'utilitzar-se sempre amb HTTPS, ja que les credencials viatgen codificades però no xifrades.

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

**Pas 2: Configurar Virtual Host o Directori**

Afegir al mateix fitxer del Virtual Host.

`Nota:` Per la prova crea una carpeta a "exemple1/html/" que es digui "admin" i afegeix un index.html

```apache
<Directory "/var/www/exemple1/html/admin">
    AuthType Basic
    AuthName "Àrea Restringida :)"
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Directory>
```

**Pas 3: Reiniciar Apache**

```bash
sudo systemctl reload apache2
```

**Pas 4: Accedir al lloc web**

Comprovar que apareix el login i provar d'iniciar sessió.

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

**Pas 2: Configurar Nginx**

`Nota:` Per la prova crea una carpeta a "exemple2/html/" que es digui "admin" i afegeix un index.html

```nginx
server {
    listen 80;
    server_name exemple2.cat;
    root /var/www/exemple2;

    # Autenticació per tot el lloc
    auth_basic "Àrea Restringida Nginx :)";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # O només per una ubicació concreta
    location /admin {
        auth_basic "Zona de l'Administrador";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Pas 3: Recarregar Nginx**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Pas 4: Accedir al lloc web**

Comprovar que apareix el login i provar d'iniciar sessió.
