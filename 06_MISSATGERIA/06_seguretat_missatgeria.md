## 06. Seguretat i Administració del Servei de Missatgeria

Un servei de missatgeria d'empresa ha de garantir quatre coses:

- Que només hi entren els usuaris autoritzats.
- Que les converses viatgen xifrades.
- Que el servei no és accessible des d'on no cal.
- Que les dades (comptes, contactes i historial) es protegeixen i es guarden només el temps necessari.

### Només Usuaris Autoritzats

**Registre des del client desactivat.** XMPP permet que un usuari es creï el compte des del mateix client (*in-band registration*). Als servidors públics, aquesta opció s'aprofita per crear comptes falsos i enviar correu brossa. En una empresa, els comptes els crea l'administrador:

```lua
-- prosody.cfg.lua, secció global
allow_registration = false
```

És el valor per defecte. Escriure'l a la configuració deixa clar que és una decisió de l'administrador.

**Contrasenyes.** Amb `authentication = "internal_hashed"`, Prosody guarda les contrasenyes en format hash (SCRAM) a `/var/lib/prosody`, no en clar. Per gestionar els comptes:

```bash
sudo prosodyctl passwd usuari2@educem.cat       # canviar la contrasenya
sudo prosodyctl deluser usuari2@educem.cat      # esborrar el compte i les seves dades
sudo prosodyctl shell c2s close usuari2@educem.cat   # tancar les sessions obertes d'un usuari
```

**Integració amb el directori.** En una empresa, el servei de missatgeria es connecta al directori LDAP (com el del mòdul 0374, `dc=educem,dc=cat`) o a l'Active Directory:

- Cada usuari té una sola contrasenya per a tots els serveis.
- Quan un treballador marxa, se'n desactiva el compte en un sol lloc.

Prosody ho fa amb el mòdul `mod_auth_ldap`, del paquet `prosody-modules`. No el configurarem al laboratori.

### Xifratge Obligatori

```lua
-- prosody.cfg.lua, secció global
c2s_require_encryption = true
```

Amb aquesta opció, que també és el valor per defecte, Prosody rebutja els clients que intenten autenticar-se sense TLS. Per comprovar-ho:

```bash
sudo prosodyctl shell c2s show_tls     # versió de TLS i algorisme de xifratge de cada sessió
```

### Servei Intern o Federat

Al laboratori, el servei de missatgeria és **intern**: només el fan servir els usuaris d'`educem.cat`. Si no cal la federació, és millor desactivar-la. Així el servidor no accepta connexions d'altres servidors i el port 5269 queda tancat:

```lua
-- prosody.cfg.lua, secció global (ja existeix, amb les línies comentades)
modules_disabled = {
	"s2s"; -- Handle server-to-server connections
}
```

```bash
sudo prosodyctl check config
sudo systemctl restart prosody
sudo ss -tlnp | grep -E ':(5222|5269) '     # ja només hi ha el 5222
```

Si l'empresa volgués federar el servei amb altres organitzacions, caldria:

- Un domini real i un certificat d'una CA reconeguda (Let's Encrypt).
- El registre SRV `_xmpp-server._tcp` (apartat 02).
- El port 5269 obert al tallafoc.
- `s2s_secure_auth = true`, que ja és a la configuració per defecte. Amb aquesta opció, Prosody només accepta servidors amb un certificat vàlid.

### Límits

| Opció                                 | Valor del laboratori             | Funció                                                       |
| ------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| `limits` (`c2s`, `rate`)              | `"10kb/s"` (configuració per defecte) | Velocitat màxima de cada client: evita que un client saturi el servidor |
| `restrict_room_creation`              | `"local"`                        | Qui pot crear sales (apartat 05)                             |
| `http_file_share_size_limit`          | 10 MB                            | Mida màxima de cada fitxer (apartat 05)                      |
| `http_file_share_daily_quota`         | 100 MB                           | Quota diària de cada usuari (apartat 05)                     |

### Privacitat i Conservació de les Dades

Al servidor hi ha dades personals: qui són els contactes de cada usuari, quan es connecta, l'historial de converses i els fitxers enviats. Segons el RGPD:

