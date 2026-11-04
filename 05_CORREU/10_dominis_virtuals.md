## 10. Dominis Virtuals de Correu

`IMPORTANT:` Aquest apartat és una **ampliació opcional**.

- La configuració de *virtual mailbox domains* canvia com Dovecot autentica els usuaris (fitxer propi en lloc dels usuaris del sistema) i on desa les bústies.
- Feu-la en una còpia o instantània del servidor: la configuració amb usuaris del sistema (`usuari1`, `usuari2`) deixa de funcionar.
- Els dominis dels exemples acaben en `.test`, un domini reservat per a proves (RFC 6761).
- Per fer proves des del client, cal crear-ne les zones al DNS (Tema 1).

### Què són els Dominis Virtuals?

Els **dominis virtuals** permeten gestionar múltiples dominis de correu en un únic servidor, de manera similar als Virtual Hosts en servidors web. Cada domini pot tenir els seus propis usuaris i bústies.

**Avantatges:**

- Gestionar múltiples dominis en un sol servidor
- Usuaris independents per cada domini
- Estalvi de recursos (un sol servidor per diversos dominis)
- Ideal per hosting de correu

**Exemple:**

```
Servidor: mail.educem.cat

Domini 1: empresa1.test
  - usuari@empresa1.test
  - vendes@empresa1.test
  - suport@empresa1.test

Domini 2: empresa2.test
  - usuari@empresa2.test
  - info@empresa2.test
  - admin@empresa2.test
```

### Tipus de Dominis Virtuals

**1. Virtual Alias Domains**

- Àlies de dominis (redirecció)
- No emmagatzemen correu
- Redirigeixen a altres adreces

**2. Virtual Mailbox Domains**

- Dominis amb bústies pròpies
- Usuaris independents del sistema
- Emmagatzematge en ubicació personalitzada

### Configuració de Virtual Alias Domains

Els alias virtuals redirigeixen correus d'un domini a adreces reals.

**Pas 1: Definir dominis virtuals d'alias**

```bash
# Editar main.cf
sudo nano /etc/postfix/main.cf
```

```bash
# Domini virtual alias
virtual_alias_domains = alies.test, altredomini.test

# Fitxer de mapes d'alias virtuals
virtual_alias_maps = hash:/etc/postfix/virtual
```

**Pas 2: Crear fitxer de mapes**

```bash
# Crear fitxer d'alias
sudo nano /etc/postfix/virtual
```

```bash
# Format: adreça_virtual    adreça_real

# Àlies individuals
info@alies.test              usuari@educem.cat
vendes@alies.test            comercial@educem.cat
suport@alies.test            tecnic@educem.cat

# Redirigir tot un domini
@altredomini.test            usuari@educem.cat

# Múltiples destinataris
equip@alies.test             usuari1@educem.cat, usuari2@educem.cat

# Àlies específics d'usuari
usuari@alies.test            usuari.real@educem.cat
```

**Pas 3: Generar base de dades i aplicar**

```bash
# Generar hash
sudo postmap /etc/postfix/virtual

# Recarregar Postfix
sudo postfix reload

# Verificar
postmap -q info@alies.test hash:/etc/postfix/virtual
```

### Configuració de Virtual Mailbox Domains

Els mailbox virtuals creen bústies independents del sistema.

**Pas 1: Crear usuari i directoris**

```bash
# Crear usuari virtual per gestionar bústies
sudo groupadd -g 5000 vmail
sudo useradd -g vmail -u 5000 vmail -d /var/mail/vhosts -m

# Crear estructura de directoris
sudo mkdir -p /var/mail/vhosts/empresa1.test
sudo mkdir -p /var/mail/vhosts/empresa2.test

# Establir permisos
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

**Pas 2: Configurar Postfix**

```bash
# Editar main.cf
sudo nano /etc/postfix/main.cf
```

```bash
# ============================================
# DOMINIS VIRTUALS MAILBOX
# ============================================

# Dominis virtuals (separats per comes)
virtual_mailbox_domains = empresa1.test, empresa2.test

