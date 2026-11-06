## 04. Correu: Servidor de Reserva (MX Secundari)

### La Recepció de Correu Ja és Tolerant a les Fallades

El correu no necessita una IP virtual. El protocol SMTP ja té mecanismes perquè no es perdin missatges:

1. El servidor que envia un missatge a `usuari1@educem.cat` consulta els registres **MX** del domini i prova primer el de **preferència més baixa**.
2. Si no respon, prova el següent registre MX.
3. Si no en respon cap, guarda el missatge a la seva cua i **ho torna a provar durant dies** (5 dies per defecte a Postfix).

Per tant, si el 0375 està aturat una estona, el correu que li envien no es perd: arriba més tard. Un **MX secundari** fa que el correu es guardi a l'organització mentre el principal està aturat i que arribi tan bon punt es recupera:

```
                  educem.cat   MX 10  mail.educem.cat   (0375, principal)
                               MX 20  mail2.educem.cat  (0374, reserva)

 Servidor que envia
   1. prova l'MX 10 ──── X ──── 0375 aturat
   2. prova l'MX 20 ───────────► 0374: accepta el missatge i el guarda a la cua
                                    3. quan el 0375 torna ──SMTP──► 0375 ──► bústia
```

**Inconvenients d'un MX secundari:**

- **Correu brossa:** els emissors de correu brossa envien sovint als MX secundaris, perquè acostumen a tenir menys filtres que el principal.
- **Adreces vàlides:** el secundari ha de saber quines adreces existeixen. Si no, accepta correu per a adreces inexistents. Després, quan el principal el rebutja, el secundari envia un avís de retorn al remitent, que sovint és fals: és el ***backscatter***.
- **Configuració duplicada:** cal mantenir la llista d'adreces i els filtres als dos servidors.

Per això, avui moltes organitzacions no tenen MX secundari, o el contracten a un proveïdor especialitzat. Al laboratori el configurem per entendre'n el funcionament.

### Configurar el 0374 com a MX Secundari

**Al 0374**, instal·leu Postfix si no hi és. A la configuració inicial, trieu **Internet Site** i el nom `mail2.educem.cat`:

```bash
sudo apt install postfix
```

Si el 0374 ja tenia Postfix (per exemple, per enviar les alertes del projecte de monitoratge del mòdul 0374), es pot aprofitar. Els canvis d'aquest apartat no afecten l'enviament del correu del mateix servidor.

```bash
sudo postconf -e 'myhostname=mail2.educem.cat'
sudo postconf -e 'inet_interfaces=all'
sudo postconf -e 'mydestination=$myhostname, localhost'
sudo postconf -e 'relay_domains=educem.cat'
sudo postconf -e 'relay_recipient_maps=hash:/etc/postfix/relay_recipients'
sudo postconf -e 'transport_maps=hash:/etc/postfix/transport'
sudo postconf -e 'maximal_queue_lifetime=10d'
```

| Paràmetre                  | Funció                                                                          |
| -------------------------- | ------------------------------------------------------------------------------- |
| `mydestination`            | **No** hi ha d'haver `educem.cat`: el 0374 no té les bústies, només guarda el correu i el reenvia |
| `relay_domains`            | Dominis per als quals el 0374 accepta correu per reenviar-lo                    |
| `relay_recipient_maps`     | Adreces vàlides d'aquests dominis. Les altres es rebutgen d'entrada             |
| `transport_maps`           | A quin servidor es lliura el correu del domini: al principal                    |
| `maximal_queue_lifetime`   | Temps que el 0374 guarda un missatge abans de retornar-lo al remitent (10 dies) |

**El servidor on es lliura el correu:**

```bash
sudo nano /etc/postfix/transport
```

```
# Correu d'educem.cat: lliurar-lo al servidor principal.
# Els claudàtors fan que no es consultin els registres MX (s'hi connecta directament)
educem.cat    smtp:[mail.educem.cat]:25
```

**Les adreces vàlides:**

```bash
sudo nano /etc/postfix/relay_recipients
```

```
# Adreces d'educem.cat que existeixen al servidor principal
usuari1@educem.cat       OK
usuari2@educem.cat       OK
postmaster@educem.cat    OK
suport@educem.cat        OK
info@educem.cat          OK
```

Afegiu-hi també els àlies i les llistes de distribució que tingueu al 0375 (Temes 5 i 6). Aquesta llista s'ha de mantenir igual que la del principal. En un entorn real, es genera automàticament o es consulta al directori LDAP.

```bash
sudo postmap /etc/postfix/transport /etc/postfix/relay_recipients
sudo postfix check
sudo systemctl restart postfix
sudo ufw allow from 192.168.1.0/24 to any port 25 proto tcp
```

### Al 0375: Acceptar el Correu que Reenvia el 0374

