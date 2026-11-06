## 06. Proves de Fallada, Supervisió i Documentació

Un muntatge d'alta disponibilitat que no es prova no és fiable: quan falla de veritat, es descobreix que la reserva no tenia un certificat, que la sincronització feia setmanes que no funcionava o que ningú no recorda el procediment de tornada. Per això les proves s'han de fer periòdicament i s'ha d'apuntar el resultat.

### Pla de Proves

Feu les proves amb el client mirant contínuament el web i l'FTP (apartats 03 i 05), i ompliu l'última columna:

| # | Prova                               | Com es provoca                                       | Resultat esperat                                                         | Resultat obtingut |
| - | ----------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------ | ----------------- |
| 1 | Cau Apache al 0375                  | `sudo systemctl stop apache2`                        | La IP virtual passa al 0374 en menys de 10 s; el web continua            |                   |
| 2 | Cau vsftpd al 0375                  | `sudo systemctl stop vsftpd`                         | La IP virtual passa al 0374; l'FTP continua (les sessions obertes es tallen) |               |
| 3 | Cau keepalived al 0375              | `sudo systemctl stop keepalived`                     | La IP virtual passa al 0374                                              |                   |
| 4 | El 0375 s'apaga de cop              | VirtualBox: **Close** → **Power off the machine**    | El web i l'FTP passen al 0374; el correu espera a la cua del 0374; el DNS i el DHCP continuen (secundaris dels temes 1 i 2) |   |
| 5 | Tornada al 0375                     | Procediment de l'apartat 05                          | La IP virtual torna al 0375 sense perdre fitxers                          |                   |
| 6 | Cau el 0374                         | `sudo poweroff` al 0374                              | Els serveis continuen al 0375, però ja no hi ha reserva, i el client es queda sense Internet (el 0374 és la seva porta d'enllaç) |  |
| 7 | Es bloquegen els anuncis VRRP       | Vegeu "Cervell Dividit"                              | Els dos servidors tenen la IP virtual (*split-brain*)                     |                   |

La prova 6 mostra un altre **punt únic de fallada**: la porta d'enllaç del client. Es podria fer redundant amb una altra IP virtual de VRRP, l'ús per al qual es va crear el protocol, i amb el NAT configurat també al 0375.

### Mesurar el Temps sense Servei

Al client, aquest bucle apunta cada segon si el web respon, i ho desa en un fitxer:

```bash
while true; do
    if curl -k -s --max-time 1 -o /dev/null https://web1.educem.cat/servidor.txt; then
        estat=OK
    else
        estat=ERROR
    fi
    echo "$(date +%T) $estat"
    sleep 1
done | tee prova-ha.log
```

Atureu el bucle amb `Ctrl+C` després de la prova. Cada `ERROR` és, aproximadament, un segon sense servei:

```bash
grep -c ERROR prova-ha.log
```

Compareu el resultat de la prova 1 amb el de la 4. Quan Apache falla, keepalived ho detecta amb la comprovació de salut. Quan el servidor s'apaga, la reserva ho detecta perquè deixa de rebre anuncis.

### Cervell Dividit (*Split-Brain*)

Si els servidors deixen de rebre els anuncis de l'altre, tots dos es creuen principals i activen la IP virtual. Els clients reben respostes d'un o de l'altre, segons quin hagi contestat l'última petició ARP. Si a més els usuaris escriuen dades, cada servidor en guarda una part.

**Provocar-lo** (al 0374): bloquejar els anuncis del 0375, amb una regla que va davant de la que els permet:

```bash
sudo ufw insert 1 deny from 192.168.1.2 to any proto vrrp
```

```bash
# Als dos servidors: tots dos tenen la IP virtual
ip -br addr show enp0s8
sudo journalctl -u keepalived -n 5 --no-pager      # Entering MASTER STATE als dos
```

Al client, `ip neigh show 192.168.1.10` pot canviar d'adreça MAC entre una consulta i una altra.

**Resoldre'l** (al 0374): esborreu la regla. El 0374 torna a rebre els anuncis del 0375, que té més prioritat, i deixa la IP virtual:

```bash
sudo ufw delete deny from 192.168.1.2 to any proto vrrp
```

**Com s'evita en entorns reals:**

- Diversos camins de comunicació entre els servidors, com una segona xarxa dedicada als anuncis.
- Supervisió que avisi quan la IP virtual és a dos llocs.
- Als clústers, mecanismes de **tancament** (*fencing* o STONITH, *Shoot The Other Node In The Head*): abans de prendre el control, un servidor desconnecta o apaga l'altre. Es treballen al mòdul 0378.

### Avisos de Canvi de Servidor

keepalived pot executar un script cada vegada que canvia l'estat d'un servidor. Als **dos servidors**:

```bash
sudo apt install mailutils           # l'ordre mail (al 0375 ja hi és, del Tema 5)
sudo nano /usr/local/bin/avisa-ha.sh
```

```bash
#!/bin/bash
# avisa-ha.sh - keepalived l'executa cada vegada que canvia l'estat de la instància.
# Arguments que afegeix keepalived: $1 = INSTANCE, $2 = nom de la instància,
#                                   $3 = estat nou (MASTER, BACKUP, FAULT o STOP), $4 = prioritat

missatge="$(hostname): la instància $2 passa a l'estat $3 (prioritat $4)"
logger -t keepalived-avis "$missatge"
echo "$missatge - $(date)" | mail -s "[HA] $(hostname): $3" usuari1@educem.cat
```

```bash
sudo chmod 755 /usr/local/bin/avisa-ha.sh
```

Afegiu-lo a `vrrp_instance` de `/etc/keepalived/keepalived.conf`, als dos servidors:

```
vrrp_instance SERVEIS {
    # ...
    notify /usr/local/bin/avisa-ha.sh
}
```

```bash
sudo keepalived -t
sudo systemctl reload keepalived
```

Després d'una prova de fallada, l'avís és al registre del sistema (`grep keepalived-avis /var/log/syslog`) i a la bústia de `usuari1`.

`Nota:` Si el servidor de correu principal és el que ha fallat, el correu de l'avís espera a la cua del 0374 fins que es recupera. Per això, els sistemes de supervisió reals envien els avisos per un altre camí, com la missatgeria del Tema 6 o els bots de Telegram del projecte de monitoratge del mòdul 0374.

### Configuració Final de keepalived

Després dels apartats 02, 03, 05 i 06, el fitxer del 0375 queda així:

```
# /etc/keepalived/keepalived.conf - servidor 0375 (principal)

global_defs {
    router_id srv0375
    enable_script_security
}

# Comprovacions de salut (apartats 03 i 05)
vrrp_script chk_apache {
    script "/usr/bin/systemctl is-active --quiet apache2"
    interval 2
    fall 2
    rise 2
    weight -60
}

vrrp_script chk_vsftpd {
    script "/usr/bin/systemctl is-active --quiet vsftpd"
    interval 2
    fall 2
    rise 2
    weight -60
}

vrrp_instance SERVEIS {
    state BACKUP              # apartat 05: nopreempt necessita l'estat inicial BACKUP
    nopreempt
    interface enp0s8
    virtual_router_id 51
    priority 150
    advert_int 1

    unicast_src_ip 192.168.1.2
    unicast_peer {
        192.168.1.1
    }

    virtual_ipaddress {
        192.168.1.10/24 dev enp0s8 label enp0s8:vip
    }

    track_script {
        chk_apache
        chk_vsftpd
    }

    notify /usr/local/bin/avisa-ha.sh
}
```

Al 0374 és igual, excepte `router_id srv0374`, `priority 100`, `unicast_src_ip 192.168.1.1` i `unicast_peer` amb `192.168.1.2`.

### Manteniment sense Aturar el Servei

L'alta disponibilitat també permet fer el manteniment sense aturar el servei: actualitzar el sistema, reiniciar o canviar maquinari.

**Actualitzar el 0375:**

1. Al 0375: `sudo systemctl stop keepalived`. La IP virtual passa al 0374.
2. Al 0375: `sudo apt update && sudo apt upgrade` i, si cal, `sudo reboot`.
3. Comproveu que els serveis del 0375 funcionen (`systemctl is-active apache2 vsftpd`).
4. Procediment de tornada (apartat 05): `sudo recupera.sh` al 0375 i `sudo systemctl restart keepalived` al 0374.

**Actualitzar el 0374:** com que és la reserva, es pot fer directament. Mentre es reinicia, però, no hi ha reserva, i el client es queda sense Internet.

### Documentació del Servei

La documentació ha de permetre que una altra persona entengui el muntatge i actuï en una caiguda, encara que no l'hagi instal·lat:

| Apartat                         | Contingut                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------- |
| **Arquitectura**                | Esquema amb els dos servidors, la IP virtual, les prioritats i els serveis de cadascun |
| **Serveis redundants i no redundants** | Web, FTP i recepció de correu; i els que només són al 0375              |
| **Fitxers de configuració**     | `keepalived.conf` (als dos), `sincronitza.llista`, `sincronitza.sh`, `recupera.sh`, `avisa-ha.sh`, `/etc/cron.d/sincronitza`, excepció de l'SSH, Postfix del 0374 |
| **Comprovacions de l'estat**    | On és la IP virtual, registres de keepalived, cua de correu del 0374            |
| **Procediments**                | Tornada al 0375, manteniment de cada servidor, i què cal fer als dos servidors per afegir un usuari, una web o una adreça de correu |
| **Proves periòdiques**          | El pla de proves, amb la data i el resultat de cada execució                    |
| **Avisos**                      | Qui els rep i per quin camí                                                     |

**Coses que s'han de fer sempre als dos servidors** (i que s'obliden fàcilment):

- Crear o esborrar usuaris d'FTP (o fer servir LDAP).
- Afegir adreces de correu a `/etc/postfix/relay_recipients` del 0374.
- Canviar la configuració d'Apache o de vsftpd.
- Renovar els certificats.

### Llista de Comprovació Final

- [ ] En funcionament normal, la IP virtual és al 0375 i keepalived està actiu als dos servidors.
- [ ] El tallafoc dels dos servidors permet VRRP de l'altre servidor.
- [ ] keepalived comprova Apache i vsftpd als dos servidors.
- [ ] La sincronització funciona cada minut, i només des del servidor actiu.
- [ ] Els dos servidors tenen els mateixos certificats, configuracions i usuaris.
- [ ] L'MX secundari té la llista d'adreces vàlides actualitzada.
- [ ] La tornada al 0375 es fa a mà amb `recupera.sh` (`nopreempt`).
- [ ] Els avisos de canvi d'estat arriben.
- [ ] S'ha executat el pla de proves i s'han apuntat els resultats.
- [ ] Les còpies de seguretat continuen funcionant: l'alta disponibilitat no les substitueix.
