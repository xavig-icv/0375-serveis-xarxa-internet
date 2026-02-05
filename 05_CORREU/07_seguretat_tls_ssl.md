## 10. Seguretat TLS/SSL en Correu

### Introducció a TLS/SSL

**TLS/SSL** (Transport Layer Security / Secure Sockets Layer) són protocols que xifren les comunicacions entre clients i servidors, protegint:

- **Confidencialitat:** El contingut dels correus està xifrat
- **Integritat:** Els missatges no poden ser modificats
- **Autenticació:** Verifica la identitat del servidor

### Diferència entre SSL i TLS

| Protocol    | Any  | Estat             | Ús           |
| ----------- | ---- | ----------------- | ------------ |
| **SSL 2.0** | 1995 | Obsolet (insegur) | NO utilitzar |
| **SSL 3.0** | 1996 | Obsolet (insegur) | NO utilitzar |
| **TLS 1.0** | 1999 | Deprecated        | NO utilitzar |
| **TLS 1.1** | 2006 | Deprecated        | NO utilitzar |
| **TLS 1.2** | 2008 | **Recomanat**     | ✅ Utilitzar |
| **TLS 1.3** | 2018 | **Recomanat**     | ✅ Utilitzar |

**Recomanació:** Només permetre **TLS 1.2** i **TLS 1.3**.

### Tipus de Xifrat en Correu

**1. Xifrat Oportunista (STARTTLS):**

- Comença sense xifrar
- Negocia TLS si el servidor ho suporta
- Port estàndard + comanda STARTTLS
- **Vulnerable** a atacs downgrade si no es força

```
CLIENT                    SERVIDOR
  │                          │
  │  EHLO                    │
  │─────────────────────────>│
  │  250-STARTTLS            │
  │<─────────────────────────│
  │  STARTTLS                │
  │─────────────────────────>│
  │  220 Ready               │
  │<─────────────────────────│
  │  [Negociació TLS]        │
  │<────────────────────────>│
  │  [Comunicació xifrada]   │
```

**2. TLS/SSL Directe (Implicit TLS):**

- Connexió xifrada des de l'inici
- Port dedicat
- Més segur (no hi ha fase sense xifrar)

```
CLIENT                    SERVIDOR
  │                          │
  │  [Connexió xifrada]      │
  │<────────────────────────>│
  │  [Negociació TLS]        │
  │<────────────────────────>│
  │  220 Ready               │
  │<─────────────────────────│
```

### Ports i Protocols

