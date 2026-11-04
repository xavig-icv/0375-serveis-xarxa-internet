## 07. Accés Anònim

L'accés **anònim** permet que qualsevol persona es connecti al servidor **sense tenir un compte**. Històricament era la forma de distribuir programari a Internet (servidors de distribucions Linux, universitats, fabricants de maquinari). Avui moltes d'aquestes descàrregues s'han traslladat a HTTPS, però l'accés anònim encara s'utilitza en xarxes internes per compartir documentació, controladors o imatges ISO.

### Com Funciona l'Accés Anònim

- El client inicia sessió amb l'usuari **`anonymous`** (o **`ftp`**).
- Tradicionalment es demana el correu electrònic com a "contrasenya" (no es comprova).
- vsftpd executa la sessió amb l'usuari del sistema **`ftp`**, creat en instal·lar el paquet.
- L'usuari anònim queda **sempre engabiat** a la seva carpeta arrel: `/srv/ftp`.

```
                                      SERVIDOR (vsftpd)
CLIENT                                 │
  USER anonymous  ───────────────────► │  usuari del sistema: ftp
  PASS (qualsevol)───────────────────► │  arrel (chroot): /srv/ftp
                                       │
                                       │  /srv/ftp/            (root, 755)  arrel
                                       │  ├── pub/             (root, 755)  descàrregues públiques
                                       │  └── incoming/        (root:ftp, 730)  bústia de pujades (opcional)
```

### Riscos de l'Accés Anònim

- **Pujades descontrolades**: si els anònims poden pujar i també descarregar, el servidor es pot convertir en un magatzem de fitxers il·legals (pel·lícules, programari pirata, *malware*).
- **Consum de recursos**: amplada de banda i espai en disc sense cap control.
- **Fuga d'informació**: fitxers publicats per error a la carpeta pública.

**Regla bàsica:** si s'activa la pujada anònima, s'ha de fer amb una **bústia** (*drop box*): els anònims poden pujar fitxers però **no poden veure ni descarregar** el que s'ha pujat.

### Pas 1: Preparar les Carpetes

```bash
# Comprovar l'usuari ftp i la seva carpeta personal
getent passwd ftp
# ftp:x:1xx:6xx:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin

# L'arrel anònima NO ha de ser modificable per l'usuari ftp
sudo chown root:root /srv/ftp
sudo chmod 755 /srv/ftp

# Carpeta pública de descàrregues
sudo mkdir -p /srv/ftp/pub
sudo chmod 755 /srv/ftp/pub

# Fitxers de prova per descarregar
echo "Manual d'ús de la xarxa de domini.cat" | sudo tee /srv/ftp/pub/manual.txt
sudo dd if=/dev/urandom of=/srv/ftp/pub/fitxer_10MB.bin bs=1M count=10 status=none
sudo chmod 644 /srv/ftp/pub/*

# Missatge de benvinguda (es mostra en entrar a la carpeta, dirmessage_enable=YES)
echo "Zona de descàrregues públiques de domini.cat" | sudo tee /srv/ftp/pub/.message
```

### Pas 2: Activar l'Accés Anònim (Només Descàrrega)

Afegiu o modifiqueu aquestes directives a `/etc/vsftpd.conf`:

```
# --- ACCÉS ANÒNIM ---
anonymous_enable=YES
# Arrel dels anònims
anon_root=/srv/ftp
# No demanar contrasenya als anònims
no_anon_password=YES
# Mostrar "ftp" com a propietari en lloc dels UID reals
hide_ids=YES
# Els anònims només poden descarregar fitxers llegibles per tothom
anon_world_readable_only=YES
```

**Molt important:** si teniu activada la **llista blanca** del punt anterior (`userlist_deny=NO`), cal afegir-hi els noms dels usuaris anònims o seran rebutjats amb `530 Permission denied.`:

```bash
printf "anonymous\nftp\n" | sudo tee -a /etc/vsftpd.userlist

sudo systemctl restart vsftpd
```

**Prova des del client:**

```
$ ftp 192.168.1.2
Name (192.168.1.2:alumne): anonymous
230 Login successful.
ftp> cd pub
250-Zona de descàrregues públiques de domini.cat
250 Directory successfully changed.
ftp> ls
-rw-r--r--    1 ftp      ftp      10485760 Oct 09 12:00 fitxer_10MB.bin
-rw-r--r--    1 ftp      ftp            39 Oct 09 12:00 manual.txt
ftp> get manual.txt
226 Transfer complete.
ftp> put prova.txt
550 Permission denied.
```

Els anònims poden descarregar però **no poden pujar** fitxers.

