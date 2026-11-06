## 05. FTP en Alta Disponibilitat

L'FTP es fa redundant igual que el web: el 0374 té la mateixa configuració, els mateixos usuaris i els fitxers sincronitzats. Però té tres diferències importants:

- **Adreça d'escolta:** el vsftpd del Tema 4 només escolta a `192.168.1.2`, i no respondria a la IP virtual.
- **Sessions:** si el servidor canvia, les sessions FTP obertes es tallen i el client s'ha de tornar a connectar. Molts clients, com FileZilla, ho fan sols.
- **Els usuaris escriuen dades.** Quan el 0374 és l'actiu, els fitxers que pugen els usuaris queden al 0374. Cal tenir-ho en compte en tornar al 0375 (punt 5).

### 1. vsftpd del 0375: Escoltar a la IP Virtual

Al Tema 4 (apartat 14) vam limitar vsftpd a l'adreça interna amb `listen_address=192.168.1.2`. Amb aquesta línia, les connexions a la IP virtual es rebutgen. Comenteu-la perquè vsftpd escolti a totes les adreces IPv4:

```bash
sudo nano /etc/vsftpd.conf
```

```
listen=YES
listen_ipv6=NO
#listen_address=192.168.1.2      # comentat: escolta a totes les adreces, també a la IP virtual
```

```bash
sudo systemctl restart vsftpd
sudo ss -tlnp | grep ':21 '          # 0.0.0.0:21
```

- El tallafoc del Tema 4 ja limita l'FTP a la xarxa interna.
- **Alternativa:** `listen_address=192.168.1.10` als dos servidors. Llavors, el servidor de reserva no té la IP virtual i vsftpd no podria arrencar, perquè no pot escoltar en una adreça que no té. Es resol permetent-ho al nucli: `net.ipv4.ip_nonlocal_bind = 1` a `/etc/sysctl.d/`.
- **Mode passiu:** com que `pasv_address` no està definida, vsftpd anuncia l'adreça a la qual s'ha connectat el client, és a dir, la IP virtual. Les connexions de dades també van al servidor actiu.

### 2. vsftpd al 0374

**Al 0374**, instal·leu vsftpd i creeu els mateixos usuaris, grups i shells que al Tema 4 (apartat 06). Feu servir les mateixes contrasenyes que al 0375:

```bash
sudo apt install vsftpd
sudo add-shell /usr/sbin/nologin
sudo addgroup ftpgrup
sudo adduser --shell /usr/sbin/nologin usuari1
sudo usermod -aG ftpgrup usuari1
sudo adduser --shell /usr/sbin/nologin --ingroup ftpgrup usuari2
```

Els identificadors numèrics (UID) dels usuaris poden ser diferents als dos servidors: `rsync` assigna els fitxers pel nom de l'usuari, no pel número. En un entorn real, els usuaris no es creen a cada servidor, sinó que es fan servir els del **directori LDAP** (mòdul 0374, Tema 1), i així són iguals a tot arreu.

**Al 0375**, copieu la configuració i el certificat de l'FTPS al 0374:

```bash
sudo tar czf - /etc/vsftpd.conf /etc/vsftpd.userlist \
    /etc/ssl/certs/vsftpd.crt /etc/ssl/private/vsftpd.key \
  | sudo ssh -i /root/.ssh/sincronitzacio root@192.168.1.1 'tar xzf - -C /'
```

Si feu servir configuració per usuari (`/etc/vsftpd/user_conf/`) o altres fitxers del Tema 4, afegiu-los a la llista. Si un fitxer de la llista no existeix, `tar` avisa i continua amb els altres.

**Al 0374**, reinicieu vsftpd i obriu els ports:

```bash
sudo systemctl restart vsftpd
sudo ufw allow from 192.168.1.0/24 to any port 21 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 40000:40100 proto tcp
```

### 3. Sincronitzar els Fitxers

**Al 0375**, afegiu els directoris de l'FTP a la llista de l'apartat 03:

```bash
sudo nano /etc/sincronitza.llista
```

```
# Directoris que se sincronitzen del 0375 al 0374 (un per línia)
/var/www/web1.educem.cat/html
/var/www/web2.educem.cat/html
/srv/ftp
/home/usuari1/ftp
/home/usuari2/ftp
```

L'script `sincronitza.sh` (apartat 03) ja els copiarà cada minut, amb els mateixos propietaris i permisos (la gàbia de `nobody` i la carpeta `fitxers` de cada usuari).

**Fitxer que identifica el servidor**, a l'arrel de l'FTP anònim:

```bash
echo "srv0375" | sudo tee /srv/ftp/servidor.txt     # al 0375
echo "srv0374" | sudo tee /srv/ftp/servidor.txt     # al 0374
```

### 4. Comprovació de Salut de vsftpd

Als **dos servidors**, afegiu una altra comprovació a `/etc/keepalived/keepalived.conf` i incloeu-la a `track_script`:

```
vrrp_script chk_vsftpd {
    script "/usr/bin/systemctl is-active --quiet vsftpd"
    interval 2
    fall 2
    rise 2
    weight -60
}

vrrp_instance SERVEIS {
    # ...

    track_script {
        chk_apache
        chk_vsftpd
    }
}
```

```bash
sudo keepalived -t
sudo systemctl reload keepalived
```

Hi ha una sola IP virtual per a tots els serveis: si falla l'FTP del 0375, també el web passa al 0374. Si es volgués que cada servei canviés de servidor per separat, caldria una instància VRRP i una IP virtual per a cada servei.

**Prova:** al client, una petició cada segon a l'FTP anònim:

```bash
while true; do
    echo "$(date +%T) $(curl -s --max-time 2 ftp://ftp.educem.cat/servidor.txt)"
    sleep 1
done
```

