## 17. Logs i Monitorització dels Servidors Web

Els logs (registres) són la primera eina per saber què passa en un servidor web: qui hi accedeix, quines pàgines demana, quins errors es produeixen i per què falla un servei.

Hi ha dos tipus de log:

- **Log d'accessos (`access.log`):** una línia per cada petició que el servidor ha respost, amb el codi d'estat HTTP.
- **Log d'errors (`error.log`):** els problemes del servidor:
  - errors de configuració;
  - fitxers que no existeixen;
  - accessos denegats;
  - autenticacions fallides.

### Ubicació dels Logs

| Servidor | Fitxer                                     | Contingut                                                                 |
| -------- | ------------------------------------------ | ------------------------------------------------------------------------- |
| Apache   | `/var/log/apache2/access.log`              | Accessos del lloc per defecte (`000-default`)                             |
| Apache   | `/var/log/apache2/error.log`               | Errors generals del servei                                                |
| Apache   | `/var/log/apache2/other_vhosts_access.log` | Accessos dels Virtual Hosts sense `CustomLog` propi                       |
| Apache   | `/var/log/apache2/web1.educem.cat-*.log`   | Logs propis de cada Virtual Host (`ErrorLog` i `CustomLog`, apartat 09)   |
| Nginx    | `/var/log/nginx/access.log`                | Accessos dels llocs sense `access_log` propi                              |
| Nginx    | `/var/log/nginx/error.log`                 | Errors generals del servei                                                |
| Nginx    | `/var/log/nginx/web2.educem.cat-*.log`     | Logs propis de cada server block (`access_log` i `error_log`, apartat 10) |

```bash
# Veure els logs en temps real (Ctrl+C per sortir)
sudo tail -f /var/log/apache2/web1.educem.cat-ssl-access.log
sudo tail -f /var/log/nginx/web2.educem.cat-ssl-error.log

# Veure diversos logs alhora
sudo tail -f /var/log/apache2/*.log
```

### Format del Log d'Accessos

Apache (format `combined`) i Nginx fan servir el mateix format per defecte:

```
192.168.1.50 - usuari1 [04/Nov/2026:10:15:32 +0100] "GET /admin/ HTTP/1.1" 200 312 "-" "curl/8.5.0"
│              │        │                            │                     │   │   │   └─ User-Agent (navegador o eina)
│              │        │                            │                     │   │   └───── Referer (pàgina d'origen)
│              │        │                            │                     │   └───────── Mida de la resposta (bytes)
│              │        │                            │                     └───────────── Codi d'estat HTTP
│              │        │                            └─────────────────────────────────── Petició (mètode, ruta, versió)
│              │        └──────────────────────────────────────────────────────────────── Data i hora
│              └───────────────────────────────────────────────────────────────────────── Usuari (autenticació bàsica) o "-"
└──────────────────────────────────────────────────────────────────────────────────────── IP del client
```

**Codis d'estat més habituals:**

| Codi  | Significat            | Quan apareix                                                   |
| ----- | --------------------- | -------------------------------------------------------------- |
| `200` | OK                    | Accés correcte                                                 |
| `301` | Redirecció permanent  | Redirecció d'HTTP a HTTPS                                      |
| `304` | No modificat          | El navegador ja tenia el recurs a la memòria cau               |
| `401` | No autoritzat         | Falten les credencials o són incorrectes (autenticació bàsica) |
| `403` | Prohibit              | Accés denegat per IP o per permisos                            |
| `404` | No trobat             | El fitxer no existeix                                          |
| `500` | Error intern          | Error del servidor (per exemple, un `.htaccess` incorrecte)    |
| `502` | Bad Gateway           | El proxy invers no pot contactar amb el backend                |

**Filtrar el log d'accessos:**

```bash
LOG=/var/log/apache2/web1.educem.cat-ssl-access.log

# Accessos correctes (el codi és el camp 9)
sudo awk '$9 == 200' $LOG

# Errors d'autenticació (401) i accessos denegats (403)
sudo awk '$9 == 401 || $9 == 403' $LOG

# Nombre de peticions per IP
sudo awk '{print $1}' $LOG | sort | uniq -c | sort -rn

# Nombre de peticions per codi d'estat
sudo awk '{print $9}' $LOG | sort | uniq -c
```

### Missatges del Log d'Errors

| Situació                                 | Apache (`error.log`)                                           | Nginx (`error.log`)                                         |
| ---------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| Petició sense credencials                | (només el `401` a l'`access.log`)                              | (només el `401` a l'`access.log`)                           |
| Usuari inexistent                        | `AH01618: user usuari9 not found`                              | `user "usuari9" was not found in "/etc/nginx/.htpasswd"`    |
| Contrasenya incorrecta                   | `AH01617: user usuari1: authentication failure ... Password Mismatch` | `user "usuari1": password mismatch`                  |
| Accés denegat per IP                     | `AH01630: client denied by server configuration`               | `access forbidden by rule`                                  |
| Fitxer inexistent                        | (només el `404` a l'`access.log` amb el `LogLevel warn` per defecte) | `open() "..." failed (2: No such file or directory)`  |

```bash
# Autenticacions fallides i accessos denegats a Apache
sudo grep -E 'AH01617|AH01618|AH01630' /var/log/apache2/*error.log

# El mateix a Nginx
sudo grep -E 'password mismatch|was not found|forbidden by rule' /var/log/nginx/*error.log
```

### Errors de Connexió

Quan el client **no arriba a connectar** amb el servidor, la petició no queda registrada a cap log d'accessos: el servidor no l'ha rebuda o no l'ha pogut respondre. Aquests errors es diagnostiquen des del client i amb l'estat del servei:

| Error al client (`curl`)                         | Causa probable                                       | On mirar al servidor                                      |
| ------------------------------------------------ | ---------------------------------------------------- | --------------------------------------------------------- |
| `Could not resolve host`                         | El nom no es resol (DNS)                             | Zona DNS, `resolvectl dns` al client                      |
| `Failed to connect ... Connection refused`       | El servei està aturat o no escolta en aquest port    | `systemctl status`, `ss -tlnp`                            |
| `Failed to connect ... Connection timed out`     | Tallafoc o ruta                                      | Tallafoc del servidor, gateway                            |
| `SSL certificate problem: self-signed certificate` | Certificat autosignat (cal `-k`)                   | Normal al laboratori                                      |

**Errors en arrencar el servei** (configuració incorrecta):

```bash
# Comprovar la configuració
sudo apache2ctl configtest
sudo nginx -t

# Missatges del servei
sudo systemctl status apache2
sudo journalctl -u apache2 -n 20
sudo journalctl -u nginx -n 20
```

**Exemple per provocar i observar un error de connexió:**

```bash
# Al servidor: aturar Nginx
sudo systemctl stop nginx

# Al client: la connexió és rebutjada i no queda cap línia a l'access.log de Nginx
curl -k https://web2.educem.cat:8443

# Al servidor: tornar a engegar-lo
sudo systemctl start nginx
```
