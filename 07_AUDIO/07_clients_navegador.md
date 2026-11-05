## 07. Reproductors i Àudio al Navegador

### Reproductors al Client

| Reproductor   | Tipus                                | Instal·lació              | Com obrir una emissió                                   |
| ------------- | ------------------------------------ | ------------------------- | ------------------------------------------------------- |
| **VLC**       | Gràfic (i `cvlc`, de text)           | `sudo apt install vlc`    | **Media** → **Open Network Stream** i escriure l'adreça |
| **mpv**       | Línia d'ordres                       | `sudo apt install mpv`    | `mpv adreça`                                            |
| **ffplay**    | Línia d'ordres (paquet `ffmpeg`)     | Ja instal·lat             | `ffplay -nodisp adreça`                                 |
| **Navegador** | Element `<audio>` de les pàgines web | -                         | Obrir l'adreça o la web de la ràdio                     |
| **Mòbil**     | Aplicacions de ràdio i de podcasts   | -                         | Adreça de l'emissió o llista `.m3u`                     |

```bash
# Al client
sudo apt install vlc mpv

mpv http://radio.educem.cat:8000/radio.mp3          # mostra el títol del que sona
cvlc http://radio.educem.cat:8000/radio.opus        # VLC sense interfície gràfica
```

A VLC, **Tools** → **Codec Information** mostra el còdec, la freqüència de mostreig i la taxa de bits que arriben. Comproveu que coincideixen amb els de cada punt de muntatge. Els noms dels menús surten en l'idioma del sistema.

### Llistes de Reproducció (`.m3u`)

Icecast genera automàticament una llista de reproducció per a cada emissió: n'hi ha prou d'afegir `.m3u` a l'adreça.

```bash
curl http://radio.educem.cat:8000/radio.mp3.m3u
# http://radio.educem.cat:8000/radio.mp3
```

Una pàgina web pot enllaçar la llista `.m3u`: en fer-hi clic, el sistema l'obre amb el reproductor que té configurat. Per això és important el valor de `<hostname>` (apartat 05): Icecast el fa servir per escriure l'adreça dins de la llista.

### La Web de la Ràdio amb HTTPS

La ràdio necessita una pàgina web amb un reproductor. Hi ha un problema: si la pàgina és HTTPS i el reproductor carrega l'emissió per HTTP (`http://radio.educem.cat:8000/...`), el navegador ho bloqueja per **contingut mixt**. La solució és que Apache serveixi també les emissions per HTTPS, fent de servidor intermediari (*proxy* invers) cap a Icecast:

```
 Navegador ──HTTPS 443──► Apache (radio.educem.cat)
                            ├─ /               → pàgina web (/var/www/radio)
                            ├─ /podcast/       → podcast (apartat 08)
                            └─ /radio.mp3 ...  → proxy ──HTTP──► Icecast (127.0.0.1:8000)
```

**1. Mòduls de proxy i certificat:**

```bash
sudo a2enmod proxy proxy_http
sudo mkdir -p /var/www/radio/podcast/episodis

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/radio.educem.cat.key \
    -out /etc/ssl/certs/radio.educem.cat.crt \
    -subj "/C=ES/ST=Catalunya/L=Barcelona/O=EDUCEM/CN=radio.educem.cat" \
    -addext "subjectAltName=DNS:radio.educem.cat"
```

**2. El lloc web:**

```bash
sudo nano /etc/apache2/sites-available/radio.educem.cat.conf
```

```apache
<VirtualHost *:443>
    ServerName radio.educem.cat
    DocumentRoot /var/www/radio

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/radio.educem.cat.crt
    SSLCertificateKeyFile /etc/ssl/private/radio.educem.cat.key

    # Emissions d'Icecast servides per HTTPS
    ProxyPass /radio.mp3       http://127.0.0.1:8000/radio.mp3
    ProxyPass /radio.opus      http://127.0.0.1:8000/radio.opus
    ProxyPass /directe.mp3     http://127.0.0.1:8000/directe.mp3
    ProxyPass /status-json.xsl http://127.0.0.1:8000/status-json.xsl

    ErrorLog ${APACHE_LOG_DIR}/radio.educem.cat-error.log
    CustomLog ${APACHE_LOG_DIR}/radio.educem.cat-access.log combined
</VirtualHost>
```

Igual que amb `llistes.educem.cat` (Tema 6), no hi afegim un lloc al port 80: els usuaris entren per `https://radio.educem.cat`.

**3. La pàgina amb els reproductors:**

```bash
sudo nano /var/www/radio/index.html
```

