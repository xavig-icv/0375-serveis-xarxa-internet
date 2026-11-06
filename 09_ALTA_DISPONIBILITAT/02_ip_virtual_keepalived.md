## 02. IP Virtual amb keepalived (VRRP)

### Què és una IP Virtual?

Una **IP virtual** (VIP) és una adreça que no pertany a cap servidor concret: la té el servidor actiu en cada moment. Els clients sempre fan servir aquesta adreça (`192.168.1.10`) i no saben quin servidor els respon.

El protocol que decideix qui té la IP virtual és **VRRP** (*Virtual Router Redundancy Protocol*, RFC 5798). Es va crear perquè dos encaminadors poguessin compartir la porta d'enllaç d'una xarxa, però funciona igual per a qualsevol servei:

1. Els servidors del grup tenen una **prioritat**. El de prioritat més alta és el **principal** (*MASTER*): té la IP virtual i envia un **anunci** cada segon per dir que funciona.
2. Els altres són de **reserva** (*BACKUP*): escolten els anuncis.
3. Si la reserva deixa de rebre anuncis durant uns 3 segons, es converteix en principal: afegeix la IP virtual a la seva interfície i envia un **ARP gratuït**. Aquest missatge anuncia a la xarxa que la IP virtual ara té l'adreça MAC del servidor nou, i els clients i els commutadors actualitzen la seva taula.

```
 t = 0 s    0375 (MASTER, té la VIP) ── anunci cada segon ──► 0374 (BACKUP)
 t = 10 s   el 0375 cau i deixa d'enviar anuncis
 t ≈ 13 s   el 0374 passa a MASTER: afegeix 192.168.1.10 i envia un ARP gratuït
            els clients continuen fent servir 192.168.1.10, ara cap al 0374
```

**keepalived** és el programa que implementa VRRP a Linux. A més, executa **comprovacions de salut**: si un servei falla, baixa la prioritat del servidor perquè l'altre passi a ser el principal (apartats 03 i 05).

### Instal·lació

Als **dos servidors**:

```bash
sudo apt update
sudo apt install keepalived

# Usuari sense privilegis que executa les comprovacions de salut
sudo useradd --system --no-create-home --shell /usr/sbin/nologin keepalived_script
```

El servei no arrenca fins que no existeix el fitxer `/etc/keepalived/keepalived.conf`.

### Configuració del 0375 (Principal)

```bash
sudo nano /etc/keepalived/keepalived.conf
```

```
# /etc/keepalived/keepalived.conf - servidor 0375 (principal)

global_defs {
    router_id srv0375
    enable_script_security
}

vrrp_instance SERVEIS {
    state MASTER
    interface enp0s8
    virtual_router_id 51
    priority 150
    advert_int 1

    # Anuncis directes a l'altre servidor (unicast), en lloc de multidifusió
    unicast_src_ip 192.168.1.2
    unicast_peer {
        192.168.1.1
    }

    virtual_ipaddress {
        192.168.1.10/24 dev enp0s8 label enp0s8:vip
    }
}
```

| Directiva                    | Significat                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------- |
| `router_id`                  | Nom del servidor als registres                                                  |
| `enable_script_security`     | Les comprovacions de salut s'executen amb l'usuari `keepalived_script`, no amb root |
| `vrrp_instance SERVEIS`      | Un grup VRRP. El nom és lliure                                                  |
| `state`                      | Estat inicial: `MASTER` al principal i `BACKUP` a la reserva                    |
| `interface`                  | Interfície per on s'envien els anuncis: la de la xarxa interna                  |
| `virtual_router_id`          | Identificador del grup (de 1 a 255). Ha de ser igual als dos servidors i diferent de qualsevol altre grup de la mateixa xarxa |
| `priority`                   | Prioritat (de 1 a 254): la més alta és la del principal                         |
| `advert_int`                 | Segons entre anuncis                                                            |
| `unicast_src_ip`, `unicast_peer` | Adreça pròpia i adreça de l'altre servidor. Amb unicast, els anuncis no depenen de la multidifusió de la xarxa |
| `virtual_ipaddress`          | La IP virtual, amb la interfície i una etiqueta (`enp0s8:vip`) per reconèixer-la |

### Configuració del 0374 (Reserva)

```bash
sudo nano /etc/keepalived/keepalived.conf
```

