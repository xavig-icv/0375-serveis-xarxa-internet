## 09. Seguretat, Rendiment i Documentació del Servei

### Contrasenyes i Accés

| Credencial                         | On es configura                                | Qui la necessita                                   |
| ---------------------------------- | ---------------------------------------------- | -------------------------------------------------- |
| Contrasenya general de les fonts   | `<source-password>`                            | Només l'emissió automàtica (`/etc/radio/radio.env`) |
| Contrasenya del directe            | `<password>` del punt de muntatge `/directe.mp3` | Qui fa el programa en directe (BUTT)             |
| Contrasenya dels relleus           | `<relay-password>`                             | Altres servidors Icecast que repeteixen l'emissió   |
| Usuari i contrasenya d'administració | `<admin-user>`, `<admin-password>`           | L'administrador del servei                          |

- **Cap contrasenya no pot ser `hackme`**, i cada funció ha de tenir la seva. Qui fa el directe no ha de poder emetre als altres punts de muntatge.
- **`icecast.xml` i `radio.env` contenen contrasenyes:** comproveu que no els pot llegir tothom.

```bash
ls -l /etc/icecast2/icecast.xml /etc/radio/radio.env
# -rw-r----- root icecast  icecast.xml
# -rw------- root root     radio.env
```

- **Les contrasenyes viatgen sense xifrar.** Icecast fa servir l'autenticació bàsica d'HTTP al port 8000. Dins de la xarxa interna és acceptable. Per emetre des de fora calen TLS (Icecast i ffmpeg ho permeten) o una VPN.
- **Contrasenya visible als processos.** La contrasenya de l'emissió automàtica forma part de l'adreça `icecast://` de l'ordre d'ffmpeg, i qualsevol usuari del servidor la pot veure amb `ps aux`. Per això, al servidor no hi ha d'haver usuaris que no hagin de saber-la.

**Administració per un túnel SSH.** La interfície d'administració també envia la contrasenya sense xifrar. Per administrar Icecast des d'un altre equip, és millor fer servir un túnel SSH (mòdul 0374):

```bash
# A l'equip de l'administrador
ssh -L 8000:localhost:8000 usuari@192.168.1.2
# Mentre el túnel està obert: http://localhost:8000/admin/ al navegador
```

La connexió va xifrada dins d'SSH fins al servidor.

### Límits i Ús de Recursos

| Límit                    | Valor del laboratori | Efecte                                                         |
| ------------------------ | -------------------- | -------------------------------------------------------------- |
| `<clients>`              | 100                  | Connexions simultànies màximes de tot el servidor              |
| `<sources>`              | 4                    | Emissions simultànies màximes                                  |
| `<max-listeners>`        | 50 per emissió       | Limita l'ample de banda de cada emissió                        |

Amb aquests límits, l'ample de banda màxim de sortida és: 50 × 128 + 50 × 64 + 50 × 128 kbps = **16 Mbps** (apartat 04). Si la connexió del servidor no en té prou, cal reduir els oients o la taxa de bits.

### Tallafoc

| Port   | Accés                    | Motiu                                                          |
| ------ | ------------------------ | -------------------------------------------------------------- |
| 8000   | Només `192.168.1.0/24`   | Fonts d'emissió (BUTT al client) i oients de la xarxa interna  |
| 443    | Ja obert (Tema 3)        | Web, emissions i podcast per HTTPS                             |

Si totes les fonts fossin al mateix servidor i tots els oients entressin per Apache, Icecast podria escoltar només a `127.0.0.1` (`<bind-address>127.0.0.1</bind-address>` dins de `<listen-socket>`), i el port 8000 quedaria tancat.

### Registres i Privacitat

- **Registre d'accessos d'Icecast** (`/var/log/icecast2/access.log`) i **d'Apache:** guarden l'adreça IP i el programa de cada oient. Segons el RGPD, l'adreça IP és una dada personal.
- **Rotació:** el paquet configura una rotació setmanal i en guarda 7 còpies (`/etc/logrotate.d/icecast2`). Cal decidir quant de temps es guarden els registres i informar-ne els oients a la web (política de privacitat).
- **Gravacions de persones:** per publicar la veu d'una persona, cal la seva autorització. En un centre educatiu, cal l'autorització de l'alumnat i, si són menors, de les famílies.

### Drets d'Autor i Llicències

- **Música comercial.** Emetre-la o incloure-la en un podcast és **comunicació pública**. Cal la llicència de les entitats de gestió de drets: a l'Estat espanyol, l'SGAE (autors), AGEDI (productors) i AIE (intèrprets). Les plataformes de ràdio en línia ja inclouen aquestes llicències.
- **Creative Commons.** Hi ha música que l'autor permet fer servir amb condicions:

