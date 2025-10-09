## 01. Introducció als Serveis de Resolució de Noms

El DNS (Domain Name System) és un servei essencial pel funcionament d'Internet i de les xarxes actuals. La seva funció principal és la de traduir noms de domini, fàcils de recordar pels humans, com "`www.google.com`", en adreces IP numèriques com "`142.250.200.110`", que són els identificadors que utilitzen els dispositius per comunicar-se entre ells.

El servei DNS és essencial per diverses raons:

- **Dominis i IPs**: El DNS manté actualitzada una base de dades que emmagatzema noms de domini i les seves corresponents adreces IP, facilitant la localització de dispositius a Internet.
  - Les adreces IP són identificadors de dispositius difícil de recordar pels humans (ex: `142.250.200.110`)
  - Noms de domini: S'associa un nom simbòlic a una adreça IP per facilitar l'accés als recursos de xarxa com les pàgines web de: `www.google.com`, gestió de correu: `gmail.com`, emmagatzematge al núvol: `drive.google.com`, sistema de videoconferència: `meet.google.com`, etc.
- **Escalabilitat**: El sistema DNS està dissenyat de manera jeràrquica i distribuïda, permetent així gestionar milions de noms de domini i respondre consultes a nivell mundial.
- **Flexibilitat**: Hi ha la possiblitat de canviar l'adreça IP d'un servidor i mantenir el nom de domini, això permet canviar la infraestructura dels servidors quan sigui necessàri sense afectar als usuaris finals de les aplicacions.

### Exemple pràctic

```
Sense DNS:
usuari → http://142.250.185.78 (difícil de recordar) → connexió exitosa

Amb DNS:
usuari → www.google.com → DNS resol → 142.250.185.78 → connexió exitosa
```
