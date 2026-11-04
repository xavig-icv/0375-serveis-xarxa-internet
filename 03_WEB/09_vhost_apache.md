## 09. Virtual Hosts amb Apache

Apache gestiona els Virtual Hosts mitjançant fitxers de configuració que defineixen com s'han de servir els diferents llocs web. Aquests fitxers permeten associar dominis, IPs o ports a directoris concrets del servidor.

**Apache disposa de 2 directoris que emmagatzemen tots els fitxers de configuració:**

- **sites-available/**: Configuracions de tots els Virtual Hosts `disponibles`
- **sites-enabled/**: Configuracions de tots els Virtual Hosts `actius`

A l'exemple crearem dos llocs al servidor 0375: `web1.educem.cat` i `web2.educem.cat`.

#### Crear Virtual Host

**Pas 1: Registres DNS dels llocs web**

El client ha de poder resoldre els noms dels llocs. Al servidor DNS primari (Tema 1):

1. Afegiu els registres a la zona `educem.cat` (`/etc/bind/zones/db.educem.cat`).
2. **Incrementeu el serial**.

```
web1        IN  A       192.168.1.2
web2        IN  A       192.168.1.2
```

```bash
# Al servidor DNS (0375)
sudo named-checkzone educem.cat /etc/bind/zones/db.educem.cat
sudo rndc reload

# Al client: comprovar la resolució
dig +short web1.educem.cat
dig +short web2.educem.cat
```

`Nota:` No editeu `/etc/hosts` per fer les proves. Si el client tingués `127.0.0.1` per a aquests noms, intentaria connectar amb ell mateix.

**Pas 2: Crear directori del lloc**

```bash
sudo mkdir -p /var/www/web1.educem.cat/html
sudo mkdir -p /var/www/web2.educem.cat/html
```

**Pas 3: Crear pàgina d'exemple**

```bash
# web1.educem.cat
echo "<h1>Hola món! web1.educem.cat</h1>" | sudo tee /var/www/web1.educem.cat/html/index.html

# web2.educem.cat
echo "<h1>Hola món! web2.educem.cat</h1>" | sudo tee /var/www/web2.educem.cat/html/index.html

# Comprovar permisos: Apache (usuari www-data) ha de poder llegir els fitxers (644) i les carpetes (755)
ls -l /var/www/web1.educem.cat/html/
```

`Nota:` Escrivim amb `sudo tee` perquè la carpeta és de root. Amb `echo ... > fitxer` sense `sudo`, la redirecció la fa l'usuari normal i dona `Permission denied`.

**Pas 4: Crear configuració Virtual Host**

```bash
sudo nano /etc/apache2/sites-available/web1.educem.cat.conf
```

```apache
<VirtualHost *:80>
    # Nom del servidor
    ServerName web1.educem.cat

    # Email de l'administrador
    ServerAdmin admin@educem.cat

    # Document root
    DocumentRoot /var/www/web1.educem.cat/html

    # Configuració del directori
    <Directory /var/www/web1.educem.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Logs específics
    ErrorLog ${APACHE_LOG_DIR}/web1.educem.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/web1.educem.cat-access.log combined
</VirtualHost>
```

`Nota:` Si el lloc té altres noms (per exemple `www.web1.educem.cat`), s'afegeixen amb `ServerAlias` i cal crear també el registre DNS corresponent.

**Repeteix el procés per web2.educem.cat** (canviant `web1` per `web2` a tot el fitxer):

```bash
sudo nano /etc/apache2/sites-available/web2.educem.cat.conf
```

**Pas 5: Activar els Virtual Hosts**

```bash
# Activar els llocs web
sudo a2ensite web1.educem.cat.conf
sudo a2ensite web2.educem.cat.conf

# Desactivar lloc per defecte (opcional)
sudo a2dissite 000-default.conf

# Verificar si hi ha errors de configuració
sudo apache2ctl configtest

# Veure els Virtual Hosts carregats
sudo apache2ctl -S

# Recarregar Apache (també es pot reiniciar amb restart)
sudo systemctl reload apache2
```

**Notes:**

- Si desactiveu `000-default`, les peticions que no coincideixen amb cap `ServerName` (per exemple, `http://192.168.1.2`) les respon el primer VirtualHost carregat (`web1`).
- L'avís `AH00558: Could not reliably determine the server's fully qualified domain name` no és cap error. Desapareix si definiu un `ServerName` global:

```bash
echo "ServerName web1.educem.cat" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl reload apache2
```

**Pas 6: Proves al servidor i al client**

```bash
curl http://web1.educem.cat
curl http://web2.educem.cat
```

Al servidor, els noms es resolen amb el seu propi BIND (netplan del Tema 1). Al client, també es pot provar des del navegador.
