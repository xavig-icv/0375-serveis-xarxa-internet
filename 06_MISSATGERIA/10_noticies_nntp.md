## 10. Servei de Notícies (NNTP) i Fòrums

### Què és el Servei de Notícies?

El servei de **notícies** (*Usenet*) és un dels serveis més antics d'Internet (1980). Funciona com un gran tauler d'anuncis dividit per temes:

- Els missatges, anomenats **articles**, es publiquen en **grups de notícies** (*newsgroups*), i qualsevol lector del grup els pot llegir.
- Els grups s'organitzen en **jerarquies**, com els noms de domini però d'esquerra a dreta: `comp.os.linux.networking`, `es.comp.os.linux`, `educem.general`.
- Els servidors de notícies s'intercanvien els articles entre ells. Així, un article publicat en un servidor arriba a tots els servidors que tenen el grup.
- El protocol és **NNTP** (*Network News Transfer Protocol*, RFC 3977), al port **119/tcp**. Amb TLS fa servir el port **563**.

**Jerarquies clàssiques:**

| Jerarquia  | Temes                                        |
| ---------- | -------------------------------------------- |
| `comp.*`   | Informàtica                                  |
| `sci.*`    | Ciència                                      |
| `rec.*`    | Lleure                                       |
| `news.*`   | El funcionament de la mateixa Usenet         |
| `alt.*`    | Temes alternatius, sense control central     |
| `es.*`, `fr.*`... | Grups en altres llengües              |

Una organització també pot tenir una jerarquia pròpia, només als seus servidors, per als debats interns: al laboratori farem `educem.*`.

### Notícies, Llistes i Fòrums

| Característica          | Notícies (NNTP)                     | Llista de distribució                | Fòrum web                         |
| ----------------------- | ----------------------------------- | ------------------------------------ | --------------------------------- |
| On són els missatges    | Al servidor; el client els descarrega | A la bústia de cada membre          | Al servidor web                   |
| Com es llegeixen        | Client de notícies (Thunderbird)    | Client de correu                     | Navegador                         |
| Cal subscriure's        | Al client (no ho sap ningú més)     | Sí, a la llista                      | Normalment cal un compte          |
| Organització            | Grups i fils de conversa            | Fils al client de correu             | Categories i fils                 |
| Ús actual               | Molt reduït                         | Projectes, butlletins                | Comunitats, suport d'usuaris      |

Avui el servei de notícies gairebé no es fa servir. Els **fòrums web** (Discourse, phpBB, Flarum), les plataformes com GitHub Discussions o Reddit i l'arxiu web de les llistes (HyperKitty, apartat 09) n'han ocupat el lloc. Encara així, NNTP és la base de tots ells: grups temàtics, fils de conversa, moderació i lectura en diferit.

### Servidors i Clients

- **Servidors:** **INN** (InterNetNews), el servidor de referència, que farem servir; **Leafnode**, per a xarxes petites.
- **Clients gràfics:** **Thunderbird**, que també és client de notícies.
- **Clients de text:** `tin` i `slrn`.

### Instal·lació d'INN2

Al servidor 0375:

```bash
sudo apt update
sudo apt install inn2
systemctl status inn2
```

El paquet instal·la un servidor que només accepta lectors des del mateix equip (`localhost`).

**Fitxers principals:**

| Fitxer                        | Contingut                                                    |
| ----------------------------- | ------------------------------------------------------------ |
| `/etc/news/inn.conf`          | Configuració general (nom del servidor, organització)        |
| `/etc/news/readers.conf`      | Qui pot llegir i publicar, i a quins grups                    |
| `/etc/news/expire.ctl`        | Quants dies es guarden els articles                          |
| `/etc/news/moderators`        | Adreça del moderador de cada grup moderat                    |
| `/var/lib/news/active`        | Llista de grups del servidor                                 |
| `/var/lib/news/newsgroups`    | Descripció de cada grup                                      |
| `/var/spool/news/articles/`   | Els articles                                                 |
| `/var/log/news/`              | Registres del servei                                         |

### Configuració General

```bash
sudo cp /etc/news/inn.conf /etc/news/inn.conf.orig
sudo nano /etc/news/inn.conf
```

Modifiqueu aquestes línies (si alguna no hi és, afegiu-la):

```
organization:   "EDUCEM - Laboratori ASIX"
pathhost:       news.educem.cat
domain:         educem.cat
```

- **`organization`:** apareix a la capçalera `Organization` dels articles publicats.
- **`pathhost`:** el nom amb què el servidor s'identifica a la capçalera `Path` dels articles.

```bash
sudo systemctl restart inn2
systemctl is-active inn2
sudo ss -tlnp | grep ':119 '
```

### Crear els Grups

```bash
# ctlinnd envia ordres al servidor en marxa; "y" vol dir que s'hi pot publicar
sudo ctlinnd newgroup educem.general y
sudo ctlinnd newgroup educem.asix y

# Descripció dels grups (separada per un tabulador)
printf 'educem.general\tDebats generals del centre\n' | sudo tee -a /var/lib/news/newsgroups
printf 'educem.asix\tGrup del cicle ASIX\n' | sudo tee -a /var/lib/news/newsgroups

# Comprovar-ho
grep educem /var/lib/news/active
```

