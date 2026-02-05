## 06. Configuració Bàsica de Dovecot

### Fitxers de Configuració Principals

Dovecot utilitza una configuració modular amb diversos fitxers a `/etc/dovecot/conf.d/`.

**Fitxer principal:**

- `/etc/dovecot/dovecot.conf` - Inclou tots els altres fitxers

**Fitxers de configuració modular:**

```
/etc/dovecot/conf.d/
├── 10-auth.conf           # Autenticació
├── 10-director.conf       # Director (clustering)
├── 10-logging.conf        # Logging
├── 10-mail.conf           # Ubicació de bústies
├── 10-master.conf         # Serveis i processos
├── 10-ssl.conf            # SSL/TLS
├── 15-lda.conf            # Local Delivery Agent
├── 15-mailboxes.conf      # Carpetes per defecte
├── 20-imap.conf           # Configuració IMAP
├── 20-lmtp.conf           # LMTP
├── 20-pop3.conf           # Configuració POP3
├── 90-acl.conf            # ACLs
├── 90-plugin.conf         # Plugins
├── 90-quota.conf          # Quotes
└── auth-*.conf.ext        # Backends d'autenticació
```

### Configuració de Protocols (dovecot.conf)

```bash
# Editar configuració principal
sudo nano /etc/dovecot/dovecot.conf
```

```bash
# Protocols actius
protocols = imap pop3 lmtp

# Només si acceptem IMAP
# protocols = imap

# Incloure altres configuracions
!include_try /usr/share/dovecot/protocols.d/*.protocol
!include conf.d/*.conf
```

### Configuració de Bústies (10-mail.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

**Paràmetres importants:**

```bash
# Ubicació de les bústies - Format Maildir (recomanat)
mail_location = maildir:~/Maildir

# O format mbox
# mail_location = mbox:~/mail:INBOX=/var/mail/%u

# Altres opcions de mail_location:
# maildir:/var/mail/%d/%n/Maildir  # Per dominis virtuals
# mbox:~/mail:INBOX=/var/mail/%u:INDEX=/var/indexes/%u

# Variables disponibles:
# %u = nom d'usuari complet (usuari@domini.cat)
# %n = part local (usuari)
# %d = domini (domini.cat)
# %h = directori home

# Usuari i grup del procés de correu (per usuaris virtuals)
mail_uid = vmail
mail_gid = vmail
# O utilitzar l'usuari del sistema:
# mail_uid =
# mail_gid =

# Directori base per bústies
# mail_home = /var/mail/%d/%n

# Permisos de fitxers nous
mail_access_groups = mail

# Privilegis
first_valid_uid = 1000
last_valid_uid = 0

# Plugins de mail
mail_plugins =
```

**Exemples de mail_location:**

```bash
# Maildir al home de l'usuari
mail_location = maildir:~/Maildir

# Maildir amb índexs separats (proporciona un millor rendiment)
mail_location = maildir:~/Maildir:INDEX=/var/indexes/%u

# Maildir per dominis virtuals
mail_location = maildir:/var/mail/vhosts/%d/%n

# mbox tradicional
mail_location = mbox:~/mail:INBOX=/var/mail/%u

# mbox amb separació de carpetes
mail_location = mbox:~/mail:INBOX=/var/mail/%u:INDEX=/var/indexes/%u
```

### Configuració d'Autenticació (10-auth.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

```bash
# Deshabilitar autenticació en text pla (excepte amb SSL/TLS)
disable_plaintext_auth = yes

# Mecanismes d'autenticació permesos
auth_mechanisms = plain login

# Altres mecanismes disponibles:
# auth_mechanisms = plain login cram-md5 digest-md5

# Cache d'autenticació
auth_cache_size = 0
auth_cache_ttl = 1 hour
auth_cache_negative_ttl = 1 hour

# Usuari per defecte (si no es troba l'usuari real)
# auth_default_user =

# Backends d'autenticació (descomenta el que necessitis)
!include auth-system.conf.ext
# !include auth-sql.conf.ext
# !include auth-ldap.conf.ext
# !include auth-passwdfile.conf.ext
# !include auth-checkpassword.conf.ext
# !include auth-static.conf.ext
```

**Fitxer auth-system.conf.ext (usuaris del sistema):**

```bash
# Editar
sudo nano /etc/dovecot/conf.d/auth-system.conf.ext
```

