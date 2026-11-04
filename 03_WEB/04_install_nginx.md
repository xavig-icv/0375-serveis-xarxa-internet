## 04. Instal·lació de Nginx

Al laboratori, Apache i Nginx funcionen **alhora** al servidor 0375 (vegeu l'[escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md)):

- Apache fa servir els ports estàndard (80 i 443).
- Nginx fa servir els ports 8080 (HTTP) i 8443 (HTTPS).

### Instal·lació a Debian/Ubuntu

En instal·lar-se, Nginx intenta arrencar amb el seu lloc per defecte al port 80, que ja ocupa Apache. Per això aturem Apache un moment, canviem el port del lloc per defecte de Nginx i tornem a engegar Apache.

```bash
# 1. Aturar Apache temporalment (allibera el port 80 durant la instal·lació)
sudo systemctl stop apache2

# 2. Actualitzar repositoris i instal·lar Nginx
sudo apt update
sudo apt install nginx

# Verificar instal·lació
nginx -v

# Sortida:
# nginx version: nginx/1.24.0 (Ubuntu)
```

**3. Canviar el port del lloc per defecte de Nginx (del 80 al 8080):**

```bash
sudo nano /etc/nginx/sites-available/default
```

```nginx
    listen 8080 default_server;
    listen [::]:8080 default_server;
```

```bash
# Verificar la configuració i reiniciar Nginx
sudo nginx -t
sudo systemctl restart nginx

# 4. Tornar a engegar Apache
sudo systemctl start apache2

# 5. Comprovar els ports: Apache al 80 i Nginx al 8080
sudo ss -tlnp | grep -E 'apache2|nginx'
```

`Nota:` Si heu instal·lat Nginx amb Apache en marxa, la instal·lació avisarà que Nginx no ha pogut arrencar (el port 80 està ocupat). Canvieu el port com al pas 3 i després executeu:

```bash
sudo dpkg --configure -a
sudo systemctl restart nginx
```

### Gestió del Servei

```bash
# Iniciar Nginx
sudo systemctl start nginx

# Aturar Nginx
sudo systemctl stop nginx

# Reiniciar Nginx
sudo systemctl restart nginx

# Recarregar configuració
sudo systemctl reload nginx

# Habilitar en l'arrencada
sudo systemctl enable nginx

# Verificar estat
sudo systemctl status nginx
```

### Estructura de Directoris

```
/etc/nginx/
├── nginx.conf                # Configuració principal
├── conf.d/                   # Configuracions addicionals
├── sites-available/          # Virtual hosts disponibles (Ubuntu)
│   └── default
├── sites-enabled/            # Virtual hosts actius (Ubuntu)
│   └── default -> ../sites-available/default
├── modules-available/        # Mòduls disponibles
├── modules-enabled/          # Mòduls actius
└── snippets/                 # Fragments reutilitzables

/var/www/
└── html/                     # Document root per defecte
    └── index.nginx-debian.html

/var/log/nginx/
├── access.log                # Log d'accessos
└── error.log                 # Log d'errors
```

### Verificar Funcionament

```bash
# Verificar configuració
sudo nginx -t

# Sortida si correcte:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Provar amb curl al mateix servidor (pàgina de benvinguda de Nginx)
curl http://localhost:8080

# Navegador del client
http://192.168.1.2:8080
```
