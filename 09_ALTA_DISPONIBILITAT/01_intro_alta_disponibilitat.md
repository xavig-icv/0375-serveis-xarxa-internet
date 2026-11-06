## 01. Introducció a l'Alta Disponibilitat

### Què és l'Alta Disponibilitat?

La **disponibilitat** d'un servei és el percentatge de temps que funciona. Sovint s'expressa en "nous":

| Disponibilitat | Temps aturat en un any | Temps aturat en un mes |
| -------------- | ---------------------- | ---------------------- |
| 99 %           | 3,65 dies (87,6 h)     | 7,2 h                  |
| 99,9 %         | 8,76 h                 | 43,2 min               |
| 99,99 %        | 52,6 min               | 4,3 min                |
| 99,999 %       | 5,3 min                | 26 s                   |

Un sol servidor ben administrat pot arribar al 99,9 %, però cada actualització del sistema que demana reiniciar i cada avaria aturen el servei. L'**alta disponibilitat** (HA, *High Availability*) consisteix a dissenyar el servei perquè continuï funcionant encara que falli un component:

1. **Redundància:** els components importants estan duplicats.
2. **Detecció:** es comprova contínuament que els components funcionen.
3. **Commutació automàtica:** quan un component falla, el de reserva n'ocupa el lloc sense intervenció humana.

### Conceptes Clau

| Concepte                               | Significat                                                           | Al laboratori                                          |
| -------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------ |
| **Punt únic de fallada** (SPOF)        | Component que, si falla, atura el servei                             | El 0375: si cau, no hi ha web, FTP ni correu           |
| **Redundància**                        | Duplicar els components                                              | Dos servidors amb els mateixos serveis                 |
| **Commutació per error** (*failover*)  | Pas automàtic al component de reserva                                | La IP virtual passa del 0375 al 0374                   |
| **Retorn** (*failback*)                | Tornada al component principal quan es recupera                      | La IP virtual torna al 0375                            |
| **Actiu-passiu**                       | Un servidor treballa i l'altre espera                                | El nostre muntatge                                     |
| **Actiu-actiu**                        | Els dos servidors treballen alhora i es reparteixen la feina         | Un balancejador de càrrega (no el farem)               |
| **Comprovació de salut** (*health check*) | Prova periòdica que el servei respon                              | `systemctl is-active apache2`, cada 2 segons           |
| **Cervell dividit** (*split-brain*)    | Els dos servidors es creuen principals alhora                        | Si un tallafoc bloqueja els missatges entre ells (apartat 06) |
| **RTO** (*Recovery Time Objective*)    | Temps màxim que el servei pot estar aturat                           | Uns segons: el temps que tarda a moure's la IP virtual |
| **RPO** (*Recovery Point Objective*)   | Dades que es poden perdre com a màxim                                | Fins a 1 minut: els continguts se sincronitzen cada minut |

### L'Alta Disponibilitat No és una Còpia de Seguretat

La redundància protegeix de la **fallada d'un equip o d'un servei**, però no protegeix dels incidents més habituals:

- Si algú esborra una pàgina web per error, la sincronització l'esborra també al servidor de reserva al cap d'un minut.
- Si un virus o un *ransomware* xifra els fitxers, se sincronitzen els fitxers xifrats.
- Si un canvi de configuració trenca el servei, el trenca als dos servidors.

Per això, **replicar no és fer una còpia de seguretat**: calen totes dues coses. En una empresa, abans de pensar en l'alta disponibilitat cal tenir còpies de seguretat que s'hagin provat de restaurar, supervisió amb alertes, les actualitzacions al dia i el servei documentat. A més, un muntatge d'alta disponibilitat mal fet també pot provocar caigudes. Per exemple, dos servidors que es creuen principals alhora, o configuracions diferents a cada servidor.

### Com es Fa Redundant Cada Servei

Cada servei té el seu mecanisme. Alguns protocols ja el porten incorporat:

| Servei              | Mecanisme                                                          | On es treballa              |
| ------------------- | ------------------------------------------------------------------ | --------------------------- |
| DNS                 | Servidor secundari, i el client té configurats tots dos servidors  | Tema 1, apartat 11          |
| DHCP                | Commutació per error (*failover*) entre dos servidors              | Tema 2, apartat 06          |
| Directori LDAP      | Replicació entre servidors                                         | Mòdul 0374, Tema 1          |
| Web                 | IP virtual i contingut sincronitzat                                | Apartats 02 i 03            |
| Correu (recepció)   | Servidor MX secundari. A més, l'SMTP ja reintenta els enviaments durant dies | Apartat 04        |
| FTP                 | IP virtual i contingut sincronitzat                                | Apartat 05                  |
| Màquines senceres   | Clúster de virtualització (Proxmox, VMware): si cau un equip físic, les màquines virtuals s'engeguen en un altre | Mòdul 0378 |

