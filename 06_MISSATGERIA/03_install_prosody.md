## 03. Instal·lació i Configuració de Prosody

**Prosody** és un servidor XMPP lleuger escrit en Lua. Té una configuració senzilla i ocupa pocs recursos, per això és habitual en organitzacions petites i mitjanes. Les alternatives més conegudes són **ejabberd**, pensat per a serveis molt grans, i **Openfire**, que té un panell d'administració web.

### Requisits Previs

Al servidor 0375:

- Els registres DNS de l'apartat 01 (`xmpp`, `fitxers` i el registre SRV).
- Accés a Internet per instal·lar paquets.

```bash
dig +short xmpp.educem.cat                       # 192.168.1.2
dig +short SRV _xmpp-client._tcp.educem.cat      # 0 5 5222 xmpp.educem.cat.
```

### Instal·lació

```bash
sudo apt update
sudo apt install prosody

# Versió instal·lada (Ubuntu 24.04: 0.12.x)
apt policy prosody

# El servei s'engega automàticament
systemctl status prosody
```

`Nota:` Ubuntu 26.04 i Debian 13 instal·len **Prosody 13**. Les opcions i les ordres d'aquest tema existeixen a totes dues versions.

### Fitxers i Directoris

```
/etc/prosody/
├── prosody.cfg.lua        # Configuració global
├── conf.avail/            # Configuració de cada domini (com sites-available d'Apache)
│   ├── localhost.cfg.lua
│   └── example.com.cfg.lua
├── conf.d/                # Dominis actius: enllaços als fitxers de conf.avail
└── certs/                 # Certificats i claus privades (root:prosody, permisos 750)

/var/lib/prosody/          # Dades: comptes, llistes de contactes, sales i historial
/var/log/prosody/
├── prosody.log            # Registre general
└── prosody.err            # Errors
```

### La Sintaxi de la Configuració

La configuració està escrita en **Lua**:

```lua
-- Això és un comentari
opcio = "valor"                       -- text entre cometes
altra_opcio = true                    -- true / false
llista = { "element1"; "element2" }   -- llistes entre claus

VirtualHost "educem.cat"              -- a partir d'aquí, opcions només d'aquest domini
```

**Secció global i seccions de domini.** Les opcions que hi ha abans de la primera línia `VirtualHost` o `Component` són **globals**. Les que hi ha després s'apliquen només a aquell domini o component, fins a la línia `VirtualHost` o `Component` següent. Per això, les opcions globals s'han de posar sempre al principi de `prosody.cfg.lua`.

### Configuració Global

Feu una còpia del fitxer original i editeu-lo:

```bash
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.orig
sudo nano /etc/prosody/prosody.cfg.lua
```

**1. Administradors.** Al principi del fitxer, indiqueu quin compte administra el servidor:

```lua
admins = { "usuari1@educem.cat" }
```

**2. Historial al servidor.** Dins de `modules_enabled`, traieu el comentari (`--`) de la línia del mòdul `mam`:

```lua
		"mam"; -- Store recent messages to allow multi-device synchronization
```

Amb el mòdul `mam`, el servidor guarda els missatges i els clients poden recuperar l'historial des de qualsevol dispositiu. Per defecte els guarda una setmana (`archive_expires_after = "1w"`).

Les altres opcions es poden deixar com estan. Les més importants ja tenen valors segurs:

| Opció                          | Valor           | Significat                                                       |
| ------------------------------ | --------------- | ---------------------------------------------------------------- |
| `authentication`               | `"internal_hashed"` | Comptes propis de Prosody, amb les contrasenyes en format hash |
| `allow_registration`           | `false` (per defecte) | Els usuaris no es poden crear el compte des del client     |
| `c2s_require_encryption`       | `true` (per defecte) | Els clients s'han de connectar amb TLS                      |
| `archive_expires_after`        | `"1w"`          | Temps que es guarda l'historial                                  |
| `certificates`                 | `"certs"`       | Directori dels certificats (`/etc/prosody/certs`)                |

### El Domini `educem.cat`

Igual que els llocs web d'Apache, cada domini té el seu fitxer a `conf.avail/` i s'activa amb un enllaç a `conf.d/`:

```bash
sudo nano /etc/prosody/conf.avail/educem.cat.cfg.lua
```

```lua
-- Servei de missatgeria del domini educem.cat
VirtualHost "educem.cat"

-- Els components (sales de xat i fitxers) s'afegeixen a l'apartat 05
```

```bash
sudo ln -s /etc/prosody/conf.avail/educem.cat.cfg.lua /etc/prosody/conf.d/
ls -l /etc/prosody/conf.d/
```

El domini és `educem.cat`, no `xmpp.educem.cat`: és la part de l'adreça que hi ha després de l'`@`.

### Certificat del Servei

Prosody busca automàticament el certificat a `/etc/prosody/certs/`, amb el nom del domini: `educem.cat.crt` i `educem.cat.key`. Generem un certificat autosignat, igual que al Tema 3 (apartat 12):

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/prosody/certs/educem.cat.key \
    -out /etc/prosody/certs/educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=educem.cat" \
    -addext "subjectAltName=DNS:educem.cat,DNS:xmpp.educem.cat,DNS:sales.educem.cat,DNS:fitxers.educem.cat"

