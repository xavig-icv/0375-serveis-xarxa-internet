## 02. Protocols de Correu Electrònic

### Protocol SMTP (Simple Mail Transfer Protocol)

**SMTP** és el protocol estàndard per a l'enviament i transferència de correu electrònic entre servidors. Defineix com es comuniquen els servidors de correu entre ells.

**Característiques:**

- Protocol de **capa d'aplicació** (Capa 7 del Model OSI)
- Es basa en comandes en **text pla** (comandes llegibles pels humans)
- **Orientat a connexió** (utilitza el protocol TCP)
- S'encarrega d'emmagatzemar i reenvia la informació (Store-and-Forward)
- Ports:
  - **25**: SMTP amb STARTTLS estàndard (servidor a servidor)
  - **587**: SMTP amb STARTTLS submission (clients amb autenticació - recomanat)
  - **465**: SMTPS (SMTP sobre SSL/TLS implícit)

### Sessió SMTP (Enviament de Correu)

Exemple de comunicació SMTP entre client i servidor:

```
CLIENT                                    SERVIDOR
  │                                          │
  │  Connexió TCP al port 25/587             │
  │<─────────────────────────────────────────│
  │  220 mail.domini.cat ESMTP Postfix       │
  │                                          │
  │  EHLO client.domini.cat                  │
  │──────────────────────────────────────────>│
  │  250-mail.domini.cat                     │
  │  250-SIZE 10240000                       │
  │  250-STARTTLS                            │
  │  250 AUTH PLAIN LOGIN                    │
  │<─────────────────────────────────────────│
  │                                          │
  │  STARTTLS                                │
  │──────────────────────────────────────────>│
  │  220 Ready to start TLS                  │
  │<─────────────────────────────────────────│
  │                                          │
  │  [Negociació TLS]                        │
  │<────────────────────────────────────────>│
  │                                          │
  │  AUTH PLAIN [credentials]                │
  │──────────────────────────────────────────>│
  │  235 Authentication successful           │
  │<─────────────────────────────────────────│
  │                                          │
  │  MAIL FROM:<usuari@domini.cat>           │
  │──────────────────────────────────────────>│
  │  250 Ok                                  │
  │<─────────────────────────────────────────│
  │                                          │
  │  RCPT TO:<desti@example.com>             │
  │──────────────────────────────────────────>│
  │  250 Ok                                  │
  │<─────────────────────────────────────────│
  │                                          │
  │  DATA                                    │
  │──────────────────────────────────────────>│
  │  354 End data with <CR><LF>.<CR><LF>     │
  │<─────────────────────────────────────────│
  │                                          │
  │  From: usuari@domini.cat                 │
  │  To: desti@example.com                   │
  │  Subject: Prova                          │
  │                                          │
  │  Contingut del missatge...               │
  │  .                                       │
  │──────────────────────────────────────────>│
  │  250 Ok: queued as 12345                 │
  │<─────────────────────────────────────────│
  │                                          │
  │  QUIT                                    │
  │──────────────────────────────────────────>│
  │  221 Bye                                 │
  │<─────────────────────────────────────────│
```

### Comandes SMTP Principals

| Comanda       | Descripció                            | Exemple                         |
| ------------- | ------------------------------------- | ------------------------------- |
| **HELO**      | Identificació del client (SMTP bàsic) | `HELO client.domini.cat`        |
| **EHLO**      | Identificació del client (ESMTP)      | `EHLO client.domini.cat`        |
| **MAIL FROM** | Especifica el remitent                | `MAIL FROM:<usuari@domini.cat>` |
| **RCPT TO**   | Especifica el destinatari             | `RCPT TO:<desti@example.com>`   |
| **DATA**      | Inicia la transferència del missatge  | `DATA`                          |
| **STARTTLS**  | Inicia xifrat TLS                     | `STARTTLS`                      |
| **AUTH**      | Autentica l'usuari                    | `AUTH LOGIN`                    |
| **QUIT**      | Tanca la connexió                     | `QUIT`                          |
| **RSET**      | Reinicia la transacció                | `RSET`                          |
| **VRFY**      | Verifica si un usuari existeix        | `VRFY usuari`                   |
| **NOOP**      | No operació (manté connexió viva)     | `NOOP`                          |

### Codis de Resposta SMTP

