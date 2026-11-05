## 08. Instal·lació de Mailman 3

### Components de Mailman 3

**GNU Mailman 3** té diverses peces, que s'instal·len juntes amb el paquet `mailman3-full`:

| Component              | Funció                                                        | On escolta                                   |
| ---------------------- | ------------------------------------------------------------- | -------------------------------------------- |
| **Mailman Core**       | Rep els missatges, aplica les regles de cada llista i els reenvia | LMTP `127.0.0.1:8024`, API REST `localhost:8001` |
| **Postorius**          | Web d'administració i de subscripció a les llistes            | Aplicació web (Django)                       |
| **HyperKitty**         | Arxiu web dels missatges, amb cerca i fils de conversa        | Aplicació web (Django)                       |
| **mailman3-web**       | Executa Postorius i HyperKitty amb uWSGI                      | Sòcol `/run/mailman3-web/uwsgi.sock`         |
| **Postfix** (Tema 5)   | Rep el correu de la llista i lliura les còpies als membres    | Port 25                                      |
| **Apache** (Tema 3)    | Publica la web de Postorius i HyperKitty                      | Ports 80 i 443                               |

```
            missatge a projectes@educem.cat
 Client ──SMTP──► Postfix ──LMTP 8024──► Mailman Core ──SMTP 25──► Postfix ──► bústies dels membres
                                              ▲
                                              │ API REST (8001)
 Navegador ──HTTPS──► Apache ──uWSGI──► Postorius / HyperKitty
```

Mailman **no lliura mai el correu directament**: el rep de Postfix i li torna perquè el lliuri.

### Requisits Previs

- **Correu del Tema 5 funcionant**, amb la configuració bàsica: el domini `educem.cat` a `mydestination` i els usuaris `usuari1` i `usuari2` amb bústia. Si heu fet l'ampliació de dominis virtuals (Tema 5, apartat 10), la integració amb Postfix és diferent i no la tractem aquí.
- **Apache del Tema 3** als ports 80 i 443.
- **Registre DNS** `llistes.educem.cat` (apartat 01).

`IMPORTANT:` Aquest apartat modifica la configuració de Postfix. Feu una instantània del servidor abans de començar.

```bash
# El correu local funciona?
echo "Prova abans de Mailman" | mail -s "Prova" usuari2@educem.cat
sudo ls /home/usuari2/Maildir/new/ | tail -1

dig +short llistes.educem.cat           # www.educem.cat. i 192.168.1.2
```

### Instal·lació

El paquet fa unes quantes preguntes de configuració, però Ubuntu només mostra les preguntes importants. Amb `DEBIAN_PRIORITY=medium` també mostra les de Mailman:

```bash
sudo apt update
sudo DEBIAN_PRIORITY=medium apt install mailman3-full
```

| Pregunta                                                    | Resposta                                       |
| ----------------------------------------------------------- | ---------------------------------------------- |
| Configure database for mailman3 / mailman3-web with dbconfig-common? | **Yes**                               |
| Database type                                               | **sqlite3**                                    |
| Add the HyperKitty configuration to mailman.cfg?            | **Yes**                                        |
| Domain name for sender email addresses                      | `educem.cat`                                   |
| Username of the Postorius superuser                         | `admin`                                        |
| Email address of the Postorius superuser                    | `usuari1@educem.cat`                           |
| Password for the Postorius superuser                        | una contrasenya per a `admin`                  |
| Web server(s) to configure automatically                    | **apache2**                                    |
| Should the webserver(s) be restarted now?                   | **Yes**                                        |

- **SQLite** és suficient per al laboratori. En producció es fa servir PostgreSQL o MariaDB.
- **L'adreça del superusuari** ha de ser una bústia real del servidor, perquè hi arribarà un correu de verificació (apartat 09). Al laboratori, `usuari1` fa d'administrador de les llistes.
- **apache2:** l'instal·lador activa el mòdul `proxy_uwsgi` d'Apache i publica la web de Mailman al camí `/mailman3` de tots els llocs web.

**Comprovacions:**

