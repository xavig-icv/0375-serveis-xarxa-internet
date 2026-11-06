## 11. Seguretat, Rendiment i Documentació del Servei

### Seguretat del Servei de Vídeo (Servidor 0375)

| Aspecte                       | Mesura                                                                                  |
| ----------------------------- | --------------------------------------------------------------------------------------- |
| **Xifratge**                  | Web, vídeos i HLS només per HTTPS (`video.educem.cat`)                                  |
| **Vídeos privats**            | Autenticació d'Apache al directori (apartat 05). La protecció també cobreix els fragments HLS, perquè són fitxers del mateix directori |
| **Qui pot emetre en directe** | `allow publish` només per a la xarxa interna. Per exigir una clau secreta, `on_publish` (apartat 07) |
| **RTMP sense xifrar**         | Només a la xarxa interna. Des de fora, RTMPS o SRT amb xifratge                         |
| **Pujar vídeos al servidor**  | Per SFTP, amb un usuari sense accés a la resta del sistema (Tema 4)                     |
| **Ús des d'altres webs**      | Una altra web podria enllaçar els vídeos i fer servir el vostre ample de banda. Es pot evitar amb enllaços signats i amb caducitat (per exemple, el mòdul `secure_link` de Nginx) |
| **Tallafoc**                  | Port 1935 només per a la xarxa interna; 443 per a la web                                |

### Seguretat de la Videoconferència (Servidor Jitsi)

- **Accés a les sales:** noms difícils d'endevinar, contrasenya i sala d'espera. El moderador decideix qui entra.
- **Qui pot crear sales:** per defecte, qualsevol que arribi a la web. En producció, s'integra amb l'autenticació de l'organització (JWT).
- **Xifratge extrem a extrem:** Jitsi el pot activar en els navegadors compatibles; el Videobridge no pot veure el contingut.
- **Actualitzacions:** Jitsi s'actualitza des del seu repositori. Feu `sudo apt update && sudo apt upgrade` sovint: és un servei exposat als usuaris.
- **Tallafoc:** només els ports de l'apartat 10, i només per a la xarxa interna.

### Privacitat i Drets d'Imatge

- **La imatge i la veu d'una persona són dades personals** (RGPD), i a més hi ha el dret a la pròpia imatge.
- **Centres educatius:** per gravar o publicar imatges de l'alumnat, cal el consentiment. A l'Estat espanyol, els menors de 14 anys necessiten el de la família.
- **Gravar una videoconferència:** cal avisar abans de començar i explicar per a què es grava, qui hi tindrà accés i quant de temps es guardarà. Millor no gravar per defecte.
- **Registres:** Apache, Nginx i Jitsi guarden les adreces IP. Cal decidir quant de temps es conserven.
- **Drets d'autor:** cal citar les obres amb llicència Creative Commons (*Big Buck Bunny*: «© Blender Foundation | peach.blender.org»).

### Prestacions i Ample de Banda

**Vídeo sota demanda i directes:** l'ample de banda de sortida és el nombre d'espectadors per la taxa de bits (Tema 7, apartat 04).

| Escenari                                       | Càlcul                         | Ample de banda |
| ---------------------------------------------- | ------------------------------ | -------------- |
| 100 alumnes miren un vídeo de 360p (1 Mbps)    | 100 × 1 Mbps                   | 100 Mbps       |
| 50 espectadors d'un directe en 720p (2,9 Mbps) | 50 × 2,9 Mbps                  | 145 Mbps       |

**Videoconferència amb 25 alumnes**, amb vídeo de 360p i *simulcast* (cada participant envia uns 0,7 Mbps i rep 24 miniatures de 180p a 0,2 Mbps):

- El Videobridge rep 25 × 0,7 = 17,5 Mbps.
- Cada participant rep 24 × 0,2 = 4,8 Mbps, i el Videobridge envia 25 × 4,8 = **120 Mbps**.

Per això, limitar la resolució (apartat 10) i demanar als participants que tanquin la càmera quan no cal fa que una classe funcioni amb la connexió d'un centre.

| Recurs        | Servidor de vídeo (0375)                                       | Servidor Jitsi                                        |
| ------------- | -------------------------------------------------------------- | ----------------------------------------------------- |
| **CPU**       | Molta per codificar: un directe de 720p en H.264 ocupa un nucli; cada qualitat de l'HLS, més. AV1 és molt lent | Poca: l'SFU no descodifica el vídeo. Prosody fa servir un sol nucli |
| **Memòria**   | 2 GB                                                           | 4 GB per a reunions petites; 8 GB recomanats          |
| **Xarxa**     | Taxa × espectadors                                             | És el recurs que limita el servei                     |
| **Disc**      | 1 hora de 720p a 2,5 Mbps ocupa 1,125 GB; l'HLS amb dues qualitats, uns 1,7 GB | Poc: registres                          |