```html
<!DOCTYPE html>
<html lang="ca">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Ràdio EDUCEM</title>
</head>
<body>
  <h1>Ràdio EDUCEM</h1>

  <h2>En directe</h2>
  <audio controls preload="none" src="/directe.mp3"></audio>
  <p>Oients connectats: <span id="oients">-</span></p>

  <h2>Programació en format eficient</h2>
  <!-- El navegador fa servir el primer format que sap reproduir -->
  <audio controls preload="none">
    <source src="/radio.opus" type="audio/ogg; codecs=opus">
    <source src="/radio.mp3" type="audio/mpeg">
    El navegador no pot reproduir àudio.
  </audio>

  <h2>Podcast</h2>
  <p><a href="/podcast/feed.xml">Subscriu-te al podcast (RSS)</a></p>

  <script>
    // Estadístiques d'Icecast en JSON: suma els oients de totes les emissions
    fetch('/status-json.xsl')
      .then(resposta => resposta.json())
      .then(dades => {
        let fonts = dades.icestats.source || [];
        if (!Array.isArray(fonts)) fonts = [fonts];     // amb una sola emissió no és una llista
        const total = fonts.reduce((suma, f) => suma + (f.listeners || 0), 0);
        document.getElementById('oients').textContent = total;
      });
  </script>
</body>
</html>
```

**4. Activar-ho:**

```bash
sudo a2ensite radio.educem.cat.conf
sudo apache2ctl configtest
sudo systemctl restart apache2      # restart, perquè hem activat mòduls nous
```

**Prova des del client:**

```bash
# Capçaleres de l'emissió a través d'Apache
curl -k -s --max-time 3 -D - -o /dev/null https://radio.educem.cat/radio.mp3 \
    | grep -i -E '^HTTP|content-type|icy-name'
```

```
HTTP/1.1 200 OK
Content-Type: audio/mpeg
icy-name: Ràdio EDUCEM
```

curl acaba amb un error de temps esgotat, perquè l'emissió no s'acaba mai: és normal. Obriu `https://radio.educem.cat` al navegador i accepteu l'avís del certificat autosignat.

### Cada Navegador, el seu Format

El reproductor de la programació ofereix dos formats. El navegador fa servir el primer que sap reproduir:

- **Firefox i Chrome** reprodueixen Opus: fan servir `/radio.opus`, que gasta la meitat d'ample de banda.
- Un navegador que no reprodueixi Ogg Opus passa a `/radio.mp3`.

A les eines de desenvolupament del navegador (`F12`), la pestanya **Network** mostra quina de les dues adreces s'ha demanat.

### Estadístiques i Registres

**Estadístiques en directe:**

```bash
curl -s http://radio.educem.cat:8000/status-json.xsl | python3 -m json.tool
```

Per a cada emissió apareixen l'adreça (`listenurl`), els oients (`listeners`), la taxa de bits i el títol del que sona. La pàgina d'administració (`http://radio.educem.cat:8000/admin/`) també llista els oients de cada punt de muntatge i permet desconnectar-los o moure'ls a una altra emissió.

**Registre d'accessos d'Icecast** (`/var/log/icecast2/access.log`). Cada línia s'escriu quan un oient es desconnecta. L'últim camp són els segons que ha estat escoltant:

```
192.168.1.50 - - [05/Nov/2026:10:31:02 +0100] "GET /radio.mp3 HTTP/1.1" 200 1925120 "-" "mpv 0.37.0" 120
```

```bash
# Connexions i temps mitjà d'escolta de cada emissió
sudo awk '$7 ~ /\.(mp3|opus)$/ {n[$7]++; s[$7]+=$NF}
          END {for (m in n) printf "%-15s %4d connexions  %6.0f s de mitjana\n", m, n[m], s[m]/n[m]}' \
    /var/log/icecast2/access.log
```

**Oients a través d'Apache.** Icecast veu les connexions que arriben per Apache com si vinguessin de `127.0.0.1`. Les adreces reals dels oients són al registre d'Apache (`/var/log/apache2/radio.educem.cat-access.log`).

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                                          |
| -------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------- |
| La pàgina es carrega, però el reproductor no sona              | Emissió per HTTP dins d'una pàgina HTTPS (contingut mixt) | Fer servir les adreces del proxy (`/radio.mp3`), no les del port 8000 |
| Error 503 a `https://radio.educem.cat/radio.mp3`               | Icecast aturat                                           | `systemctl status icecast2`                                      |
| Error 404 a l'emissió                                          | El punt de muntatge no té font i no té emissió de reserva | Comprovar el servei `radio-auto` (apartat 06)                    |
| `AH01144: No protocol handler was valid for the URL`           | El mòdul `proxy_http` no està activat                    | `sudo a2enmod proxy_http` i `sudo systemctl restart apache2`     |
| El nombre d'oients surt sempre com a `-`                       | `/status-json.xsl` no passa pel proxy o hi ha un error de JavaScript | Afegir-ne el `ProxyPass`; consola del navegador (`F12`)  |
| Tots els oients d'Icecast tenen l'adreça `127.0.0.1`           | Arriben a través d'Apache                                | És normal: les adreces reals són al registre d'Apache            |

### Punt de Control

- [ ] VLC i mpv reprodueixen les tres emissions, i VLC mostra el còdec i la taxa de bits de cadascuna.
- [ ] `curl http://radio.educem.cat:8000/radio.mp3.m3u` retorna l'adreça de l'emissió.
- [ ] `https://radio.educem.cat` mostra la pàgina i els reproductors funcionen des del navegador.
- [ ] La pàgina mostra el nombre d'oients.
- [ ] Sabeu calcular les connexions i el temps mitjà d'escolta a partir de `access.log`.
