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
    reject_rbl_client zen.spamhaus.org=127.0.0.[2..11],
    reject_rbl_client bl.spamcop.net,
    reject_unknown_client_hostname,
    permit
```

Les llistes negres (RBL) s'expliquen més avall. Fixeu-vos en el filtre `=127.0.0.[2..11]` de Spamhaus.

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

Les taules `hash:` només busquen coincidències exactes (una IP o un nom concret). Per a xarxes en format CIDR (`192.168.1.0/24`) cal una taula `cidr:`.

```bash
# Xarxes i IPs (taula cidr: no cal postmap)
sudo nano /etc/postfix/client_access.cidr
```

```
# Format: xarxa/màscara    acció
192.168.1.0/24          OK
203.0.113.0/24          REJECT
```

```bash
# Noms de domini dels clients (taula hash: cal postmap)
sudo nano /etc/postfix/client_access
```

```
# Format: domini    acció
spammer.test            REJECT
confianca.test          OK
```

**Generar base de dades (només la taula hash):**

```bash
sudo postmap /etc/postfix/client_access
```

**Configurar a main.cf:**

```bash
smtpd_client_restrictions =
    check_client_access cidr:/etc/postfix/client_access.cidr,
    check_client_access hash:/etc/postfix/client_access,
    permit_mynetworks,
    reject
```

`Nota:` Amb `reject` al final, només els clients de la llista (i `mynetworks`) es poden connectar. El servidor deixa de rebre correu de la resta de servidors d'Internet. És una configuració per a un servidor intern.

### Llistes Negres (RBL/DNSBL)

**RBL més utilitzades:**

```bash
smtpd_client_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_rbl_client zen.spamhaus.org=127.0.0.[2..11],
    reject_rbl_client bl.spamcop.net,
    permit
```

**Notes importants:**

- **Spamhaus no respon a consultes fetes a través de DNS públics** (8.8.8.8, 1.1.1.1...): en lloc de "no llistat", hi respon amb un codi d'error (`127.255.255.254`).
  - El nostre BIND fa servir `8.8.8.8` com a forwarder (Tema 1).
  - Sense el filtre `=127.0.0.[2..11]`, Postfix interpretaria aquesta resposta com "llistat" i **rebutjaria tot el correu**.
  - Amb el filtre, només es rebutja quan la resposta és un dels codis de llistat reals.
- **Llistes obsoletes:** `dnsbl.sorbs.net` va tancar el 2024, i `cbl.abuseat.org` està integrada a `zen.spamhaus.org`. No les feu servir.
- **Al laboratori**, els servidors no reben correu d'Internet: les RBL no es poden provar de manera real.

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
maxretry = 5
bantime = 3600

[postfix-sasl]
enabled = true
maxretry = 3
bantime = 3600

[dovecot]
enabled = true
maxretry = 3
bantime = 3600
```

El filtre, els ports i el fitxer de log de cada presó ja estan definits a `/etc/fail2ban/jail.conf`. A `jail.local` només les activem i n'ajustem els valors.

No hi poseu `filter = postfix-sasl`: a les versions actuals de fail2ban aquest filtre no existeix com a fitxer, perquè forma part del filtre `postfix` (mode `auth`). Si no troba el filtre, fail2ban no arrenca.

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
confianca.test    OK
empresa-segura.test      OK

# IP de confiança
192.168.1.100          OK
```

**Blacklist (llista negra):**

```bash
sudo nano /etc/postfix/blacklist
```

```
# Bloquejar domini
spam-domini.test        REJECT Spam domain
spammer2.test            REJECT Known spammer

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
# Des d'un altre equip (per exemple, el servidor 0374)
telnet mail.educem.cat 25

EHLO test.example.com
MAIL FROM:<spam@spammer.test>
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
