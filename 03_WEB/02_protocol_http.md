## 02. Protocols HTTP i HTTPS

### Protocol HTTP

**HTTP** (HyperText Transfer Protocol) és la base de la comunicació entre clients i servidors web. Defineix com es realitzen les peticions i com es responen.

**Característiques:**

- Protocol de **capa d'aplicació** (Capa 7 del Model OSI)
- Basat en **text pla** (és completament llegible pels humans)
- **Stateless** (sense estat entre peticions, cada petició és independent)
- Port per defecte: **80**

**Versions:**

- **HTTP/1.0** (1996): Una connexió per cada petició
- **HTTP/1.1** (1997): Connexions persistents (keep-alive) que milloren el rendiment
- **HTTP/2** (2015): Multiplexing (diverses peticions en una única connexió), compressió de les capçaleres (header compression)
- **HTTP/3** (2022): Funciona sobre QUIC (utilitza UDP en lloc de TCP)

### Petició HTTP (Request)

Missatge que envia el client al servidor per sol·licitar un recurs.

**Estructura d'una petició HTTP:**

```
GET /index.html HTTP/1.1
Host: www.exemple.cat
User-Agent: Mozilla/5.0 (Linux)
Accept: text/html,application/xhtml+xml
Accept-Language: ca,es;q=0.9,en;q=0.8
Connection: keep-alive

[Cos del missatge - opcional]
```

**Parts d'una petició:**

1. **Línia de petició**: Mètode HTTP + URL o ruta del recurs + Versió HTTP
2. **Headers**: Metadades (informació addicional sobre el client i la petició)
3. **Línia en blanc** (és el final dels headers)
4. **Cos** (opcional): Dades enviades al servidor amb mètodes com (POST o PUT)

**Mètodes HTTP:**

| Mètode  | Descripció                             |
| ------- | -------------------------------------- |
| GET     | Obtenir un recurs                      |
| POST    | Enviar dades / Crear un recurs         |
| PUT     | Actualitzar un recurs                  |
| DELETE  | Eliminar un recurs                     |
| HEAD    | Com GET però sense cos                 |
| OPTIONS | Opcions de comunicació amb el servidor |
| PATCH   | Modificació parcial                    |

### Resposta HTTP (Response)

Missatge que envia el servidor al servidor després de processar la seva petició.

**Estructura d'una resposta HTTP:**

```
HTTP/1.1 200 OK
Date: Thu, 09 Jan 2026 10:00:00 GMT
Server: Apache/2.4.52 (Ubuntu)
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Last-Modified: Wed, 08 Jan 2026 15:30:00 GMT
Connection: keep-alive

<!DOCTYPE html>
<html>
<head><title>Exemple</title></head>
<body><h1>Hola món!</h1></body>
</html>
```

**Parts d'una resposta:**

1. **Línia d'estat**: Versió HTTP + Codi estat + Missatge descriptiu
2. **Headers**: Metadades (informació addicional del servidor i del contingut)
3. **Línia en blanc**
4. **Cos**: Contingut retornat (HTML, JSON, fitxers, etc.)

**Codis d'estat HTTP:**

| Rang | Categoria      | Exemples                                                            |
| ---- | -------------- | ------------------------------------------------------------------- |
| 1xx  | Informatiu     | 100 Continue                                                        |
| 2xx  | Èxit           | 200 OK, 201 Created, 204 No Content                                 |
| 3xx  | Redirecció     | 301 Moved Permanently, 302 Found, 304 Not Modified                  |
| 4xx  | Error Client   | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found     |
| 5xx  | Error Servidor | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

### Protocol HTTPS

**HTTPS** (HyperText Transfer Protocol Secure) és la versió segura del protocol HTTP. Disposa d'una capa de xifrat mitjançant els protocols SSL/TLS.

**Diferències clau amb HTTP:**

- Comunicacions xifrdes (confidencialitat)
- Autenticació del servidor (mitjançant un certificat digital)
- Integritat de dades (les dades no es poden manipular)
- Port per defecte: 443

**Procés de connexió HTTPS (TLS Handshake):**

```
CLIENT                                   SERVIDOR
  │                                          │
  │  1. ClientHello                          │
  │  (versions TLS, xifratges suportats)     │
  │──────────────────────────────────────────>│
  │                                          │
  │  2. ServerHello                          │
  │  (versió TLS escollida, xifratge)        │
  │  + Certificat del servidor               │
  │<──────────────────────────────────────────│
  │                                          │
  │  3. Client verifica certificat           │
  │  (cadena de confiança, CA)               │
  │                                          │
  │  4. ClientKeyExchange                    │
  │  (clau de sessió xifrada)                │
  │──────────────────────────────────────────>│
  │                                          │
  │  5. Finished                             │
  │<─────────────────────────────────────────>│
  │                                          │
  │  6. Comunicació xifrada amb clau sessió  │
  │<─────────────────────────────────────────>│
```

**Beneficis d'HTTPS:**

- ✅ Protegeix dades sensibles (contrasenyes, targetes, dades de salut, etc.)
- ✅ Millora el SEO (Google prioritza HTTPS)
- ✅ Confiança dels usuaris (candau verd)
- ✅ Requisit per fer ús de HTTP/2 i HTTP/3
- ✅ Evita advertències del navegador
