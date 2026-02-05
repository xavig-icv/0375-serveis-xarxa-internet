## 07. Autenticació SASL

### Què és SASL?

**SASL** (Simple Authentication and Security Layer) és un framework que permet als clients autenticar-se amb el servidor SMTP abans d'enviar correus. És essencial per prevenir que el servidor es converteixi en un relay obert.

**Per què és necessari?**

- Permet als usuaris autenticats enviar correu des de qualsevol lloc
- Evita que el servidor s'utilitzi per enviar spam (open relay)
- Protegeix el servidor de ser inclòs en llistes negres (blacklists)
- Requereix credencials per enviar correu (port 587)

### Components de SASL

**Postfix + Dovecot SASL:**

```
CLIENT                    POSTFIX                   DOVECOT
  │                          │                         │
  │  1. EHLO                 │                         │
  │─────────────────────────>│                         │
  │  250-AUTH PLAIN LOGIN    │                         │
  │<─────────────────────────│                         │
  │                          │                         │
  │  2. AUTH PLAIN [creds]   │                         │
  │─────────────────────────>│                         │
  │                          │  3. Verificar creds     │
  │                          │────────────────────────>│
  │                          │  4. OK/FAIL             │
  │                          │<────────────────────────│
  │  235 Authentication OK   │                         │
  │<─────────────────────────│                         │
  │                          │                         │
  │  5. MAIL FROM/RCPT TO    │                         │
  │─────────────────────────>│                         │
```

### Configuració de SASL amb Dovecot

**Pas 1: Configurar Dovecot per autenticació SASL**