```bash
# Els dos serveis funcionen
systemctl is-active mailman3 mailman3-web

# Mailman Core escolta només a localhost
sudo ss -tlnp | grep -E ':(8001|8024) '

# La configuració d'HyperKitty és a mailman.cfg
sudo grep -A3 'archiver.hyperkitty' /etc/mailman3/mailman.cfg

# Apache publica /mailman3
ls /etc/apache2/conf-enabled/ | grep mailman
sudo apache2ctl -M | grep uwsgi
```

Si alguna comprovació falla perquè l'instal·lador no ha fet les preguntes, vegeu la taula de Resolució de Problemes.

### Fitxers de Configuració

| Fitxer                                   | Contingut                                                        |
| ---------------------------------------- | ---------------------------------------------------------------- |
| `/etc/mailman3/mailman.cfg`              | Configuració de Mailman Core (MTA, API REST, base de dades)      |
| `/etc/mailman3/mailman-web.py`           | Configuració de Postorius i HyperKitty (Django)                  |
| `/etc/mailman3/mailman-hyperkitty.cfg`   | Connexió entre Mailman Core i l'arxiu HyperKitty                 |
| `/etc/mailman3/apache.conf`              | Configuració d'Apache per al camí `/mailman3`                    |
| `/var/lib/mailman3/data/`                | Mapes de Postfix que genera Mailman (`postfix_lmtp`, `postfix_domains`) |
| `/var/log/mailman3/`                     | Registres de Mailman Core (`mailman.log`, `smtp.log`...) i de la web (`web/`) |

### Configuració de Mailman Core

Indiqueu l'adreça de l'administrador del servidor de llistes, que rep els avisos de Mailman:

```bash
sudo nano /etc/mailman3/mailman.cfg
```

```ini
[mailman]
site_owner: usuari1@educem.cat
```

La secció `[mta]` ja és correcta per a Postfix al mateix servidor. No cal canviar-la:

```ini
[mta]
incoming: mailman.mta.postfix.LMTP
outgoing: mailman.mta.deliver.deliver
lmtp_host: 127.0.0.1
lmtp_port: 8024
smtp_host: localhost
smtp_port: 25
configuration: python:mailman.config.postfix
```

Reinicieu Mailman i genereu els mapes per a Postfix:

```bash
sudo systemctl restart mailman3

# Mailman s'executa amb l'usuari list
sudo -u list mailman info
sudo -u list mailman aliases
ls -l /var/lib/mailman3/data/
```

`mailman aliases` crea `postfix_lmtp` i `postfix_domains` i els seus fitxers `.db`. Cada vegada que es crea o s'esborra una llista, Mailman els actualitza automàticament.

### Integració amb Postfix

Postfix ha de saber quines adreces són de llistes i lliurar-les a Mailman per LMTP, en lloc de buscar una bústia:

```bash
sudo postconf -e 'owner_request_special=no'
sudo postconf -e 'transport_maps=hash:/var/lib/mailman3/data/postfix_lmtp'
sudo postconf -e 'local_recipient_maps=proxy:unix:passwd.byname $alias_maps hash:/var/lib/mailman3/data/postfix_lmtp'

sudo postfix check
sudo systemctl reload postfix
postconf recipient_delimiter        # recipient_delimiter = +  (Tema 5)
```

| Paràmetre                   | Funció                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------- |
| `transport_maps`            | Les adreces de les llistes es lliuren per LMTP a Mailman (`lmtp:[127.0.0.1]:8024`)     |
| `local_recipient_maps`      | Adreces locals vàlides: els usuaris del sistema, els àlies i, ara, les llistes. Si no hi fossin, Postfix rebutjaria `projectes@educem.cat` perquè no és cap usuari |
| `owner_request_special = no` | Postfix no tracta de manera especial les adreces `-owner` i `-request`: les passa a Mailman |
| `recipient_delimiter = +`   | Ja configurat al Tema 5. Mailman el fa servir per identificar els rebots (VERP)       |

**No afegiu `relay_domains`.** La documentació de Mailman el fa servir per als dominis que no són locals. Al laboratori, `educem.cat` ja és a `mydestination`. Si també el posàvem a `relay_domains`, Postfix avisaria que el domini és a tots dos llocs.

### Web de les Llistes amb HTTPS

L'instal·lador ha publicat Mailman a `/mailman3` de tots els llocs web d'Apache. Creem un lloc propi, `llistes.educem.cat`, amb HTTPS, igual que `web1` al Tema 3 (apartats 11 i 12):

