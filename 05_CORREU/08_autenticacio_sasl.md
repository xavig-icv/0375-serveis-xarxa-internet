## 08. Autenticació SASL

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

Postfix no comprova les contrasenyes: les demana a Dovecot a través d'un socket. Així els usuaris fan servir la mateixa contrasenya per enviar (SMTP) i per llegir el correu (IMAP/POP3).

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

# Ruta al socket d'autenticació de Dovecot (relativa a /var/spool/postfix)
smtpd_sasl_path = private/auth

# Mecanismes permesos: de moment, també sense TLS (per a les proves amb telnet)
smtpd_sasl_security_options = noanonymous

# Permetre autenticació per usuaris autenticats
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

### Configuració del Port Submission (587)

El port 587 és el recomanat per a l'enviament de correu per part de clients autenticats. El servei `submission` de `/etc/postfix/master.cf` ja el vam configurar a l'apartat 07. Les seves opcions `smtpd_sasl_auth_enable=yes` i `smtpd_client_restrictions=permit_sasl_authenticated,reject` funcionen a partir d'ara, quan Postfix ja pot consultar Dovecot.

```bash
# Veure la configuració activa del submission
postconf -M submission/inet
postconf -P | grep '^submission'
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

**Configuració recomanada (Dovecot, `/etc/dovecot/conf.d/10-auth.conf`):**

```bash
auth_mechanisms = plain login
# Només amb SSL/TLS
disable_plaintext_auth = yes
```

**Configuració recomanada (Postfix, `/etc/postfix/main.cf`):** la configurarem a la fase 2 de les proves.

```bash
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous
```

### Aplicar Configuració

```bash
# Recarregar Dovecot
sudo systemctl reload dovecot

# Recarregar Postfix
sudo systemctl reload postfix

# Verificar que existeix el socket i que escolta al port 587
sudo ls -l /var/spool/postfix/private/auth
sudo ss -plutn | grep :587
```

### Proves d'Autenticació SASL

Les proves es fan en dues fases:

1. **Fase 1 (temporal):** l'autenticació també es permet sense xifrat al port 25, per poder veure-la amb `telnet`.
2. **Fase 2 (definitiva):** l'autenticació només es permet dins de TLS, i es prova amb `openssl`.

Amb la configuració definitiva, Postfix **no anuncia AUTH fins que el client ha fet STARTTLS**. Per això les proves amb `telnet` s'han de fer abans, igual que fem temporalment amb `disable_plaintext_auth = no` a Dovecot.

#### Fase 1: proves amb telnet (sense xifrat, temporal)

```bash
# Comprovar que l'autenticació sense TLS està permesa temporalment
postconf smtpd_tls_auth_only smtpd_sasl_security_options
# smtpd_tls_auth_only = no
# smtpd_sasl_security_options = noanonymous
```

**Test 1: Verificar que SASL està actiu**

```bash
# Connectar al port 25 (al mateix servidor)
telnet localhost 25

# Comandes:
EHLO localhost

# Ha de mostrar, entre altres:
# 250-STARTTLS
# 250-AUTH PLAIN LOGIN
```

**Test 2: Provar autenticació amb telnet**

```bash
# Generar credencials en base64
# Format: \0usuari\0contrasenya
printf '\0usuari1\0contrasenya' | base64

# Connectar
telnet localhost 25

# Comandes:
EHLO localhost
AUTH PLAIN [string_base64_generat]

# Si l'autenticació és correcta:
# 235 2.7.0 Authentication successful
```

Si l'autenticació falla, consulteu `/var/log/mail.log` i comproveu la configuració temporal de Dovecot (`disable_plaintext_auth = no`, apartat 06).

#### Fase 2: autenticació només amb TLS (configuració definitiva)

```bash
sudo postconf -e 'smtpd_tls_auth_only=yes'
sudo postconf -e 'smtpd_sasl_security_options=noanonymous, noplaintext'
sudo postconf -e 'smtpd_sasl_tls_security_options=noanonymous'
sudo systemctl reload postfix
```

- `smtpd_tls_auth_only = yes`: AUTH només s'anuncia i s'accepta després de STARTTLS.
- `noplaintext`: sense TLS, no es permeten els mecanismes que envien la contrasenya en clar (PLAIN, LOGIN).

Ara, amb `telnet localhost 25`, l'`EHLO` ja no mostra `250-AUTH`. És el comportament correcte.

**Test 3: Provar amb openssl (TLS)**

```bash
# Connectar amb TLS al port submission
openssl s_client -connect localhost:587 -starttls smtp -crlf

# Comandes (després de STARTTLS cal tornar a fer EHLO):
EHLO localhost
AUTH PLAIN [base64_credentials]
MAIL FROM:<usuari1@educem.cat>
RCPT TO:<usuari2@educem.cat>
DATA
Subject: Test SASL
From: usuari1@educem.cat
To: usuari2@educem.cat

