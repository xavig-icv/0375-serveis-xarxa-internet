## 09. Clients FTP: Línia d'Ordres, Gràfics i Navegador

Un cop configurat el servidor, cal saber utilitzar-lo des de diferents tipus de clients. Cada client és útil en un context diferent:

| Tipus de client      | Exemples                              | Quan s'utilitza                                   |
| -------------------- | ------------------------------------- | ------------------------------------------------- |
| Línia d'ordres       | `ftp`, `lftp`, `curl`, `wget`         | Servidors sense entorn gràfic, scripts, diagnòstic |
| Gràfics              | FileZilla, WinSCP, Cyberduck          | Usuaris finals, gestió de molts fitxers           |
| Gestor de fitxers    | Nautilus (GNOME), Explorador de Windows | Accés ràpid sense instal·lar res               |
| Navegador web        | Firefox, Chrome (fins a 2021)         | Descàrregues anònimes (avui obsolet)              |

### Client `ftp` (Línia d'Ordres)

És el client clàssic, disponible a Linux (paquet `ftp`) i també a Windows (`ftp.exe`).

```bash
# Connectar al servidor
ftp 192.168.1.2

# Connectar sense iniciar sessió automàticament
ftp -n 192.168.1.2
ftp> user usuari1
```

**Ordres principals del client `ftp`:**

| Ordre                    | Descripció                                                       |
| ------------------------ | ---------------------------------------------------------------- |
| `open servidor`          | Connectar a un servidor                                          |
| `user nom`               | Iniciar sessió amb un usuari                                     |
| `ls` / `dir`             | Llistar el directori remot                                       |
| `pwd`                    | Directori remot actual                                           |
| `cd carpeta`             | Canviar de directori remot                                       |
| `lcd carpeta`            | Canviar de directori **local**                                   |
| `!ls`                    | Executar una ordre al sistema **local** (`!` + ordre)            |
| `get fitxer`             | Descarregar un fitxer                                            |
| `put fitxer`             | Pujar un fitxer                                                  |
| `mget *.txt`             | Descarregar diversos fitxers                                     |
| `mput *.txt`             | Pujar diversos fitxers                                           |
| `prompt`                 | Activar/desactivar la confirmació a `mget`/`mput`                |
| `delete fitxer`          | Esborrar un fitxer remot                                         |
| `mkdir` / `rmdir`        | Crear / esborrar carpetes remotes                                |
| `rename origen destí`    | Canviar el nom d'un fitxer remot                                 |
| `binary` / `ascii`       | Tipus de transferència                                           |
| `passive`                | Alternar entre mode passiu i actiu                               |
| `hash`                   | Mostrar `#` durant la transferència (indicador de progrés)       |
| `status`                 | Estat de la connexió i de les opcions                            |
| `debug`                  | Mostrar les ordres del protocol que s'envien (`---> RETR ...`)   |
| `bye` / `quit`           | Tancar la sessió                                                 |

**Exemple de sessió:**

```
ftp> debug
Debugging on (debug=1).
ftp> lcd /tmp
Local directory now: /tmp
ftp> cd fitxers
---> CWD fitxers
250 Directory successfully changed.
ftp> prompt
Interactive mode off.
ftp> mget *.txt
---> TYPE I
---> EPSV
---> RETR prova.txt
226 Transfer complete.
---> RETR prova2.txt
226 Transfer complete.
ftp> bye
```

Amb `debug` podem veure com cada ordre del client (`mget`) es tradueix a ordres del protocol (`TYPE`, `EPSV`, `RETR`).

**Client `ftp` de Windows:** el client integrat de Windows (`ftp.exe`) **només funciona en mode actiu**. Si el tallafoc de Windows està activat o hi ha NAT, el llistat fallarà amb `425`. És un bon exemple pràctic del problema del mode actiu.

### Client `lftp` (Línia d'Ordres Avançat)

`lftp` és un client molt més potent: reintents automàtics, transferències en paral·lel, sincronització de carpetes, scripts i suport per a FTP, FTPS, SFTP i HTTP.

```bash
# Connectar amb un usuari (demana la contrasenya)
lftp -u usuari1 192.168.1.2

# Connectar indicant el protocol a la URL
lftp ftp://usuari1@192.168.1.2
```

**Ordres destacades de `lftp`:**