```bash
# Certificat autosignat per a llistes.educem.cat
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/llistes.educem.cat.key \
    -out /etc/ssl/certs/llistes.educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=llistes.educem.cat" \
    -addext "subjectAltName=DNS:llistes.educem.cat"

sudo nano /etc/apache2/sites-available/llistes.educem.cat.conf
```

```apache
<VirtualHost *:443>
    ServerName llistes.educem.cat

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/llistes.educem.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/llistes.educem.cat.key

    # La web de Mailman és a /mailman3 (/etc/mailman3/apache.conf)
    RedirectMatch ^/$ /mailman3/

    ErrorLog ${APACHE_LOG_DIR}/llistes.educem.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/llistes.educem.cat-access.log combined
</VirtualHost>
```

```bash
sudo a2ensite llistes.educem.cat.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

**Per què no fem un VirtualHost al port 80 que redirigeixi a HTTPS?** Mailman Core envia els missatges a l'arxiu HyperKitty amb una petició a `http://localhost/mailman3/hyperkitty/`, i aquesta adreça ha de funcionar sense redirecció. Si el nou lloc fos el primer del port 80, la redirecció trencaria l'arxiu. Al laboratori, els usuaris entren directament per `https://llistes.educem.cat`.

**Prova (des del client):** obriu `https://llistes.educem.cat` al navegador i accepteu l'avís del certificat autosignat. Ha d'aparèixer la pàgina de Postorius amb la llista de llistes, encara buida.

```bash
curl -k -s -o /dev/null -w '%{http_code}\n' https://llistes.educem.cat/mailman3/postorius/lists/   # 200
```

### Tallafoc

No cal obrir cap port nou: el correu fa servir el 25 (Tema 5) i la web, el 443 (Tema 3). Els ports 8001 i 8024 escolten només a `localhost` i no s'han d'obrir mai.

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                                          |
| -------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| L'instal·lador no ha fet les preguntes de Mailman              | Ubuntu només mostra les preguntes d'alta prioritat       | `sudo dpkg-reconfigure mailman3-web` (si pregunta si ha de tornar a instal·lar la base de dades, responeu **No**) |
| No hi ha `archiver.hyperkitty` a `mailman.cfg`                 | No s'ha respost **Yes** a la pregunta d'HyperKitty       | `cat /usr/share/mailman3/mailman_cfg_hyperkitty_snippet.cfg \| sudo tee -a /etc/mailman3/mailman.cfg` i `sudo systemctl restart mailman3` |
| `https://llistes.educem.cat/mailman3/` dona error 404          | Apache no té la configuració de Mailman                  | `sudo ln -s /etc/mailman3/apache.conf /etc/apache2/conf-available/mailman3.conf`, `sudo a2enmod proxy_uwsgi`, `sudo a2enconf mailman3` i `sudo systemctl reload apache2` |
| Error 502 o 503 a `/mailman3`                                   | `mailman3-web` (uWSGI) aturat                            | `sudo systemctl restart mailman3-web`; registre a `/var/log/mailman3/web/mailman-web.log` |
| La web es veu sense estils                                     | Falten els fitxers estàtics                              | `sudo mailman-web collectstatic --noinput` i `sudo mailman-web compress` |
| No existeix el superusuari `admin`                             | No s'ha respost la pregunta del superusuari              | `sudo mailman-web createsuperuser`                               |
| Postfix registra `open database /var/lib/mailman3/data/postfix_lmtp.db` | Els mapes no existeixen                        | `sudo -u list mailman aliases`                                   |
| `mailman` diu que no s'ha d'executar com a root                | S'ha executat amb `sudo mailman`                         | Fer servir `sudo -u list mailman ...`                            |

Registres: `/var/log/mailman3/mailman.log`, `/var/log/mail.log` (Postfix) i `/var/log/apache2/llistes.educem.cat-error.log`.

### Punt de Control

- [ ] `systemctl is-active mailman3 mailman3-web` retorna `active` dues vegades.
- [ ] Mailman escolta a `127.0.0.1:8024` (LMTP) i a `localhost:8001` (API).
- [ ] `postconf transport_maps local_recipient_maps` mostra els mapes de Mailman.
- [ ] `https://llistes.educem.cat` mostra Postorius.