```bash
# Autenticació amb usuaris del sistema (/etc/passwd)
passdb {
  driver = pam
  # args =
}

# Base de dades d'usuaris del sistema
userdb {
  driver = passwd
  # Sobreescriure el directori home si cal
  # override_fields = home=/var/mail/%u
}

# O utilitzar passwd-file per usuaris virtuals
# passdb {
#   driver = passwd-file
#   args = scheme=CRYPT username_format=%u /etc/dovecot/users
# }
#
# userdb {
#   driver = passwd-file
#   args = username_format=%u /etc/dovecot/users
# }
```

### Configuració de Serveis (10-master.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/10-master.conf
```

```bash
# Servei IMAP
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }

  # Nombre de processos
  # process_min_avail = 0
  # service_count = 1
}

service imap {
  # Límit de processos IMAP simultanis
  # process_limit = 1024
}

# Servei POP3
service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}

service pop3 {
  # process_limit = 1024
}

# Servei d'autenticació
service auth {
  # Socket per Postfix SASL
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }

  # Socket per auth-userdb
  unix_listener auth-userdb {
    mode = 0600
    user = vmail
    # group =
  }

  # Usuari del procés d'autenticació
  # user = $default_internal_user
}

# Servei de lliurament local (LDA)
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

# Servei auth-worker
service auth-worker {
  # user = root
}
```

### Configuració SSL/TLS (10-ssl.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

```bash
# SSL/TLS activat
ssl = yes

# Certificats (generar primer el certificat)
ssl_cert = </etc/dovecot/private/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.key

# O amb Let's Encrypt:
# ssl_cert = </etc/letsencrypt/live/mail.domini.cat/fullchain.pem
# ssl_key = </etc/letsencrypt/live/mail.domini.cat/privkey.pem

# Certificat de la CA
# ssl_ca = </etc/ssl/certs/ca-certificates.crt

# Requerir SSL per tots els clients
# ssl_required = yes

# Protocols SSL/TLS permesos
ssl_min_protocol = TLSv1.2

# Xifratges preferits (Mozilla Modern)
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

# Preferir els xifratges del servidor
ssl_prefer_server_ciphers = yes

# Paràmetres DH
ssl_dh = </etc/dovecot/dh.pem

# Generar dh.pem:
# openssl dhparam -out /etc/dovecot/dh.pem 2048
```

### Configuració IMAP (20-imap.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/20-imap.conf
```

```bash
protocol imap {
  # Mida màxima de línia (per compatibilitat amb clients)
  # imap_max_line_length = 64k

  # Timeout d'inactivitat
  # imap_idle_notify_interval = 2 mins

  # Plugins IMAP
  # mail_plugins = $mail_plugins imap_quota imap_sieve

  # Nombre màxim de carpetes
  # imap_max_folders = unlimited
}
```

### Configuració POP3 (20-pop3.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/20-pop3.conf
```

```bash
protocol pop3 {
  # Mantenir missatges al servidor després de descarregar
  # pop3_no_flag_updates = no

  # UIDL format
  pop3_uidl_format = %08Xu%08Xv

  # Plugins POP3
  # mail_plugins = $mail_plugins

  # Deixar missatges al servidor
  # pop3_delete_type = flag  # Marca com esborrat però no esborra
  # pop3_delete_type = expunge  # Esborra immediatament (per defecte)
}
```

### Configuració de Logging (10-logging.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/10-logging.conf
```

```bash
# Ubicació dels logs (buit = syslog)
# log_path = /var/log/dovecot.log
# info_log_path = /var/log/dovecot-info.log
# debug_log_path = /var/log/dovecot-debug.log

# Utilitzar syslog (per defecte a /var/log/mail.log)
log_path = syslog
syslog_facility = mail

# Nivell de verbositat
# auth_verbose = no
# auth_debug = no
# auth_debug_passwords = no
# mail_debug = no
# verbose_ssl = no

# Format del log
# log_timestamp = "%Y-%m-%d %H:%M:%S "
```

### Configuració de Mailboxes (15-mailboxes.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/15-mailboxes.conf
```

```bash
namespace inbox {
  # Prefix de les carpetes (buit per INBOX)
  # prefix =

  # Separador de jerarquia de carpetes
  separator = /
  # separator = .  # Per compatibilitat amb clients antics

  # Carpetes especials (auto-crear)
  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }

  mailbox Junk {
    auto = subscribe
    special_use = \Junk
  }

  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }

  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }

  mailbox "Sent Messages" {
    special_use = \Sent
  }

  mailbox Spam {
    auto = no
    special_use = \Junk
  }
}
```

### Configuració de Quotes (90-quota.conf)

```bash
# Editar
sudo nano /etc/dovecot/conf.d/90-quota.conf
```

```bash
# Plugin de quota
plugin {
  # Tipus de backend de quota
  # quota = maildir:User quota
  # quota = count:User quota

  # Regla de quota (500 MB per usuari)
  # quota_rule = *:storage=500M

  # Excepcions
  # quota_rule2 = Trash:storage=+100M

  # Avís quan s'arriba al 90%
  # quota_warning = storage=90%% quota-warning 90 %u
  # quota_warning2 = storage=80%% quota-warning 80 %u
}