```
lftp usuari1@192.168.1.2:~> ls
lftp usuari1@192.168.1.2:/> cd fitxers
lftp usuari1@192.168.1.2:/fitxers> get -c fitxer_gran.iso   # -c: continua una descàrrega interrompuda
lftp usuari1@192.168.1.2:/fitxers> pget -n 4 fitxer.iso     # descarrega en 4 parts paral·leles
lftp usuari1@192.168.1.2:/fitxers> mirror . /tmp/copia      # descarrega tota la carpeta
lftp usuari1@192.168.1.2:/fitxers> mirror -R ./web .        # puja tota una carpeta local (Reverse)
lftp usuari1@192.168.1.2:/fitxers> bye
```

**Ús en scripts (sense interacció):**

```bash
# Executar ordres i sortir
lftp -u usuari1,Contrasenya -e "cd fitxers; ls; bye" 192.168.1.2

# Sincronitzar una carpeta local cap al servidor (només puja els canvis)
lftp -u usuari1,Contrasenya -e "mirror -R --only-newer ./documents /fitxers/documents; bye" 192.168.1.2
```

`Nota:` escriure la contrasenya a la línia d'ordres és insegur (queda a l'historial i és visible amb `ps`). Als scripts reals s'utilitza el fitxer `~/.netrc` amb permisos `600` o, millor encara, **SFTP amb claus** (punt 12).

### Clients `curl` i `wget`

Són eines de transferència d'URL que també entenen FTP. Molt útils en scripts.

```bash
# Llistar una carpeta (demana la contrasenya amb -u usuari)
curl -u usuari1 ftp://192.168.1.2/fitxers/

# Descarregar un fitxer
curl -u usuari1 -O ftp://192.168.1.2/fitxers/prova.txt

# Pujar un fitxer (-T = upload)
curl -u usuari1 -T informe.pdf ftp://192.168.1.2/fitxers/

# Forçar el mode actiu (per defecte curl fa servir el passiu)
curl -u usuari1 --ftp-port - ftp://192.168.1.2/fitxers/

# wget: descarregar recursivament la carpeta pública anònima
wget -r -np -nH ftp://192.168.1.2/pub/
```

### Clients Gràfics

#### FileZilla (Linux, Windows, macOS)

És el client gràfic FTP més conegut. És programari lliure i suporta FTP, FTPS i SFTP.

```bash
# Instal·lació a Ubuntu/Debian
sudo apt install filezilla
```

`Nota:` a Windows, descarregueu-lo **només des del web oficial** i aneu amb compte durant la instal·lació: l'instal·lador pot oferir programari addicional no desitjat. Una alternativa a Windows és **WinSCP**.

**Connexió ràpida:** a la barra superior escriviu Servidor `192.168.1.2`, Usuari, Contrasenya i Port `21`.

**Gestor de llocs (recomanat):** `Fitxer → Gestor de llocs → Lloc nou`

```
Protocol:          FTP - Protocol de transferència de fitxers
Servidor:          ftp.educem.cat         Port: 21
Xifratge:          Usa només FTP pla (insegur)     ← ho canviarem al punt 10
Tipus d'accés:     Normal
Usuari:            usuari1
Contrasenya:       ********

Pestanya "Paràmetres de transferència":
Mode de transferència:  Passiu
```

**Interfície de FileZilla:**

```
┌─────────────────────────────────────────────────────────────────┐
│ Registre de missatges (ordres i respostes del protocol)         │
│  Ordre:   PASV                                                  │
│  Resposta: 227 Entering Passive Mode (192,168,1,2,156,80).      │
├────────────────────────────────┬────────────────────────────────┤
│ LLOC LOCAL (el vostre equip)   │ LLOC REMOT (servidor)          │
│  /home/alumne/Documents        │  /fitxers                      │
│   informe.pdf    ── arrossegar ──►                              │
├────────────────────────────────┴────────────────────────────────┤
│ Cua de transferències: fitxers en cua / fallits / correctes     │
└─────────────────────────────────────────────────────────────────┘
```

El **registre de missatges** mostra totes les ordres FTP i és molt útil per diagnosticar problemes.

#### WinSCP (Windows)

Client lliure per a Windows amb suport per a SFTP, SCP, FTP, FTPS, WebDAV i S3. Permet crear **scripts** de transferència i és molt utilitzat per administradors de sistemes Windows.

#### Gestors de Fitxers del Sistema

**Nautilus (Ubuntu Desktop / GNOME):** `Altres ubicacions → Connecta al servidor`