### Supervisió

```bash
# Servidor 0375
systemctl is-active apache2 nginx
ls /var/www/video/directe/          # hi ha un directe en marxa si hi ha fitxers .m3u8
sudo tail /var/log/nginx/error.log
du -sh /var/www/video /srv/video/gravacions

# Servidor Jitsi
systemctl is-active prosody jicofo jitsi-videobridge2 nginx
sudo tail /var/log/jitsi/jvb.log
```

**Les gravacions omplen el disc.** Per veure les que tenen més de 30 dies i, després de revisar-les, esborrar-les:

```bash
sudo find /srv/video/gravacions -name '*.flv' -mtime +30 -print
sudo find /srv/video/gravacions -name '*.flv' -mtime +30 -delete
```

Aquesta segona ordre es pot programar amb `cron` (mòdul 0374) quan la política de conservació estigui decidida.

### Còpies de Seguretat

```bash
# Servidor 0375: configuració, web, scripts i vídeos originals
sudo tar czf /root/video-config-$(date +%F).tar.gz \
    /etc/nginx/rtmp.conf /etc/apache2/sites-available/video.educem.cat.conf \
    /etc/ssl/certs/video.educem.cat.crt /etc/ssl/private/video.educem.cat.key \
    /usr/local/bin/genera-videofeed.sh /var/www/video/*.html

# Servidor Jitsi: configuració
sudo tar czf /root/jitsi-config-$(date +%F).tar.gz /etc/jitsi /etc/prosody /etc/nginx/sites-available
```

Els vídeos ocupen molt. El més important és guardar els **originals**: les versions per a la web i l'HLS es poden tornar a generar amb ffmpeg i els scripts dels apartats 04 i 06.

### Documentació del Servei

| Apartat                        | Servei de vídeo (0375)                                        | Videoconferència (Jitsi)                              |
| ------------------------------ | ------------------------------------------------------------- | ----------------------------------------------------- |
| **Descripció**                 | Vídeo sota demanda, HLS, directes i videopodcast              | Videoconferència al navegador                         |
| **Equips, noms i ports**       | 0375, `video.educem.cat`, 443 i 1935                          | `meet.educem.cat` (`192.168.1.3`), 443 i 10000/udp    |
| **Programari i versions**      | Apache, Nginx i el mòdul RTMP, ffmpeg                         | Jitsi Meet, Jicofo, Videobridge, Prosody, Nginx       |
| **Fitxers de configuració**    | `rtmp.conf`, lloc d'Apache, scripts                           | `config.js`, `jvb.conf`, `jicofo.conf`, Prosody       |
| **Procediments**               | Publicar un vídeo, fer un directe, publicar un episodi        | Crear una sala segura, actualitzar el servei          |
| **Credencials**                | On es guarden i qui les té (sense escriure les contrasenyes)  | Igual                                                 |
| **Supervisió i còpies**        | Registres, espai de disc, còpies                              | Registres, estat dels serveis, còpies                 |
| **Privacitat**                 | Consentiments, conservació de les gravacions i dels registres | Gravacions, registres                                 |
| **Problemes coneguts**         | Taules de resolució de problemes dels apartats 04-08          | Taula de resolució de problemes de l'apartat 10       |

```bash
# Versions instal·lades, per a la documentació
apt policy apache2 nginx libnginx-mod-rtmp ffmpeg 2>/dev/null | grep -E '^[a-z]|Installed'     # 0375
apt policy jitsi-meet jicofo jitsi-videobridge2 prosody 2>/dev/null | grep -E '^[a-z]|Installed' # Jitsi
```

### Llista de Comprovació de Seguretat

- [ ] El web, els vídeos, l'HLS i Jitsi només funcionen per HTTPS.
- [ ] Només es pot emetre en directe des de la xarxa interna.
- [ ] Els ports 1935 (0375) i 10000/udp (Jitsi) només estan oberts per a la xarxa interna.
- [ ] Les sales de videoconferència tenen contrasenya o sala d'espera quan cal.
- [ ] Els vídeos privats estan protegits amb autenticació.
- [ ] Hi ha consentiments per a les imatges de persones i una política de conservació de les gravacions.
- [ ] Hi ha còpia de la configuració dels dos servidors i dels vídeos originals.
- [ ] El servei està documentat.
