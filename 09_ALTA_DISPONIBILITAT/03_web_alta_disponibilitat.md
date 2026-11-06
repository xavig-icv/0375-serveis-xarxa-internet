## 03. Web en Alta Disponibilitat

Perquè el 0374 pugui servir les webs quan el 0375 falli, ha de tenir quatre coses iguals que el 0375:

1. El **programari**: Apache amb els mateixos mòduls.
2. La **configuració**: els llocs virtuals `web1` i `web2` i els seus certificats.
3. El **contingut**: les pàgines, sincronitzades cada minut.
4. Una **comprovació de salut** a keepalived, perquè la IP virtual canviï de servidor si Apache falla.

### 1. Canal de Sincronització entre els Servidors

La sincronització la fa `root` del 0375, amb una clau SSH sense contrasenya que només pot fer servir des de la seva adreça.

**Al 0375**, creeu la clau:

```bash
sudo ssh-keygen -t ed25519 -N "" -C "sincronitzacio 0375" -f /root/.ssh/sincronitzacio
sudo cat /root/.ssh/sincronitzacio.pub
```

**Al 0374**, autoritzeu-la per a `root`, amb restriccions:

```bash
sudo mkdir -p -m 700 /root/.ssh
echo 'from="192.168.1.2",restrict CLAU_PUBLICA' | sudo tee -a /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

Substituïu `CLAU_PUBLICA` per la línia sencera del fitxer `.pub` (`ssh-ed25519 AAAA... sincronitzacio 0375`).

- `from="192.168.1.2"`: la clau només funciona des del 0375.
- `restrict`: desactiva el reenviament de ports, l'agent i el terminal interactiu. La clau serveix per executar ordres com `rsync`, però no per obrir una sessió normal.

**Si el 0374 té l'SSH bastionat** (mòdul 0374, Tema 4: `PermitRootLogin no` i `AllowGroups`), afegiu una excepció només per al 0375 **al final** de `/etc/ssh/sshd_config`:

```
# Excepció: el 0375 pot entrar com a root, només amb clau, per sincronitzar els serveis
Match Address 192.168.1.2 User root
    PermitRootLogin prohibit-password
    AllowGroups root
```

```bash
sudo sshd -t
sudo systemctl restart ssh
```

Els blocs `Match` s'apliquen fins al final del fitxer. Per això han d'anar sempre al final.

El bastionat també limita l'SSH del tallafoc del 0374 a la xarxa del centre. Permeteu-lo des del 0375:

```bash
sudo ufw allow from 192.168.1.2 to any port 22 proto tcp
```

**Prova, al 0375.** El primer cop cal acceptar l'empremta de la clau del 0374:

```bash
sudo ssh -i /root/.ssh/sincronitzacio root@192.168.1.1 hostname
```

`Nota:` Aquest canal dona a `root` del 0375 accés total al 0374. Per això es limita a una clau, a una adreça i a un ús concret. En un entorn real, també es pot restringir la clau a determinats directoris (amb `rrsync`, del paquet `rsync`) o gestionar els dos servidors amb una eina d'automatització com Ansible (mòdul 0374).

### 2. Apache al 0374

**Al 0374:**

```bash
sudo apt install apache2
sudo a2enmod ssl
```

Compareu els mòduls actius dels dos servidors (`apache2ctl -M`) i activeu al 0374 els que hi faltin.

**Al 0375**, copieu la configuració i els certificats al 0374 a través del canal SSH. `tar` els empaqueta, `ssh` els transporta xifrats i el `tar` del 0374 els desempaqueta al mateix lloc:

```bash
sudo tar czf - \
    /etc/apache2/sites-available/web1.educem.cat.conf \
    /etc/apache2/sites-available/web2.educem.cat.conf \
    /etc/ssl/certs/web1.educem.cat.crt /etc/ssl/private/web1.educem.cat.key \
    /etc/ssl/certs/web2.educem.cat.crt /etc/ssl/private/web2.educem.cat.key \
  | sudo ssh -i /root/.ssh/sincronitzacio root@192.168.1.1 'tar xzf - -C /'
