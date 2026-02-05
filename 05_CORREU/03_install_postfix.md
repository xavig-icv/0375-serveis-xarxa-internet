## 03. Instal·lació de Postfix (MTA)

### Instal·lació a Debian/Ubuntu

```bash
# Actualitzar els repositoris
sudo apt update

# Instal·lar Postfix
sudo apt install postfix

# Durant la instal·lació apareixerà un assistent de configuració:
# - Tipus de configuració: "Internet Site"
# - System mail name: domini.cat (el vostre domini --> educem.cat)
```

### Verificar la Instal·lació

```bash
# Verificar la versió de Postfix
postconf mail_version

# Verificar que el servei està actiu
sudo systemctl status postfix

# Verificar que escolta al port correcte (el 25)
sudo ss -plutn | grep master

# Sortida esperada:
# tcp   LISTEN  0  100  0.0.0.0:25   0.0.0.0:*   users:(("master",pid=...))
```

### Gestió bàsica del Servei

```bash
# Iniciar Postfix
sudo systemctl start postfix

# Aturar Postfix
sudo systemctl stop postfix

# Reiniciar Postfix
sudo systemctl restart postfix

# Recarregar configuració (sense aturar el servei)
sudo systemctl reload postfix

# Habilitar en l'arrencada
sudo systemctl enable postfix

# Verificar estat
sudo systemctl status postfix
```

### Estructura de Directoris

```
/etc/postfix/
├── main.cf                    # Configuració principal
├── master.cf                  # Configuració de processos
├── access                     # Control d'accés
├── virtual                    # Dominis virtuals
├── header_checks              # Filtres de capçaleres
├── body_checks                # Filtres de contingut
└── sasl/                      # Configuració SASL

/var/spool/postfix/            # Cua de correus
├── active/                    # Correus en cua activa
├── deferred/                  # Correus ajornats
├── incoming/                  # Correus entrants
├── maildrop/                  # Correus pendents de processar
└── hold/                      # Correus retinguts

/var/log/
└── mail.log                   # Log de Postfix (Debian/Ubuntu)

/var/mail/                     # Bústies d'usuaris (format mbox)
└── usuari                     # Bústia de l'usuari
```

### Verificar el Funcionament

**Comprovar que el servei escolta al port 25:**

```bash
sudo ss -plutn | grep :25

# Sortida esperada:
# tcp   0   0 0.0.0.0:25   0.0.0.0:*   LISTEN   1234/master
```

**Provar l'enviament de correu local:**

```bash
# Instal·lar l'eina "mail" per enviar correus
sudo apt install mailutils

# Enviar un correu de prova a un usuari local (el del vostre admin)
echo "Prova de correu" | mail -s "Test" usuari@localhost

# Verificar que s'ha rebut
sudo tail -f /var/log/mail.log

# O llegir-lo de la bústia
mail
```

### Eines per provar Protocols de Correu

Podeu fer ús de telnet per connectar-vos a ports TCP i enviar comandes manualment (així simulariem al servidor SMTP)

```bash
# Instal·lar l'eina "telnet" per fer connexió a ports d'un servidor
sudo apt install telnet

# Connectar al servidor SMTP
telnet localhost 25

# Un cop connectat, escriure les comandes SMTP:
EHLO localhost
MAIL FROM:<usuari1@domini.cat>
RCPT TO:<usuari2@domini.cat>
DATA
From: usuari1@domini.cat
To: usuari2@domini.cat
Subject: Prova

Aquest és un correu de prova.
.
QUIT
```

### Fitxers de Configuració Principals

**1. /etc/postfix/main.cf**

El fitxer `main.cf` disposa de la configuració principal de Postfix. Els paràmetres més importants són el de "domini" i "hostname":

```bash
# Veure configuració actual
postconf | grep mydomain
postconf | grep myhostname

# Veure un paràmetre específic
postconf myhostname

# Exemple de com modificar un paràmetre després de la instal·lació
sudo postconf -e 'myhostname=mail.domini.cat'
```

**2. /etc/postfix/master.cf**

Defineix els processos i serveis que executa Postfix:

El servei (smtp port 25) accepta connexions sense xifrar (amb STARTTLS opcional).
El servei (submission port 587) obliga l'ús de TLS.

