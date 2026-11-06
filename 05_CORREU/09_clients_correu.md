## 09. Configuració de Clients de Correu

### Introducció

Els clients de correu (MUA - Mail User Agent) són les aplicacions que utilitzen els usuaris per llegir i enviar correus. En aquest tema configurarem **Thunderbird** al client Ubuntu i l'eina de línia de comandes `mail` al servidor.

### Thunderbird (Client Gràfic)

**Thunderbird** és un client de correu gratuït i de codi obert desenvolupat per Mozilla.

**Instal·lació a Ubuntu (al client):**

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar Thunderbird (a Ubuntu s'instal·la com a paquet snap)
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
   - **Email address:** usuari1@educem.cat
   - **Password:** contrasenya de l'usuari al servidor

**Pas 2: Configuració Manual**

Clicar **"Configure manually"**. La configuració automàtica busca el servidor a Internet i amb el nostre domini de laboratori no el trobaria.

**Servidor Entrant (IMAP):**

| Camp               | Valor                                  |
| ------------------ | -------------------------------------- |
| **Protocol**       | IMAP                                   |
| **Hostname**       | mail.educem.cat                        |
| **Port**           | 993                                    |
| **SSL**            | SSL/TLS                                |
| **Authentication** | Normal password                        |
| **Username**       | usuari1                                |

**Servidor Sortint (SMTP):**

| Camp               | Valor           |
| ------------------ | --------------- |
| **Hostname**       | mail.educem.cat |
| **Port**           | 587             |
| **SSL**            | STARTTLS        |
| **Authentication** | Normal password |
| **Username**       | usuari1         |

**Nom d'usuari:** els usuaris de correu són usuaris del sistema, i el seu nom és `usuari1` (sense el domini).

- Thunderbird proposa per defecte l'adreça completa (`usuari1@educem.cat`). Si no s'ha configurat `auth_username_format = %Ln` a Dovecot (apartat 06), l'autenticació falla.
- Amb aquesta directiva, funcionen totes dues formes.

**Pas 3: Altres Ports**

Altres combinacions possibles:

- **IMAP amb STARTTLS:** port 143, Connection security **STARTTLS**.
- **POP3 amb SSL/TLS:** port 995, Connection security **SSL/TLS**.
- **SMTP amb TLS implícit:** port 465, Connection security **SSL/TLS**. Cal activar el servei `submissions` a `master.cf`. L'RFC 8314 el recomana per als clients.

No configureu IMAP o POP3 sense xifrat (143/110 amb "None"). Dovecot no permet enviar la contrasenya sense xifrar des d'un altre equip (`disable_plaintext_auth = yes`).

### Configuració amb Certificats Autosignats

Si utilitzes certificats autosignats, Thunderbird mostrarà una advertència.

**Acceptar Certificat:**

1. Thunderbird mostrarà: "This server is using a security certificate that cannot be verified"
2. Clicar **"Confirm Security Exception"**
3. Clicar **"View"** per veure el certificat
4. Clicar **"Confirm Security Exception"**

L'excepció es guarda per a cada servidor i port. El primer cop que envieu un correu (servidor SMTP, port 587), Thunderbird també pot demanar que accepteu el certificat. Les excepcions es poden revisar a **Settings → Privacy & Security → Manage Certificates → Servers**.

**Important:** Només fer això en entorns de proves o amb certificats autosignats de confiança.

### Eines de Terminal

L'ordre `mail` (paquet `mailutils`) es fa servir **al servidor**, on hi ha els usuaris i les bústies. No l'instal·leu al client: el paquet hi instal·laria també un servidor de correu (Postfix).

**Instal·lació (al servidor):**

```bash
sudo apt install mailutils
```

**Enviar correu:**

```bash
# Enviar correu simple
echo "Contingut del missatge" | mail -s "Assumpte" usuari2@educem.cat

# Enviar amb fitxer com a contingut
mail -s "Assumpte" usuari2@educem.cat < missatge.txt

# Enviar amb fitxer adjunt
echo "Missatge" | mail -s "Assumpte" -A document.pdf usuari2@educem.cat
```

**Llegir correu:**

Les bústies són en format **Maildir** (`~/Maildir`), i `mail` sense arguments busca el correu a `/var/mail/usuari`. Cal indicar-li la bústia:

```bash
# Com a usuari2: llegir la bústia Maildir
mail -f ~/Maildir

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

**Test 1: Enviar correu amb mail (al servidor)**

```bash
# Enviar correu de prova
echo "Test de configuració del client" | mail -s "Prova" usuari1@educem.cat

# Verificar als logs del servidor
sudo tail -f /var/log/mail.log
```

**Test 2: Connectar amb openssl (IMAPS) des del client**

```bash
# Provar connexió IMAPS
openssl s_client -connect mail.educem.cat:993 -crlf

a001 LOGIN usuari1 contrasenya
a002 LIST "" "*"
a003 SELECT INBOX
a004 LOGOUT
```

Si proveu `telnet mail.educem.cat 143` des del client, el servidor anuncia `LOGINDISABLED`: no accepta contrasenyes sense xifrar des d'un altre equip. És el comportament correcte.

**Test 3: Verificar enviament SMTP des del client**

```bash
# Amb telnet només es pot arribar fins a STARTTLS
telnet mail.educem.cat 587

EHLO client.educem.cat
STARTTLS
# (després de STARTTLS cal utilitzar openssl)

# O directament amb openssl:
openssl s_client -connect mail.educem.cat:587 -starttls smtp -crlf -nocommands

EHLO client.educem.cat
AUTH PLAIN [base64_credentials]
MAIL FROM:<usuari1@educem.cat>
RCPT TO:<usuari2@educem.cat>
DATA
Subject: Test
From: usuari1@educem.cat
To: usuari2@educem.cat

Missatge de prova.
.
QUIT
```

`Nota:` Sense `-nocommands`, `openssl` no envia la línia `RCPT TO` al servidor (apartat 08, Test 3).

### Resolució de Problemes

**Thunderbird no connecta:**

1. Verificar configuració de ports i SSL
2. Verificar que el client resol el servidor i que els ports responen:
   ```bash
   dig +short mail.educem.cat
   openssl s_client -connect mail.educem.cat:993 </dev/null
   openssl s_client -connect mail.educem.cat:587 -starttls smtp </dev/null
   ```
3. Verificar logs del servidor:
   ```bash
   sudo tail -f /var/log/mail.log
   ```
4. Provar amb SSL/TLS vs STARTTLS

**Error: "Certificate not trusted":**

- Si és un certificat autosignat, acceptar l'excepció
- Si és un certificat d'una CA (per exemple Let's Encrypt), verificar que és vàlid i que el nom coincideix

**Error: "Authentication failed":**

1. Verificar usuari i contrasenya (el nom d'usuari és el del sistema: `usuari1`)
2. Provar autenticació des del servidor:
   ```bash
   doveadm auth test usuari1 contrasenya
   ```
3. Verificar logs:
   ```bash
   sudo grep "auth" /var/log/mail.log | tail -20
   ```
