## 09. Virtual Hosts amb Apache

Apache gestiona els Virtual Hosts mitjançant fitxers de configuració que defineixen com s'han de servir els diferents llocs web. Aquests fitxers permeten associar dominis, IPs o ports a directoris concrets del servidor.

**Apache disposa de 2 directoris que emmagatzemen tots els fitxers de configuració:**

- **sites-available/**: Configuracions de tots els Virtual Hosts `disponibles`
- **sites-enabled/**: Configuracions de tota els Virtual Hosts `actius`

#### Crear Virtual Host

**Pas 0: Aturar nginx i encende Apache**

```bash
sudo systemctl stop nginx
sudo systemctl start apache2
```

**Pas 1: Crear directori del lloc**

```bash
sudo mkdir -p /var/www/exemple1.cat/html
sudo mkdir -p /var/www/exemple2.cat/html

# Establir el propietari
sudo chown -R www-data:www-data /var/www/exemple1.cat
sudo chown -R www-data:www-data /var/www/exemple2.cat

# Permisos (executae després de crear fitxers i subdirectoris del web)
sudo chmod -R 755 /var/www
```

**Pas 2: Crear pàgina d'exemple**

```bash
# exemple1.at
echo "<h1>Hola món! exemple1.cat</h1>" > /var/www/exemple1.cat/html/index.html

# exemple2.cat
echo "<h1>Hola món! exemple2.cat</h1>" > /var/www/exemple2.cat/html/index.html
```

**Pas 3: Crear configuració Virtual Host**

```bash
sudo nano /etc/apache2/sites-available/exemple1.cat.conf
```

```apache
<VirtualHost *:80>
    # Nom del servidor
    ServerName exemple1.cat
    ServerAlias www.exemple1.cat

    # Email de l'administrador
    ServerAdmin admin@exemple1.cat

    # Document root
    DocumentRoot /var/www/exemple1.cat/html

    # Configuració del directori
    <Directory /var/www/exemple1.cat/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Logs específics
    ErrorLog ${APACHE_LOG_DIR}/exemple1.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/exemple1.cat-access.log combined
</VirtualHost>
```

**Repeteix el procés per l'exemple2.cat:**

```bash
sudo nano /etc/apache2/sites-available/exemple2.cat.conf
```

**Pas 4: Activar els Virtual Hosts**

```bash
# Activar els llocs web
sudo a2ensite exemple1.cat.conf
sudo a2ensite exemple2.cat.conf

# Desactivar lloc per defecte (opcional)
sudo a2dissite 000-default.conf

# Verificar si hi ha errors de configuració
sudo apache2ctl configtest

# Recarregar Apache (també es pot reiniciar amb restart)
sudo systemctl reload apache2
```

**Pas 5: Configurar DNS/hosts al servidor i al client**

```bash
# Per proves locals, editar /etc/hosts
# Per la pràctica haureu de tenir el servidor DNS operatiu
sudo nano /etc/hosts

# Afegir:
127.0.0.1    exemple1.cat www.exemple1.cat
127.0.0.1    exemple2.cat www.exemple2.cat
```

**Pas 6: Proves al servidor i al client**

```bash
curl http://exemple1.cat
curl http://exemple2.cat
```
