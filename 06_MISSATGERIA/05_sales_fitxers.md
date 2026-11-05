## 05. Sales de Xat i Compartició de Fitxers

### Sales de Xat (MUC)

Les sales de xat (*Multi-User Chat*, XEP-0045) permeten converses de grup. Cada sala té una adreça, com `asix@sales.educem.cat`, i cada participant hi entra amb un **àlies** (*nick*).

Les sales les ofereix un **component** del servidor, amb un domini propi (`sales.educem.cat`). Els usuaris del mateix servidor no necessiten cap registre DNS per a aquest domini: el seu client envia les peticions a Prosody, i Prosody les passa al component.

**Opcions de cada sala:**

| Opció (Prosody)                                            | Significat                                                           |
| ---------------------------------------------------------- | -------------------------------------------------------------------- |
| **Persistent** (*room should remain even when it is empty*) | La sala es manté quan en surt l'últim participant. Si no, desapareix |
| **Pública** (*Include room information in public lists*)   | La sala apareix a la llista de sales del servidor                    |
| **Només membres** (*Only allow members to join*)           | Només hi poden entrar els usuaris de la llista de membres            |
| **Moderada** (*Moderated, require permission to speak*)    | Només hi poden escriure els participants amb veu                     |
| **Contrasenya** (*Password*)                               | Cal la contrasenya per entrar-hi                                     |
| **Qui veu les adreces reals** (*Addresses (JIDs) of room occupants may be viewed by*) | Només els moderadors o tothom          |
| **Historial** (*Archive chat on server*)                   | El servidor guarda la conversa de la sala                            |

### Afiliacions i Rols

Cada usuari té una **afiliació** a la sala, que es manté entre sessions, i un **rol**, que només dura mentre és a la sala:

| Afiliació   | Permisos                                                                 |
| ----------- | ------------------------------------------------------------------------ |
| `owner`     | Propietari: configura la sala, nomena administradors i la pot destruir   |
| `admin`     | Administrador: gestiona els membres i els usuaris vetats                 |
| `member`    | Membre: pot entrar a les sales de només membres                          |
| `none`      | Cap afiliació (usuari qualsevol)                                         |
| `outcast`   | Vetat: no hi pot entrar                                                  |

| Rol           | Permisos dins de la sala                                     |
| ------------- | ------------------------------------------------------------ |
| `moderator`   | Pot expulsar participants i donar o treure la veu            |
| `participant` | Pot escriure                                                 |
| `visitor`     | Només pot llegir (a les sales moderades)                     |

### Configurar el Servei de Sales

Afegiu el component al fitxer del domini, a continuació del `VirtualHost`:

```bash
sudo nano /etc/prosody/conf.avail/educem.cat.cfg.lua
```

```lua
-- Servei de missatgeria del domini educem.cat
VirtualHost "educem.cat"

-- Sales de xat
Component "sales.educem.cat" "muc"
    name = "Sales de xat d'educem.cat"
    restrict_room_creation = "local"     -- només els usuaris d'educem.cat poden crear sales
    modules_enabled = { "muc_mam" }      -- historial de les sales al servidor
```

```bash
sudo prosodyctl check config
sudo systemctl restart prosody
```

Amb `restrict_room_creation = "local"`, només els comptes del domini `educem.cat` poden crear sales. Els valors possibles són:

- `false`: qualsevol usuari en pot crear (valor per defecte).
- `true`: només els administradors del servidor.
- `"local"`: els usuaris del domini.

### Crear una Sala Pública

**Amb Gajim** (`usuari1`): creeu un xat de grup nou amb l'adreça `asix@sales.educem.cat` i marqueu-lo com a públic. Gajim crea la sala amb `usuari1` com a propietari.

**Amb Profanity** (`usuari2`): entreu a la sala que ha creat `usuari1`:

```
/join asix@sales.educem.cat nick Usuari2
Hola a tothom!
/occupants show
```

Si la sala no existeix, Profanity la crea i la deixa **bloquejada** fins que el propietari en confirma la configuració. `/room accept` accepta la configuració per defecte, i `/room config` permet modificar-la.

**Al servidor**, l'administrador pot llistar les sales:

```bash
sudo prosodyctl shell muc list sales.educem.cat
```

### Crear una Sala Només per a Membres

Una sala per al professorat del departament, a la qual no pugui entrar ningú més:

1. Amb Gajim, `usuari1` crea la sala `departament@sales.educem.cat`.
2. A la configuració de la sala, activa **Persistent**, **Only allow members to join** i **Archive chat on server**, i desactiva **Include room information in public lists**.
3. Afegeix `usuari2@educem.cat` amb l'afiliació `member`.

