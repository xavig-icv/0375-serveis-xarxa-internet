## 08. Virtual Hosts (Llocs Web Virtuals)

Els Virtual Hosts permeten allotjar diversos llocs web en un únic servidor, compartint els mateixos recursos de maquinari i sistema operatiu.

Gràcies a aquesta tècnica o funcionalitat, un servidor web com Apache o Nginx pot servir diferents pàgines web segons la petició del client.

Aquest tipus d'infraestructura és la que utilitzen els hostings compartits.

**Tipus de Virtual Host:**

**1. Named-based (Basat en nom)**

El servidor determina quin lloc web ha de servir a partir del nom de domini sol·licitat pel client (el camp "host" de la petició HTTP).

- És el tipus més habitual i eficient
- Permet múltiples dominis amb una sola adreça IP

```
Client sol·licita: www.exemple1.cat
    ↓
Apache/Nginx: Serveix /var/www/exemple1
```

```
Client sol·licita: www.exemple2.cat
    ↓
Apache/Nginx: Serveix /var/www/exemple2
```

**2. IP-based (Basat en IP)**

Cada lloc web està associat a una adreça IP diferent. El servidor decideix quin lloc servir segons la IP a la qual s'ha connectat el client.

- És útil quan cal separar serveis per IP
- Però requereix diverses adreces IP operatives

```
Client sol·licita: 192.168.1.10
    ↓
Apache/Nginx: Serveix lloc 1

Client sol·licita: 192.168.1.11
    ↓
Apache/Nginx: Serveix lloc 2
```

**3. Port-based (Basat en port)**

El servidor diferencia els llocs web segons el port utilitzat en la connexió.

```
Client sol·licita: example.cat:80
    ↓
Apache/Nginx: Serveix lloc 1

Client sol·licita: example.cat:8080
    ↓
Apache/Nginx: Serveix lloc 2
```

- És útil per entorns de proves o serveis interns
- És menys habitual en producció, però pot fer-se servir
