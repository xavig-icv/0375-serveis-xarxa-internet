## 13. Filtres Antispam amb SpamAssassin

### Introducció a SpamAssassin

**SpamAssassin** és un filtre antispam de codi obert que analitza els correus i els classifica com a spam o legítims (ham) mitjançant múltiples tècniques.

**Característiques:**

- Anàlisi de contingut amb regles
- Llistes negres DNS (RBL/DNSBL)
- Llistes blanques i personalització
- Aprenentatge bayesià
- Puntuació de spam (score)
- Integració amb Postfix i Dovecot

### Instal·lació de SpamAssassin

A Ubuntu 24.04 (SpamAssassin 4.0), el servei `spamd` és en un paquet separat:

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar SpamAssassin, el servei spamd i el client spamc
sudo apt install spamassassin spamd spamc

# Verificar instal·lació
spamassassin --version
```

### Configuració Bàsica

**Habilitar i iniciar el servei:**

```bash
sudo systemctl enable --now spamd
sudo systemctl status spamd
```

Les opcions d'arrencada del servei són a `/etc/default/spamd` (per exemple, `OPTIONS="--create-prefs --max-children 5 --helper-home-dir"`).

**Verificar que escolta:**

```bash
sudo ss -plutn | grep spamd
# Ha de mostrar: tcp ... 127.0.0.1:783
```

### Configurar SpamAssassin

```bash
# Fitxer de configuració principal
sudo nano /etc/spamassassin/local.cf
```

```bash
# ============================================
# CONFIGURACIÓ DE SPAMASSASSIN
# ============================================

# Puntuació necessària per considerar spam (per defecte: 5.0)
required_score 5.0

# Reescriure Subject dels correus spam
rewrite_header Subject [***SPAM***]

# Afegir l'informe a les capçaleres sense modificar el missatge original
report_safe 0

# Utilitzar Bayes (aprenentatge)
use_bayes 1
bayes_auto_learn 1

# Xarxes de confiança (els seus servidors no es consideren origen d'spam)
trusted_networks 192.168.1.0/24
```

`Nota:` Les opcions de Razor, Pyzor i DCC (xarxes col·laboratives de detecció d'spam) necessiten programari addicional. Si s'activen sense tenir-lo, `spamassassin --lint` mostra errors.

**Validar la configuració i actualitzar regles:**

```bash
# Validar la configuració (sense sortida = correcta)
sudo spamassassin --lint

# Actualitzar regles de SpamAssassin
sudo sa-update

# Reiniciar el servei
sudo systemctl restart spamd
```

### Integració amb Postfix

Feu servir **un sol** dels dos mètodes. Si els activeu tots dos, cada correu s'analitza dues vegades.

**Mètode 1: Utilitzant spamass-milter**

```bash
# Instal·lar spamass-milter
sudo apt install spamass-milter

# Postfix ha de poder fer servir el socket del milter
sudo adduser postfix spamass-milter

# Opcions del milter
sudo nano /etc/default/spamass-milter
```

```bash
# -i 127.0.0.1: no analitzar el correu que surt del mateix servidor
OPTIONS="-u spamass-milter -i 127.0.0.1"
```

El socket del milter és a `/var/spool/postfix/spamass/spamass.sock`. El servei `smtpd` de Postfix s'executa engabiat (chroot) a `/var/spool/postfix`. Per això, a `main.cf` la ruta del socket s'escriu **relativa a aquesta carpeta**:

```bash
# Afegir a Postfix
sudo nano /etc/postfix/main.cf
```

```bash
# Milter per SpamAssassin (ruta relativa al chroot de Postfix)
smtpd_milters = unix:/spamass/spamass.sock
milter_default_action = accept

# Si també feu servir OpenDKIM (apartat 11), els dos milters van a la mateixa línia:
# smtpd_milters = inet:localhost:8891, unix:/spamass/spamass.sock
```

```bash
sudo systemctl restart spamass-milter
sudo systemctl reload postfix
```

`Nota:` Amb `-i 127.0.0.1`, els correus enviats des del mateix servidor (per exemple, amb `mail` o `swaks --server localhost`) no s'analitzen. Feu les proves enviant des del client.

**Mètode 2: Content filter**

```bash
# Usuari sense privilegis per executar el filtre
sudo useradd -r -s /usr/sbin/nologin spamfilter

# Crear script de filtre
sudo nano /etc/postfix/spamfilter.sh
```

```bash
#!/bin/bash
# Analitza el correu amb spamc i el torna a injectar a Postfix
SENDMAIL="/usr/sbin/sendmail -G -i"
/usr/bin/spamc | $SENDMAIL "$@"
exit $?
```

```bash
# Fer executable
sudo chmod 755 /etc/postfix/spamfilter.sh

# Configurar Postfix master.cf
sudo nano /etc/postfix/master.cf
```

```bash
# Afegir filtre de spam (el remitent es passa amb -f i els destinataris després de --)
spamfilter unix  -       n       n       -       -       pipe
  flags=Rq user=spamfilter argv=/etc/postfix/spamfilter.sh -f ${sender} -- ${recipient}

# Modificar smtp (port 25): el correu rebut passa pel filtre
smtp      inet  n       -       y       -       -       smtpd
  -o content_filter=spamfilter:dummy
