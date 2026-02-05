## 05. Configuració Bàsica de Postfix

### Fitxer de Configuració Principal

El fitxer `/etc/postfix/main.cf` conté la configuració global de Postfix (paràmetres generals que afecten tot el servidor de correu).

**Paràmetres importants:**

```bash
# Editar configuració
sudo nano /etc/postfix/main.cf

# O utilitzar postconf per modificar paràmetres
sudo postconf -e 'paràmetre=valor'
```

### Paràmetres Bàsics de main.cf

```bash
# Nom del sistema de correu (FQDN del servidor)
myhostname = mail.domini.cat

# Nom de domini del sistema
mydomain = domini.cat

# Origen dels correus enviats localment
myorigin = $mydomain

# Interfícies de xarxa on escoltar
# inet_interfaces = all
# inet_interfaces = localhost  # Només local
inet_interfaces = 192.168.1.2, localhost  # IPs específiques

# Protocols IP (IPv4, IPv6 o ambdós)
inet_protocols = ipv4
# inet_protocols = all  # IPv4 + IPv6

# Xarxes de confiança (poden enviar correus sense autenticació servidor-servidor)
mynetworks = 127.0.0.0/8, 192.168.1.0/24

# Destinacions finals (els correus pels següents dominis es lliuren localment)
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Carpeta de bústies (format recomanat Maildir)
# home_mailbox = mail/  # Per format mbox
home_mailbox = Maildir/

# Mida màxima del missatge (en bytes)
message_size_limit = 10485760  # 10 MB

# Mida màxima de la bústia
mailbox_size_limit = 0  # 0 = il·limitat

# Recipient delimiter (per adreces amb + )
recipient_delimiter = +

# Alias map
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
```

### Estructura de main.cf (Recomanada)

```bash
# ============================================
# IDENTIFICACIÓ DEL SERVIDOR
# ============================================
# Que coincideixi amb el MX del servidor DNS que teniu configurat
myhostname = mail.domini.cat
mydomain = domini.cat
myorigin = $mydomain

# ============================================
# INTERFÍCIES DE XARXA
# ============================================
inet_interfaces = all
inet_protocols = ipv4

# ============================================
# DESTINACIONS I XARXES
# ============================================
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8, 192.168.1.0/24
relayhost =

# ============================================
# BÚSTIES I EMMAGATZEMATGE
# ============================================
home_mailbox = Maildir/
mailbox_size_limit = 0
message_size_limit = 10485760

# ============================================
# ALIAS I MAPS
# ============================================
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
recipient_delimiter = +

# ============================================
# COMPATIBILITAT
# ============================================
compatibility_level = 2

# ============================================
# SEGURETAT BÀSICA
# ============================================
smtpd_banner = $myhostname ESMTP
biff = no
append_dot_mydomain = no
readme_directory = no
```

### Fitxer master.cf

El fitxer `/etc/postfix/master.cf` defineix els processos i serveis que executa Postfix.

Aquí principalment es podrà configurar el format d'enviament de correus del servidor:

- SMTP estàndard (port 25) - Transferència entre servidors
- Submission (port 587) - Enviament per clients autenticats
- SMTPS (port 465) - SMTP amb SSL/TLS (no el recomano)

### Configuració d'Alias

Els alias permeten establir una redirecció de correus d'una adreça a una altra.

**Editar fitxer d'alias:**

```bash
sudo nano /etc/aliases
```

**Contingut típic:**

```
# Alias del sistema
postmaster:    root
abuse:         root
spam:          root
webmaster:     root

# Usuaris específics
root:          admin@domini.cat
suport:        usuari1, usuari2
info:          usuari1
vendes:        comercial@domini.cat

# Llistes de distribució
equip:         usuari1, usuari2, usuari3
administratius: usuari4, usuari5
```

**Aplicar canvis:**

```bash
# Regenerar la base de dades d'alias
sudo newaliases
# O també:
sudo postalias /etc/aliases

# Verificar
postmap -q root hash:/etc/aliases
```

### Configuració de Límits i Quotes

```bash
# Mida màxima de missatge (10 MB)
message_size_limit = 10485760

# Mida màxima de bústia (0 = il·limitat)
mailbox_size_limit = 0

# Límit de recipients per missatge
smtpd_recipient_limit = 100

# Límit d'errors SMTP abans de desconnectar
smtpd_error_sleep_time = 1s
smtpd_soft_error_limit = 10
smtpd_hard_error_limit = 20

# Timeout de connexió idle
smtpd_timeout = 300s

# Màxim de processos SMTP simultanis
default_process_limit = 100
```

### Configuració de Logging

```bash
# Nivell de detall dels logs
# 0 = només errors crítics
# 1 = errors i advertències (per defecte)
# 2 = informació addicional
# 3 = debug
maillog_file = /var/log/postfix.log
maillog_file_prefixes = /var/log

# O utilitzar syslog (per defecte)
# Els logs aniran a /var/log/mail.log
```