```
# Servei   Tipus   Privat  Reús    Wake    Max     Comanda + args
smtp       inet    n       -       y       -       -       smtpd
submission inet    n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=encrypt
pickup     unix    n       -       y       60      1       pickup
cleanup    unix    n       -       y       -       0       cleanup
```

### Comandes Útils de Postfix

```bash
# Verificar sintaxi de configuració
sudo postfix check

# Recarregar configuració
sudo postfix reload

# Mostrar configuració actual (tots els paràmetres)
postconf

# Mostrar només configuració modificada (diferent de la configuració per defecte)
postconf -n

# Mostrar valors per defecte
postconf -d

# Modificar un paràmetre concreta de la configuració
sudo postconf -e 'paràmetre=valor'

# Veure la cua de correus
mailq

# Per veure la cua també pots fer servir:
postqueue -p

# Eliminar tots els correus de la cua
sudo postsuper -d ALL

# Eliminar correus ajornats (deferred)
sudo postsuper -d ALL deferred

# Forçar el reenviament de la cua
sudo postqueue -f

# Eliminar un correu específic per ID
sudo postsuper -d ID_DEL_CORREU

# Mostrar el contingut d'un correu de la cua
sudo postcat -q ID_DEL_CORREU
```

### Logs de Postfix

**Ubicació dels logs:**

```bash
# Debian/Ubuntu
/var/log/mail.log
/var/log/mail.err
```

**Visualitzar els logs a temps real:**

```bash
# Veure els logs de correu en directe
sudo tail -f /var/log/mail.log

# Filtrar només els missatges de Postfix
sudo tail -f /var/log/mail.log | grep postfix

# Cercar errors
sudo grep -i error /var/log/mail.log

# Veure correus enviats avui
sudo grep "$(date '+%b %d')" /var/log/mail.log | grep "status=sent"

# Veure correus rebutjats
sudo grep "$(date '+%b %d')" /var/log/mail.log | grep "reject"
```

### Resolució de Problemes Inicials

**El servei de Postfix no arrenca:**

```bash
# Verificar errors de sintaxi
sudo postfix check

# Verificar logs
sudo tail -50 /var/log/mail.log

# Verificar que no hi ha altre servei al port 25
sudo ss -plutn | grep :25

# Verificar permisos dels directoris
ls -ld /var/spool/postfix/
```

**No es poden enviar correus:**

```bash
# Verificar que el servei està actiu
sudo systemctl status postfix

# Verificar la cua
mailq

# Verificar logs
sudo tail -f /var/log/mail.log

# Provar enviament manual
echo "Test" | mail -s "Prova" usuari@localhost
```

**Els correus queden a la cua:**

```bash
# Veure per què estan retinguts
mailq

# Veure detalls d'un correu específic
sudo postcat -q ID_CORREU

# Forçar reenviament
sudo postqueue -f

# Verificar connectivitat DNS
dig MX mail.educem.cat
```

### Proves d'Enviament de Correus

**Mètode 1: Amb la comanda mail**

```bash
# Instal·lar mailutils si no ho estava previament
sudo apt install mailutils

# Enviar un correu simple
echo "Contingut del missatge" | mail -s "Assumpte" destinatari@domini.cat

# Enviar un correu amb un fitxer adjunt
echo "Missatge" | mail -s "Prova" -A /ruta/fitxer.pdf usuari@domini.cat
```

**Mètode 2: Amb sendmail**

```bash
# Enviar correu amb sendmail
sendmail destinatari@example.com << EOF
From: remitent@domini.cat
To: destinatari@example.com
Subject: Prova

Aquest és un correu de prova.
EOF
```

**Mètode 3: Amb telnet (manual SMTP)**

```bash
telnet localhost 25

# Comandes:
EHLO domini.cat
MAIL FROM:<remitent@domini.cat>
RCPT TO:<destinatari@example.com>
DATA
Subject: Prova SMTP
From: remitent@domini.cat
To: destinatari@example.com

Missatge de prova enviat manualment via SMTP.
.
QUIT
```

**Mètode 4: Amb l'eina SWAKS (Swiss Army Knife SMTP)**

```bash
sudo apt install swaks
# Enviar un correu simple
swaks --to destinatari@gmail.com --from usuari@domini.cat --server mail.domini.cat
```