```
# /etc/keepalived/keepalived.conf - servidor 0374 (reserva)

global_defs {
    router_id srv0374
    enable_script_security
}

vrrp_instance SERVEIS {
    state BACKUP
    interface enp0s8
    virtual_router_id 51
    priority 100
    advert_int 1

    unicast_src_ip 192.168.1.1
    unicast_peer {
        192.168.1.2
    }

    virtual_ipaddress {
        192.168.1.10/24 dev enp0s8 label enp0s8:vip
    }
}
```

Només canvien `router_id`, `state`, `priority` i les adreces de l'unicast.

### Tallafoc

VRRP no és TCP ni UDP: és un protocol IP propi (número 112). Si el tallafoc està actiu, cal deixar passar els anuncis de l'altre servidor:

```bash
# Al 0375
sudo ufw allow from 192.168.1.1 to any proto vrrp

# Al 0374
sudo ufw allow from 192.168.1.2 to any proto vrrp
```

Sense aquesta regla, cap servidor no rep els anuncis de l'altre, i tots dos es creuen principals (*split-brain*, apartat 06).

### Engegar i Comprovar

Als dos servidors, començant pel 0375:

```bash
sudo keepalived -t                    # comprova la sintaxi del fitxer
sudo systemctl enable keepalived
sudo systemctl restart keepalived
sudo journalctl -u keepalived -n 15 --no-pager
```

Al registre del 0375 ha d'aparèixer `(SERVEIS) Entering MASTER STATE`, i al del 0374, `(SERVEIS) Entering BACKUP STATE`.

```bash
# On és la IP virtual?
ip -br addr show enp0s8
# 0375: enp0s8  UP  192.168.1.2/24 192.168.1.10/24 ...
# 0374: enp0s8  UP  192.168.1.1/24 ...

# Al 0374: els anuncis del 0375 (un per segon, prioritat 150)
sudo tcpdump -ni enp0s8 vrrp -c 5
```

**Des del client:**

```bash
ping -c 3 192.168.1.10
ip neigh show 192.168.1.10          # l'adreça MAC és la de l'enp0s8 del 0375
```

### Primera Prova de Commutació

1. Al client, feu un `ping` continu a la IP virtual, amb l'hora de cada resposta:

```bash
ping -D 192.168.1.10
```

2. Al 0375, atureu keepalived (simula que el servidor falla):

```bash
sudo systemctl stop keepalived
```

3. Al cap d'uns segons, el 0374 té la IP virtual (`ip -br addr show enp0s8`) i el `ping` continua. Potser es perden un o dos paquets. Al client, `ip neigh show 192.168.1.10` mostra ara l'adreça MAC del 0374.
4. Torneu a engegar keepalived al 0375. Com que té més prioritat, recupera la IP virtual (**preempció**).

Proveu també d'apagar del tot el 0375 (`sudo poweroff`). El resultat ha de ser el mateix.

**La preempció.** Per defecte, quan el principal es recupera, torna a agafar la IP virtual. Per al web és el que volem. En canvi, per a un servei on els usuaris escriuen dades, com l'FTP, pot ser un problema (apartat 05).

### Resolució de Problemes

| Problema                                                   | Causa                                                        | Solució                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| keepalived no arrenca                                      | Error al fitxer de configuració                              | `sudo keepalived -t` i `journalctl -u keepalived`                |
| Avís `default user 'keepalived_script' ... does not exist` | No s'ha creat l'usuari per a les comprovacions               | Crear-lo amb `useradd` (apartat d'instal·lació)                  |
| Els dos servidors tenen la IP virtual                      | No reben els anuncis de l'altre (tallafoc o `virtual_router_id` diferent) | Regla `proto vrrp`; `tcpdump -ni enp0s8 vrrp`        |
| La IP virtual no apareix enlloc                            | Nom de la interfície incorrecte                              | `ip -br link` i corregir `interface` i `dev`                     |
| El client continua enviant a l'antic servidor              | El client no ha rebut l'ARP gratuït                          | Esperar uns segons o buidar la taula: `sudo ip neigh flush all`  |
| El canvi tarda massa                                       | `advert_int` alt                                             | Amb `advert_int 1`, el canvi tarda uns 3 segons                  |

### Punt de Control

- [ ] keepalived està actiu als dos servidors, amb els estats MASTER (0375) i BACKUP (0374).
- [ ] La IP virtual `192.168.1.10` és a l'`enp0s8` del 0375 i respon al `ping` del client.
- [ ] `tcpdump` al 0374 mostra els anuncis VRRP del 0375.
- [ ] En aturar keepalived al 0375, la IP virtual passa al 0374 en pocs segons; en tornar-lo a engegar, torna al 0375.