```
ftp://usuari1@192.168.1.2
ftp://192.168.1.2            (anònim)
```

La carpeta remota s'obre com una carpeta local: podeu arrossegar fitxers, obrir-los directament, etc.

**Explorador de fitxers de Windows:** escriviu a la barra d'adreces:

```
ftp://192.168.1.2
ftp://usuari1@192.168.1.2
```

També es pot afegir de forma permanent amb `Aquest equip → Afegeix una ubicació de xarxa`.

### El Navegador com a Client FTP

Fins fa pocs anys, els navegadors web eren el client FTP més utilitzat per descarregar fitxers de servidors anònims: n'hi havia prou d'escriure `ftp://servidor/` a la barra d'adreces.

**Situació actual:**

- **Chrome** (i els navegadors basats en Chromium, com Edge) va eliminar el suport FTP l'any **2021** (versió 95).
- **Firefox** el va eliminar el mateix any (versió 90).
- Motius: l'FTP no és segur, s'utilitzava molt poc i el codi era difícil de mantenir.

Si avui escriviu `ftp://192.168.1.2` en un navegador, us oferirà obrir una aplicació externa (el gestor de fitxers o FileZilla).

**Alternativa moderna: publicar els fitxers per HTTP(S)**

Els grans servidors de descàrregues (com els repositoris de Debian, que van tancar el seu servei FTP el 2017) publiquen ara els fitxers per HTTPS amb un **llistat de directori**. Podem fer el mateix amb el servidor web del Tema 3, publicant la carpeta `pub` de l'FTP anònim:

**Amb Apache:**

```bash
sudo nano /etc/apache2/conf-available/ftp-pub.conf
```

```apache
# Publicar la carpeta pública de l'FTP a http://servidor/pub/
Alias /pub /srv/ftp/pub

<Directory /srv/ftp/pub>
    Options +Indexes
    IndexOptions FancyIndexing NameWidth=* HTMLTable
    Require all granted
</Directory>
```

```bash
sudo a2enconf ftp-pub
sudo apache2ctl configtest
sudo systemctl reload apache2
```

**Amb Nginx** (dins del bloc `server {}` del lloc per defecte, que al laboratori escolta al port 8080):

```nginx
location /pub/ {
    alias /srv/ftp/pub/;
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Ara, des del navegador del client, el llistat de fitxers es pot veure i descarregar a `http://192.168.1.2/pub/` (Apache) o a `http://192.168.1.2:8080/pub/` (Nginx). És el mateix contingut que es publica per FTP anònim, accessible des de qualsevol navegador i, si s'activa HTTPS, **xifrat**.

```
                ┌──────────────── /srv/ftp/pub ─────────────────┐
                │                                               │
  ftp://servidor/pub/  (vsftpd, anònim)        http://servidor/pub/  (Apache/Nginx)
  clients FTP, scripts                         qualsevol navegador
```

### Resolució de Problemes amb els Clients

| Símptoma                                         | Causa probable                                      |
| ------------------------------------------------ | --------------------------------------------------- |
| `ls` es queda penjat i acaba amb `425`           | Tallafoc bloquejant la connexió de dades (rang passiu tancat o client en mode actiu) |
| FileZilla: "No s'ha pogut obtenir el llistat"    | El mateix: reviseu el mode de transferència i el tallafoc |
| `530 Login incorrect.`                           | Contrasenya incorrecta, usuari a `/etc/ftpusers` o shell no vàlida |
| `530 Permission denied.`                         | Usuari fora de la `userlist`                        |
| `550 Permission denied.`                         | Sense permís d'escriptura (`write_enable`, permisos de carpeta, `cmds_denied`) |
| `421 There are too many connections...`          | S'ha superat `max_per_ip` (tanqueu sessions o reduïu connexions simultànies del client) |
| Fitxers binaris corromputs                       | Transferència en mode ASCII: feu servir `binary`    |

### Punt de Control

- [ ] Heu pujat i descarregat fitxers amb `ftp`, `lftp` i `curl`.
- [ ] Heu fet servir `debug` per veure les ordres del protocol.
- [ ] Heu sincronitzat una carpeta amb `lftp mirror -R`.
- [ ] Heu creat un lloc al gestor de llocs de FileZilla i heu revisat el registre de missatges.
- [ ] Heu accedit al servidor des del gestor de fitxers (Nautilus o Windows).
- [ ] La carpeta `pub` és accessible des del navegador per HTTP.