```

```bash
sudo postfix check
sudo systemctl reload postfix
```

### Integració amb Dovecot (Sieve)

Els filtres Sieve els aplica **Dovecot en lliurar el correu**. Fins ara, Postfix desava el correu directament a la bústia (`home_mailbox`). Perquè s'executin els filtres Sieve, cal que Postfix lliuri el correu a Dovecot a través de **LMTP**.

**Instal·lar els paquets:**

```bash
sudo apt install dovecot-lmtpd dovecot-sieve dovecot-managesieved
```

**Socket LMTP per a Postfix (`/etc/dovecot/conf.d/10-master.conf`):**

```bash
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

**Activar Sieve en el lliurament LMTP (`/etc/dovecot/conf.d/20-lmtp.conf`):**

```bash
protocol lmtp {
  mail_plugins = $mail_plugins sieve
}
```

**Configurar Sieve (`/etc/dovecot/conf.d/90-sieve.conf`):**

```bash
plugin {
  sieve = file:~/sieve;active=~/.dovecot.sieve
  # Filtre per defecte per als usuaris que no en tenen cap de propi
  sieve_default = /var/lib/dovecot/sieve/default.sieve
}
```

**Postfix lliura a Dovecot (`/etc/postfix/main.cf`):**

```bash
mailbox_transport = lmtp:unix:private/dovecot-lmtp
```

Amb usuaris del sistema, Dovecot ha de trobar l'usuari a partir de l'adreça (`usuari1@educem.cat` → `usuari1`). Per això cal `auth_username_format = %Ln` (apartat 06).

**Crear filtre Sieve per spam:**

```bash
sudo mkdir -p /var/lib/dovecot/sieve
sudo nano /var/lib/dovecot/sieve/default.sieve
```

```sieve
require ["fileinto", "mailbox"];

# Moure spam a carpeta Spam (:create la crea si no existeix)
if header :contains "X-Spam-Flag" "YES" {
  fileinto :create "Spam";
}
```

**Compilar i aplicar:**

```bash
sudo sievec /var/lib/dovecot/sieve/default.sieve
sudo systemctl restart dovecot
sudo systemctl reload postfix

# Comprovar que Postfix lliura via LMTP (relay=...dovecot-lmtp)
sudo tail -f /var/log/mail.log
```

### Entrenament Bayesià

L'aprenentatge bayesià no s'aplica fins que SpamAssassin ha après com a mínim **200 correus spam i 200 de legítims**. Els exemples següents serveixen per veure com funciona l'ordre `sa-learn`.

**Entrenar amb correus spam:**

```bash
mkdir -p ~/spam-samples
printf 'Subject: Oferta\n\nGuanya 1000 euros ara! Clica aqui!\n' > ~/spam-samples/spam1.eml
printf 'Subject: Urgent\n\nOferta exclusiva, actua rapid!\n' > ~/spam-samples/spam2.eml
```

```bash
# Entrenar amb un directori de spam
sa-learn --spam ~/spam-samples/
```

**Entrenar amb correus legítims (ham):**

```bash
mkdir -p ~/ham-samples
printf 'Subject: Reunio\n\nReunio dema a les 10:00. Salutacions, Joan\n' > ~/ham-samples/ham1.eml
printf 'Subject: Informe\n\nInforme de vendes mensual adjunt\n' > ~/ham-samples/ham2.eml
```

```bash
# Entrenar amb correus bons
sa-learn --ham ~/ham-samples/

# Entrenar amb els correus llegits d'una bústia Maildir
sa-learn --ham ~/Maildir/cur/
```

**Verificar base de dades bayesiana:**

```bash
sa-learn --dump magic
```

### Proves i Verificació

**Provar SpamAssassin amb un correu:**

```bash
# Crear correu de prova
cat > /tmp/test-spam.txt << 'EOL'
Subject: Test spam
From: spam@example.com

URGENT! Click here to win a million dollars!
Buy cheap viagra now!
EOL

# Analitzar amb SpamAssassin
spamassassin -t < /tmp/test-spam.txt
```

**Test GTUBE (test de spam garantit):**

```bash
# Enviar GTUBE (Generic Test for Unsolicited Bulk Email)
cat > /tmp/gtube.txt << 'EOL'
Subject: Test GTUBE
From: test@example.com

XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X
EOL

spamassassin -t < /tmp/gtube.txt
# Ha de retornar score molt alt (spam)
```

Per provar tot el circuit (Postfix + SpamAssassin + Sieve), envieu el text GTUBE en un correu **des del client** (Thunderbird o `swaks` al client). Comproveu que arriba amb `[***SPAM***]` a l'assumpte i a la carpeta Spam.

### Millors Pràctiques

1. Actualitzar regles regularment: `sudo sa-update`
2. Entrenar Bayes amb correus reals
3. Ajustar `required_score` segons necessitats
4. Monitoritzar logs: `/var/log/mail.log`
5. Crear llistes blanques per remitents de confiança

```bash
# Configurar llista blanca
sudo nano /etc/spamassassin/local.cf
```

```bash
# A SpamAssassin 4.0 la directiva es diu welcomelist_from (abans whitelist_from)
welcomelist_from *@confianca.test
welcomelist_from *@empresa-segura.test
```