En aturar vsftpd al 0375, el client passa a rebre `srv0374`.

### 5. El Problema de la Tornada al 0375

Feu aquesta prova per veure el problema:

1. Al 0375, atureu vsftpd. La IP virtual passa al 0374.
2. Al client, `usuari1` puja un fitxer, que queda **al 0374**:

```bash
echo "Informe pujat durant la caiguda" > informe.txt
curl --ssl-reqd -k -u usuari1 -T informe.txt ftp://ftp.educem.cat/fitxers/
```

3. Al 0375, torneu a engegar vsftpd. Com que té més prioritat, el 0375 recupera la IP virtual immediatament (preempció).
4. `usuari1` llista els seus fitxers: **`informe.txt` no hi és**, perquè és al 0374.
5. Al cap d'un minut, `sincronitza.sh` (ara el 0375 torna a ser l'actiu) deixa el 0374 igual que el 0375, i amb `--delete` **esborra `informe.txt` del 0374. El fitxer s'ha perdut.**

Per evitar-ho, el retorn al 0375 no pot ser automàtic. Abans, cal recuperar els canvis fets al 0374. Ho fem en dos passos.

**1. Desactivar la preempció.** Als **dos servidors**, a `vrrp_instance`, poseu `state BACKUP` i afegiu `nopreempt`:

```
vrrp_instance SERVEIS {
    state BACKUP          # també al 0375: nopreempt no funciona amb l'estat inicial MASTER
    nopreempt
    # ... la resta igual ...
}
```

```bash
sudo keepalived -t
sudo systemctl restart keepalived
```

Ara, quan el 0374 passa a ser l'actiu, s'hi queda fins que l'administrador decideix tornar al 0375. Si tots dos arrenquen alhora, continua guanyant el de més prioritat (el 0375).

**2. Script de recuperació** (al 0375). Porta al 0375 el que ha canviat al 0374:

```bash
sudo nano /usr/local/bin/recupera.sh
```

```bash
#!/bin/bash
# recupera.sh - Porta al 0375 els canvis fets al 0374 mentre era el servidor actiu.
# S'executa a mà, abans de tornar la IP virtual al 0375.

VIP=192.168.1.10
REMOT=root@192.168.1.1
SSH="ssh -i /root/.ssh/sincronitzacio -o BatchMode=yes"

if ip -4 addr show dev enp0s8 | grep -q " $VIP/"; then
    echo "El 0375 ja té la IP virtual: no s'ha de recuperar res." >&2
    exit 1
fi

for dir in $(grep -v '^#' /etc/sincronitza.llista); do
    echo "Recuperant $dir..."
    rsync -az --delete --exclude servidor.txt -e "$SSH" "$REMOT:$dir/" "$dir/"
done

echo "Fet. Ara reinicieu keepalived al 0374 perquè el 0375 torni a ser el servidor actiu."
```

```bash
sudo chmod 755 /usr/local/bin/recupera.sh
```

**Procediment de tornada al 0375:**

1. Comproveu que el 0375 i els seus serveis funcionen.
2. Al 0375: `sudo recupera.sh`. Copia els canvis del 0374 al 0375, en sentit contrari al de cada minut.
3. Al 0374: `sudo systemctl restart keepalived`. En aturar-se, el 0374 deixa la IP virtual i el 0375 la pren. En tornar a arrencar, el 0374 queda de reserva.
4. Comproveu que `informe.txt` és al 0375 i que la sincronització de cada minut torna a funcionar.

Repetiu la prova del principi del punt 5: ara el fitxer no es perd.

**Per què s'esborrava el fitxer?** La sincronització d'un sol sentit suposa que les dades només canvien al servidor principal. Les solucions professionals guarden les dades en un **emmagatzematge compartit o replicat** (NFS en un servidor redundant, DRBD, clústers de fitxers), que no depèn de quin servidor sigui l'actiu. Es treballen al mòdul 0378.

### Resolució de Problemes

| Problema                                                     | Causa                                                       | Solució                                                          |
| ------------------------------------------------------------ | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| `curl: (7) Failed to connect` a `ftp.educem.cat`             | vsftpd encara escolta només a `192.168.1.2`                 | Comentar `listen_address` i reiniciar vsftpd                     |
| La connexió de control funciona, però el llistat es penja    | Ports passius tancats al 0374, o `pasv_address` amb l'adreça del 0375 | Obrir el 40000-40100; no definir `pasv_address`        |
| `530 Login incorrect` al 0374                                | L'usuari no existeix, té una altra contrasenya o falta `nologin` a `/etc/shells` | Crear-lo igual que al 0375; `sudo add-shell /usr/sbin/nologin` |
| Els fitxers del 0374 tenen propietaris incorrectes           | L'usuari o el grup no existien al 0374 quan es va sincronitzar | Crear-los i tornar a executar `sincronitza.sh`                |
| `recupera.sh` diu que el 0375 ja té la IP virtual            | El 0375 ja és l'actiu (no cal recuperar res, o és massa tard) | Comprovar on és la IP virtual abans de recuperar               |
| Després de reiniciar keepalived al 0375, no recupera la IP virtual | És l'efecte de `nopreempt`                            | Seguir el procediment de tornada                                 |

### Punt de Control

- [ ] vsftpd del 0375 escolta a totes les adreces i el client hi accedeix per `ftp.educem.cat` (IP virtual).
- [ ] El 0374 té els mateixos usuaris, la mateixa configuració i el mateix certificat.
- [ ] Els directoris de l'FTP se sincronitzen cada minut.
- [ ] En aturar vsftpd al 0375, l'FTP passa al 0374.
- [ ] Heu reproduït la pèrdua del fitxer amb la preempció i l'heu evitat amb `nopreempt` i `recupera.sh`.
