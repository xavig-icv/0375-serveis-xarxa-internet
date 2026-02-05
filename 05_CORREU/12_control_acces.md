## 12. Control d'Accés i Restriccions

### Introducció

El control d'accés permet definir qui pot enviar i rebre correus a través del servidor, protegint-lo contra ús no autoritzat i spam.

### Restriccions a Postfix

**Tipus de restriccions:**

1. **Client restrictions** - Basades en qui connecta
2. **Sender restrictions** - Basades en el remitent
3. **Recipient restrictions** - Basades en el destinatari
4. **Data restrictions** - Basades en el contingut

### Restriccions per Client

**Configuració:**

```bash
sudo nano /etc/postfix/main.cf
```

```bash
smtpd_client_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_rbl_client zen.spamhaus.org,
    reject_rbl_client bl.spamcop.net,
    reject_unknown_client_hostname,
    permit
```

**Explicació:**

| Restricció                       | Descripció                    |
| -------------------------------- | ----------------------------- |
| `permit_mynetworks`              | Permet xarxes de confiança    |
| `permit_sasl_authenticated`      | Permet usuaris autenticats    |
| `reject_rbl_client`              | Rebutja IPs en llistes negres |
| `reject_unknown_client_hostname` | Rebutja si no té DNS invers   |

### Restriccions per Remitent

```bash
smtpd_sender_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit
```

**Explicació:**

| Restricció                     | Descripció                   |
| ------------------------------ | ---------------------------- |
| `reject_non_fqdn_sender`       | Rebutja remitents sense FQDN |
| `reject_unknown_sender_domain` | Rebutja dominis sense DNS    |

### Restriccions per Destinatari

```bash
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain,
    reject_unauth_destination,
    reject_unverified_recipient,
    permit
```

**Important:** `reject_unauth_destination` evita que el servidor sigui un open relay.

### Control d'Accés per IP

**Permetre només IPs específiques:**

```bash
# Crear fitxer d'accés
sudo nano /etc/postfix/access
```

```
# Format: adreça/xarxa    acció

# Permetre IPs específiques
192.168.1.0/24           OK
10.0.0.0/8              OK

# Rebutjar IPs específiques
203.0.113.0/24          REJECT
spammer.com             REJECT

# Permetre domini específic
domini-confianca.cat    OK
```

**Generar base de dades:**

```bash
sudo postmap /etc/postfix/access
```

**Configurar a main.cf:**

```bash
smtpd_client_restrictions =
    check_client_access hash:/etc/postfix/access,
    permit_mynetworks,
    reject
```

### Llistes Negres (RBL/DNSBL)

**RBL més utilitzades:**

```bash
smtpd_client_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_rbl_client zen.spamhaus.org,
    reject_rbl_client bl.spamcop.net,
    reject_rbl_client cbl.abuseat.org,
    reject_rbl_client dnsbl.sorbs.net,
    permit
```

### Límits de Velocitat

**Limitar missatges per connexió:**

```bash
smtpd_error_sleep_time = 1s
smtpd_soft_error_limit = 10
smtpd_hard_error_limit = 20

# Després de 20 errors, desconnecta el client
```

**Limitar recipients:**

```bash
smtpd_recipient_limit = 50
# Màxim 50 destinataris per missatge
```

**Limitar mida de missatge:**

```bash
message_size_limit = 10485760
# 10 MB màxim
```

### Control amb fail2ban

**Instal·lar fail2ban:**

```bash
sudo apt install fail2ban
```

**Configurar per Postfix:**

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[postfix]
enabled = true
port = smtp,submission
filter = postfix
logpath = /var/log/mail.log
maxretry = 5
bantime = 3600

[postfix-sasl]
enabled = true
port = smtp,submission
filter = postfix-sasl
logpath = /var/log/mail.log
maxretry = 3
bantime = 3600

[dovecot]
enabled = true
port = pop3,pop3s,imap,imaps
filter = dovecot
logpath = /var/log/mail.log
maxretry = 3
bantime = 3600
```

**Reiniciar fail2ban:**

```bash
sudo systemctl restart fail2ban
```

**Veure IPs bloquejades:**

```bash
sudo fail2ban-client status postfix
sudo fail2ban-client status dovecot
```

### Whitelist i Blacklist

**Whitelist (llista blanca):**

```bash
sudo nano /etc/postfix/whitelist
```

```
# Domini de confiança
domini-confianca.cat    OK
empresa-segura.com      OK

# IP de confiança
192.168.1.100          OK
```

**Blacklist (llista negra):**

```bash
sudo nano /etc/postfix/blacklist
```

```
# Bloquejar domini
spam-domain.com        REJECT Spam domain
spammer.net            REJECT Known spammer

# Bloquejar IP
203.0.113.50          REJECT Blocked IP
```

**Generar i aplicar:**

```bash
sudo postmap /etc/postfix/whitelist
sudo postmap /etc/postfix/blacklist

# Afegir a main.cf
smtpd_client_restrictions =
    check_client_access hash:/etc/postfix/whitelist,
    check_client_access hash:/etc/postfix/blacklist,
    permit_mynetworks,
    permit
```

### Verificació i Logs

**Veure connexions rebutjades:**

```bash
sudo grep "reject" /var/log/mail.log | tail -20
```

**Veure IPs bloquejades per RBL:**

```bash
sudo grep "RBL" /var/log/mail.log | tail -10
```

**Provar restriccions:**

```bash
# Des d'un altre servidor
telnet mail.domini.cat 25

EHLO test.example.com
MAIL FROM:<spam@spammer.com>
# Ha de rebutjar si està a la blacklist
```

### Millors Pràctiques

1. **Sempre incloure `permit_mynetworks`** primer
2. **Utilitzar `reject_unauth_destination`** per evitar open relay
3. **Configurar RBL** per filtrar spam
4. **Utilitzar fail2ban** per bloquejar atacs
5. **Monitoritzar logs** regularment
6. **Mantenir whitelist actualitzada**
7. **Provar abans de bloquejar** dominis legítims