Els serveis **sense estat**, com el web, són els més fàcils: cada petició HTTP és independent, i després d'un canvi de servidor el navegador continua com si res. Els serveis **amb estat** són més difícils. Una sessió FTP oberta es talla, i les bústies de correu o els fitxers pujats només són al servidor on s'han escrit.

### L'Escenari d'Aquest Bloc

```
                              CLIENT (192.168.1.50)
                                       │  web1.educem.cat → 192.168.1.10
                                       ▼
                        IP VIRTUAL 192.168.1.10 (la té el servidor actiu)
                  ┌────────────────────┴────────────────────┐
                  │                                         │
       SERVIDOR 0375 (192.168.1.2)               SERVIDOR 0374 (192.168.1.1)
       Principal: prioritat 150   ◄─── VRRP ───►  Reserva: prioritat 100
       Apache, vsftpd, Postfix                   Apache, vsftpd, Postfix (MX secundari)
                  │                                         ▲
                  └───── rsync cada minut: els continguts ──┘
```

| Element                 | Servidor 0375                         | Servidor 0374                                     |
| ----------------------- | ------------------------------------- | ------------------------------------------------- |
| IP virtual              | La té normalment (principal)          | La pren si el 0375 o els seus serveis fallen      |
| Web (Apache, 80 i 443)  | Llocs `web1` i `web2` (Tema 3)        | Els mateixos llocs, amb el contingut sincronitzat |
| FTP (vsftpd)            | Tema 4                                | La mateixa configuració i els mateixos usuaris    |
| Correu                  | Servidor principal (MX 10, Tema 5)    | Servidor de reserva (MX 20): guarda el correu i el reenvia |
| Altres funcions         | La resta de serveis del mòdul         | Porta d'enllaç, secundari DNS i DHCP, serveis del 0374 |

`Nota:` Només fem redundants el web, l'FTP i la recepció de correu. Els altres serveis del 0375 (llistes, missatgeria, ràdio, vídeo, SFTP i TFTP) continuen només al 0375.

### Canvis al DNS

Els noms dels serveis redundants han d'apuntar a la **IP virtual**. Els altres han de continuar apuntant a l'**adreça real del 0375**. Al servidor DNS primari, modifiqueu aquests registres de la zona `educem.cat`, sense duplicar-los:

```
; Bloc d'alta disponibilitat: serveis redundants a la IP virtual
serveis  IN  A      192.168.1.10        ; nom de la IP virtual
www      IN  A      192.168.1.10        ; abans 192.168.1.2
web1     IN  A      192.168.1.10        ; abans 192.168.1.2
web2     IN  A      192.168.1.10        ; abans 192.168.1.2
ftp      IN  A      192.168.1.10        ; abans 192.168.1.2

; Serveis no redundants: a l'adreça real del 0375
llistes  IN  A      192.168.1.2         ; abans CNAME www
sftp     IN  A      192.168.1.2         ; abans CNAME ftp
tftp     IN  A      192.168.1.2         ; abans CNAME ftp

; Correu: servidor de reserva (apartat 04)
mail2    IN  A      192.168.1.1
@        IN  MX 20  mail2.educem.cat.
```

I a la zona inversa:

```
10       IN  PTR    serveis.educem.cat.
```

Incrementeu el serial de totes dues zones i feu `sudo rndc reload`.

- `llistes` era un àlies de `www`, però la web de les llistes (Mailman) només és al 0375. Si apuntés a la IP virtual, deixaria de funcionar en cas de canvi de servidor.
- `sftp` es queda al 0375 perquè cada servidor té les seves claus d'SSH. Si el servei canviés de servidor, el client detectaria una clau diferent i avisaria d'un possible atac.
- `www`, `web1`, `web2` i `ftp` funcionen igual mentre la IP virtual sigui al 0375.

```bash
# Comprovació des del client (la IP virtual encara no existeix: s'activa a l'apartat 02)
dig +short web1.educem.cat         # 192.168.1.10
dig +short llistes.educem.cat      # 192.168.1.2
dig +short MX educem.cat           # 10 mail.educem.cat. i 20 mail2.educem.cat.
```

### Abans de Començar

- **Feu una instantània de les dues màquines virtuals.** Aquest bloc modifica tots dos servidors.
- Els serveis dels temes 3, 4 i 5 han de funcionar al 0375.
- Heu de poder entrar per SSH als dos servidors.
- Els rellotges dels dos servidors han d'estar sincronitzats (`timedatectl`): és important per comparar registres.
- El 0374 també és el servidor del mòdul 0374. Els serveis que hi afegim conviuen amb els que ja té.

### Relació amb el Mòdul 0378

L'alta disponibilitat és un dels resultats d'aprenentatge del mòdul **0378, Seguretat i Alta Disponibilitat**. S'hi treballen els clústers, l'emmagatzematge compartit, els balancejadors de càrrega i la virtualització. En aquest bloc apliquem les idees bàsiques als serveis del mòdul 0375.