# Prosody s'executa amb l'usuari prosody: ha de poder llegir la clau
sudo chown root:prosody /etc/prosody/certs/educem.cat.key
sudo chmod 640 /etc/prosody/certs/educem.cat.key
```

- **`educem.cat`:** el domini de les adreces. És el nom que comprova el client.
- **`sales.educem.cat` i `fitxers.educem.cat`:** els components de l'apartat 05. Prosody fa servir el mateix certificat perquè els noms hi són inclosos.
- **`xmpp.educem.cat`:** el nom de l'equip; no és obligatori, però facilita les proves amb `openssl`.

Amb un domini real, el certificat s'obtindria de Let's Encrypt (Tema 3, apartat 16) i s'importaria amb `sudo prosodyctl --root cert import /etc/letsencrypt/live`.

### Comprovar i Aplicar la Configuració

```bash
# Revisar la configuració abans de reiniciar
sudo prosodyctl check config
sudo prosodyctl check certs

# Aplicar els canvis
sudo systemctl restart prosody
systemctl is-active prosody

# Ports en què escolta
sudo ss -tlnp | grep -E ':(5222|5269) '
```

- `check config` indica errors de sintaxi i opcions mal posades.
- `check certs` comprova, per a cada domini, que hi ha certificat i clau, que el certificat no ha caducat i que inclou el nom del domini. Si avisa que no troba el certificat de `localhost`, no cal fer-hi res: és el domini de proves que porta la configuració per defecte.
- El servidor escolta al **5222** (clients) i al **5269** (altres servidors). El 5269 el tancarem a l'apartat 06.

### Comptes d'Usuari

Els comptes de Prosody són **propis del servei**: no són els usuaris del sistema ni els comptes de correu del Tema 5. Poden tenir el mateix nom, però cada compte té la seva contrasenya.

```bash
# Crear els comptes (demana la contrasenya dues vegades)
sudo prosodyctl adduser usuari1@educem.cat
sudo prosodyctl adduser usuari2@educem.cat

# Llistar els comptes del domini
sudo prosodyctl shell user list educem.cat

# Canviar la contrasenya d'un compte
sudo prosodyctl passwd usuari2@educem.cat

# Esborrar un compte (no ho feu ara)
# sudo prosodyctl deluser usuari2@educem.cat
```

En una empresa, Prosody es connecta al directori LDAP o a l'Active Directory perquè cada usuari tingui una sola contrasenya (apartat 06).

### Tallafoc

Si el tallafoc del servidor està actiu (Tema 4, apartat 14), obriu el port dels clients només per a la xarxa interna:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 5222 proto tcp
sudo ufw status
```

### Prova des del Client

Abans de configurar cap client de missatgeria, comproveu que el servei respon i quin certificat presenta:

```bash
# El port 5222 és accessible?
nc -zv xmpp.educem.cat 5222

# Connexió amb STARTTLS: certificat que presenta el servidor
openssl s_client -connect xmpp.educem.cat:5222 -starttls xmpp -xmpphost educem.cat </dev/null 2>/dev/null \
    | grep -E 'subject=|Verify return code'
```

**Sortida esperada:**

```
subject=C = ES, ST = Catalunya, L = Barcelona, O = EDUCEM, CN = educem.cat
Verify return code: 18 (self-signed certificate)
```

L'error 18 indica que el client no confia en el certificat perquè és autosignat. Ho resoldrem a l'apartat 04.

### Resolució de Problemes

| Problema                                              | Causa                                                    | Solució                                                          |
| ----------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| Prosody no arrenca després d'editar la configuració   | Error de sintaxi (cometes, claus o `;`)                  | `sudo prosodyctl check config` indica el fitxer i la línia       |
| `check certs` diu que no troba el certificat          | El nom del fitxer no coincideix amb el domini            | Els fitxers s'han de dir `educem.cat.crt` i `educem.cat.key`     |
| Error en carregar la clau privada                     | L'usuari `prosody` no pot llegir la clau                 | `chown root:prosody` i `chmod 640` a la clau                     |
| Una opció global no fa efecte                         | S'ha posat després d'una línia `VirtualHost`             | Moure-la al principi de `prosody.cfg.lua`                        |
| `dig SRV` no retorna res al client                    | Registre no afegit, serial no incrementat o memòria cau  | Revisar la zona, `sudo rndc reload` i `resolvectl flush-caches`  |
| `nc` diu `Connection refused` o es queda esperant     | Prosody aturat o tallafoc                                | `systemctl status prosody` i `sudo ufw status`                   |

Els registres són a `/var/log/prosody/prosody.log` i `/var/log/prosody/prosody.err`, i també amb `sudo journalctl -u prosody`.

### Punt de Control

- [ ] `systemctl is-active prosody` retorna `active`.
- [ ] `sudo prosodyctl check config` no mostra errors.
- [ ] Prosody escolta al port 5222.
- [ ] Els comptes `usuari1@educem.cat` i `usuari2@educem.cat` existeixen.
- [ ] Des del client, `openssl s_client` mostra el certificat `CN = educem.cat`.
