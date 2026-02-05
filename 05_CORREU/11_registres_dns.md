## 15. Registres DNS per Correu (MX, SPF, DKIM, DMARC)

### Introducció

Els registres DNS són essencials per al correu electrònic. Defineixen com els altres servidors troben i verifiquen el teu servidor de correu.

### Registre MX (Mail eXchanger)

**Funció:** Indica quin servidor gestiona el correu d'un domini.

**Format:**

```
domini.cat.    IN    MX    10    mail.domini.cat.
```

**Prioritat:** Número més baix = prioritat més alta.

**Exemple amb múltiples servidors:**

```dns
domini.cat.    IN    MX    10    mail1.domini.cat.
domini.cat.    IN    MX    20    mail2.domini.cat.
```

**Configurar:**

```bash
# A la zona DNS
@ IN MX 10 mail.domini.cat.

# També cal el registre A per al servidor
mail IN A 192.168.1.2
```

**Verificar:**

```bash
dig MX domini.cat
nslookup -type=MX domini.cat
```

### Registre A/AAAA

**Funció:** Resol el nom del servidor a una IP.

```dns
mail.domini.cat.    IN    A      192.168.1.2
```

### Registre PTR (DNS Invers)

**Funció:** Resol IP a nom de host. Crucial per evitar ser marcat com spam.

**Configurar al proveïdor d'IP:**

```
192.168.1.2    →    mail.domini.cat
```

**Verificar:**

```bash
dig -x 192.168.1.2
host 192.168.1.2
```

### Registre SPF (Sender Policy Framework)

**Funció:** Defineix quins servidors poden enviar correu pel domini.

**Format:** Registre TXT

```dns
domini.cat.    IN    TXT    "v=spf1 ip4:192.168.1.2 -all"
```

Amb el `-all` rebutgem tota la resta de servidors que no siguin 192.168.1.2

**Exemples:**

```dns
# Només permet el servidor propi de correu
v=spf1 mx a -all

# Incloure Google Workspace com a servidor de correu
v=spf1 include:_spf.google.com -all

# Múltiples IPs
v=spf1 ip4:192.168.1.1 ip4:192.168.1.2 -all
```

**Verificar SPF en local:**

```bash
dig TXT domini.cat
nslookup -type=TXT domini.cat
```

### Registre DKIM (DomainKeys Identified Mail)

**Funció:** Signa els correus amb una clau criptogràfica per verificar l'autenticitat.

**Instal·lar OpenDKIM:**

```bash
sudo apt install opendkim opendkim-tools
```

**Generar claus:**

```bash
# Crear directori
sudo mkdir -p /etc/opendkim/keys/domini.cat

# Generar clau
sudo opendkim-genkey -b 2048 -d domini.cat -D /etc/opendkim/keys/domini.cat -s default -v

# Resultat:
# - default.private (clau privada)
# - default.txt (clau pública per DNS)
```

**Configurar OpenDKIM:**

```bash
sudo nano /etc/opendkim.conf
```

```
Domain domini.cat
KeyFile /etc/opendkim/keys/domini.cat/default.private
Selector default
Socket inet:8891@localhost
```

**Veure clau pública:**

```bash
sudo cat /etc/opendkim/keys/domini.cat/default.txt
```

**Afegir registre DNS:**

```dns
default._domainkey.domini.cat.    IN    TXT    "v=DKIM1; k=rsa; p=MIGfMA0GCSq...clau-privada...QIDAQAB"
```

**Integrar amb Postfix:**

```bash
sudo nano /etc/postfix/main.cf
```

```bash
# OpenDKIM
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

**Iniciar OpenDKIM:**

```bash
sudo systemctl enable opendkim
sudo systemctl start opendkim
sudo systemctl restart postfix
```

**Verificar:**

```bash
dig TXT default._domainkey.domini.cat
```

### Registre DMARC

**Funció:** Política de validació SPF i DKIM. Indica què fer amb correus que fallen la verificació.

**Format:**

```dns
_dmarc.domini.cat.    IN    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@domini.cat"
```

**Paràmetres DMARC:**

| Paràmetre | Descripció           | Valors                         |
| --------- | -------------------- | ------------------------------ |
| `v`       | Versió               | `DMARC1`                       |
| `p`       | Política             | `none`, `quarantine`, `reject` |
| `sp`      | Subdomini policy     | `none`, `quarantine`, `reject` |
| `rua`     | Report aggregate URI | `mailto:reports@domini.cat`    |
| `ruf`     | Report forensic URI  | `mailto:forensic@domini.cat`   |
| `pct`     | Percentatge aplicat  | `100` (per defecte)            |
| `adkim`   | Alineació DKIM       | `r` (relaxed), `s` (strict)    |
| `aspf`    | Alineació SPF        | `r` (relaxed), `s` (strict)    |

**Polítiques:**

- `p=none`: Només monitoritza (recomanat inicialment)
- `p=quarantine`: Marca com spam
- `p=reject`: Rebutja directament

**Exemples:**

```dns
# Política suau (monitorització)
"v=DMARC1; p=none; rua=mailto:dmarc@domini.cat"

# Política estricta
"v=DMARC1; p=reject; rua=mailto:dmarc@domini.cat; ruf=mailto:forensic@domini.cat"

# Quarantena amb 50% aplicació
"v=DMARC1; p=quarantine; pct=50; rua=mailto:dmarc@domini.cat"
```

**Verificar:**

```bash
dig TXT _dmarc.domini.cat
```

### Configuració Completa DNS

**Exemple complet per domini.cat:**

```dns
; Registre A del servidor
mail                IN    A      192.168.1.2

; Registre MX
@                   IN    MX 10  mail.domini.cat.

; SPF
@                   IN    TXT    "v=spf1 mx a ip4:192.168.1.2 -all"

; DKIM
default._domainkey  IN    TXT    "v=DKIM1; k=rsa; p=MIGfMA0GCS.......clau-privada.......DCBiQKBgQC..."

; DMARC
_dmarc              IN    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@domini.cat"
```

### Verificació i Test

**Verificar tots els registres:**

```bash
# MX
dig MX domini.cat +short

# A
dig A mail.domini.cat +short

# PTR (DNS invers)
dig -x 192.168.1.2 +short

# SPF
dig TXT domini.cat +short | grep spf

# DKIM
dig TXT default._domainkey.domini.cat +short

# DMARC
dig TXT _dmarc.domini.cat +short
```

### Millors Pràctiques

1. **Configurar sempre MX i PTR**
2. **SPF: Començar amb `~all`, després `-all`**
3. **DKIM: Claus de 2048 bits mínim**
4. **DMARC: Començar amb `p=none`, després `p=quarantine` o `p=reject`**
5. **Monitoritzar informes DMARC**
6. **Verificar registres amb eines online**
7. **Documentar tots els canvis DNS**