| Indicador | Tipus de grup                                                         |
| --------- | --------------------------------------------------------------------- |
| `y`       | Es pot publicar directament                                           |
| `m`       | Moderat: els articles s'envien per correu al moderador (`/etc/news/moderators`), que decideix si es publiquen |
| `n`       | Els usuaris locals no hi poden publicar; només arriben articles d'altres servidors |

Per esborrar un grup: `sudo ctlinnd rmgroup educem.asix`.

### Accés dels Lectors

Per defecte, només es pot llegir des del mateix servidor. Afegiu al final de `/etc/news/readers.conf` un bloc per a la xarxa interna:

```bash
sudo nano /etc/news/readers.conf
```

```
# Xarxa interna del laboratori: llegir i publicar als grups educem.*
auth "xarxa-interna" {
    hosts: "192.168.1.0/24"
    default: "<xarxa>"
}

access "xarxa-interna" {
    users: "<xarxa>"
    newsgroups: "educem.*"
    access: RP
}
```

- **`auth`:** identifica els clients. Aquí, per l'adreça IP; també es pot fer amb usuari i contrasenya.
- **`access`:** què poden fer els clients identificats. `R` és llegir, `P` és publicar.
- **`newsgroups: "educem.*"`:** només els grups de la jerarquia `educem`.

No cal reiniciar: el procés que atén els lectors (`nnrpd`) llegeix `readers.conf` a cada connexió nova.

```bash
sudo ufw allow from 192.168.1.0/24 to any port 119 proto tcp
```

### Prova del Protocol des del Client

Igual que vam fer amb SMTP al Tema 5, podem parlar NNTP directament. `nc -C` envia els finals de línia que exigeix el protocol (CRLF):

```bash
nc -C news.educem.cat 119
```

```
200 news.educem.cat InterNetNews NNRP server INN 2.7.2 ready (posting ok)
LIST
215 Newsgroups in form "group high low status"
educem.asix 0000000000 0000000001 y
educem.general 0000000000 0000000001 y
.
POST
340 Ok, recommended message-ID <...>
From: usuari1@educem.cat
Newsgroups: educem.general
Subject: Primer article

Hola! Aquest article l'he publicat des de nc.
.
240 Article received <...>
GROUP educem.general
211 1 1 1 educem.general
ARTICLE 1
220 1 <...> article
...
QUIT
205 Bye!
```

Les línies que comencen amb un número són les respostes del servidor. L'article acaba amb una línia que només té un punt (`.`), com al correu. El text exacte de les respostes pot variar una mica segons la versió d'INN.

### Clients de Notícies

**Thunderbird.** **File** → **New** → **Newsgroup Account**:

1. Nom i adreça de correu de l'usuari (apareixen a la capçalera `From`).
2. **Newsgroup Server:** `news.educem.cat`.
3. Feu clic dret al compte nou → **Subscribe** i marqueu `educem.general` i `educem.asix`.

Els articles es llegeixen com els correus, agrupats per fils. El botó **Reply** respon al grup.

**tin (text):**

```bash
sudo apt install tin
NNTPSERVER=news.educem.cat tin -r
```

| Tecla   | Acció                                    |
| ------- | ---------------------------------------- |
| `y`     | Mostrar tots els grups del servidor      |
| `s`     | Subscriure's al grup seleccionat         |
| `Enter` | Entrar al grup o llegir l'article        |
| `w`     | Escriure un article nou                  |
| `q`     | Tornar enrere o sortir                   |

### Conservació dels Articles

El fitxer `/etc/news/expire.ctl` indica quants dies es guarden els articles. Cada nit, la tasca programada `news.daily` esborra els articles caducats. Per exemple, perquè els grups `educem.*` es guardin 90 dies:

```
# patró:modificador:mínim:per_defecte:màxim
educem.*:A:1:90:90
```

### Resolució de Problemes

| Problema                                                  | Causa                                                    | Solució                                                          |
| --------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| `502 You have no permission to talk` en connectar         | La IP del client no és a cap bloc `auth` de `readers.conf` | Afegir el bloc de la xarxa interna                              |
| Es pot llegir però no publicar (`440 Posting not permitted`) | L'accés no inclou `P`                                 | `access: RP`                                                     |
| El client no veu el grup                                  | El grup no existeix o no és dins de `newsgroups:`        | `grep educem /var/lib/news/active` i revisar `readers.conf`      |
| `ctlinnd` diu que no pot connectar amb el servidor        | `innd` aturat                                            | `sudo systemctl restart inn2`; registres a `/var/log/news/`      |
| `Connection refused` des del client                       | Tallafoc o servei aturat                                 | `sudo ufw status` i `sudo ss -tlnp \| grep ':119 '`              |

### Punt de Control

- [ ] `systemctl is-active inn2` retorna `active` i el servidor escolta al port 119.
- [ ] Els grups `educem.general` i `educem.asix` són a `/var/lib/news/active`.
- [ ] Des del client, `nc -C news.educem.cat 119` permet publicar i llegir un article.
- [ ] Thunderbird o `tin` mostren l'article publicat.