| Condició                  | Significat                                                          |
| ------------------------- | ------------------------------------------------------------------- |
| **BY** (reconeixement)    | Cal citar l'autor i l'obra                                          |
| **SA** (compartir igual)  | Les obres derivades s'han de publicar amb la mateixa llicència      |
| **NC** (no comercial)     | No es pot fer servir amb finalitats comercials                       |
| **ND** (sense obres derivades) | No es pot modificar (per exemple, retallar o barrejar)         |

Per exemple, una obra **CC BY-NC** es pot emetre en una ràdio escolar sense publicitat, citant-ne l'autor.

### Prestacions Mínimes del Servidor

| Recurs                    | Laboratori                     | Ràdio d'un centre amb 200 oients                         |
| ------------------------- | ------------------------------ | -------------------------------------------------------- |
| CPU                       | 1 nucli                        | 2 nuclis (codificar dos formats en directe)              |
| Memòria                   | 1 GB                           | 2 GB                                                     |
| Ample de banda de sortida | Xarxa interna                  | 200 × 128 kbps = 25,6 Mbps, més un 10 %: uns 28 Mbps     |
| Disc                      | Pocs GB                        | Gravacions i podcast: 57,6 MB per hora a 128 kbps; 28,8 MB a 64 kbps |

Icecast gasta molt poca CPU. Qui en gasta és ffmpeg, perquè codifica cada format en temps real.

### Supervisió

Una comprovació senzilla per a un script de supervisió (com el projecte de monitoratge del mòdul 0374):

```bash
# La ràdio està emetent?
curl -s http://localhost:8000/status-json.xsl | grep -q '/radio.mp3' \
    && echo "OK: /radio.mp3 emet" || echo "ALERTA: /radio.mp3 no emet"

# Estat dels serveis
systemctl is-active icecast2 radio-auto apache2
```

### Còpies de Seguretat

```bash
sudo tar czf /root/radio-$(date +%F).tar.gz \
    /etc/icecast2 /etc/radio /etc/systemd/system/radio-auto.service \
    /usr/local/bin/radio-auto.sh /usr/local/bin/genera-feed.sh \
    /etc/apache2/sites-available/radio.educem.cat.conf \
    /etc/ssl/certs/radio.educem.cat.crt /etc/ssl/private/radio.educem.cat.key \
    /srv/radio /var/www/radio
```

La còpia conté contrasenyes i la clau privada del certificat: guardeu-la en un lloc protegit.

### Documentació del Servei

La documentació ha de permetre que una altra persona administri el servei sense haver-lo instal·lat. Apartats mínims:

| Apartat                        | Contingut                                                                     |
| ------------------------------ | ----------------------------------------------------------------------------- |
| **Descripció**                 | Què fa el servei i per a qui                                                  |
| **Programari i versions**      | Icecast, ffmpeg, Apache i les seves versions                                  |
| **Equips, noms i ports**       | Servidor 0375, `radio.educem.cat`, ports 8000 i 443                           |
| **Emissions**                  | Punts de muntatge, format, font i límits de cadascun                          |
| **Fitxers de configuració**    | `icecast.xml`, `radio.env`, `radio-auto.service`, lloc d'Apache               |
| **Credencials**                | Quines hi ha i on es guarden. **No hi escriviu les contrasenyes**             |
| **Procediments**               | Engegar i aturar, fer un directe, publicar un episodi, canviar una contrasenya |
| **Supervisió i registres**     | On són els registres i com es comprova que la ràdio emet                      |
| **Còpies de seguretat**        | Què es copia, on, cada quan i com es restaura                                 |
| **Problemes coneguts**         | Les taules de resolució de problemes d'aquest tema                            |

```bash
# Versions instal·lades, per a la documentació
apt policy icecast2 ffmpeg apache2 2>/dev/null | grep -E '^[a-z]|Installed'
```

### Llista de Comprovació de Seguretat

- [ ] Cap contrasenya no és `hackme`, i cada funció té la seva.
- [ ] `icecast.xml` i `radio.env` no els pot llegir tothom.
- [ ] El port 8000 només és accessible des de la xarxa interna.
- [ ] L'administració d'Icecast es fa des de la xarxa interna o per un túnel SSH.
- [ ] La web i les emissions es poden fer servir per HTTPS.
- [ ] Cada emissió té un límit d'oients d'acord amb l'ample de banda disponible.
- [ ] Teniu els drets de tot el que s'emet i l'autorització de les persones gravades.
- [ ] Els registres tenen rotació i la web informa de les dades que es guarden.
- [ ] Hi ha una còpia de seguretat de la configuració, la música, les gravacions i el podcast.
- [ ] El servei està documentat.
