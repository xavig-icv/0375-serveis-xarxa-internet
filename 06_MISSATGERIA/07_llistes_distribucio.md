## 07. Llistes de Distribució

### Què és una Llista de Distribució?

Una **llista de distribució** (*mailing list*) és una adreça de correu que reenvia cada missatge a tots els membres de la llista. Si `usuari1` escriu a `projectes@educem.cat`, el servidor de llistes envia una còpia del missatge a cada subscriptor.

```
                                                          ┌──► usuari2@educem.cat
 usuari1@educem.cat ──► projectes@educem.cat ──► Mailman ─┼──► usuari3@educem.cat
                         (adreça de la llista)            └──► maria@altra.test
```

El programari de llistes s'encarrega de tot el que una simple adreça de reenviament no fa:

- **Subscripcions i baixes:** els usuaris s'hi poden apuntar i desapuntar sense l'administrador.
- **Control de qui hi pot escriure:** tothom, només els membres o només els moderadors.
- **Moderació:** un moderador revisa els missatges abans d'enviar-los.
- **Arxiu:** guarda els missatges per consultar-los més endavant, normalment des d'una web.
- **Recopilacions (*digest*):** el membre rep un sol correu diari amb tots els missatges, en lloc d'un correu per missatge.
- **Gestió de rebots:** dona de baixa les adreces que ja no existeixen.
- **Capçaleres estàndard** (`List-Id`, `List-Unsubscribe`): els clients de correu les fan servir per filtrar els missatges i mostrar l'opció de donar-se de baixa.

### Àlies de Correu o Llista de Distribució?

Al Tema 5 (apartat 05) vam crear àlies de Postfix com `equip: usuari1, usuari2, usuari3` a `/etc/aliases`. També reenvien un missatge a diverses persones, però són molt més limitats:

| Funció                                  | Àlies (`/etc/aliases`)                | Llista de distribució (Mailman)       |
| --------------------------------------- | ------------------------------------- | ------------------------------------- |
| Reenviar a diversos destinataris        | Sí                                    | Sí                                    |
| Alta i baixa per part dels usuaris      | No: cal editar el fitxer (root)       | Sí: per correu o per web              |
| Decidir qui hi pot escriure             | No: qualsevol hi pot escriure         | Sí: membres, moderadors o tothom      |
| Moderació                               | No                                    | Sí                                    |
| Arxiu dels missatges                    | No                                    | Sí                                    |
| Membres d'altres dominis                | Sí                                    | Sí                                    |
| Ús típic                                | Adreces de rol (`info@`, `suport@`)   | Grups, projectes, butlletins          |

### Tipus de Llistes

| Tipus                           | Qui hi escriu                     | Exemple                                                  |
| ------------------------------- | --------------------------------- | -------------------------------------------------------- |
| **Llista d'anuncis**            | Només els propietaris o moderadors | Butlletí del centre, avisos de manteniment, novetats d'un producte |
| **Llista de debat**             | Els membres                       | Equip de projecte, departament, comunitat d'usuaris      |
| **Llista oberta**               | Qualsevol, encara que no en sigui membre | Contacte amb un grup de treball, llistes de suport públiques |

### Modes d'Accés

**Subscripció: com s'hi entra.** Mailman ofereix quatre polítiques:

| Política (Mailman)          | Com funciona                                                                   |
| --------------------------- | ------------------------------------------------------------------------------ |
| **Open**                    | L'alta és immediata. No es recomana: qualsevol podria apuntar una adreça aliena |
| **Confirm**                 | L'usuari rep un correu i ha de confirmar l'alta. Comprova que l'adreça és seva |
| **Moderate**                | L'alta l'ha d'aprovar un moderador                                             |
| **Confirm, then moderate**  | L'usuari confirma l'adreça i, després, un moderador aprova l'alta              |

**Publicació: qui hi pot escriure.** Per a cada missatge, Mailman decideix què fer segons si el remitent és membre o no:

| Acció                    | Efecte                                                                    |
| ------------------------ | ------------------------------------------------------------------------- |
| **Accept**               | S'envia a la llista sense més comprovacions                               |
| **Default processing**   | Passa les comprovacions habituals (mida, nombre de destinataris...) i s'envia |
| **Hold**                 | Queda retingut fins que un moderador l'aprova o el rebutja                |
| **Reject**               | Es rebutja i s'avisa el remitent                                          |
| **Discard**              | S'elimina sense avisar ningú. Útil per al correu brossa                   |

**Arxiu: qui pot llegir els missatges antics.** L'arxiu pot ser públic, privat (només per als membres) o es pot desactivar.

**Visibilitat de la llista.** La llista pot aparèixer o no a l'índex de llistes del servidor (*advertised*). La llista de membres no l'han de poder veure els usuaris qualssevol: les adreces dels membres són dades personals.

### Rols d'una Llista

| Rol                        | Què pot fer                                                             |
| -------------------------- | ----------------------------------------------------------------------- |
| **Propietari** (*owner*)   | Configura la llista, nomena moderadors i gestiona els membres           |
| **Moderador**              | Aprova o rebutja els missatges retinguts i les sol·licituds d'alta      |
| **Membre**                 | Rep els missatges i, segons la configuració, hi pot escriure            |
| **No membre**              | Segons la configuració, hi pot escriure (els missatges es poden retenir) |
| **Administrador del servidor** | Crea dominis i llistes, i té accés a totes les llistes              |

### Adreces d'una Llista

Cada llista té diverses adreces de correu, que Mailman crea automàticament. Per a la llista `projectes@educem.cat`:

| Adreça                               | Funció                                                         |
| ------------------------------------ | -------------------------------------------------------------- |
| `projectes@educem.cat`               | Enviar un missatge a la llista                                 |
| `projectes-join@educem.cat`          | Demanar l'alta                                                 |
| `projectes-leave@educem.cat`         | Demanar la baixa                                               |
| `projectes-request@educem.cat`       | Enviar ordres per correu (`help` retorna l'ajuda)              |
| `projectes-owner@educem.cat`         | Escriure als propietaris de la llista                          |
| `projectes-bounces@educem.cat`       | Recollir els rebots (adreces que ja no existeixen)             |
| `projectes-confirm@educem.cat`       | Rebre les confirmacions d'alta i de baixa                      |

### Programari de Llistes

| Programari         | Característiques                                                              |
| ------------------ | ----------------------------------------------------------------------------- |
| **GNU Mailman 3**  | El més estès en programari lliure. Web d'administració (Postorius) i arxiu (HyperKitty). **És el que farem servir** |
| **Sympa**          | Molt utilitzat a universitats i administracions; integració amb LDAP          |
| **mlmmj**          | Molt lleuger, sense interfície web                                            |
| **Grups de Google, Microsoft 365** | Llistes integrades en els serveis al núvol                    |
| **Listmonk, Mailchimp** | Butlletins (*newsletters*) i campanyes: només anuncis, amb estadístiques |

### Bones Pràctiques i Legislació

- **Consentiment.** Per enviar butlletins a persones de fora de l'organització cal el seu consentiment. Per això les llistes fan servir la política de doble confirmació (*double opt-in*) per a les altes.
- **Baixa fàcil.** Cada missatge ha de permetre donar-se de baixa (capçalera `List-Unsubscribe` i peu del missatge).
- **Privacitat.** Les adreces dels membres no s'han de publicar.
- **Reputació del servidor.** Una llista que envia a adreces que no existeixen o que fa enviaments massius sense consentiment pot fer que altres servidors rebutgin tot el correu del domini (SPF, DKIM i DMARC, Tema 5, apartat 11).