**Prova amb curl** (molt útil per a scripts, l'usuari per defecte de curl és `anonymous`):

```bash
# Llistar la carpeta pública
curl ftp://192.168.1.2/pub/

# Descarregar un fitxer
curl -O ftp://192.168.1.2/pub/manual.txt
```

### Pas 3: Bústia de Pujades Anònimes (Opcional)

Ara permetrem que els anònims puguin **deixar** fitxers a la carpeta `incoming`, però sense poder-los **llistar** ni **descarregar**.

```bash
# Carpeta de pujades: el grup ftp pot escriure i entrar, però no llistar
sudo mkdir /srv/ftp/incoming
sudo chown root:ftp /srv/ftp/incoming
sudo chmod 730 /srv/ftp/incoming

sudo ls -ld /srv/ftp/incoming
# drwx-wx--- 2 root ftp 4096 ... /srv/ftp/incoming
```

**Per què `730`?**

| Qui                 | Permisos | Pot fer                                             |
| ------------------- | -------- | --------------------------------------------------- |
| `root` (propietari) | `rwx`    | Tot                                                 |
| grup `ftp` (anònims)| `-wx`    | Crear fitxers i entrar, però **no llistar** (sense `r`) |
| altres              | `---`    | Res                                                 |

Afegiu a `/etc/vsftpd.conf`:

```
# --- PUJADES ANÒNIMES (bústia) ---
# write_enable=YES ja està activat des del punt 5
anon_upload_enable=YES
anon_mkdir_write_enable=NO
anon_other_write_enable=NO
# Els fitxers pujats es creen amb permisos 600 (no descarregables)
anon_umask=077
# Canviar el propietari dels fitxers pujats
chown_uploads=YES
chown_username=usuari1
```

- `anon_upload_enable`: permet pujar fitxers (requereix `write_enable=YES`).
- `anon_mkdir_write_enable=NO`: els anònims no poden crear carpetes.
- `anon_other_write_enable=NO`: els anònims no poden esborrar ni canviar el nom de fitxers.
- `anon_umask=077`: els fitxers pujats tenen permisos `600`. Com que `anon_world_readable_only=YES`, **cap anònim els podrà descarregar**.
- `chown_uploads` + `chown_username`: els fitxers pujats passen a ser propietat de `usuari1`, l'encarregat de revisar-los.

```bash
sudo systemctl restart vsftpd
```

**Prova des del client:**

```
ftp> cd incoming
250 Directory successfully changed.
ftp> put informe.pdf
226 Transfer complete.
ftp> ls
150 Here comes the directory listing.
226 Transfer done (but failed to open directory).
ftp> get informe.pdf
550 Failed to open file.
ftp> delete informe.pdf
550 Permission denied.
```

**Al servidor**, l'administrador veu el fitxer pujat:

```bash
sudo ls -l /srv/ftp/incoming
# -rw------- 1 usuari1 ... 48213 ... informe.pdf
```

### Usuaris Locals i Anònims alhora

Amb aquesta configuració el servidor ofereix els dos tipus d'accés:

| Usuari                 | Arrel (chroot)      | Pot llegir                 | Pot escriure                   |
| ---------------------- | ------------------- | -------------------------- | ------------------------------ |
| `anonymous` / `ftp`    | `/srv/ftp`          | `pub/`                     | Només pujar a `incoming/`      |
| `usuari1`, `usuari2`   | `/home/<usuari>/ftp`| La seva carpeta            | `fitxers/` i `projectes/`      |

### Resolució de Problemes

**`500 OOPS: vsftpd: refusing to run with writable anonymous root`**

```bash
# L'arrel anònima no pot ser modificable per l'usuari ftp
sudo chown root:root /srv/ftp
sudo chmod 755 /srv/ftp
```

**`530 Permission denied.` amb l'usuari anonymous**

- Teniu `userlist_deny=NO` i `anonymous` no és a `/etc/vsftpd.userlist`.

**`550 Permission denied.` en pujar a `incoming`**

- Comproveu `write_enable=YES`, `anon_upload_enable=YES` i els permisos `730 root:ftp` de la carpeta.

**`550 Failed to open file.` en descarregar de `pub`**

- El fitxer no és llegible per tothom. Amb `anon_world_readable_only=YES` cal que tingui permisos `644`: `sudo chmod 644 /srv/ftp/pub/*`.

### Punt de Control

- [ ] Podeu entrar com a `anonymous` sense contrasenya.
- [ ] Podeu descarregar fitxers de `pub/` però no pujar-hi res.
- [ ] Podeu pujar fitxers a `incoming/` però no llistar-los ni descarregar-los.
- [ ] Els fitxers pujats a `incoming/` són propietat de `usuari1` amb permisos `600`.
- [ ] Els usuaris locals continuen funcionant igual que al punt anterior.
