## 09. Configuració de Clients de Correu

### Introducció

Els clients de correu (MUA - Mail User Agent) són les aplicacions que utilitzen els usuaris per llegir i enviar correus. En aquest tema configurarem **Thunderbird** i l'eina de línia de comandes.

### Thunderbird (Client Gràfic)

**Thunderbird** és un client de correu gratuït i de codi obert desenvolupat per Mozilla.

**Instal·lació a Ubuntu:**

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar Thunderbird
sudo apt install thunderbird

# Executar
thunderbird &
```

### Configuració Manual de Thunderbird

**Pas 1: Afegir Compte de Correu**

1. Obrir Thunderbird
2. Menú → **File** → **New** → **Existing Mail Account**
3. Introduir dades:
   - **Your name:** Nom de l'usuari
   - **Email address:** usuari@domini.cat
   - **Password:** contrasenya

**Pas 2: Configuració Manual**

Si la configuració automàtica no funciona, clicar **"Manual config"**:

**Servidor Entrant (IMAP o POP3):**

| Camp               | Valor IMAP        | Valor POP3        |
| ------------------ | ----------------- | ----------------- |
| **Protocol**       | IMAP              | POP3              |
| **Hostname**       | mail.domini.cat   | mail.domini.cat   |
| **Port**           | 143               | 110               |
| **SSL**            | STARTTLS          | STARTTLS          |
| **Authentication** | Normal password   | Normal password   |
| **Username**       | usuari@domini.cat | usuari@domini.cat |

**Servidor Sortint (SMTP):**

| Camp               | Valor             |
| ------------------ | ----------------- |
| **Hostname**       | mail.domini.cat   |
| **Port**           | 587               |
| **SSL**            | STARTTLS          |
| **Authentication** | Normal password   |
| **Username**       | usuari@domini.cat |

**Pas 3: Configuració Avançada (Ports SSL/TLS)**

Si s'utilitzen ports amb SSL/TLS directe:

**IMAP amb SSL/TLS:**

- Port: **993**
- Connection security: **SSL/TLS**

**POP3 amb SSL/TLS:**

- Port: **995**
- Connection security: **SSL/TLS**

**SMTP amb SSL/TLS:**

- Port: **465**
- Connection security: **SSL/TLS**

### Configuració amb Certificats Autosignats

Si utilitzes certificats autosignats, Thunderbird mostrarà una advertència.

**Acceptar Certificat:**

1. Thunderbird mostrarà: "This server is using a security certificate that cannot be verified"
2. Clicar **"Confirm Security Exception"**
3. Clicar **"View"** per veure el certificat
4. Clicar **"Confirm Security Exception"**

**Important:** Només fer això en entorns de proves o amb certificats autosignats de confiança.

### Eines de Terminal

**Instal·lació:**

```bash
sudo apt install mailutils
```

**Enviar correu:**

```bash
# Enviar correu simple
echo "Contingut del missatge" | mail -s "Assumpte" destinatari@example.com

# Enviar amb fitxer com a contingut
mail -s "Assumpte" destinatari@example.com < missatge.txt

# Enviar amb fitxer adjunt
echo "Missatge" | mail -s "Assumpte" -A document.pdf destinatari@example.com
```

**Llegir correu:**

```bash
# Llegir correus
mail

# Comandes dins de mail:
# h        - Llistar correus
# 1        - Llegir correu número 1
# d 1      - Esborrar correu 1
# q        - Sortir i aplicar canvis
# x        - Sortir sense aplicar canvis

# Verificar als logs del servidor
sudo tail -f /var/log/mail.log
```

### Proves de Configuració

**Test 1: Enviar correu amb mail**

```bash
# Enviar correu de prova
echo "Test de configuració del client" | mail -s "Prova" usuari@domini.cat

# Verificar als logs del servidor
sudo tail -f /var/log/mail.log
```

**Test 2: Connectar amb telnet (IMAP)**

```bash
# Provar connexió IMAP
telnet mail.domini.cat 143

EHLO localhost
a001 LOGIN usuari@domini.cat contrasenya
a002 LIST "" "*"
a003 SELECT INBOX
a004 LOGOUT
```

**Test 3: Connectar amb openssl (IMAPS)**

```bash
# Provar connexió IMAPS
openssl s_client -connect mail.domini.cat:993 -crlf

a001 LOGIN usuari@domini.cat contrasenya
a002 LIST "" "*"
a003 LOGOUT
```

**Test 4: Verificar enviament SMTP**

```bash
# Provar enviament amb telnet
telnet mail.domini.cat 587

EHLO localhost
STARTTLS
# (després de STARTTLS cal utilitzar openssl)

# O directament amb openssl:
openssl s_client -connect mail.domini.cat:587 -starttls smtp -crlf

EHLO localhost
AUTH PLAIN [base64_credentials]
MAIL FROM:<usuari@domini.cat>
RCPT TO:<desti@example.com>
DATA
Subject: Test
From: usuari@domini.cat
To: desti@example.com

Missatge de prova.
.
QUIT
```

### Resolució de Problemes

**Thunderbird no connecta:**

1. Verificar configuració de ports i SSL
2. Verificar que el servidor està actiu:
   ```bash
   telnet mail.domini.cat 143
   telnet mail.domini.cat 587
   ```
3. Verificar logs del servidor:
   ```bash
   sudo tail -f /var/log/mail.log
   ```
4. Provar amb SSL/TLS vs STARTTLS

**Error: "Certificate not trusted":**

- Si és un certificat autosignat, acceptar l'excepció
- Si és Let's Encrypt, verificar que el certificat és vàlid

**Error: "Authentication failed":**

1. Verificar usuari i contrasenya
2. Provar autenticació des del servidor:
   ```bash
   doveadm auth test usuari@domini.cat contrasenya
   ```
3. Verificar logs:
   ```bash
   sudo grep "auth" /var/log/mail.log | tail -20
   ```