```bash
# Editar configuració de serveis de Dovecot
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Afegir o modificar el servei d'autenticació:

```bash
service auth {
  # Socket Unix per Postfix
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

**Pas 2: Configurar Postfix per utilitzar Dovecot SASL**

```bash
# Editar configuració de Postfix
sudo nano /etc/postfix/main.cf
```

Afegir paràmetres SASL:

```bash
# ============================================
# AUTENTICACIÓ SASL (Dovecot)
# ============================================

# Activar autenticació SASL
smtpd_sasl_auth_enable = yes

# Tipus de servidor SASL (Dovecot)
smtpd_sasl_type = dovecot

# Ruta al socket d'autenticació de Dovecot
smtpd_sasl_path = private/auth

# Domini SASL (opcional)
# smtpd_sasl_local_domain = $mydomain

# Mecanismes d'autenticació permesos
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

# Trencar la compatibilitat amb clients antics (Microsoft)
# broken_sasl_auth_clients = yes

# Permetre autenticació per usuaris autenticats
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

### Configuració del Port Submission (587)

El port 587 és el recomanat per a l'enviament de correu per part de clients autenticats.

**Verificar la configuració a master.cf:**

```bash
sudo nano /etc/postfix/master.cf
```

```bash
submission inet n       -       y       -       -       smtpd
...
...
```

### Mecanismes d'Autenticació SASL

**Mecanismes disponibles:**

| Mecanisme      | Seguretat             | Descripció                          |
| -------------- | --------------------- | ----------------------------------- |
| **PLAIN**      | Baixa (només amb TLS) | Usuari i contrasenya en text pla    |
| **LOGIN**      | Baixa (només amb TLS) | Similar a PLAIN (Microsoft)         |
| **CRAM-MD5**   | Mitjana               | Challenge-response amb MD5          |
| **DIGEST-MD5** | Mitjana               | Similar a CRAM-MD5 però més complex |
| **GSSAPI**     | Alta                  | Kerberos                            |
| **EXTERNAL**   | Alta                  | Certificats client SSL/TLS          |

**Configuració recomanada (Dovecot):**

```bash
# /etc/dovecot/conf.d/10-auth.conf
auth_mechanisms = plain login
disable_plaintext_auth = yes  # Només amb SSL/TLS
```

**Configuració recomanada (Postfix):**

```bash
# /etc/postfix/main.cf
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous
```

### Aplicar Configuració

```bash
# Recarregar Dovecot
sudo systemctl reload dovecot

# Recarregar Postfix
sudo systemctl reload postfix

# Verificar que escolta al port 587
sudo ss -plutn | grep :587
```

### Proves d'Autenticació SASL

**Test 1: Verificar que SASL està actiu**

```bash
# Connectar al port 587
telnet localhost 587

# Comandes:
EHLO localhost

# Ha de mostrar:
# 250-AUTH PLAIN LOGIN
# 250-STARTTLS
```

**Test 2: Provar autenticació amb telnet**

```bash
# Generar credencials en base64
# Format: \0usuari\0contrasenya
printf '\0usuari@domini.cat\0contrasenya' | base64

# Connectar
telnet localhost 587

# Comandes:
EHLO localhost
AUTH PLAIN [string_base64_generat]

# Si l'autenticació és correcta:
# 235 2.7.0 Authentication successful
```

**Test 3: Provar amb openssl (TLS)**

```bash
# Connectar amb TLS
openssl s_client -connect localhost:587 -starttls smtp -crlf

# Comandes:
EHLO localhost
AUTH PLAIN [base64_credentials]
MAIL FROM:<usuari@domini.cat>
RCPT TO:<desti@example.com>
DATA
Subject: Test SASL
From: usuari@domini.cat
To: desti@example.com

Prova d'autenticació SASL.
.
QUIT
```

**Test 4: Amb Thunderbird**

Configuració al client:

- **Servidor sortint (SMTP):** mail.domini.cat
- **Port:** 587
- **Seguretat de connexió:** STARTTLS
- **Mètode d'autenticació:** Contrasenya normal
- **Nom d'usuari:** usuari@domini.cat

### Verificació als Logs

```bash
# Veure autenticacions correctes
sudo grep "sasl_method" /var/log/mail.log

# Sortida esperada:
# ... mail postfix/submission/smtpd[1234]: ABC123: client=unknown[192.168.1.100], sasl_method=PLAIN, sasl_username=usuari@domini.cat

# Veure autenticacions fallides
sudo grep "authentication failed" /var/log/mail.log

# Veure qui s'ha autenticat avui
sudo grep "$(date '+%b %d')" /var/log/mail.log | grep sasl_method
```

### Restriccions amb SASL

**Permetre només usuaris autenticats:**

```bash
# /etc/postfix/main.cf
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject
```

**Permetre relay només a autenticats:**

```bash
smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    defer_unauth_destination
```

**Restriccions per remitent:**

```bash
smtpd_sender_login_maps = hash:/etc/postfix/sender_login

# Fitxer /etc/postfix/sender_login:
# usuari@domini.cat    usuari@domini.cat
# usuari2@domini.cat   usuari2@domini.cat

smtpd_sender_restrictions =
    reject_sender_login_mismatch,
    permit_mynetworks,
    permit_sasl_authenticated,
    reject
```

### Resolució de Problemes SASL

**Problema: "authentication failed"**

```bash
# Verificar logs de Dovecot
sudo grep "auth" /var/log/mail.log | grep dovecot

# Provar autenticació directament
doveadm auth test usuari@domini.cat contrasenya

# Verificar socket
ls -la /var/spool/postfix/private/auth
```

**Problema: "SASL authentication not available"**

```bash
# Verificar configuració de Postfix
postconf smtpd_sasl_auth_enable
postconf smtpd_sasl_type
postconf smtpd_sasl_path

# Ha de ser:
# smtpd_sasl_auth_enable = yes
# smtpd_sasl_type = dovecot
# smtpd_sasl_path = private/auth
```

**Problema: Socket no existeix**

```bash
# Verificar configuració de Dovecot
doveconf -n | grep -A 10 "service auth"

# Ha d'haver-hi:
# unix_listener /var/spool/postfix/private/auth

# Verificar permisos
ls -la /var/spool/postfix/private/
```

**Problema: "Must issue a STARTTLS command first"**

```bash
# El client intenta autenticar sense TLS
# Configuració de Postfix:
smtpd_tls_auth_only = yes  # Força TLS abans d'autenticar

# Assegurar que el client utilitza STARTTLS o SSL/TLS
```

### Seguretat amb SASL

**1. Requerir TLS per autenticació:**

```bash
# main.cf
smtpd_tls_auth_only = yes
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous
```

**2. Limitar intents d'autenticació:**

```bash
# main.cf
smtpd_error_sleep_time = 1s
smtpd_soft_error_limit = 10
smtpd_hard_error_limit = 20

# Després de 20 errors, es desconnecta el client
```

**3. Monitoritzar autenticacions fallides:**

```bash
# Veure intents fallits
sudo grep "authentication failed" /var/log/mail.log

# Comptar per IP
sudo grep "authentication failed" /var/log/mail.log | awk '{print $(NF-1)}' | sort | uniq -c | sort -rn
```

**4. Bloquejar IPs amb fail2ban:**

```bash
# Instal·lar fail2ban
sudo apt install fail2ban

# Configurar filtre per Postfix SASL
# /etc/fail2ban/jail.local
[postfix-sasl]
enabled = true
port = smtp,submission
filter = postfix-sasl
logpath = /var/log/mail.log
maxretry = 3
bantime = 3600
```

### Millors Pràctiques

1. **Utilitzar sempre port 587 amb STARTTLS** (no 465 ni 25 per clients)
2. **Requerir TLS** abans de permetre autenticació
3. **Utilitzar contrasenyes fortes** per usuaris
4. **Monitoritzar logs** regularment per detectar intents d'autenticació
5. **Limitar intents** d'autenticació fallits
6. **No permetre autenticació anònima** (noanonymous)
7. **Utilitzar fail2ban** per bloquejar atacs de força bruta

### Verificació Final

```bash
# Checklist SASL:
# 1. Dovecot té el socket configurat
doveconf -n | grep -A 5 "service auth"

# 2. Postfix té SASL activat
postconf smtpd_sasl_auth_enable

# 3. Port 587 escolta
sudo ss -plutn | grep :587

# 4. Es pot autenticar
doveadm auth test usuari@domini.cat contrasenya

# 5. Els logs mostren autenticacions
sudo grep sasl_method /var/log/mail.log | tail -5
```