# Mapa de bústies virtuals
virtual_mailbox_maps = hash:/etc/postfix/vmailbox

# Directori base per bústies virtuals
virtual_mailbox_base = /var/mail/vhosts

# UID i GID de l'usuari virtual
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

# Límit de mida de bústia (0 = il·limitat)
virtual_mailbox_limit = 0

# Mínim d'UID permès
virtual_minimum_uid = 100

# Àlies virtuals (opcional). Si ja feu servir virtual_alias_maps per als virtual alias domains,
# poseu tots els fitxers a la mateixa línia: una segona línia virtual_alias_maps substitueix la primera.
virtual_alias_maps = hash:/etc/postfix/virtual, hash:/etc/postfix/virtual_aliases
```

**Pas 3: Crear fitxer de bústies virtuals**

```bash
# Crear fitxer de bústies
sudo nano /etc/postfix/vmailbox
```

```bash
# Format: adreça_email    ruta_maildir/

# Domini 1
usuari1@empresa1.test       empresa1.test/usuari1/
usuari2@empresa1.test       empresa1.test/usuari2/
admin@empresa1.test         empresa1.test/admin/
info@empresa1.test          empresa1.test/info/

# Domini 2
usuari1@empresa2.test       empresa2.test/usuari1/
vendes@empresa2.test        empresa2.test/vendes/
suport@empresa2.test        empresa2.test/suport/
```

**Pas 4: Crear fitxer d'àlies virtuals (opcional)**

```bash
sudo nano /etc/postfix/virtual_aliases
```

```bash
# Àlies per dominis virtuals
postmaster@empresa1.test    admin@empresa1.test
webmaster@empresa1.test     admin@empresa1.test

abuse@empresa2.test         admin@empresa2.test
```

**Pas 5: Generar bases de dades**

```bash
# Generar hash de bústies
sudo postmap /etc/postfix/vmailbox

# Generar hash d'àlies (si existeix)
sudo postmap /etc/postfix/virtual_aliases

# Recarregar Postfix
sudo postfix reload
```

**Pas 6: Directoris Maildir**

No cal crear-los a mà. El lliurament virtual de Postfix crea el Maildir de cada usuari (amb l'usuari `vmail`) quan rep el primer correu, perquè la ruta del fitxer `vmailbox` acaba en `/`.

```bash
# Comprovar-ho després d'enviar el primer correu
sudo ls -la /var/mail/vhosts/empresa1.test/usuari1/
```

### Configurar Dovecot per Dominis Virtuals

**Pas 1: Configurar ubicació de bústies**

```bash
# Editar configuració de mail
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

```bash
# Ubicació de bústies virtuals
mail_location = maildir:/var/mail/vhosts/%d/%n

# %d = domini (empresa1.test)
# %n = nom usuari (usuari1)
# %u = usuari complet (usuari1@empresa1.test)

# Usuari i grup
mail_uid = vmail
mail_gid = vmail

# Directori home virtual
mail_home = /var/mail/vhosts/%d/%n

# Privilegis
first_valid_uid = 5000
last_valid_uid = 5000
first_valid_gid = 5000
last_valid_gid = 5000
```

**Pas 2: Configurar autenticació amb fitxer passwd**

```bash
# Editar configuració d'autenticació
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

```bash
# Deshabilitar autenticació del sistema
#!include auth-system.conf.ext

# Activar autenticació amb passwd-file
!include auth-passwdfile.conf.ext

# Els usuaris virtuals entren amb l'adreça completa (usuari1@empresa1.test):
# traieu auth_username_format = %Ln (apartat 06), que n'eliminaria el domini
#auth_username_format = %Ln
```

**Pas 3: Crear fitxer d'usuaris virtuals**

```bash
# Crear fitxer de contrasenyes
sudo nano /etc/dovecot/conf.d/auth-passwdfile.conf.ext
```

```bash
passdb {
  driver = passwd-file
  args = scheme=CRYPT username_format=%u /etc/dovecot/users
}