| Rang    | Categoria                | Exemples                                                        |
| ------- | ------------------------ | --------------------------------------------------------------- |
| **2xx** | Èxit                     | 220 Service ready, 250 OK, 235 Authentication successful        |
| **3xx** | Necessita més informació | 354 Start mail input                                            |
| **4xx** | Error temporal           | 421 Service not available, 450 Mailbox unavailable              |
| **5xx** | Error permanent          | 500 Syntax error, 550 Mailbox not found, 554 Transaction failed |

**Codis de resposta detallats:**

| Codi | Significat                         |
| ---- | ---------------------------------- |
| 220  | Service ready                      |
| 221  | Closing connection                 |
| 250  | Requested action completed         |
| 235  | Authentication successful          |
| 354  | Start mail input                   |
| 421  | Service not available              |
| 450  | Mailbox unavailable (temporalment) |
| 451  | Local error in processing          |
| 500  | Syntax error, command unrecognized |
| 501  | Syntax error in parameters         |
| 550  | Mailbox unavailable (permanent)    |
| 551  | User not local                     |
| 552  | Exceeded storage allocation        |
| 553  | Mailbox name not allowed           |
| 554  | Transaction failed                 |

### Protocol POP3 (Post Office Protocol version 3)

**POP3** és un protocol per **descarregar** correus del servidor al client. Està dissenyat per accedir a correus de manera simple.

**Característiques:**

- Protocol de **capa d'aplicació**
- **Descàrrega** de correus (normalment els esborra del servidor)
- **Simple** i lleuger
- **Un sol dispositiu** (no sincronitza entre dispositius)
- Ports:
  - **110**: POP3 sense xifrat
  - **995**: POP3S (POP3 sobre SSL/TLS)

**Comandes POP3:**

| Comanda  | Descripció                                 | Exemple                  |
| -------- | ------------------------------------------ | ------------------------ |
| **USER** | Nom d'usuari                               | `USER usuari@domini.cat` |
| **PASS** | Contrasenya                                | `PASS secret123`         |
| **STAT** | Estat de la bústia                         | `STAT` → `+OK 3 1024`    |
| **LIST** | Llista missatges                           | `LIST`                   |
| **RETR** | Recupera un missatge                       | `RETR 1`                 |
| **DELE** | Marca missatge per esborrar                | `DELE 1`                 |
| **QUIT** | Tanca connexió i esborra missatges marcats | `QUIT`                   |
| **RSET** | Reseteja marques d'esborrat                | `RSET`                   |
| **NOOP** | No operació                                | `NOOP`                   |

**Exemple de sessió POP3:**

```
CLIENT                          SERVIDOR
  │                                │
  │  Connexió TCP al port 110      │
  │<───────────────────────────────│
  │  +OK POP3 server ready         │
  │                                │
  │  USER usuari@domini.cat        │
  │────────────────────────────────>│
  │  +OK                           │
  │<───────────────────────────────│
  │                                │
  │  PASS secret123                │
  │────────────────────────────────>│
  │  +OK Logged in                 │
  │<───────────────────────────────│
  │                                │
  │  STAT                          │
  │────────────────────────────────>│
  │  +OK 2 3200                    │
  │<───────────────────────────────│
  │                                │
  │  LIST                          │
  │────────────────────────────────>│
  │  +OK 2 messages                │
  │  1 1200                        │
  │  2 2000                        │
  │  .                             │
  │<───────────────────────────────│
  │                                │
  │  RETR 1                        │
  │────────────────────────────────>│
  │  +OK 1200 octets               │
  │  [contingut del missatge]      │
  │  .                             │
  │<───────────────────────────────│
  │                                │
  │  DELE 1                        │
  │────────────────────────────────>│
  │  +OK                           │
  │<───────────────────────────────│
  │                                │
  │  QUIT                          │
  │────────────────────────────────>│
  │  +OK Bye                       │
  │<───────────────────────────────│
```

### Protocol IMAP (Internet Message Access Protocol)

**IMAP** és un protocol per **accedir** i **gestionar** correus que romanen al servidor. Permet sincronització entre múltiples dispositius.

**Característiques:**

- Protocol de **capa d'aplicació**
- **Sincronització** de correus (es mantenen al servidor)
- **Gestió avançada** de carpetes i etiquetes
- **Múltiples dispositius** (sincronització completa)
- **Accés parcial** (pot descarregar només capçaleres)
- Ports:
  - **143**: IMAP sense xifrat
  - **993**: IMAPS (IMAP sobre SSL/TLS)

**Comandes IMAP principals:**