```

Si els llocs fan servir altres fitxers, com el fitxer de contrasenyes de l'autenticació bàsica (Tema 3, apartat 13), afegiu-los a la llista.

**Els dos servidors han de tenir el mateix certificat.** Si cada servidor en tingués un de diferent, després d'un canvi de servidor el navegador tornaria a avisar que el certificat no és de confiança.

**Al 0374**, activeu els llocs:

```bash
sudo mkdir -p /var/www/web1.educem.cat/html /var/www/web2.educem.cat/html
sudo a2ensite web1.educem.cat.conf web2.educem.cat.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo ufw allow from 192.168.1.0/24 to any port 80,443 proto tcp
```

### 3. Sincronitzar el Contingut

**Fitxer que identifica el servidor.** Per veure en les proves quin servidor respon, cada servidor té un fitxer `servidor.txt` propi, que no se sincronitza:

```bash
# Al 0375
echo "srv0375" | sudo tee /var/www/web1.educem.cat/html/servidor.txt

# Al 0374
echo "srv0374" | sudo tee /var/www/web1.educem.cat/html/servidor.txt
```

**Llista dels directoris que cal sincronitzar** (al 0375):

```bash
sudo nano /etc/sincronitza.llista
```

```
# Directoris que se sincronitzen del 0375 al 0374 (un per línia)
/var/www/web1.educem.cat/html
/var/www/web2.educem.cat/html
```

**Script de sincronització** (al 0375):

```bash
sudo nano /usr/local/bin/sincronitza.sh
```

```bash
#!/bin/bash
# sincronitza.sh - Copia al 0374 els continguts dels serveis redundants (del 0375 al 0374).
# cron l'executa cada minut. Només actua si el 0375 té la IP virtual, és a dir, si és el servidor actiu.

VIP=192.168.1.10
REMOT=root@192.168.1.1
SSH="ssh -i /root/.ssh/sincronitzacio -o BatchMode=yes"

# Si el 0375 no té la IP virtual, el servidor actiu és el 0374: no s'hi ha de copiar res
ip -4 addr show dev enp0s8 | grep -q " $VIP/" || exit 0

for dir in $(grep -v '^#' /etc/sincronitza.llista); do
    rsync -az --delete --exclude servidor.txt -e "$SSH" "$dir/" "$REMOT:$dir/"
done
```

```bash
sudo chmod 755 /usr/local/bin/sincronitza.sh
sudo sincronitza.sh                                    # primera sincronització, a mà

# Comprovar-ho al 0374
sudo ssh -i /root/.ssh/sincronitzacio root@192.168.1.1 ls /var/www/web1.educem.cat/html
```

| Opció de `rsync`            | Funció                                                                        |
| --------------------------- | ----------------------------------------------------------------------------- |
| `-a`                        | Conserva permisos, propietaris i dates                                        |
| `-z`                        | Comprimeix les dades durant la transferència                                  |
| `--delete`                  | Esborra al 0374 el que ja no existeix al 0375: el directori queda idèntic     |
| `--exclude servidor.txt`    | No toca el fitxer que identifica cada servidor                                |
| `-e "ssh ..."`              | Fa servir la clau de sincronització. `BatchMode=yes` fa que falli en lloc de demanar una contrasenya |

**Programar-lo cada minut** amb cron:

```bash
echo '* * * * * root /usr/local/bin/sincronitza.sh' | sudo tee /etc/cron.d/sincronitza
```

- Si el 0375 cau, es poden perdre els canvis de l'últim minut: és el **RPO** del servei.
- Per sincronitzar en pocs segons, hi ha eines que detecten cada canvi, com **lsyncd**.
- `--delete` sincronitza també els esborrats, incloent-hi els errors (apartat 01).

### 4. Comprovació de Salut d'Apache

Si Apache s'atura al 0375 però el servidor continua funcionant, keepalived continua enviant anuncis i no hi ha canvi de servidor. Cal una **comprovació de salut** que baixi la prioritat del 0375 si Apache falla.

Als **dos servidors**, afegiu a `/etc/keepalived/keepalived.conf` el bloc `vrrp_script` abans de `vrrp_instance`, i el bloc `track_script` dins de `vrrp_instance`:

```
vrrp_script chk_apache {
    script "/usr/bin/systemctl is-active --quiet apache2"
    interval 2
    fall 2
    rise 2
    weight -60
}

