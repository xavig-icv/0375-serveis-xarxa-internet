## 04. Clients de Missatgeria: Gràfics i de Text

### Clients XMPP Habituals

Com que XMPP és un protocol obert, hi ha clients de molts desenvolupadors diferents. Tots es poden connectar al mateix servidor:

| Client             | Tipus  | Sistemes                  | Paquet a Ubuntu |
| ------------------ | ------ | ------------------------- | --------------- |
| **Gajim**          | Gràfic | Linux, Windows            | `gajim`         |
| **Dino**           | Gràfic | Linux                     | `dino-im`       |
| **Pidgin**         | Gràfic, multiprotocol | Linux, Windows | `pidgin`       |
| **Conversations**  | Mòbil  | Android                   | -               |
| **Monal**          | Mòbil  | iOS, macOS                | -               |
| **Profanity**      | Text   | Linux                     | `profanity`     |
| **mcabber**        | Text   | Linux                     | `mcabber`       |

Al laboratori farem servir **Gajim** (gràfic) al client Ubuntu i **Profanity** (text) al client o al servidor.

### Confiar en el Certificat del Laboratori

El certificat de Prosody és autosignat (apartat 03). Hi ha dues opcions perquè els clients l'acceptin:

- Acceptar-lo des del mateix client, quan avisa que no és de confiança. Cal fer-ho a cada client i a cada equip.
- **Afegir-lo als certificats de confiança del sistema**. Així el reconeixen tots els programes de l'equip. És el que farem.

**1. Obtenir el certificat des del client**, a partir de la connexió al servidor:

```bash
openssl s_client -connect xmpp.educem.cat:5222 -starttls xmpp -xmpphost educem.cat </dev/null 2>/dev/null \
    | openssl x509 -out educem.cat.crt

openssl x509 -in educem.cat.crt -noout -subject -fingerprint -sha256
```

**2. Comprovar que és el certificat del servidor.** Algú podria fer-se passar pel servidor. Per això, abans de confiar en un certificat, cal comparar-ne l'empremta (*fingerprint*) amb la del servidor original. **Al servidor 0375:**

```bash
sudo openssl x509 -in /etc/prosody/certs/educem.cat.crt -noout -fingerprint -sha256
```

Les dues empremtes han de ser idèntiques.

**3. Afegir-lo als certificats de confiança del client:**

```bash
# El fitxer ha d'acabar en .crt
sudo cp educem.cat.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates            # 1 added

# Ara la verificació és correcta
openssl s_client -connect xmpp.educem.cat:5222 -starttls xmpp -xmpphost educem.cat </dev/null 2>/dev/null \
    | grep 'Verify return code'
# Verify return code: 0 (ok)
```

Amb un certificat de Let's Encrypt aquest pas no cal, perquè els sistemes ja confien en aquesta CA.

### Gajim (Client Gràfic)

**Instal·lació (al client):**

```bash
sudo apt install gajim
gajim &
```

**Afegir el compte.** El primer cop que s'obre, Gajim mostra l'assistent de comptes. Trieu **Log In** i escriviu:

| Camp              | Valor                           |
| ----------------- | ------------------------------- |
| **XMPP Address**  | `usuari1@educem.cat`            |
| **Password**      | contrasenya del compte de Prosody (apartat 03) |

No cal indicar el servidor: Gajim el troba amb el registre SRV `_xmpp-client._tcp.educem.cat`. Si no hi hagués registre SRV, caldria indicar el servidor (`xmpp.educem.cat`) i el port (5222) a la configuració avançada de la connexió.

**Afegir un contacte.** Afegiu `usuari2@educem.cat`. Gajim li envia una petició de subscripció de presència (apartat 02), que `usuari2` ha d'acceptar des del seu client.

**Estat.** Al selector d'estat, canvieu entre **Available**, **Away** i **Busy** (no molestar), i comproveu que l'altre usuari veu el canvi.

### Profanity (Client de Text)

Profanity funciona dins del terminal. És útil als servidors sense entorn gràfic i per entendre què fa el protocol.

```bash
sudo apt install profanity
profanity
```

A Profanity, les ordres comencen amb `/`. La sessió de `usuari2`:

```
/connect usuari2@educem.cat
/sub show
/sub allow usuari1@educem.cat
/roster add usuari1@educem.cat Usuari1
/sub request usuari1@educem.cat
/msg usuari1@educem.cat Hola des de Profanity!
/status set away "Dinant"
/quit
```

