## 05. Procés de Resolució de Noms

### Consultes Recursives vs Iteratives

#### Consulta Recursiva

El resolver realitza una consulta de nom complet (FQDN) i retorna la resposta final al client.

```
Client → Resolver: "Què és www.exemple.cat?"
                ↓
        Resolver fa totes les consultes
                ↓
Resolver → Client: "És 174.18.120.12"
```

**Flux complet:**

```
[Client] → Resolver DNS
             ↓ (recursiu)
         Root Server
             ↓
         TLD Server (.com)
             ↓
   Authoritative Server
             ↓
        Resposta final → Client
```

#### Consulta Iterativa

El resolver retorna la millor resposta parcial que disposa i el client continua consultant. Això passa quan un servidor rep una consulta recursiva per la que no té resposta, comença un procés iteratiu consultant a altres servidors.

```
Client → Root: "On està .cat?"
Root → Client: "Pregunta a 192.5.6.30"
Client → TLD: "On està exemple.cat?"
TLD → Client: "Pregunta a 90.118.206.30"
Client → Auth: "Què és www.exemple.cat?"
Auth → Client: "És 174.18.120.12"
```

### Funcionament del Resolver

El **resolver** és el component que actua d'intermediari entre les aplicacions i els serveis de DNS, gestionant les consultes DNS que es realitzen. El "resolver" pot ser:

- **Stub resolver**: Client DNS simple (només fa consultes recursives)
- **Recursive resolver**: Servidor que resol consultes completes
- **Forwarding resolver**: Reenvia consultes a un altre resolver

**Configuració típica a Linux (`/etc/resolv.conf`):**

```
nameserver 8.8.8.8
nameserver 1.1.1.1
search local.domain
```

### Cache DNS i Optimització

#### Funcionament del Cache

```
1a consulta de www.google.com:
Client → Resolver → Root → TLD → Auth → Resolver → Client
(lent: ~100-500ms)

2a consulta de www.google.com (dins de TTL):
Client → Resolver (cache) → Client
(ràpid: ~1-5ms)
```

#### Time To Live (TTL)

El TTL determina quant temps es guarda un registre a cache:

```
exemple.cat.    300    IN    A    174.18.120.12
                 ↑
                TTL (en segons)
```

**Recomanacions de TTL:**

- **86400** (24h): Registres molt estables
- **3600** (1h): Registres normals
- **300** (5min): Registres que canvien sovint
- **60** (1min): Quan es realitza una migració

#### Optimització de Consultes

**Tècniques:**

1. **Prefetching**: Renovar cache abans que expiri
2. **Negative caching**: Guardar respostes NXDOMAIN
3. **DNSSEC**: Validació criptogràfica (més lent però segur)
4. **Cache compartida**: Usar resolver local per múltiples clients

**Exemple de configuració de cache en BIND:**

```
options {
    max-cache-size 256M;
    max-cache-ttl 86400;
    max-ncache-ttl 3600;
};
```

### Privacitat i Seguretat DNS

Tota la informació que es transmet a través del protocol DNS viatja en text clar, per tant:

- Qualsevol dispositiu en la xarxa pot veure quins dominis visita un usuari.
- Això pot exposar preferències, navegació i perfils d’usuari

**Solucions de xifrat:**

| Protocol                 | Com funciona                           | Port típic |
| ------------------------ | -------------------------------------- | ---------- |
| **DoT (DNS over TLS)**   | Xifra les consultes DNS utilitzant TLS | 853        |
| **DoH (DNS over HTTPS)** | Xifra les consultes DNS dins HTTPS     | 443        |

**Avantatges:**

- El contingut de les consultes DNS no es veu a la xarxa.
- DoH és especialment útil quan el trànsit DNS ha de passar per proxies o xarxes públiques, ja que es barreja amb HTTPS normal.

Flux resum amb DoT/DoH

```
[Client DNS]
   │
   ▼ (xifrat: TLS / HTTPS)
[Servidor Recursiu]
   │
   ▼
Root → TLD → Authoritative
   │
   ▼ (xifrat de tornada)
[Resposta final al client]
```