El 0375 ja accepta correu per al seu domini des de qualsevol servidor. Però si li heu aplicat les restriccions del Tema 5 (apartat 12), com les llistes negres i les comprovacions del remitent, afegiu el 0374 a la llista blanca. Així, el correu que el 0374 reenvia no es rebutja per error:

```bash
echo "192.168.1.1    OK" | sudo tee -a /etc/postfix/whitelist
sudo postmap /etc/postfix/whitelist
sudo systemctl reload postfix
```

### Proves

Al **client**, instal·leu `swaks` per enviar correu directament a cada servidor, com faria un servidor de fora:

```bash
sudo apt install swaks
dig +short MX educem.cat               # 10 mail.educem.cat. i 20 mail2.educem.cat.
```

**1. Funcionament normal.** El 0374 reenvia el missatge de seguida:

```bash
swaks --server mail2.educem.cat --to usuari1@educem.cat --from usuari2@educem.cat \
    --header "Subject: Prova a través de l'MX secundari"
```

Al 0374, el registre mostra el lliurament al principal:

```bash
sudo grep 'relay=mail.educem.cat' /var/log/mail.log | tail -2      # status=sent
```

**2. Adreça inexistent.** El 0374 la rebutja d'entrada, sense acceptar el missatge:

```bash
swaks --server mail2.educem.cat --to noexisteix@educem.cat --from usuari2@educem.cat
# 550 5.1.1 <noexisteix@educem.cat>: Recipient address rejected: User unknown in relay recipient table
```

**3. El principal no funciona.** Al 0375, atureu Postfix:

```bash
sudo systemctl stop postfix          # al 0375
```

Al client, intenteu enviar al principal (no respon) i, després, al secundari, com faria un servidor remitent:

```bash
swaks --server mail.educem.cat --to usuari1@educem.cat --from usuari2@educem.cat     # falla: Connection refused
swaks --server mail2.educem.cat --to usuari1@educem.cat --from usuari2@educem.cat \
    --header "Subject: Enviat mentre el principal estava aturat"                    # 250 OK
```

Al 0374, el missatge espera a la cua:

```bash
mailq
# ... (connect to mail.educem.cat[192.168.1.2]:25: Connection refused)
```

**4. El principal es recupera.**

```bash
sudo systemctl start postfix         # al 0375
sudo postqueue -f                    # al 0374: reintenta ara els missatges de la cua
mailq                                # Mail queue is empty
```

Si no feu `postqueue -f`, Postfix ho torna a provar sol al cap d'uns minuts. Comproveu que el missatge és a la bústia de `usuari1` (Thunderbird o `/home/usuari1/Maildir/new/`).

### I Llegir el Correu?

L'MX secundari garanteix la **recepció**, però les bústies (IMAP i POP3, Dovecot) només són al 0375. Mentre el principal està aturat, els usuaris no poden llegir el correu. Per fer redundants les bústies hi ha diverses opcions:

- **Emmagatzematge compartit:** les bústies són en un servidor de fitxers (NFS, mòdul 0374) que tots dos servidors de correu munten. Llavors, l'emmagatzematge ha de ser redundant.
- **Replicació de les bústies:** Dovecot 2.3 (Ubuntu 24.04) pot replicar les bústies entre dos servidors. **Dovecot 2.4 (Ubuntu 26.04) ha eliminat aquesta funció** i recomana l'emmagatzematge compartit.
- **Correu al núvol:** molts centres i empreses deixen el correu en mans d'un proveïdor (Microsoft 365, Google Workspace), que ja és redundant.

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                                          |
| -------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| `Relay access denied` al 0374                                  | `educem.cat` no és a `relay_domains`                     | `postconf relay_domains`                                         |
| `User unknown in relay recipient table` amb una adreça vàlida  | No és a `relay_recipients`, o no s'ha fet `postmap`      | Afegir-la i fer `sudo postmap /etc/postfix/relay_recipients`     |
| `mail for educem.cat loops back to myself`                     | `educem.cat` és a `mydestination` del 0374, o falta el transport | Treure-la de `mydestination`; revisar `transport_maps`    |
| El missatge es queda a la cua del 0374 amb el principal funcionant | El principal el rebutja                              | El motiu és a `mailq`; afegir el 0374 a la llista blanca del 0375 |
| El client no pot connectar al port 25 del 0374                 | Postfix només escolta a `localhost`, o el tallafoc       | `postconf inet_interfaces` (ha de ser `all`); `sudo ufw status`  |

### Punt de Control

- [ ] `dig MX educem.cat` mostra els dos servidors, amb preferències 10 i 20.
- [ ] El 0374 accepta correu per a les adreces vàlides d'`educem.cat` i el reenvia al 0375.
- [ ] El 0374 rebutja les adreces que no existeixen.
- [ ] Amb Postfix aturat al 0375, el correu queda a la cua del 0374 i arriba quan el 0375 es recupera.