| Comanda     | Descripció                      | Exemple                       |
| ----------- | ------------------------------- | ----------------------------- |
| **LOGIN**   | Autenticació                    | `a001 LOGIN usuari pass`      |
| **SELECT**  | Selecciona una carpeta          | `a002 SELECT INBOX`           |
| **EXAMINE** | Examina carpeta (només lectura) | `a003 EXAMINE INBOX`          |
| **LIST**    | Llista carpetes                 | `a004 LIST "" "*"`            |
| **FETCH**   | Recupera missatges              | `a005 FETCH 1 BODY[]`         |
| **SEARCH**  | Cerca missatges                 | `a006 SEARCH FROM "usuari"`   |
| **STORE**   | Modifica flags                  | `a007 STORE 1 +FLAGS (\Seen)` |
| **COPY**    | Copia missatges                 | `a008 COPY 1 Trash`           |
| **LOGOUT**  | Tanca sessió                    | `a009 LOGOUT`                 |

**Exemple de sessió IMAP:**

```
CLIENT                                SERVIDOR
  │                                      │
  │  Connexió TCP al port 143            │
  │<─────────────────────────────────────│
  │  * OK IMAP4 server ready             │
  │                                      │
  │  a001 LOGIN usuari@domini.cat pass   │
  │──────────────────────────────────────>│
  │  a001 OK Logged in                   │
  │<─────────────────────────────────────│
  │                                      │
  │  a002 SELECT INBOX                   │
  │──────────────────────────────────────>│
  │  * 3 EXISTS                          │
  │  * 0 RECENT                          │
  │  a002 OK [READ-WRITE] SELECT done    │
  │<─────────────────────────────────────│
  │                                      │
  │  a003 FETCH 1 BODY[]                 │
  │──────────────────────────────────────>│
  │  * 1 FETCH (BODY[] {1234}            │
  │  [contingut del missatge]            │
  │  )                                   │
  │  a003 OK FETCH completed             │
  │<─────────────────────────────────────│
  │                                      │
  │  a004 LOGOUT                         │
  │──────────────────────────────────────>│
  │  * BYE Logging out                   │
  │  a004 OK Logout completed            │
  │<─────────────────────────────────────│
```

### Comparació: POP3 vs IMAP

| Característica            | POP3                              | IMAP                            |
| ------------------------- | --------------------------------- | ------------------------------- |
| **Emmagatzematge**        | Descarrega i esborra del servidor | Manté correus al servidor       |
| **Sincronització**        | No                                | Sí (entre tots els dispositius) |
| **Accés offline**         | Sí (després de descarregar)       | Parcial (caché local)           |
| **Gestió de carpetes**    | No                                | Sí (carpetes al servidor)       |
| **Ús de disc servidor**   | Baix                              | Alt                             |
| **Ús de disc client**     | Alt                               | Baix                            |
| **Velocitat inicial**     | Més ràpid                         | Més lent (sincronitza tot)      |
| **Múltiples dispositius** | Problemàtic                       | Excel·lent                      |
| **Cerca de correus**      | Local (client)                    | Servidor                        |
| **Recomanat per**         | Un sol dispositiu                 | Múltiples dispositius           |

### Ports Utilitzats - Resum

| Protocol | Port | Xifrat                 | Ús                                |
| -------- | ---- | ---------------------- | --------------------------------- |
| SMTP     | 25   | No (opcional STARTTLS) | Transferència entre servidors     |
| SMTP     | 587  | STARTTLS               | Enviament per clients (recomanat) |
| SMTPS    | 465  | SSL/TLS                | Enviament xifrat (deprecated)     |
| POP3     | 110  | No                     | Descàrrega de correu              |
| POP3S    | 995  | SSL/TLS                | Descàrrega xifrada                |
| IMAP     | 143  | No (opcional STARTTLS) | Sincronització de correu          |
| IMAPS    | 993  | SSL/TLS                | Sincronització xifrada            |

### Xifrat en Protocols de Correu

**STARTTLS:**

- Comença amb connexió sense xifrar
- Client demana **STARTTLS**
- Servidor respon i inicia negociació TLS
- A partir d'aquí tot va xifrat
- Vulnerable a atacs **downgrade** si no es força

**SSL/TLS directe (ports dedicats):**

- Connexió xifrada des de l'inici
- Més segur (no hi ha fase sense xifrar)
- Utilitza ports dedicats (465, 993, 995)

**Recomanacions de seguretat:**

- Utilitzar **sempre STARTTLS o SSL/TLS**
- Port **587** amb STARTTLS per enviar correus
- Port **993** (IMAPS) o **995** (POP3S) per rebre
- **Deshabilitar** protocols sense xifrat en producció
