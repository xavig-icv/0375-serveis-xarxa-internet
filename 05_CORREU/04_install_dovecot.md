## 04. Instal·lació de Dovecot (MDA)

### Instal·lació a Debian/Ubuntu

```bash
# Actualitzar els repositoris
sudo apt update

# Instal·lar Dovecot amb suport per IMAP i POP3
sudo apt install dovecot-core dovecot-imapd dovecot-pop3d

# Verificar la instal·lació
dovecot --version
```

**Paquets inclosos a Dovecot:**

| Paquet                   | Descripció                          |
| ------------------------ | ----------------------------------- |
| **dovecot-core**         | Components bàsics de Dovecot        |
| **dovecot-imapd**        | Servidor IMAP                       |
| **dovecot-pop3d**        | Servidor POP3                       |
| **dovecot-lmtpd**        | LMTP (Local Mail Transfer Protocol) |
| **dovecot-sieve**        | Suport per filtres Sieve            |
| **dovecot-managesieved** | Gestió remota de filtres Sieve      |
| **dovecot-mysql**        | Autenticació contra MySQL/MariaDB   |
| **dovecot-pgsql**        | Autenticació contra PostgreSQL      |
| **dovecot-ldap**         | Autenticació contra LDAP            |

### Gestió bàsica del Servei

```bash
# Iniciar Dovecot
sudo systemctl start dovecot

# Aturar Dovecot
sudo systemctl stop dovecot

# Reiniciar Dovecot
sudo systemctl restart dovecot

# Recarregar la configuració (sense aturar el servei)
sudo systemctl reload dovecot

# Habilitar a l'arrencada del sistema
sudo systemctl enable dovecot

# Comprovar l'estat actual del servei
sudo systemctl status dovecot
```

### Verificació del Servei

```bash
# Comprovar que el servei està actiu
sudo systemctl status dovecot

# Verificar que escolta als ports correctes
sudo ss -plutn | grep dovecot

# Sortida esperada:
# tcp   LISTEN  0  100  0.0.0.0:110   0.0.0.0:*   users:(("dovecot",pid=...))  # POP3
# tcp   LISTEN  0  100  0.0.0.0:143   0.0.0.0:*   users:(("dovecot",pid=...))  # IMAP
# tcp   LISTEN  0  100  0.0.0.0:993   0.0.0.0:*   users:(("dovecot",pid=...))  # IMAPS
# tcp   LISTEN  0  100  0.0.0.0:995   0.0.0.0:*   users:(("dovecot",pid=...))  # POP3S

# Verificar protocols actius
doveconf protocols

# Sortida esperada:
# protocols = imap pop3
```

### Estructura de Directoris de Dovecot

```
/etc/dovecot/
├── dovecot.conf                    # Configuració principal (inclou altres fitxers)
├── conf.d/                         # Fitxers de configuració modular
│   ├── 10-auth.conf                # Autenticació
│   ├── 10-mail.conf                # Ubicació de bústies
│   ├── 10-master.conf              # Processos i sockets
│   ├── 10-ssl.conf                 # Configuració SSL/TLS
│   ├── 15-mailboxes.conf           # Carpetes per defecte
│   ├── 20-imap.conf                # Configuració IMAP
│   ├── 20-pop3.conf                # Configuració POP3
│   ├── 90-quota.conf               # Quotes de disc
│   └── auth-system.conf.ext        # Autenticació del sistema
├── dovecot-sql.conf.ext            # Autenticació SQL
└── dovecot-ldap.conf.ext           # Autenticació LDAP

/var/mail/                          # Bústies mbox (per defecte)
└── usuari

/home/usuari/Maildir/               # Bústies Maildir (el que recomanao)
├── cur/                            # Correus actuals
├── new/                            # Correus nous
└── tmp/                            # Temporals

/var/log/
├── mail.log                        # Log general de correu
└── mail.err                        # Errors de correu
```

### Verificar el Funcionament

**Provar la connexió amb POP3:**

```bash
# Connectar-se amb telnet
telnet localhost 110

# Comandes:
USER usuari
PASS contrasenya
STAT
LIST
QUIT
```

**Provar la connexió amb IMAP:**

```bash
# Connectar-se amb telnet
telnet localhost 143

# Comandes:
a001 LOGIN usuari contrasenya
a002 LIST "" "*"
a003 SELECT INBOX
a004 LOGOUT
```

**Provar amb openssl (només per connexions xifrades):**

```bash
# Provar IMAPS (port 993)
openssl s_client -connect localhost:993 -crlf

# Un cop connectat:
a001 LOGIN usuari contrasenya
a002 LIST "" "*"
a003 LOGOUT

# Provar POP3S (port 995)
openssl s_client -connect localhost:995 -crlf

# Un cop connectat:
USER usuari
PASS contrasenya
STAT
QUIT
```

### Fitxers de Configuració Principals

**1. /etc/dovecot/dovecot.conf**

Fitxer principal que inclou la resta de configuracions:

```bash
# Mostrar el contingut
cat /etc/dovecot/dovecot.conf

# Contingut típic:
!include_try /usr/share/dovecot/protocols.d/*.protocol
!include conf.d/*.conf
!include_try conf.d/auth/*.conf.ext
```

**2. /etc/dovecot/conf.d/10-mail.conf**

Defineix on s'emmagatzemen les bústies:

```bash
# Veure la configuració actual
grep mail_location /etc/dovecot/conf.d/10-mail.conf

# Format mbox (per defecte):
# mail_location = mbox:~/mail:INBOX=/var/mail/%u

# Format Maildir (la bústia recomanada):
mail_location = maildir:~/Maildir
```

**3. /etc/dovecot/conf.d/10-auth.conf**

Configuració d'autenticació:

```bash
# Veure els mecanismes d'autenticació
grep auth_mechanisms /etc/dovecot/conf.d/10-auth.conf

# Per defecte:
auth_mechanisms = plain login
```

### Comandes Útils de Dovecot

```bash
# Verificar la configuració
doveconf

# Verificar la configuració modificada (no valors per defecte)
doveconf -n

# Verificar un paràmetre específic
doveconf mail_location
doveconf protocols

# Recarregar la configuració
sudo doveadm reload

# Veure els usuaris connectats
sudo doveadm who

# Buscar missatges a la bústia d'un usuari
sudo doveadm search -u usuari ALL

# Elimina del disc missatges esborrats per l'usuari
sudo doveadm expunge -u usuari mailbox INBOX ALL

# Veure les quotes d'un usuari
sudo doveadm quota get -u usuari

# Estadístiques del servidor
sudo doveadm stats dump

# Forçar la reindexació de la bústia
sudo doveadm force-resync -u usuari INBOX
```

### Logs de Dovecot

**Ubicació dels logs:**

```bash
# Debian/Ubuntu
/var/log/mail.log
/var/log/mail.err
/var/log/dovecot.log  # Si es configura específicament
```

**Visualitzar logs:**

```bash
# Veure el logs a temps real
sudo tail -f /var/log/mail.log

# Filtrar només els missatges de Dovecot
sudo tail -f /var/log/mail.log | grep dovecot

# Veure errors
sudo grep -i error /var/log/mail.log | grep dovecot

# Veure les autenticacions d'usuaris
sudo grep "dovecot.*Login" /var/log/mail.log

# Veure les autenticacions fallides d'usuaris
sudo grep "auth failed" /var/log/mail.log
```

### Resolució de Problemes Inicials

**Dovecot no arrenca:**

```bash
# Verificar errors de configuració
doveconf -n

# Verificar logs
sudo tail -50 /var/log/mail.log | grep dovecot

# Verificar que no hi ha conflicte de ports
sudo ss -plutn | grep -E ':(110|143)'

# Verificar permisos
ls -ld /var/mail/
```

**Un usuari NO es pot autenticar:**

```bash
# Verificar que l'usuari existeix al sistema
id usuari

# Verificar la configuració de l'autenticació
doveconf -n | grep auth

# Verificar logs d'autenticació
sudo grep "auth" /var/log/mail.log | tail -20

# Provar l'autenticació manualment
doveadm auth test usuari contrasenya
```

**No es veuen els correus:**

```bash
# Verificar la ubicació de les bústies
doveconf mail_location

# Verificar que existeix la bústia de l'usuari
ls -la /var/mail/usuari
# o
ls -la /home/usuari/Maildir/

# Verificar els permisos
ls -ld /var/mail/usuari

# Verificar el propietari
stat /var/mail/usuari
```

### Test de Connexió

**Mètode 1: Amb telnet (POP3)**

```bash
telnet localhost 110

# Comandes:
+OK Dovecot ready.
USER usuari
+OK
PASS contrasenya
+OK Logged in.
STAT
+OK 2 1024
LIST
+OK 2 messages
1 512
2 512
.
QUIT
+OK Logging out.
```

**Mètode 2: Amb telnet (IMAP)**

```bash
telnet localhost 143

# Comandes:
* OK [CAPABILITY ...] Dovecot ready.
a001 LOGIN usuari contrasenya
a001 OK Logged in
a002 SELECT INBOX
* FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
* OK [PERMANENTFLAGS ...] Flags permitted.
* 2 EXISTS
* 0 RECENT
* OK [UIDVALIDITY ...] UIDs valid
a002 OK [READ-WRITE] Select completed
a003 LOGOUT
* BYE Logging out
a003 OK Logout completed
```

**Mètode 3: Amb un client de correu (Thunderbird)**

Configuració manual a Thunderbird:

- Servidor entrant: localhost (o IP servidor)
- Protocol: IMAP o POP3
- Port: 143 (IMAP) o 110 (POP3)
- Seguretat: Sense (o STARTTLS si està configurat)
- Autenticació: Contrasenya normal
- Nom d'usuari: usuari (o usuari@domini.cat)
