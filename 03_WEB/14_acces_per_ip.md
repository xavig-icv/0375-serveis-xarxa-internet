## Control d'Accés per IP

### Control d'Accés per IP (Apache)

L'Apache permet restringir l'accés a recursos web segons l'adreça IP del client. Aquest control es pot aplicar a directoris, fitxers o Virtual Hosts.

`Nota:` Per fer la prova crear una carpeta anomenada "`adminpanel`" dins de "exemple1/html/"

- Només les IP indicades podran accedir al directori /adminpanel.

```apache
<Directory "/var/www/exemple1/html/adminpanel">
    # Denegar per defecte
    Require all denied

    # Permetre IPs de la xarxa local dels clients del 0374 i 0375.
    Require ip 192.168.1.0/24
    # La vostra IP que us ha assignat l'escola
    Require ip 10.0.X.X
</Directory>
```

**Combinació IP + Autenticació (Apache):**

Apache permet combinar diverses condicions mitjançant blocs lògics:

- <RequireAny> → una condició és suficient (OR)
- <RequireAll> → totes les condicions són obligatòries (AND)

```apache
<Directory "/var/www/exemple1/html/adminpanel">
    # Requereix o una IP local o l'autenticació per accedir
    <RequireAny>
        # Accés directe des de la xarxa local (sense autenticació)
        Require ip 192.168.1.0/24
        # Autenticació obligatòria si s'està fora de la xarxa local
        <RequireAll>
            AuthType Basic
            AuthName "Login Requerit estàs fora de la xarxa"
            AuthUserFile /etc/apache2/.htpasswd
            Require valid-user
        </RequireAll>
    </RequireAny>
</Directory>
```

### Control d'Accés per IP (Nginx)

Nginx controla l'accés per IP mitjançant les directives allow i deny, aplicables a nivell de server o location.

`Nota:` Per fer la prova crear una carpeta anomenada "`adminpanel`" dins de "exemple2/html/"

- Només les IP indicades podran accedir al directori /adminpanel.

```nginx
server {
    listen 80;
    server_name exemple2.cat;

    location /adminpanel {
        # Permetre IPs específiques
        allow 192.168.1.0/24;
        # La IP que t'ha assignat el DHCP de l'escola.
        allow 10.0.X.X;

        # Denegar l'accés a la resta
        deny all;

        root /var/www/exemple2/html;
    }
}
```

**Combinació IP + Autenticació (Nginx):**

A Nginx, la combinació es fa amb la directiva satisfy:

- satisfy any; → IP o autenticació (només una de les dues)
- satisfy all; → IP i autenticació (són obligatòries les dues)

```nginx
location /adminpanel {
    # Permet l'accés per IP o per autenticació
    satisfy any;

    # Primer comprova la IP
    allow 192.168.1.0/24;
    deny all;

    # Després comprova l'autenticació bàsica
    auth_basic "Àrea Restringida";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```