### Comandes de Configuració

```bash
# Veure tots els paràmetres actuals
postconf

# Veure només paràmetres modificats (diferents del valor per defecte)
postconf -n

# Veure valors per defecte
postconf -d

# Veure un paràmetre específic
postconf myhostname
postconf mydestination

# Modificar un paràmetre
sudo postconf -e 'myhostname=mail.domini.cat'
sudo postconf -e 'mydomain=domini.cat'
sudo postconf -e 'myorigin=$mydomain'

# Múltiples paràmetres a la vegada
sudo postconf -e 'myhostname=mail.domini.cat' -e 'mydomain=domini.cat'

# Verificar sintaxi de la configuració
sudo postfix check

# Recarregar configuració (sense aturar el servei)
sudo postfix reload

# Veure configuració del master.cf
postconf -M

# Veure configuració del master.cf (només modificacions)
postconf -Mn
```

### Exemple de Configuració Completa

**Per a servidor de correu local (només usuaris del sistema):**

```bash
# Identificació
myhostname = mail.domini.cat
mydomain = domini.cat
myorigin = $mydomain

# Xarxa
inet_interfaces = all
inet_protocols = ipv4
mynetworks = 127.0.0.0/8, 192.168.1.0/24
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Bústies (Maildir)
home_mailbox = Maildir/
mailbox_size_limit = 0
message_size_limit = 10485760

# Alias
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
recipient_delimiter = +

# Seguretat bàsica
smtpd_banner = $myhostname ESMTP
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

### Verificació de la Configuració

```bash
# Verificar sintaxi
sudo postfix check

# Si hi ha errors, es mostraran aquí
# Si no hi ha sortida, la configuració és correcta

# Veure configuració activa
postconf -n

# Provar un paràmetre
postconf myhostname

# Veure on s'està utilitzant un paràmetre
postconf -n | grep mydomain
```

### Aplicar Canvis

```bash
# Després de modificar main.cf o master.cf:

# Mètode 1: Recarregar (recomanat, no interromp el servei)
sudo postfix reload

# Mètode 2: Reiniciar (si hi ha canvis importants)
sudo systemctl restart postfix

# Verificar que el servei està actiu
sudo systemctl status postfix

# Verificar logs per errors
sudo tail -f /var/log/mail.log
```

### Proves de Configuració

**Test 1: Enviar correu local**

```bash
# Enviar correu a usuari local
echo "Test de configuració" | mail -s "Prova" usuari@localhost

# Verificar als logs
sudo tail -f /var/log/mail.log

# Verificar a la bústia
ls -la ~/Maildir/new/
```

**Test 2: Verificar paràmetres**

```bash
# Verificar que el hostname és correcte
postconf myhostname

# Verificar destinacions
postconf mydestination

# Verificar xarxes de confiança
postconf mynetworks
```

**Test 3: Provar restriccions**

```bash
# Connectar amb telnet i provar restriccions
telnet localhost 25

EHLO test.example.com
MAIL FROM:<test@example.com>
RCPT TO:<usuari@domini.cat>
# Ha de permetre si domini.cat està a mydestination

RCPT TO:<extern@altredomain.com>
# Ha de rebutjar si no està autenticat (reject_unauth_destination)
```

### Resolució de Problemes de Configuració

**Error: "postfix: fatal: parameter xxx: bad parameter value"**

```bash
# Verificar sintaxi del paràmetre
postconf xxx

# Veure valor per defecte
postconf -d xxx

# Corregir i recarregar
sudo postconf -e 'xxx=valor_correcte'
sudo postfix reload
```

**Els correus no arriben a destinataris locals:**

```bash
# Verificar mydestination
postconf mydestination

# Ha d'incloure el domini local
sudo postconf -e 'mydestination=$myhostname, localhost, $mydomain'
sudo postfix reload
```

**Els correus queden a la cua:**

```bash
# Veure la cua
mailq

# Veure detalls
sudo postcat -q ID_CORREU

# Forçar reenviament
sudo postqueue -f

# Verificar logs
sudo tail -50 /var/log/mail.log
```

### Millors Pràctiques

1. **Sempre verificar abans de recarregar:**

   ```bash
   sudo postfix check
   ```

2. **Fer còpies de seguretat abans de modificar:**

   ```bash
   sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.backup
   ```

3. **Utilitzar postconf per modificar:**

   ```bash
   # Millor que editar manualment
   sudo postconf -e 'paràmetre=valor'
   ```

4. **Documentar els canvis:**

   ```bash
   # Afegir comentaris al main.cf
   # Data: 2026-02-04
   # Canvi: Augmentat message_size_limit a 20MB
   message_size_limit = 20971520
   ```

5. **Revisar logs després de canvis:**
   ```bash
   sudo tail -f /var/log/mail.log
   ```