| Ordre                                   | Què fa                                                       |
| --------------------------------------- | ------------------------------------------------------------ |
| `/connect usuari2@educem.cat`           | Inicia la sessió (demana la contrasenya)                     |
| `/sub show`                             | Mostra les peticions de subscripció rebudes                  |
| `/sub allow usuari1@educem.cat`         | Accepta que `usuari1` vegi la nostra presència               |
| `/roster add usuari1@educem.cat Usuari1`| Afegeix el contacte a la llista, amb un àlies                |
| `/sub request usuari1@educem.cat`       | Demana veure la presència de `usuari1`                       |
| `/msg usuari1@educem.cat Text`          | Obre una conversa i envia el missatge                        |
| `/status set away "Dinant"`             | Canvia l'estat (`online`, `away`, `dnd`, `xa`, `chat`)       |
| `/wins` i `Alt`+número                  | Llista les finestres i canvia de finestra                    |
| `/close`                                | Tanca la finestra actual                                     |
| `/quit`                                 | Surt de Profanity                                            |

**Certificat no reconegut.** Si no heu fet el pas de "Confiar en el Certificat del Laboratori" en aquest equip, Profanity mostra el certificat i demana què fer: `/tls allow` (només aquesta sessió), `/tls always` (desar-lo) o `/tls deny` (no connectar). Abans d'acceptar-lo, compareu-ne l'empremta amb la del servidor.

### Altres Clients

- **Dino** (`dino-im`): client gràfic modern, amb una interfície semblant a les aplicacions de mòbil. Fa servir els certificats de confiança del sistema.
- **Pidgin** (`pidgin`): client clàssic que també funciona amb altres protocols (IRC, entre d'altres). Demana acceptar el certificat el primer cop.
- **Windows:** Gajim té instal·lador per a Windows. Per confiar en el certificat, cal importar-lo a *Entitats de certificació arrel de confiança* (`certlm.msc`), o acceptar-lo quan Gajim avisi.

### Proves de Funcionament

Feu les proves amb `usuari1` a Gajim (client) i `usuari2` a Profanity:

| Prova                                              | Resultat esperat                                                    |
| -------------------------------------------------- | ------------------------------------------------------------------- |
| `usuari2` canvia l'estat a `away`                  | Gajim mostra `usuari2` com a absent                                 |
| `usuari1` envia un missatge a `usuari2` desconnectat | Quan `usuari2` es connecta, rep el missatge (missatges fora de línia) |
| `usuari1` es connecta també amb Profanity (dos dispositius) | La conversa apareix als dos clients (*Message Carbons*)      |
| Tancar i tornar a obrir Gajim                      | L'historial recent es recupera del servidor (mòdul `mam`)           |

**Al servidor**, l'administrador pot veure les connexions actives i comprovar que van xifrades:

```bash
sudo prosodyctl shell c2s show        # sessions connectades: JID complet amb el recurs i IP
sudo prosodyctl shell c2s show_tls    # versió de TLS i xifratge de cada sessió
```

Si `usuari1` té dues sessions (Gajim i Profanity), apareix dues vegades, amb recursos diferents.

### Resolució de Problemes

| Problema                                              | Causa                                                  | Solució                                                          |
| ----------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------- |
| El client diu que no troba el servidor                | Falta el registre SRV o el client no fa servir el DNS del laboratori | `dig SRV _xmpp-client._tcp.educem.cat` al client      |
| Error de certificat no vàlid                          | El client no confia en el certificat autosignat        | Afegir-lo als certificats de confiança o acceptar-lo al client   |
| Error de certificat: el nom no coincideix             | El certificat no inclou `educem.cat`                   | Tornar a generar-lo amb el SAN de l'apartat 03                   |
| `Authentication failed` / `not-authorized`            | Contrasenya incorrecta o compte inexistent            | `sudo prosodyctl shell user list educem.cat`; `prosodyctl passwd` |
| No es veu l'estat del contacte                        | La subscripció no s'ha acceptat                       | `/sub show` (Profanity) o acceptar la petició a Gajim            |
| L'historial no es recupera                            | El mòdul `mam` no està actiu                          | Treure el comentari de `"mam";` i reiniciar Prosody (apartat 03) |

### Punt de Control

- [ ] El client confia en el certificat (`Verify return code: 0 (ok)`).
- [ ] `usuari1` es connecta amb Gajim i `usuari2` amb Profanity.
- [ ] Tots dos veuen l'estat de l'altre (subscripció mútua).
- [ ] Un missatge enviat a un usuari desconnectat arriba quan es connecta.
- [ ] `sudo prosodyctl shell c2s show` mostra les sessions actives.
