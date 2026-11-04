## 14. Control d'Accés per IP

### Control d'Accés per IP (Apache)

L'Apache permet restringir l'accés a recursos web segons l'adreça IP del client. Aquest control es pot aplicar a directoris, fitxers o Virtual Hosts.

`Nota:` Per fer la prova crear una carpeta anomenada "`adminpanel`" dins de `/var/www/web1.educem.cat/html/`, amb un `index.html`.

- Només les IP indicades podran accedir al directori /adminpanel.
- El bloc va dins del `<VirtualHost *:443>` de `web1.educem.cat.conf`, igual que a l'apartat 13. La ruta ha de coincidir exactament amb la de la carpeta.

```apache
<Directory "/var/www/web1.educem.cat/html/adminpanel">
    # Només el client Ubuntu (reserva DHCP 192.168.1.50)
    Require ip 192.168.1.50

    # O bé tota la xarxa interna:
    # Require ip 192.168.1.0/24
</Directory>
```

**Notes:**

- Quan hi ha diverses línies `Require`, n'hi ha prou que se'n compleixi una.
- Les IP que no hi surten reben un `403 Forbidden`.
- El client rep la IP per DHCP. La reserva per MAC del Tema 2 garanteix que tingui sempre la `192.168.1.50` i que la regla continuï funcionant.

**Com provar-ho:** Cal fer peticions des de dues IP diferents. El servidor 0374 (`192.168.1.1`) és a la mateixa xarxa i fa d'equip no autoritzat.

```bash
# Des del client (192.168.1.50): 200 OK
curl -k -I https://web1.educem.cat/adminpanel/

# Des del servidor 0374 (192.168.1.1): 403 Forbidden
curl -k -I https://web1.educem.cat/adminpanel/
```

`Nota:` El 0374 resol `web1.educem.cat` amb el seu DNS secundari. Si no el resol, comproveu que la zona transferida està actualitzada (serial del Tema 1).

**Combinació IP + Autenticació (Apache):**

Apache permet combinar diverses condicions mitjançant blocs lògics:

- `<RequireAny>` → una condició és suficient (OR)
- `<RequireAll>` → totes les condicions són obligatòries (AND)

```apache
<Directory "/var/www/web1.educem.cat/html/adminpanel">
    # Requereix o una IP local o l'autenticació per accedir
    <RequireAny>
        # Accés directe des del client (sense autenticació)
        Require ip 192.168.1.50
        # Autenticació obligatòria per a la resta d'equips
        <RequireAll>
            AuthType Basic
            AuthName "Login Requerit"
            AuthUserFile /etc/apache2/.htpasswd
            Require valid-user
        </RequireAll>
    </RequireAny>
</Directory>
```

### Control d'Accés per IP (Nginx)

Nginx controla l'accés per IP mitjançant les directives allow i deny, aplicables a nivell de server o location.

`Nota:` Per fer la prova crear una carpeta anomenada "`adminpanel`" dins de `/var/www/web2.educem.cat/html/`, amb un `index.html`.

- Només les IP indicades podran accedir al directori /adminpanel.
- El bloc `location` va **dins del server block HTTPS que ja existeix** (port 8443) de `web2.educem.cat`. No creeu un altre bloc `server`.

```nginx
    location /adminpanel {
        # Permetre la IP del client
        allow 192.168.1.50;

        # Denegar l'accés a la resta
        deny all;
    }
```

```bash
sudo nginx -t
sudo systemctl reload nginx

# Des del client: 200 OK. Des del 0374: 403 Forbidden
curl -k -I https://web2.educem.cat:8443/adminpanel/
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
    allow 192.168.1.50;
    deny all;

    # Després comprova l'autenticació bàsica
    auth_basic "Àrea Restringida";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```
