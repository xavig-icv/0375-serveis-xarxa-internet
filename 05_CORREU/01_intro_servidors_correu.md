## 01. Introducció als Servidors de Correu

### Què és un Servidor de Correu?

Un **servidor de correu** és un sistema que gestiona l'enviament, recepció, emmagatzematge i distribució de missatges de correu electrònic. Funciona com una oficina de correus digital, encarregant-se de transferir els missatges entre remitents i destinataris a través d'Internet.

El servei de correu electrònic és responsable de:

- Rebre correus entrants de servidors externs
- Enviar correus sortints cap a altres servidors
- Emmagatzemar missatges a les bústies dels usuaris
- Autenticar usuaris i verificar permisos
- Filtrar spam i missatges maliciosos

**Funcionament bàsic:**

```
CLIENT (Thunderbird/Mail)              SERVIDOR CORREU
      │                                       │
      │  1. Enviar correu (SMTP)              │
      │    MAIL FROM: usuari@domini.cat       │
      │    RCPT TO: desti@example.com         │
      │──────────────────────────────────────>│
      │                                       │
      │                                       │ 2. Processa el missatge
      │                                       │    - Verifica el destinatari
      │                                       │    - Filtra l'spam
      │                                       │    - Emmagatzema/Reenvia
      │                                       │
      │  3. Rebre correu (POP3/IMAP)          │
      │    USER usuari@domini.cat             │
      │    PASS ********                      │
      │<──────────────────────────────────────│
      │                                       │
      │  4. Descarrega/Sincronitza missatges  │
```

### Components d'un Sistema de Correu

Un servidor de correu està composat per diversos components que treballen conjuntament:

**1. MTA (Mail Transfer Agent) - Agent de Transferència de Correu**

- S'encarrega d'**enviar** i **rebre** correus entre servidors
- Utilitza el protocol **SMTP** (port 25) comunicació entre servidors
- Exemples: **Postfix**, Sendmail, Exim

**2. MDA (Mail Delivery Agent) - Agent de Lliurament de Correu**

- S'encarrega de **lliurar** (permet descarregar) els correus a les bústies dels usuaris
- Permet als usuaris **accedir** als seus correus
- Utilitza protocols **POP3** (port 110) i **IMAP** (port 143)
- Exemples: **Dovecot**, Courier

**3. MUA (Mail User Agent) - Client de Correu**

- Aplicació que utilitza l'usuari per gestionar el correu
- Exemples: **Thunderbird**, Outlook, Apple Mail, webmail

### Servidors de Correu Populars

**Postfix (MTA)**

- És el MTA més utilitzat actualment
- Suporta el protocol SMTP
- És segurr, ràpid i proporciona bon rendiment amb un alt volum
- Compatible amb Sendmail

**Dovecot (MDA)**

- És el MDA més popular
- Suporta els protocols POP3 i IMAP
- Bon rendiment i seguretat
- Fàcil de configurar

### Casos d'Ús

**Postfix és millor per:**

- Servidors amb molt volum de correu
- És prioritza la seguretat del servei
- Integració amb sistemes antispam

**Dovecot és millor per:**

- Gestió de bústies d'usuaris
- Suport IMAP amb sincronització
- Integració amb autenticació LDAP

**Solució completa:**

```
Postfix (MTA - Envia/Rep)
   ↓
Dovecot (MDA - Emmagatzema/Serveix)
   ↓
Clients (Thunderbird, Webmail)
```

### Ports Utilitzats en Servidors de Correu

| Port    | Protocol | Descripció                          | Xifrat                 |
| ------- | -------- | ----------------------------------- | ---------------------- |
| **25**  | SMTP     | Transferència entre servidors       | No (opcional STARTTLS) |
| **587** | SMTP     | Enviament per clients (submission)  | STARTTLS               |
| **465** | SMTPS    | SMTP amb SSL/TLS (no el recomano)   | SSL/TLS                |
| **110** | POP3     | Recepció de correu (descàrrega)     | No                     |
| **995** | POP3S    | POP3 amb SSL/TLS                    | SSL/TLS                |
| **143** | IMAP     | Recepció de correu (sincronització) | No                     |
| **993** | IMAPS    | IMAP amb SSL/TLS                    | SSL/TLS                |

### Flux Complet d'un Correu Electrònic

```
1. USUARI A (usuari@domini.cat) escriu un correu
   ↓
2. CLIENT (Thunderbird) connecta a POSTFIX via SMTP:587
   ↓
3. POSTFIX autentica l'usuari (SASL)
   ↓
4. POSTFIX accepta el correu i el processa
   ↓
5. POSTFIX consulta DNS per trobar servidor MX de destí
   ↓
6. POSTFIX connecta al servidor de destí via SMTP:25
   ↓
7. SERVIDOR DESTÍ rep el correu
   ↓
8. SERVIDOR DESTÍ passa correu a DOVECOT
   ↓
9. DOVECOT emmagatzema correu a bústia d'USUARI B
   ↓
10. USUARI B consulta correu amb IMAP:143
    ↓
11. DOVECOT serveix missatge a CLIENT d'USUARI B
```