userdb {
  driver = passwd-file
  args = username_format=%u /etc/dovecot/users

  # Sobreescriure directoris
  default_fields = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
}
```

**Pas 4: Crear fitxer d'usuaris**

```bash
# Crear fitxer d'usuaris
sudo nano /etc/dovecot/users
```

Format: `usuari:contrasenya_hash:uid:gid:nom::home:shell`

```bash
# Generar hash de contrasenya
doveadm pw -s SHA512-CRYPT

# Afegir usuaris al fitxer
# usuari@empresa1.test:{SHA512-CRYPT}$6$....:5000:5000::/var/mail/vhosts/empresa1.test/usuari::

usuari1@empresa1.test:{SHA512-CRYPT}$6$random$hash...:5000:5000::/var/mail/vhosts/empresa1.test/usuari1::
usuari2@empresa1.test:{SHA512-CRYPT}$6$random$hash...:5000:5000::/var/mail/vhosts/empresa1.test/usuari2::
usuari1@empresa2.test:{SHA512-CRYPT}$6$random$hash...:5000:5000::/var/mail/vhosts/empresa2.test/usuari1::
```

**Pas 5: Establir permisos i recarregar**

```bash
# Permisos del fitxer d'usuaris
sudo chmod 640 /etc/dovecot/users
sudo chown root:dovecot /etc/dovecot/users

# Recarregar Dovecot
sudo systemctl reload dovecot
```

### Proves amb Dominis Virtuals

**Test 1: Enviar correu a usuari virtual**

```bash
# Des del mateix servidor
echo "Test domini virtual" | mail -s "Prova" usuari1@empresa1.test

# Verificar als logs
sudo tail -f /var/log/mail.log
```

**Test 2: Connectar amb IMAP**

```bash
# Amb telnet
telnet localhost 143

# Comandes:
a001 LOGIN usuari1@empresa1.test contrasenya
a002 LIST "" "*"
a003 SELECT INBOX
a004 LOGOUT
```

**Test 3: Verificar bústia**

```bash
# Veure fitxers de la bústia
sudo ls -la /var/mail/vhosts/empresa1.test/usuari1/new/

# Veure contingut d'un correu
sudo cat /var/mail/vhosts/empresa1.test/usuari1/new/[fitxer]
```

### Resolució de Problemes

**Els correus no arriben a dominis virtuals:**

```bash
# Verificar que el domini està a virtual_mailbox_domains
postconf virtual_mailbox_domains

# Verificar mapa
postmap -q usuari1@empresa1.test hash:/etc/postfix/vmailbox

# Verificar logs
sudo tail -f /var/log/mail.log
```

**No es pot autenticar amb usuari virtual:**

```bash
# Provar autenticació
doveadm auth test usuari1@empresa1.test contrasenya

# Verificar fitxer d'usuaris
sudo cat /etc/dovecot/users | grep usuari1@empresa1.test

# Verificar permisos
ls -la /etc/dovecot/users
```

**Error: "User doesn't exist":**

```bash
# Verificar mail_location
doveconf -n | grep mail_location

# Verificar que existeix el directori
sudo ls -la /var/mail/vhosts/empresa1.test/usuari1/

# Si no existeix, enviar-li un correu (Postfix el crea) i comprovar els permisos
sudo chown -R vmail:vmail /var/mail/vhosts
```

### Millors Pràctiques

1. **Utilitzar usuari vmail dedicat** per gestionar bústies
2. **Separar cada domini** en el seu propi directori
3. **Utilitzar base de dades** per entorns amb molts usuaris
4. **Fer còpies de seguretat** de `/etc/postfix/vmailbox` i `/etc/dovecot/users`
5. **Documentar cada domini** i els seus usuaris
6. **Utilitzar scripts** per automatitzar la creació d'usuaris
7. **Monitoritzar l'espai** de `/var/mail/vhosts`

```bash
# Verificar espai utilitzat per domini
sudo du -sh /var/mail/vhosts/*
```