Prova d'autenticació SASL.
.
QUIT
```

**Test 4: Amb Thunderbird**

Configuració al client:

- **Servidor sortint (SMTP):** mail.educem.cat
- **Port:** 587
- **Seguretat de connexió:** STARTTLS
- **Mètode d'autenticació:** Contrasenya normal
- **Nom d'usuari:** usuari1 (amb `auth_username_format = %Ln` a Dovecot, també `usuari1@educem.cat`)

#### Fase 3 (opcional): demostrar que el relay necessita autenticació

Mentre la xarxa interna sigui a `mynetworks` (apartat 05), el client pot enviar correu cap a qualsevol destí sense autenticar-se. Per veure la utilitat de SASL, deixem només la xarxa local del servidor:

```bash
sudo postconf -e 'mynetworks=127.0.0.0/8'
sudo systemctl reload postfix
```

Des del client, sense autenticar-se:

```bash
telnet mail.educem.cat 25

EHLO client.educem.cat
MAIL FROM:<usuari1@educem.cat>
RCPT TO:<algu@example.com>
# Resposta: Relay access denied (codi 454 o 554)

RCPT TO:<usuari2@educem.cat>
# Resposta: 250 Ok (el destinatari és local: no és relay)
QUIT
```

Els clients que s'autentiquen al port 587 (Thunderbird) continuen podent enviar correu.

### Verificació als Logs

```bash
# Veure autenticacions correctes
sudo grep "sasl_method" /var/log/mail.log

# Sortida esperada:
# ... postfix/submission/smtpd[1234]: ABC123: client=client.educem.cat[192.168.1.50], sasl_method=PLAIN, sasl_username=usuari1

# Veure autenticacions fallides
sudo grep "authentication failed" /var/log/mail.log

# Veure qui s'ha autenticat avui
sudo grep "$(date '+%b %d')" /var/log/mail.log | grep sasl_method
```

### Restriccions amb SASL

**Permetre només usuaris autenticats (relay):**

```bash
# /etc/postfix/main.cf
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

No afegiu `reject` al final de la llista: rebutjaria també el correu que altres servidors envien als vostres usuaris. `reject_unauth_destination` ja rebutja el relay no autoritzat.

**Permetre relay només a autenticats:**

```bash
smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    defer_unauth_destination
```

**Restriccions per remitent:** un usuari autenticat només pot fer servir la seva adreça com a remitent.

```bash
# /etc/postfix/main.cf
smtpd_sender_login_maps = hash:/etc/postfix/sender_login

smtpd_sender_restrictions =
    reject_sender_login_mismatch,
    permit_mynetworks,
    permit_sasl_authenticated
```

```bash
# Fitxer /etc/postfix/sender_login: adreça → usuari SASL que la pot fer servir
usuari1@educem.cat    usuari1
usuari2@educem.cat    usuari2
```

```bash
sudo postmap /etc/postfix/sender_login
sudo systemctl reload postfix
```

`Nota:` Amb `reject_sender_login_mismatch`, un correu amb remitent `usuari1@educem.cat` que no s'ha autenticat com a `usuari1` es rebutja. Això inclou, per exemple, una prova amb `telnet` al port 25.

### Resolució de Problemes SASL

**Problema: "authentication failed"**

```bash
# Verificar logs de Dovecot
sudo grep "auth" /var/log/mail.log | grep dovecot

# Provar autenticació directament
doveadm auth test usuari1 contrasenya

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

**Problema: "Must issue a STARTTLS command first" o l'EHLO no mostra AUTH**

```bash
# El client intenta autenticar sense TLS i Postfix té:
# smtpd_tls_auth_only = yes  (força TLS abans d'autenticar)
postconf smtpd_tls_auth_only

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

# /etc/fail2ban/jail.local
[postfix-sasl]
enabled = true
maxretry = 3
bantime = 3600
```

El filtre, els ports i el fitxer de log de la presó `postfix-sasl` ja estan definits a `/etc/fail2ban/jail.conf`. A `jail.local` només s'activa i s'ajusta.

### Millors Pràctiques

1. **Utilitzar el port 587 amb STARTTLS o el 465 amb TLS implícit** per als clients (no el 25)
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

# 2. Postfix té SASL activat i AUTH només amb TLS
postconf smtpd_sasl_auth_enable smtpd_tls_auth_only

# 3. Port 587 escolta
sudo ss -plutn | grep :587

# 4. Es pot autenticar
doveadm auth test usuari1 contrasenya

# 5. Els logs mostren autenticacions
sudo grep sasl_method /var/log/mail.log | tail -5
```