# Afegir plugin a protocols
# protocol imap {
#   mail_plugins = $mail_plugins imap_quota
# }
```

### Comandes de Configuració

```bash
# Veure configuració completa
doveconf

# Veure només configuració modificada (no valors per defecte)
doveconf -n

# Veure configuració d'un protocol específic
doveconf -n protocol imap
doveconf -n protocol pop3

# Veure un paràmetre específic
doveconf mail_location
doveconf protocols
doveconf ssl

# Verificar sintaxi de la configuració
doveconf > /dev/null
# Si no hi ha sortida, la configuració és correcta

# Recarregar configuració
sudo doveadm reload

# O reiniciar el servei
sudo systemctl restart dovecot
```

### Exemple de Configuració Completa

**Configuració bàsica per usuaris del sistema amb Maildir:**

**1. dovecot.conf:**

```bash
protocols = imap pop3
!include conf.d/*.conf
```

**2. 10-mail.conf:**

```bash
mail_location = maildir:~/Maildir
```

**3. 10-auth.conf:**

```bash
disable_plaintext_auth = yes
auth_mechanisms = plain login
!include auth-system.conf.ext
```

**4. 10-master.conf:**

```bash
# Configuració per integració amb Postfix
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

**5. 10-ssl.conf:**

```bash
ssl = required
ssl_cert = </etc/dovecot/private/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.key
ssl_min_protocol = TLSv1.2
```

### Verificació de la Configuració

```bash
# Verificar sintaxi
doveconf -n > /dev/null
echo $?  # Ha de retornar 0

# Veure configuració activa
doveconf -n

# Verificar protocols
doveconf protocols

# Verificar ubicació de bústies
doveconf mail_location

# Verificar SSL
doveconf ssl ssl_cert ssl_key

# Verificar autenticació
doveconf auth_mechanisms disable_plaintext_auth
```

### Aplicar Canvis

```bash
# Després de modificar qualsevol fitxer de configuració:

# Mètode 1: Recarregar (recomanat)
sudo doveadm reload

# Mètode 2: Reiniciar
sudo systemctl restart dovecot

# Verificar que el servei està actiu
sudo systemctl status dovecot

# Verificar logs per errors
sudo tail -f /var/log/mail.log | grep dovecot
```

### Proves de Configuració

**Test 1: Connexió IMAP**

```bash
# Amb telnet
telnet localhost 143

# Comandes:
a001 LOGIN usuari contrasenya
a002 LIST "" "*"
a003 SELECT INBOX
a004 LOGOUT
```

**Test 2: Connexió POP3**

```bash
# Amb telnet
telnet localhost 110

# Comandes:
USER usuari
PASS contrasenya
STAT
LIST
QUIT
```

**Test 3: Autenticació**

```bash
# Provar autenticació d'un usuari
doveadm auth test usuari contrasenya

# Sortida esperada si OK:
# passdb: usuari auth succeeded
# userdb: usuari
```

### Resolució de Problemes

**Error: "Fatal: Error in configuration file"**

```bash
# Verificar sintaxi
doveconf -n

# Buscar l'error específic
sudo tail -50 /var/log/mail.log | grep -i error
```

**No es pot autenticar:**

```bash
# Verificar mecanismes
doveconf auth_mechanisms

# Provar autenticació
doveadm auth test usuari contrasenya

# Verificar logs
sudo grep "auth" /var/log/mail.log | tail -20
```

**No es troben les bústies:**

```bash
# Verificar mail_location
doveconf mail_location

# Verificar que existeix el directori
ls -la ~/Maildir/

# Crear Maildir si no existeix
maildirmake ~/Maildir
```

### Millors Pràctiques

1. **Utilitzar Maildir en lloc de mbox:**

   ```bash
   mail_location = maildir:~/Maildir
   ```

2. **Requerir SSL/TLS:**

   ```bash
   ssl = required
   disable_plaintext_auth = yes
   ```

3. **Fer còpies de seguretat:**

   ```bash
   sudo cp -r /etc/dovecot /etc/dovecot.backup
   ```

4. **Verificar sempre abans de recarregar:**

   ```bash
   doveconf -n > /dev/null && sudo doveadm reload
   ```

5. **Monitoritzar logs:**
   ```bash
   sudo tail -f /var/log/mail.log | grep dovecot
   ```
