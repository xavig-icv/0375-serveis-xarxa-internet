## 05. Instal·lació i Configuració d'Icecast

**Icecast** és el servidor de streaming d'àudio lliure de la fundació Xiph.Org. Rep emissions de les fonts i les reparteix als oients per HTTP. Accepta MP3, AAC, Ogg Vorbis, Opus i FLAC.

### Instal·lació

Al servidor 0375:

```bash
sudo apt update
sudo apt install icecast2
```

L'instal·lador pregunta si voleu configurar Icecast (**Configure Icecast2?**). Responeu **Yes** i, a les preguntes següents:

| Pregunta                               | Resposta                                     |
| -------------------------------------- | -------------------------------------------- |
| **Icecast2 hostname**                  | `radio.educem.cat`                           |
| **Icecast2 source password**           | La contrasenya de les fonts d'emissió        |
| **Icecast2 relay password**            | Una altra contrasenya, per als relleus       |
| **Icecast2 administration password**   | Una altra contrasenya, per a l'administració |

- No deixeu la contrasenya per defecte, `hackme`: la coneix tothom.
- Feu servir només lletres i números: la contrasenya de les fonts anirà dins d'adreces com `icecast://source:contrasenya@...`, on alguns caràcters tenen un significat especial.
- Si heu respost **No**, torneu a fer les preguntes amb `sudo dpkg-reconfigure icecast2`.

Fins que no es configura, el paquet deixa el servei desactivat. Activeu-lo i engegueu-lo:

```bash
sudo systemctl enable --now icecast2
systemctl status icecast2
sudo ss -tlnp | grep ':8000 '
```

Icecast no té un servei propi de systemd, sinó un script d'inici clàssic (`/etc/init.d/icecast2`). Per això, en activar-lo, apareix el missatge `Executing: /usr/lib/systemd/systemd-sysv-install enable icecast2`. És normal.

### Fitxers i Directoris

```
/etc/icecast2/icecast.xml        # Configuració (conté les contrasenyes: només la llegeixen root i el grup icecast)
/etc/default/icecast2            # Usuari i grup amb què s'executa el servei (icecast2:icecast)
/usr/share/icecast2/web/         # Pàgines públiques d'estat (status.xsl, status-json.xsl)
/usr/share/icecast2/admin/       # Pàgines d'administració
/var/log/icecast2/access.log     # Connexions dels oients i de les fonts
/var/log/icecast2/error.log      # Errors i esdeveniments del servidor
```

### El Fitxer `icecast.xml`

```bash
sudo cp /etc/icecast2/icecast.xml /etc/icecast2/icecast.xml.orig
sudo nano /etc/icecast2/icecast.xml
```

El fitxer té molts comentaris. Aquestes són les seccions principals, amb els canvis que fem al laboratori:

```xml
<icecast>
    <location>Barcelona</location>
    <admin>usuari1@educem.cat</admin>

    <limits>
        <clients>100</clients>
        <sources>4</sources>
        <queue-size>524288</queue-size>
        <client-timeout>30</client-timeout>
        <header-timeout>15</header-timeout>
        <source-timeout>10</source-timeout>
        <burst-on-connect>1</burst-on-connect>
        <burst-size>65535</burst-size>
    </limits>

    <authentication>
        <source-password>CONTRASENYA_FONTS</source-password>
        <relay-password>CONTRASENYA_RELLEUS</relay-password>
        <admin-user>admin</admin-user>
        <admin-password>CONTRASENYA_ADMIN</admin-password>
    </authentication>

    <hostname>radio.educem.cat</hostname>

    <listen-socket>
        <port>8000</port>
    </listen-socket>

    <!-- ... paths, logging i security es deixen com estan ... -->
</icecast>
```

