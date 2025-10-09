## 02. Classificació dels Mecanismes de Resolució

### Resolució Estàtica (`/etc/hosts`)

És un arxiu local que permet associar noms de host amb adreces IP sense necessitat de realitzar una consulta externa.

**Avantatges:**

- Molt ràpid (es consulta en un fitxer del mateix dispositiu)
- Fàcil de configurar i permet fer proves locals
- No requereix connexió a la xarxa local o a Internet
- Disposa de prioritat sobre el servei de DNS

**Inconvenients:**

- No és escalable (manteniment manual)
- Cal actualitzar cada ordinador individualment
- Només disposa d'un efecte local (a l'ordinador on està situat)

**Exemple de `/etc/hosts`:**

```
127.0.0.1       localhost
192.168.1.10    servidor.local servidor
192.168.1.20    web.local
192.168.1.30    mail.local
192.168.1.40    google.com
```

Si fem un `ping google.com` primer es consultaria el fitxer `/etc/hosts` abans que el servei de DNS.

### Resolució Dinàmica (Domain Name System)

**Avantatges:**

- És escalable, pot disposar de milions de registres
- Gestió centralitzada de la informació
- Actualitzacions automàtiques (si hi ha canvis de domini o IP)
- Pot disposar de redundància (diversos servidors en fail over)

**Inconvenients:**

- Depèn de la connexió de xarxa local o a Internet
- És més dificil de configurar
- Si no disposa de redundància pot ser un punt crític si falla

Si fem un `ping google.com`, si no es troba l'entrada a `/etc/hosts` es consulta al servidor DNS que retorna la IP associada a aquest domini, per exemple: `142.250.200.110`.

### Resolució Estàtica vs Dinàmica

**Xarxa petita (menys de 10 equips):**

- Amb un fitxer de hosts pot ser suficient
- És fàcil de mantenir manualment

**Xarxa mitjana (fins a 100 equips):**

- Es recomana configurar un servei de DNS intern
- Facilita l'administració. Permet gestionar de manera senzilla els noms de cada equip

**Xarxa gran (> 100 hosts):**

- El servei de DNS intern és obligatori
- La redundància de, com a mínim, 2 servidors DNS és obligatoria
