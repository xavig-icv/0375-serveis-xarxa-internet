## 11. Filtres Antispam amb SpamAssassin

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

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar SpamAssassin i spamc
sudo apt install spamassassin spamc

# Verificar instal·lació
spamassassin --version
```

### Configuració Bàsica

**Habilitar i iniciar el servei:**

```bash
# Editar configuració del servei
sudo nano /etc/default/spamassassin
```

```bash
# Canviar ENABLED a 1
ENABLED=1

# Opcions de spamd
OPTIONS="--create-prefs --max-children 5 --helper-home-dir"

# Crear usuari si no existeix
SPAMD_USER="spamd"
SPAMD_GROUP="spamd"
```

**Iniciar SpamAssassin:**

```bash
sudo systemctl enable spamassassin
sudo systemctl start spamassassin
sudo systemctl status spamassassin
```

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

# Informar del score al header
report_safe 0

# Utilitzar Bayes (aprenentatge)
use_bayes 1
bayes_auto_learn 1

# Utilitzar llistes negres DNS (DNSBL)
use_razor2 1
use_pyzor 1
use_dcc 1
use_bayes_rules 1

# Confiar en Received headers
trusted_networks 192.168.1.0/24

# Idioma
ok_languages ca es en
ok_locales ca es en
```

**Actualitzar regles:**

```bash
# Actualitzar regles de SpamAssassin
sudo sa-update

# Reiniciar el servei
sudo systemctl restart spamassassin
```

### Integració amb Postfix

**Mètode 1: Utilitzant spamass-milter**

```bash
# Instal·lar spamass-milter
sudo apt install spamass-milter

# Configurar
sudo nano /etc/default/spamass-milter
```

```bash
OPTIONS="-u spamass-milter -i 127.0.0.1"
```

```bash
# Afegir a Postfix
sudo nano /etc/postfix/main.cf
```

```bash
# Milter per SpamAssassin
smtpd_milters = unix:/var/spool/postfix/spamass/spamass.sock
milter_default_action = accept
```

**Mètode 2: Content filter (recomanat)**

```bash
# Crear script de filtre
sudo nano /etc/postfix/spamfilter.sh
```

```bash
#!/bin/bash
SENDMAIL="/usr/sbin/sendmail -G -i"
/usr/bin/spamc | $SENDMAIL "$@"
exit $?
```

```bash
# Fer executable
sudo chmod +x /etc/postfix/spamfilter.sh

# Configurar Postfix master.cf
sudo nano /etc/postfix/master.cf
```

```bash
# Afegir filtre de spam
spamfilter unix  -       n       n       -       -       pipe
  flags=Rq user=spamd argv=/etc/postfix/spamfilter.sh ${sender} ${recipient}

# Modificar smtp
smtp      inet  n       -       y       -       -       smtpd
  -o content_filter=spamfilter:dummy
```

### Integració amb Dovecot (Sieve)

**Instal·lar sieve:**

```bash
sudo apt install dovecot-sieve dovecot-managesieved
```

**Configurar Dovecot:**

```bash
sudo nano /etc/dovecot/conf.d/90-sieve.conf
```

```bash
plugin {
  sieve = file:~/sieve;active=~/.dovecot.sieve
  sieve_global_path = /var/lib/dovecot/sieve/default.sieve
}
```

**Crear filtre Sieve per spam:**

```bash
sudo nano /var/lib/dovecot/sieve/default.sieve
```

```sieve
require ["fileinto"];

# Moure spam a carpeta Spam
if header :contains "X-Spam-Flag" "YES" {
  fileinto "Spam";
}
```

**Compilar i aplicar:**

```bash
sudo sievec /var/lib/dovecot/sieve/default.sieve
sudo systemctl reload dovecot
```

### Entrenament Bayesià

**Entrenar amb correus spam:**

```bash
mkdir -p ~/spam-samples
echo "Guanya 1000€ ara! Clica aquí!" > ~/spam-samples/spam1.eml
echo "Oferta exclusiva, actua ràpid!" > ~/spam-samples/spam2.eml
```

```bash
# Entrenar amb un directori de spam
sa-learn --spam ~/spam-samples/

# Entrenar amb un fitxer mbox
sa-learn --spam /var/mail/spam-examples
```

**Entrenar amb correus legítims (ham):**

```bash
mkdir -p ~/ham-samples
echo "Reunió demà a les 10:00. Salutacions, Joan" > ~/ham-samples/ham1.eml
echo "Informe de vendes mensual adjunt" > ~/ham-samples/ham2.eml
```

```bash
# Entrenar amb correus bons
sa-learn --ham ~/ham-samples/

# Entrenar amb bústia
sa-learn --ham /var/mail/usuari
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

### Millors Pràctiques

1. Actualitzar regles regularment: `sudo sa-update`
2. Entrenar Bayes amb correus reals
3. Ajustar `required_score` segons necessitats
4. Monitoritzar logs: `/var/log/mail.log`
5. Crear llistes blanques per remitents de confiança

```bash
# Configurar whitelist
sudo nano /etc/spamassassin/local.cf
```

```bash
whitelist_from domini-confianca.cat
whitelist_from *@empresa-segura.com
```