Amb Profanity, el propietari ho pot fer des de la finestra de la sala:

```
/affiliation set member usuari2@educem.cat
/affiliation list member
```

L'administrador del servidor també pot donar l'afiliació des del *shell* de Prosody:

```bash
sudo prosodyctl shell
```

```lua
muc:room("departament@sales.educem.cat"):set_affiliation(true, "usuari2@educem.cat", "member")
```

Per sortir del *shell*, escriviu `quit`.

**Prova:** un tercer compte que no sigui membre (per exemple, `usuari3@educem.cat`, creat amb `prosodyctl adduser`) no pot entrar a la sala.

### Compartició de Fitxers (XEP-0363)

Per enviar fitxers (imatges, documents, àudio), els clients XMPP actuals fan servir **HTTP File Upload**:

```
 Gajim (usuari1)                Prosody                          Gajim (usuari2)
      │ 1. Vull pujar foto.jpg (2 MB)  │                                 │
      │──────────────────────────────>│                                 │
      │ 2. Puja-la a aquesta URL      │                                 │
      │<──────────────────────────────│                                 │
      │ 3. PUT https://fitxers.educem.cat:5281/file_share/...           │
      │──────────────────────────────>│ (desa el fitxer)                │
      │ 4. Missatge amb l'enllaç ─────────────────────────────────────>│
      │                               │ 5. GET de l'enllaç (HTTPS)      │
      │                               │<────────────────────────────────│
```

El fitxer no viatja dins del missatge. Es puja al servidor per HTTPS, i el missatge només conté l'enllaç. Així funciona amb sales, amb diversos dispositius i amb usuaris desconnectats.

**Configuració.** Afegiu un altre component al mateix fitxer:

```lua
-- Compartició de fitxers
Component "fitxers.educem.cat" "http_file_share"
    http_file_share_size_limit = 10*1024*1024     -- 10 MB per fitxer
    http_file_share_daily_quota = 100*1024*1024   -- 100 MB per usuari i dia
```

```bash
sudo prosodyctl check config
sudo systemctl restart prosody

# Ara Prosody també escolta al 5281 (HTTPS)
sudo ss -tlnp | grep ':5281 '

# Obrir el port a la xarxa interna
sudo ufw allow from 192.168.1.0/24 to any port 5281 proto tcp
```

- El domini `fitxers.educem.cat` **sí que necessita DNS** (el CNAME de l'apartat 01), perquè els clients s'hi connecten per HTTPS.
- El certificat de l'apartat 03 ja inclou `fitxers.educem.cat`.
- Els clients troben el servei automàticament, perquè és un subdomini del domini dels usuaris.
- Per defecte, només el poden fer servir els usuaris del servidor, i els fitxers s'esborren al cap d'una setmana.

**Prova:** des de Gajim, envieu una imatge a `usuari2` i a la sala `asix`. Els fitxers queden a `/var/lib/prosody/`, dins del directori del component. Prosody escriu els punts dels noms de domini com a `%2e` (`fitxers%2eeducem%2ecat`):

```bash
sudo find /var/lib/prosody -path '*fitxers*' -type f
```

`Nota:` A Prosody 0.12, l'opció que indica quant de temps es guarden els fitxers (`http_file_share_expires_after`) és un número de segons; a Prosody 13 també accepta un text com `"1 week"`.

### Resolució de Problemes

| Problema                                              | Causa                                                    | Solució                                                          |
| ----------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| No es pot crear cap sala                              | Component no carregat o adreça de sala mal escrita       | `sudo prosodyctl check config`; la sala ha de ser `nom@sales.educem.cat` |
| Un usuari d'un altre domini no pot crear sales        | `restrict_room_creation = "local"`                       | És el comportament esperat                                       |
| `usuari2` no pot entrar a `departament`               | No té l'afiliació `member`                               | `/affiliation set member usuari2@educem.cat` (propietari)        |
| La sala desapareix quan tothom en surt                | No és persistent                                         | Activar **Persistent** a la configuració de la sala              |
| Gajim no ofereix l'opció d'enviar fitxers             | El component `http_file_share` no està carregat          | Revisar la configuració i reiniciar Prosody                      |
| L'enviament de fitxers falla                          | Falta el DNS de `fitxers`, el port 5281 està tancat o el certificat no inclou el nom | `dig fitxers.educem.cat`, `sudo ufw status`, apartat 03 |

### Punt de Control

- [ ] `sudo prosodyctl shell muc list sales.educem.cat` mostra les sales `asix` i `departament`.
- [ ] `usuari1` i `usuari2` conversen a la sala `asix`.
- [ ] Només els membres poden entrar a `departament`.
- [ ] Gajim envia una imatge a `usuari2` i l'altre usuari la veu.
