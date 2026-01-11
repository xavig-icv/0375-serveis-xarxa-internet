## 01. Introducció als Servidors Web

### Què és un Servidor Web?

Un **servidor web** és un software que serveix contingut (pàgines HTML, imatges, estils CSS, JavaScript, etc.) a clients que ho sol·liciten a través de la mateixa xarxa local (LAN) o mitjançant Internet.

El servei web és el responsable de rebre peticions dels clients (navegadors, aplicacions, APIs) i respondre amb contingut web o dades processades, utilitzant principalment els protocols HTTP i HTTPS.

El servidor web pot:

- Servir contingut estàtic (HTML, imatges, CSS)
- Servir contingut dinàmic (PHP, Python, Node.js)
- Actuar com a punt d'entrada cap a altres serveis

**Funcionament bàsic:**

```
CLIENT (Navegador)                    SERVIDOR WEB
      │                                    │
      │  1. Petició HTTP                   │
      │    GET /index.html HTTP/1.1        │
      │───────────────────────────────────>│
      │                                    │
      │                                    │ 2. Processa petició
      │                                    │    - Busca fitxer
      │                                    │    - Executa codi
      │                                    │    - Genera resposta
      │                                    │
      │  3. Resposta HTTP                  │
      │    200 OK                          │
      │    Content-Type: text/html         │
      │    <html>...</html>                │
      │<───────────────────────────────────│
      │                                    │
      │  4. Renderitza pàgina              │
```

### Servidors Web Populars

**Apache HTTP Server**

- El més utilitzat històricament
- Molt configurable i flexible
- Gran ecosistema de mòduls
- Consum de recursos mitjà-alt
- Millor per contingut dinàmic

**Nginx**

- Molt ràpid i eficient
- Baix consum de recursos
- Excel·lent com a reverse proxy
- Millor per contingut estàtic
- Arquitectura asíncrona

### Casos d'Ús

**Apache és millor per:**

- Hosting compartit (per `.htaccess`)
- Aplicacions PHP complexes
- Necessitat de molts mòduls
- Compatibilitat amb projectes antics

**Nginx és millor per:**

- Reverse proxy / Load balancer
- Servir fitxers estàtics
- Microserveis
- Molt trànsit d'usuaris

**Solució híbrida:**

```
Internet
   ↓
Nginx (Reverse Proxy)
   ↓
Apache (Backend per PHP)
```
