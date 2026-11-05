## 09. Creació i Gestió de Llistes

En aquest apartat creem dues llistes amb tipus i modes d'accés diferents (apartat 07):

| Llista                  | Tipus     | Subscripció                | Qui hi escriu                          | Arxiu   |
| ----------------------- | --------- | -------------------------- | -------------------------------------- | ------- |
| `anuncis@educem.cat`    | Anuncis   | **Confirm** (l'usuari confirma) | Només `usuari1`; la resta queda retinguda | Públic  |
| `projectes@educem.cat`  | Debat     | **Confirm, then moderate** | Els membres; els no membres queden retinguts | Privat  |

Els noms de les opcions de la web de Mailman (Postorius) s'escriuen en anglès, tal com apareixen amb la configuració per defecte.

### Primer Accés com a Administrador

1. Obriu `https://llistes.educem.cat` des del client i feu clic a **Log In**.
2. Entreu amb l'usuari `admin` i la contrasenya de la instal·lació.
3. El primer cop, Postorius demana verificar l'adreça de correu del superusuari (**Verify Your E-mail Address**) i hi envia un missatge.
4. Llegiu el correu a la bústia de `usuari1` (Thunderbird, Tema 5) i obriu l'enllaç de confirmació. Al servidor, l'enllaç també es pot trobar així:

```bash
sudo grep -rh "confirm-email" /home/usuari1/Maildir/new/ | tail -1
```

5. Confirmeu l'adreça i torneu a entrar.

Si el correu de verificació no arriba, reviseu `/var/log/mail.log`: el correu local del Tema 5 ha de funcionar.

### Nom del Lloc Web

Mailman fa servir el nom del lloc web als correus i a les pàgines. Per defecte és `example.com`. Canvieu-lo a l'administració de Django, `https://llistes.educem.cat/mailman3/admin/`:

1. Entreu amb l'usuari `admin`.
2. **Sites** → `example.com`.
3. **Domain name:** `llistes.educem.cat`; **Display name:** `Llistes d'educem.cat`.
4. Deseu (**Save**).

### Domini de Correu de les Llistes

A Postorius, **Domains** → **Add domain**:

| Camp            | Valor                    |
| --------------- | ------------------------ |
| **Mail Host**   | `educem.cat`             |
| **Description** | `Llistes d'educem.cat`   |
| **Web Host**    | `llistes.educem.cat`     |

El domini de correu és el que va després de l'`@` a les adreces de les llistes.

### Llista d'Anuncis: `anuncis@educem.cat`

**1. Crear la llista.** **Lists** → **Create New List**:

| Camp                              | Valor                                  |
| --------------------------------- | -------------------------------------- |
| **List Name**                     | `anuncis`                              |
| **Mail Host**                     | `educem.cat`                           |
| **Initial list owner address**    | `usuari1@educem.cat`                   |
| **Advertise this list?**          | Yes                                    |
| **List Style**                    | Announce only mailing list style.      |
| **Description**                   | `Avisos del centre`                    |

**2. Comprovar-la al servidor:**

```bash
sudo -u list mailman lists
sudo postmap -q anuncis@educem.cat hash:/var/lib/mailman3/data/postfix_lmtp     # lmtp:[127.0.0.1]:8024
```

Mailman ha afegit les adreces de la llista (`anuncis`, `anuncis-join`, `anuncis-request`...) al mapa de Postfix.

**3. Que només hi escrigui l'emissor dels anuncis.** A Mailman 3.3, l'estil d'anuncis no impedeix per si sol que els membres hi escriguin. Cal configurar-ho:

- **Settings** → **Message Acceptance** → **Default action to take when a member posts to the list:** **Hold for moderation**. Ara, els missatges dels membres queden retinguts.
- `usuari1` s'ha de subscriure a la llista (vegeu "Membres" més avall). Després, a **Members**, obriu el membre `usuari1@educem.cat` i, a l'opció **Moderation**, trieu **Accept immediately (bypass other rules)**.

Així, `usuari1` envia els anuncis directament, i qualsevol altre missatge espera l'aprovació d'un moderador.

### Llista de Debat: `projectes@educem.cat`

**Lists** → **Create New List**:

| Camp                              | Valor                                                   |
| --------------------------------- | ------------------------------------------------------- |
| **List Name**                     | `projectes`                                             |
| **Mail Host**                     | `educem.cat`                                            |
| **Initial list owner address**    | `usuari1@educem.cat`                                    |
| **Advertise this list?**          | No                                                      |
| **List Style**                    | Discussion mailing list style with private archives.   |
| **Description**                   | `Equip de projectes`                                    |

Aquest estil ja configura el que volem. Comproveu-ho a **Settings**:

| Secció                  | Opció                                                    | Valor                          |
| ----------------------- | -------------------------------------------------------- | ------------------------------ |
| **Member Policy**       | Subscription Policy                                      | Confirm, then moderate         |
| **Message Acceptance**  | Default action to take when a member posts to the list   | Default processing             |
| **Message Acceptance**  | Default action to take when a non-member posts to the list | Hold for moderation          |
| **Archiving**           | Archive policy                                           | Private archives               |

### Membres

Hi ha tres maneres de donar d'alta un membre.

**1. L'usuari s'hi apunta per correu.** `usuari2` envia un missatge qualsevol a `projectes-join@educem.cat` (amb Thunderbird o des del servidor):

```bash
sudo -u usuari2 bash -c 'echo "Alta" | mail -s "Alta" projectes-join@educem.cat'
```

- `usuari2` rep un correu de `projectes-confirm+...@educem.cat` que demana confirmar l'alta. Hi ha de respondre sense canviar l'assumpte.
- Com que la política és **Confirm, then moderate**, després l'alta l'ha d'aprovar un moderador. A Postorius, la llista té pendent una petició a **Subscription requests**: accepteu-la.

**2. L'usuari s'hi apunta per la web.** Si `usuari2` es crea un compte a Postorius (**Sign Up**) amb la seva adreça, pot subscriure's a les llistes publicades i gestionar les seves subscripcions.

**3. El propietari dona d'alta diversos membres alhora.** **Mass operations** → **Mass subscribe**, una adreça per línia:

| Opció                    | Efecte                                                          |
| ------------------------ | --------------------------------------------------------------- |
| **Pre confirm**          | No cal que l'usuari confirmi l'alta                             |
| **Pre approved**         | No cal l'aprovació del moderador                                |
| **Pre Verified**         | L'adreça es considera verificada                                |
| **Invitation**           | L'usuari rep una invitació i decideix si s'hi apunta            |
| **Send welcome message** | L'usuari rep el missatge de benvinguda                          |

Doneu d'alta `usuari1@educem.cat` a les dues llistes, amb **Pre confirm**, **Pre approved** i **Pre Verified**. Si l'adreça no està verificada i no es marca **Pre Verified**, Mailman envia igualment un correu de confirmació.

**Des de la línia d'ordres** (sense cap confirmació ni moderació):

```bash
echo "usuari2@educem.cat" | sudo -u list mailman addmembers - anuncis@educem.cat
sudo -u list mailman members anuncis@educem.cat
```

### Moderació

1. `usuari2` (membre) escriu a `anuncis@educem.cat`: el missatge queda retingut.
2. Una adreça que no és membre escriu a `projectes@educem.cat`: també queda retingut.
3. Els propietaris reben un avís per correu. A Postorius, **Held messages** mostra els missatges retinguts amb aquestes opcions:
   - **Accept:** s'envia a la llista.
   - **Reject:** es rebutja i s'avisa el remitent.
   - **Discard:** s'elimina sense avís.

Per provar l'enviament d'un no membre, feu servir una adreça que no sigui a la llista, com un àlies del Tema 5 o un compte nou del sistema.

### Arxiu (HyperKitty)

Cada llista té l'arxiu a `https://llistes.educem.cat/mailman3/hyperkitty/`:

- L'arxiu d'`anuncis` és públic: qualsevol el pot llegir.
- El de `projectes` és privat: cal entrar a Postorius amb un compte que sigui membre de la llista.

HyperKitty mostra els missatges agrupats per fils de conversa, i també permet fer-hi cerques. Per això, moltes comunitats el fan servir com a fòrum web de les seves llistes.

### Ordres per Correu

Els usuaris també poden gestionar les subscripcions per correu, sense la web:

| Enviar un missatge a                   | Efecte                                                  |
| -------------------------------------- | ------------------------------------------------------- |
| `projectes-join@educem.cat`            | Demanar l'alta                                          |
| `projectes-leave@educem.cat`           | Demanar la baixa                                        |
| `projectes-request@educem.cat` (cos: `help`) | Rebre l'ajuda de les ordres per correu            |
| `projectes-owner@educem.cat`           | Escriure als propietaris                                |

### Proves de Funcionament

| Prova                                                          | Resultat esperat                                                    |
| -------------------------------------------------------------- | ------------------------------------------------------------------- |
| `usuari1` escriu a `anuncis@educem.cat`                        | Arriba a tots els membres, amb la capçalera `List-Id`               |
| `usuari2` escriu a `anuncis@educem.cat`                        | Queda retingut a **Held messages**                                  |
| `usuari2` demana l'alta a `projectes`                          | Rep el correu de confirmació i, després, la petició espera el moderador |
| Un no membre escriu a `projectes@educem.cat`                   | Queda retingut                                                      |
| Obrir l'arxiu de `projectes` sense entrar a Postorius          | No es pot veure (arxiu privat)                                      |
| `usuari2` envia un missatge a `projectes-leave@educem.cat`     | Rep la confirmació de baixa                                         |

Per seguir el recorregut d'un missatge, mireu tots dos registres:

```bash
sudo tail -f /var/log/mail.log            # Postfix: relay=127.0.0.1[127.0.0.1]:8024 (lliurament a Mailman)
sudo tail -f /var/log/mailman3/smtp.log   # Mailman: còpies enviades als membres
```

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                                          |
| -------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| Postfix rebutja la llista amb `User unknown in local recipient table` | Falta el mapa de Mailman a `local_recipient_maps` | Revisar la integració amb Postfix (apartat 08)                  |
| El missatge es lliura a una bústia i no a la llista            | Falta `transport_maps`, o la llista té el mateix nom que un àlies | `sudo postmap -q llista@educem.cat hash:/var/lib/mailman3/data/postfix_lmtp`; no feu servir noms d'àlies existents |
| `postmap -q` no troba la llista                                | Mapes no regenerats                                      | `sudo -u list mailman aliases`                                   |
| El missatge d'un membre queda retingut                         | La política de la llista o del membre és **Hold**        | Revisar **Message Acceptance** i l'opció **Moderation** del membre |
| No arriba el correu de confirmació                             | El correu local no funciona                              | `/var/log/mail.log` i proves del Tema 5                          |
| Els missatges no apareixen a l'arxiu                           | La clau de l'arxiu no coincideix                         | `api_key` de `/etc/mailman3/mailman-hyperkitty.cfg` ha de ser igual que `MAILMAN_ARCHIVER_KEY` de `/etc/mailman3/mailman-web.py`; després, `sudo systemctl restart mailman3` |
| Els correus de Postorius diuen `example.com`                   | No s'ha canviat el nom del lloc web                      | Administració de Django → **Sites**                              |

### Punt de Control

- [ ] `sudo -u list mailman lists` mostra `anuncis@educem.cat` i `projectes@educem.cat`.
- [ ] Un missatge de `usuari1` a `anuncis` arriba a tots els membres.
- [ ] Un missatge de `usuari2` a `anuncis` queda retingut fins que s'aprova.
- [ ] L'alta a `projectes` demana confirmació de l'usuari i aprovació del moderador.
- [ ] L'arxiu de `projectes` només és visible per als membres.
