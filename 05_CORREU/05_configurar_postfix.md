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
myhostname = mail.educem.cat

# Nom de domini del sistema
mydomain = educem.cat

# Origen dels correus enviats localment
myorigin = $mydomain

# Interfícies de xarxa on escoltar
# inet_interfaces = all
# Només local:
# inet_interfaces = localhost
# IPs específiques (la xarxa interna i localhost):
inet_interfaces = 192.168.1.2, localhost

# Protocols IP (IPv4, IPv6 o ambdós)
inet_protocols = ipv4
# inet_protocols = all  # IPv4 + IPv6

# Xarxes de confiança: poden enviar correu cap a qualsevol destí sense autenticar-se.
# Al principi hi posem la xarxa interna per fer les primeres proves des del client.
# A l'apartat 08 (SASL) la traurem: els clients s'hauran d'autenticar per enviar.
mynetworks = 127.0.0.0/8, 192.168.1.0/24

# Destinacions finals (els correus pels següents dominis es lliuren localment)
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Carpeta de bústies (format recomanat Maildir)
# Postfix crea ~/Maildir/ amb el primer correu que lliura a cada usuari
home_mailbox = Maildir/

# Mida màxima del missatge (en bytes): 10 MB
message_size_limit = 10485760

# Mida màxima de la bústia (0 = il·limitat)
mailbox_size_limit = 0

# Recipient delimiter (per adreces amb + )
recipient_delimiter = +

# Alias map
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
```

**Notes:**

- A `main.cf`, els comentaris han d'anar en una línia pròpia, començada per `#`. Un comentari al final d'una línia es considera part del valor, i Postfix dona error (`bad parameter value`).
- Amb el format Maildir, l'ordre `mail` sense arguments busca el correu a `/var/mail/usuari` i respon `No mail for usuari`. Per llegir la bústia: `mail -f ~/Maildir` (apartat 09).

### Estructura de main.cf (Recomanada)

```bash
# ============================================
# IDENTIFICACIÓ DEL SERVIDOR
# ============================================
# Que coincideixi amb el MX del servidor DNS que teniu configurat
myhostname = mail.educem.cat
mydomain = educem.cat
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
# Valor que porta Ubuntu 24.04 (no el baixeu)
compatibility_level = 3.6

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
- SMTPS (port 465) - SMTP amb TLS implícit (l'RFC 8314 el recomana per als clients; al laboratori farem servir el 587)

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
root:          admin@educem.cat
suport:        usuari1, usuari2
info:          usuari1
vendes:        comercial@educem.cat

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

Per defecte, Postfix envia els logs al syslog i queden a **`/var/log/mail.log`**. És el fitxer que fem servir a tot el tema.

```bash
# Opcional: Postfix escriu directament en un fitxer propi (servei postlog de master.cf).
# Si ho activeu, els logs de Postfix ja no apareixen a /var/log/mail.log.
# maillog_file = /var/log/postfix.log

# Més detall als logs només per a uns clients concrets (per diagnosticar un problema)
# debug_peer_list = 192.168.1.50
# debug_peer_level = 2
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
sudo postconf -e 'myhostname=mail.educem.cat'
sudo postconf -e 'mydomain=educem.cat'
sudo postconf -e 'myorigin=$mydomain'

# Múltiples paràmetres a la vegada
sudo postconf -e 'myhostname=mail.educem.cat' -e 'mydomain=educem.cat'

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
myhostname = mail.educem.cat
mydomain = educem.cat
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
echo "Test de configuració" | mail -s "Prova" usuari1@localhost

# Verificar als logs
sudo tail -f /var/log/mail.log

# Verificar que Postfix ha creat la bústia Maildir i hi ha deixat el correu
sudo ls -la /home/usuari1/Maildir/new/
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

**Test 3: Provar destinataris locals**

```bash
# Connectar amb telnet
telnet localhost 25

EHLO test.example.com
MAIL FROM:<test@example.com>
RCPT TO:<usuari1@educem.cat>
# 250 Ok: educem.cat és a mydestination i usuari1 existeix

RCPT TO:<noexisteix@educem.cat>
# 550 5.1.1 ... User unknown in local recipient table
QUIT
```

`Nota:` Des del mateix servidor (o des de la xarxa interna, que és a `mynetworks`), Postfix accepta enviar correu a qualsevol destí: `permit_mynetworks`. La prova que el relay sense autenticació es rebutja la farem a l'apartat 08, quan traiem la xarxa interna de `mynetworks`.

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