| Element                         | Significat                                                                          |
| ------------------------------- | ----------------------------------------------------------------------------------- |
| `<location>`, `<admin>`         | Informació que mostra la pàgina d'estat                                             |
| `<clients>`                     | Connexions simultànies màximes de tot el servidor (oients i consultes a les pàgines) |
| `<sources>`                     | Emissions simultànies màximes. Per defecte és 2; en farem servir 3 (apartat 06)     |
| `<queue-size>`                  | Dades d'un flux que es guarden per als oients que van endarrerits (bytes)           |
| `<burst-on-connect>`, `<burst-size>` | En connectar-se, l'oient rep de cop uns segons de so i la reproducció comença de seguida |
| `<source-password>`             | Contrasenya de les fonts (usuari `source`)                                          |
| `<admin-user>`, `<admin-password>` | Usuari i contrasenya de les pàgines d'administració                              |
| `<hostname>`                    | Nom que fa servir Icecast per construir les adreces (llistes `.m3u`, pàgina d'estat) |
| `<listen-socket>`               | Port d'escolta. Sense `<bind-address>`, escolta a totes les interfícies              |

Comproveu la sintaxi abans de reiniciar. Un error d'XML impedeix que el servei arrenqui:

```bash
sudo apt install libxml2-utils
sudo xmllint --noout /etc/icecast2/icecast.xml     # sense sortida: el fitxer és correcte
sudo systemctl restart icecast2
sudo tail /var/log/icecast2/error.log
```

### Pàgines Web d'Icecast

Des del navegador del client:

| Adreça                                          | Contingut                                                                   |
| ----------------------------------------------- | --------------------------------------------------------------------------- |
| `http://radio.educem.cat:8000/`                 | Pàgina d'estat pública: emissions actives, oients i títol del que sona      |
| `http://radio.educem.cat:8000/admin/`           | Administració (usuari `admin`): llistar oients, moure'ls a un altre punt de muntatge, desconnectar una font |
| `http://radio.educem.cat:8000/status-json.xsl`  | Estadístiques en format JSON, per a scripts i pàgines web                   |

Mentre no hi hagi cap font connectada, la pàgina d'estat no mostra cap emissió.

### Tallafoc

Icecast escolta a totes les interfícies, també a la de la xarxa del centre (`enp0s3`). Obriu el port només a la xarxa interna:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 8000 proto tcp
sudo ufw status
```

### Primera Prova: un To en Directe

**Al servidor**, emeteu un to amb ffmpeg al punt de muntatge `/prova.mp3`:

```bash
ffmpeg -re -f lavfi -i "sine=frequency=440" -ac 2 -c:a libmp3lame -b:a 128k \
    -f mp3 icecast://source:CONTRASENYA_FONTS@localhost:8000/prova.mp3
```

- `-re` llegeix l'entrada a velocitat real. Sense aquesta opció, ffmpeg enviaria el so tan ràpid com pogués.
- El generador `sine` sense durada no s'acaba mai: atureu l'emissió amb `Ctrl+C`.

**Al client**, escolteu l'emissió i recarregueu la pàgina d'estat:

```bash
sudo apt install mpv
mpv http://radio.educem.cat:8000/prova.mp3
```

La pàgina d'estat mostra el punt de muntatge `/prova.mp3`, amb un oient. Quan atureu ffmpeg, el punt de muntatge desapareix i el reproductor es desconnecta.

### Resolució de Problemes

| Problema                                                     | Causa                                                        | Solució                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| Icecast no arrenca després d'editar la configuració          | Error de sintaxi XML                                         | `sudo xmllint --noout /etc/icecast2/icecast.xml`                 |
| `systemctl status` diu que el servei està desactivat         | El paquet no l'activa fins que es configura                  | `sudo systemctl enable --now icecast2`                           |
| Les contrasenyes continuen sent `hackme`                     | Es va respondre **No** a la configuració                     | `sudo dpkg-reconfigure icecast2` o editar `icecast.xml`          |
| ffmpeg diu `401 Unauthorized`                                | Contrasenya de la font incorrecta                            | Revisar `<source-password>`                                      |
| ffmpeg diu `403 Forbidden`                                   | Ja hi ha una font en aquest punt de muntatge, o s'ha arribat al límit de `<sources>` | Canviar de punt de muntatge o augmentar `<sources>` |
| El client no pot obrir `http://radio.educem.cat:8000`        | Tallafoc, servei aturat o DNS                                | `sudo ufw status`, `systemctl status icecast2`, `dig radio.educem.cat` |

### Punt de Control

- [ ] `systemctl is-active icecast2` retorna `active` i Icecast escolta al port 8000.
- [ ] Les contrasenyes no són `hackme`, i `<hostname>` és `radio.educem.cat`.
- [ ] `<sources>` és 4 i `xmllint` no mostra errors.
- [ ] El client escolta el to de `/prova.mp3` i el veu a la pàgina d'estat.
