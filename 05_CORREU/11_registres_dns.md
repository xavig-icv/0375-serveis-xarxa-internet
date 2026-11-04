## 11. Registres DNS per Correu (MX, SPF, DKIM, DMARC)

### Introducció

Els registres DNS són essencials per al correu electrònic. Defineixen com els altres servidors troben i verifiquen el teu servidor de correu.

Al laboratori, els registres van a la zona `educem.cat` del Tema 1 (`/etc/bind/zones/db.educem.cat`), al servidor DNS primari:

- Els registres `MX` i `A` de `mail` ja hi són des del Tema 1: comproveu-los, no els dupliqueu.
- Cada vegada que afegiu o modifiqueu un registre, **incrementeu el serial** i recarregueu la zona (`sudo rndc reload`). Si no, el secundari no s'actualitza.

### Registre MX (Mail eXchanger)

**Funció:** Indica quin servidor gestiona el correu d'un domini.

**Format:**

```
educem.cat.    IN    MX    10    mail.educem.cat.
```

**Prioritat:** Número més baix = prioritat més alta.

**Exemple amb múltiples servidors:**

```dns
educem.cat.    IN    MX    10    mail1.educem.cat.
educem.cat.    IN    MX    20    mail2.educem.cat.
```

**Configurar:**

```bash
# A la zona DNS
@ IN MX 10 mail.educem.cat.

# També cal el registre A per al servidor
mail IN A 192.168.1.2
```

**Verificar:**

```bash
dig MX educem.cat
nslookup -type=MX educem.cat
```

### Registre A/AAAA

**Funció:** Resol el nom del servidor a una IP.

```dns
mail.educem.cat.    IN    A      192.168.1.2
```

### Registre PTR (DNS Invers)

**Funció:** Resol IP a nom de host. Crucial per evitar ser marcat com spam.

A Internet, el PTR de la IP pública del servidor de correu el configura el proveïdor d'Internet, i ha de coincidir amb el nom del servidor (`myhostname`):

```
IP pública del servidor    →    mail.example.com
```

Al laboratori, la zona inversa és nostra (Tema 1) i la `192.168.1.2` hi té el PTR `ns1.educem.cat`. Cada IP ha de tenir un sol PTR, i per al correu intern del laboratori n'hi ha prou amb aquest.

**Verificar:**

```bash
dig -x 192.168.1.2
host 192.168.1.2
```

### Registre SPF (Sender Policy Framework)

**Funció:** Defineix quins servidors poden enviar correu pel domini.

**Format:** Registre TXT

```dns
educem.cat.    IN    TXT    "v=spf1 ip4:192.168.1.2 -all"
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
dig TXT educem.cat
nslookup -type=TXT educem.cat
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
sudo mkdir -p /etc/opendkim/keys/educem.cat

# Generar clau
sudo opendkim-genkey -b 2048 -d educem.cat -D /etc/opendkim/keys/educem.cat -s default -v

# Resultat:
# - default.private (clau privada)
# - default.txt (clau pública per DNS)

# El servei opendkim s'executa amb l'usuari opendkim: ha de poder llegir la clau privada
sudo chown -R opendkim:opendkim /etc/opendkim/keys
sudo chmod 600 /etc/opendkim/keys/educem.cat/default.private
```

**Configurar OpenDKIM:**

```bash
sudo nano /etc/opendkim.conf
```

Afegiu aquestes línies. Substituïu la línia `Socket` que ja hi ha al fitxer, perquè no n'hi hagi dues:

```
Domain educem.cat
KeyFile /etc/opendkim/keys/educem.cat/default.private
Selector default
Socket inet:8891@localhost
```

**Veure clau pública:**

```bash
sudo cat /etc/opendkim/keys/educem.cat/default.txt
```

**Afegir registre DNS:**

El registre TXT conté la **clau pública** (el valor `p=` del fitxer `default.txt`). La clau privada no surt mai del servidor.

```dns
default._domainkey.educem.cat.    IN    TXT    "v=DKIM1; k=rsa; p=MIIBIjANBgkq...clau-publica...IDAQAB"
```

`Nota:` El fitxer `default.txt` ja té el format d'un registre de zona i divideix la clau en diverses cadenes entre parèntesis. BIND l'accepta tal qual: el podeu copiar a la zona.

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

Si més endavant afegiu un altre milter (per exemple, SpamAssassin a l'apartat 13), tots dos van a la mateixa línia `smtpd_milters`, separats per comes. Si escriviu una segona línia `smtpd_milters`, substitueix la primera i el DKIM deixa de funcionar.

**Iniciar OpenDKIM:**

```bash
sudo systemctl enable opendkim
sudo systemctl start opendkim
sudo systemctl restart postfix
```

**Verificar:**

```bash
dig TXT default._domainkey.educem.cat
```

### Registre DMARC

**Funció:** Política de validació SPF i DKIM. Indica què fer amb correus que fallen la verificació.

**Format:**

```dns
_dmarc.educem.cat.    IN    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@educem.cat"
```

**Paràmetres DMARC:**

| Paràmetre | Descripció           | Valors                         |
| --------- | -------------------- | ------------------------------ |
| `v`       | Versió               | `DMARC1`                       |
| `p`       | Política             | `none`, `quarantine`, `reject` |
| `sp`      | Subdomini policy     | `none`, `quarantine`, `reject` |
| `rua`     | Report aggregate URI | `mailto:reports@educem.cat`    |
| `ruf`     | Report forensic URI  | `mailto:forensic@educem.cat`   |
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
"v=DMARC1; p=none; rua=mailto:dmarc@educem.cat"

# Política estricta
"v=DMARC1; p=reject; rua=mailto:dmarc@educem.cat; ruf=mailto:forensic@educem.cat"

# Quarantena amb 50% aplicació
"v=DMARC1; p=quarantine; pct=50; rua=mailto:dmarc@educem.cat"
```

**Verificar:**

```bash
dig TXT _dmarc.educem.cat
```

### Configuració Completa DNS

**Exemple complet per educem.cat:**

```dns
; Registre A del servidor
mail                IN    A      192.168.1.2

; Registre MX
@                   IN    MX 10  mail.educem.cat.

; SPF
@                   IN    TXT    "v=spf1 mx a ip4:192.168.1.2 -all"

; DKIM (clau pública del fitxer default.txt)
default._domainkey  IN    TXT    "v=DKIM1; k=rsa; p=MIIBIjANBgkq.......clau-publica.......IDAQAB"

; DMARC
_dmarc              IN    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@educem.cat"
```

### Verificació i Test

**Verificar tots els registres:**

```bash
# MX
dig MX educem.cat +short

# A
dig A mail.educem.cat +short

# PTR (DNS invers)
dig -x 192.168.1.2 +short

# SPF
dig TXT educem.cat +short | grep spf

# DKIM
dig TXT default._domainkey.educem.cat +short

# DMARC
dig TXT _dmarc.educem.cat +short
```

### Millors Pràctiques

1. **Configurar sempre MX i PTR**
2. **SPF: Començar amb `~all`, després `-all`**
3. **DKIM: Claus de 2048 bits mínim**
4. **DMARC: Començar amb `p=none`, després `p=quarantine` o `p=reject`**
5. **Monitoritzar informes DMARC**
6. **Verificar registres amb eines online**
7. **Documentar tots els canvis DNS**