- **Cal guardar-les només el temps necessari.** L'historial es conserva una setmana per defecte (`archive_expires_after = "1w"`), i els fitxers també. L'empresa ha de decidir aquest termini i informar-ne els usuaris.
- **Cal protegir-les.** `/var/lib/prosody` només és accessible per a l'usuari `prosody`. Les còpies de seguretat també contenen les converses: cal protegir-les igual.
- **Xifratge extrem a extrem (OMEMO).** Els clients com Gajim o Conversations el poden activar per a cada conversa. Amb OMEMO, el servidor guarda els missatges xifrats i ni tan sols l'administrador els pot llegir. Però el servidor continua sabent qui parla amb qui i quan.

### Tallafoc

Regles del tema per al servidor 0375 (només la xarxa interna):

```bash
sudo ufw allow from 192.168.1.0/24 to any port 5222 proto tcp   # clients XMPP
sudo ufw allow from 192.168.1.0/24 to any port 5281 proto tcp   # compartició de fitxers (HTTPS)
sudo ufw status numbered
```

No s'obren el 5269 (no hi ha federació) ni el 5280 (HTTP sense xifrar; Prosody només l'escolta a `localhost`).

### Registres i Supervisió

```bash
# Registre general i d'errors
sudo tail -f /var/log/prosody/prosody.log
sudo tail /var/log/prosody/prosody.err

# Autenticacions (correctes i fallides)
sudo grep -i auth /var/log/prosody/prosody.log | tail

# Estat del servidor
sudo prosodyctl status
sudo prosodyctl shell server uptime
sudo prosodyctl shell c2s show
```

Per depurar un problema, canvieu `info` per `debug` a la línia `info = "/var/log/prosody/prosody.log"` de la secció `log` i reinicieu Prosody. Després, torneu-lo a deixar com estava: el nivell `debug` genera molts registres i hi pot aparèixer el contingut dels missatges.

### Còpies de Seguretat

Prosody guarda la configuració a `/etc/prosody` i les dades a `/var/lib/prosody`. Per fer-ne una còpia:

```bash
sudo tar czf /root/prosody-$(date +%F).tar.gz /etc/prosody /var/lib/prosody
sudo ls -lh /root/prosody-*.tar.gz
```

Per restaurar-la, atureu Prosody, descomprimiu la còpia a `/` i torneu-lo a engegar.

### Llista de Comprovació de Seguretat

- [ ] `allow_registration = false`: els comptes només els crea l'administrador.
- [ ] `c2s_require_encryption = true` i `c2s show_tls` mostra totes les sessions xifrades.
- [ ] Federació (`s2s`) desactivada: el port 5269 no escolta.
- [ ] Només els usuaris del domini poden crear sales.
- [ ] Tallafoc: només el 5222 i el 5281, i només per a la xarxa interna.
- [ ] La clau privada del certificat només la poden llegir `root` i `prosody`.
- [ ] Termini de conservació de l'historial i dels fitxers decidit i comunicat.
- [ ] Còpia de seguretat de `/etc/prosody` i `/var/lib/prosody`.

### Resolució de Problemes

| Problema                                                    | Causa                                                   | Solució                                                        |
| ----------------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| Prosody no arrenca després de desactivar `s2s`              | Error de sintaxi a `modules_disabled`                   | `sudo prosodyctl check config`                                 |
| El port 5269 continua obert                                 | `s2s` encara és a la llista de mòduls o no s'ha reiniciat | Revisar `modules_disabled` i `sudo systemctl restart prosody` |
| Un client antic no es pot connectar                         | No suporta TLS o només suporta versions antigues        | Actualitzar el client; no desactivar el xifratge                |
| Hi ha molts intents d'autenticació fallits al registre      | Algú prova contrasenyes                                 | Revisar d'on venen, restringir el tallafoc i canviar les contrasenyes febles |
| Després de restaurar una còpia, Prosody no llegeix les dades | Propietari incorrecte dels fitxers                     | `sudo chown -R prosody:prosody /var/lib/prosody`               |