vrrp_instance SERVEIS {
    # ... la configuració de l'apartat 02 ...

    track_script {
        chk_apache
    }
}
```

| Directiva       | Funció                                                                              |
| --------------- | ----------------------------------------------------------------------------------- |
| `script`        | Ordre de comprovació: si acaba amb codi 0, el servei funciona                       |
| `interval 2`    | Es comprova cada 2 segons                                                           |
| `fall 2`, `rise 2` | Calen 2 errors seguits per considerar-lo caigut, i 2 èxits per considerar-lo recuperat |
| `weight -60`    | Si falla, la prioritat baixa 60 punts                                               |

**El càlcul:** si Apache falla al 0375, la seva prioritat passa de 150 a 90, que és menys que els 100 del 0374. El 0374 passa a ser el principal. Si Apache falla al 0374, la seva prioritat baixa a 40 i continua de reserva.

```bash
sudo keepalived -t
sudo systemctl reload keepalived
sudo journalctl -u keepalived -n 10 --no-pager      # VRRP_Script(chk_apache) succeeded
```

### 5. Proves

**Al client**, una petició cada segon, que mostra quin servidor respon:

```bash
while true; do
    echo "$(date +%T) $(curl -k -s --max-time 2 https://web1.educem.cat/servidor.txt)"
    sleep 1
done
```

| Prova                                                     | Resultat esperat                                                       |
| --------------------------------------------------------- | ---------------------------------------------------------------------- |
| Al 0375: `sudo systemctl stop apache2`                    | Al cap d'uns segons, el client rep `srv0374`                           |
| Al 0375: `sudo systemctl start apache2`                   | El 0375 recupera la IP virtual i el client torna a rebre `srv0375`     |
| Al 0375: crear una pàgina a `/var/www/web1.educem.cat/html/` | En menys d'un minut també és al 0374                                |
| Al 0375: esborrar la pàgina                               | En menys d'un minut també s'esborra al 0374: la replicació no és una còpia de seguretat |
| Apagar el 0375                                            | El 0374 serveix les webs                                               |

Al registre de keepalived del 0375 es veu tot el procés: `Script chk_apache now returning 3`, `Changing effective priority from 150 to 90` i `Entering BACKUP STATE`.

### Actiu-Actiu: Balanceig de Càrrega

En el nostre muntatge, el 0374 només treballa quan falla el 0375. Les webs amb molt de trànsit fan servir un **balancejador de càrrega** (HAProxy, Nginx), que reparteix les peticions entre diversos servidors que treballen alhora i deixa de fer servir els que no responen. Perquè el balancejador no sigui un punt únic de fallada, es duplica amb keepalived i una IP virtual, com hem fet aquí. Es treballa al mòdul 0378.

### Resolució de Problemes

| Problema                                                    | Causa                                                       | Solució                                                          |
| ----------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| `Host key verification failed` a la sincronització          | `root` del 0375 encara no coneix la clau del 0374           | Fer la primera connexió a mà i acceptar l'empremta               |
| `Permission denied (publickey)`                             | Línia d'`authorized_keys` incorrecta, permisos o SSH bastionat | Revisar la línia i els permisos (600); bloc `Match` al final de `sshd_config` |
| El contingut no se sincronitza                              | El 0375 no té la IP virtual, o cron no executa l'script     | `ip -br addr show enp0s8`; `grep sincronitza /var/log/syslog`    |
| Apache no arrenca al 0374                                   | Falta un mòdul o un certificat                              | `sudo apache2ctl configtest`                                     |
| En aturar Apache al 0375, no canvia de servidor             | Falta `track_script`, o el pes no és prou gran              | 150 − 60 ha de ser menys de 100; revisar el registre de keepalived |
| La comprovació falla sempre                                 | L'ordre no funciona amb l'usuari `keepalived_script`        | `sudo -u keepalived_script /usr/bin/systemctl is-active apache2` |
| Després del canvi, el navegador avisa del certificat        | Els dos servidors tenen certificats diferents               | Copiar el mateix certificat i la mateixa clau als dos            |

### Punt de Control

- [ ] `root` del 0375 entra al 0374 amb la clau de sincronització, només des de `192.168.1.2`.
- [ ] El 0374 té Apache amb els llocs `web1` i `web2` i els mateixos certificats.
- [ ] `sincronitza.sh` copia el contingut cada minut, i només quan el 0375 té la IP virtual.
- [ ] Si s'atura Apache al 0375, el client rep les pàgines del 0374 en pocs segons.
- [ ] Heu comprovat que un esborrat també es replica.