| Servei         | Port | Tipus TLS            | Descripció               |
| -------------- | ---- | -------------------- | ------------------------ |
| **SMTP**       | 25   | STARTTLS (opcional)  | Servidor a servidor      |
| **Submission** | 587  | STARTTLS (recomanat) | Clients a servidor       |
| **SMTPS**      | 465  | SSL/TLS directe      | Legacy (no l'aconsello)  |
| **IMAP**       | 143  | STARTTLS (opcional)  | Sense xifrar per defecte |
| **IMAPS**      | 993  | SSL/TLS directe      | Xifrat sempre            |
| **POP3**       | 110  | STARTTLS (opcional)  | Sense xifrar per defecte |
| **POP3S**      | 995  | SSL/TLS directe      | Xifrat sempre            |

### Configuració TLS/SSL a Postfix

```bash
# Opció 1: Certificat autosignat (només proves)
sudo openssl req -new -x509 -days 365 -nodes -out /etc/ssl/certs/mail.crt -keyout /etc/ssl/private/mail.key

# Opció 2: Let's Encrypt (producció amb domini real)
# sudo certbot certonly --standalone -d mail.domini.cat
```

**Configuració bàsica TLS a Postfix:**

```bash
# Editar main.cf
sudo nano /etc/postfix/main.cf
```

```bash
# ============================================
# CONFIGURACIÓ TLS/SSL
# ============================================

# Certificats autosignats
smtpd_tls_cert_file = /etc/ssl/certs/mail.crt
smtpd_tls_key_file = /etc/ssl/private/mail.key

# O amb certificats de Let's Encrypt:
#smtpd_tls_cert_file = /etc/letsencrypt/live/mail.domini.cat/fullchain.pem
#smtpd_tls_key_file = /etc/letsencrypt/live/mail.domini.cat/privkey.pem


# Activar TLS per rebre correus (SMTP)
smtpd_tls_security_level = may
# may = oportunista (si el client vol)
# encrypt = obligatori (requereix TLS)

# Activar TLS per enviar correus (client SMTP)
smtp_tls_security_level = may

# Cache de sessions TLS (millora rendiment)
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# Protocols TLS permesos (només 1.2 i 1.3)
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1

# Xifratges segurs (Mozilla Modern)
smtpd_tls_mandatory_ciphers = high
tls_high_cipherlist = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

# Logging TLS
smtpd_tls_loglevel = 1
# 0 = només errors
# 1 = informació de sessió (recomanat)
# 2 = debug
smtp_tls_loglevel = 1

# DANE (opcional - validació DNS)
# smtp_dns_support_level = dnssec
# smtp_tls_security_level = dane

# Forçar autenticació només amb TLS
smtpd_tls_auth_only = yes

# Rebutjar clients amb certificats expirats
smtpd_tls_ask_ccert = yes
```

**Configuració del port Submission (587):**

```bash
# Editar master.cf
sudo nano /etc/postfix/master.cf
```

```bash
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

**Aplicar configuració:**

```bash
# Verificar configuració
sudo postfix check

# Recarregar Postfix
sudo systemctl reload postfix

# Verificar que escolta als ports
sudo ss -plutn | grep -E ':(25|587)'
```

### Configuració TLS/SSL a Dovecot

```bash
# Editar configuració SSL
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

```bash
# Activar SSL
ssl = required
# ssl = yes      # Permet connexions sense SSL (no recomanat)
# ssl = required # Requereix SSL (recomanat)

# Certificats autosignats
ssl_cert = </etc/ssl/certs/mail.crt
ssl_key = </etc/ssl/private/mail.key

# O amb certificats de Let's Encrypt:
# ssl_cert = </etc/letsencrypt/live/mail.domini.cat/fullchain.pem
#ssl_key = </etc/letsencrypt/live/mail.domini.cat/privkey.pem


# Protocols TLS permesos (només 1.2 i 1.3)
ssl_min_protocol = TLSv1.2

# Xifratges segurs (Mozilla Modern)
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305

# Preferir xifratges del servidor
ssl_prefer_server_ciphers = yes

# Paràmetres DH (Diffie-Hellman)
ssl_dh = </etc/dovecot/dh.pem

# Generar paràmetres DH (només cal fer-ho una vegada):
# openssl dhparam -out /etc/dovecot/dh.pem 2048

# Deshabilitar compressió SSL (prevenir atac CRIME)
ssl_options = no_compression

# Requerir certificats vàlids del client (opcional)
# ssl_verify_client_cert = yes
# ssl_ca = </etc/ssl/certs/ca-certificates.crt
```

**Aplicar configuració:**

```bash
# Generar paràmetres DH (si no existeix)
sudo openssl dhparam -out /etc/dovecot/dh.pem 2048

# Recarregar Dovecot
sudo systemctl reload dovecot

# Verificar que escolta als ports SSL
sudo ss -plutn | grep -E ':(993|995)'
```

### Configuració de Ports SSL/TLS

**Habilitar IMAPS (993) i POP3S (995):**

Ja estan configurats per defecte a `/etc/dovecot/conf.d/10-master.conf`:

```bash
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}
```

### Proves de Connexió TLS/SSL

**Test 1: Provar STARTTLS amb openssl (SMTP):**

```bash
# Connectar al port 25 o 587
openssl s_client -connect mail.domini.cat:587 -starttls smtp -crlf

# Ha de mostrar informació del certificat
# Comprovar:
# - Protocol: TLSv1.2 o TLSv1.3
# - Cipher: ECDHE-RSA-AES256-GCM-SHA384 o similar
# - Verify return code: 0 (ok) si el certificat és vàlid

# Provar comandes SMTP:
EHLO localhost
QUIT
```

**Test 2: Provar IMAPS (993):**

```bash
# Connectar amb SSL/TLS directe
openssl s_client -connect mail.domini.cat:993 -crlf

# Provar comandes IMAP:
a001 LOGIN usuari@domini.cat contrasenya
a002 LIST "" "*"
a003 LOGOUT
```

**Test 3: Provar POP3S (995):**

```bash
openssl s_client -connect mail.domini.cat:995 -crlf

# Provar comandes POP3:
USER usuari@domini.cat
PASS contrasenya
STAT
QUIT
```

**Test 4: Verificar certificat:**

```bash
# Veure informació del certificat
openssl s_client -connect mail.domini.cat:993 -showcerts

# Verificar dates d'expiració
echo | openssl s_client -connect mail.domini.cat:993 2>/dev/null | openssl x509 -noout -dates

# Verificar emissor
echo | openssl s_client -connect mail.domini.cat:993 2>/dev/null | openssl x509 -noout -issuer
```

**Test 5: Test de seguretat SSL:**

```bash
# Instal·lar testssl.sh
git clone https://github.com/drwetter/testssl.sh.git
cd testssl.sh

# Provar servidor
./testssl.sh mail.domini.cat:993
./testssl.sh mail.domini.cat:587

# Analitza:
# - Protocols suportats
# - Xifratges
# - Vulnerabilitats
```

### Verificació als Logs

```bash
# Veure connexions TLS
sudo grep "TLS connection" /var/log/mail.log

# Sortida esperada:
# Feb 04 10:30:15 mail postfix/smtpd[1234]: Anonymous TLS connection established from unknown[192.168.1.100]: TLSv1.3 with cipher TLS_AES_256_GCM_SHA384 (256/256 bits)

# Veure errors TLS
sudo grep -i "tls" /var/log/mail.log | grep -i error

# Veure certificats utilitzats
sudo grep "certificate" /var/log/mail.log
```

### Millors Pràctiques

1. **Utilitzar només TLS 1.2 i 1.3**
2. **Certificats vàlids** (Let's Encrypt recomanat)
3. **Renovar certificats** abans que caduquin
4. **Forçar TLS** al port submission (587)
5. **Xifratges moderns** (ECDHE, AES-GCM)
6. **Monitoritzar expiració** de certificats
7. **Paràmetres DH de 2048 bits** mínim
8. **Deshabilitar protocols antics** (SSLv2, SSLv3, TLSv1.0, TLSv1.1)
9. **Actualitzar regularment** OpenSSL i llibreries TLS
10. **Provar amb testssl.sh** periòdicament
