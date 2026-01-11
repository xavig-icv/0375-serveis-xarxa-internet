## 03. Instal·lació d'Apache HTTP Server

### Instal·lació en Debian/Ubuntu

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar l'Apache
sudo apt install apache2

# Verificar la instal·lació
apache2 -v
```

### 3.3 Gestió del Servei

```bash
# Iniciar l'Apache
sudo systemctl start apache2     # Debian/Ubuntu

# Aturar l'Apache
sudo systemctl stop apache2

# Reiniciar l'Apache
sudo systemctl restart apache2

# Recarregar configuració (sense aturar-lo)
sudo systemctl reload apache2

# Habilitar en l'arrencada (si es reinicia el servidor)
sudo systemctl enable apache2

# Verificar estat
sudo systemctl status apache2
```

### Verificar el Funcionament

**Comprovar que escolta al port 80:**

```bash
sudo ss -plutn | grep :80

# Sortida:
# tcp   0   0 0.0.0.0:80   0.0.0.0:*   LISTEN   1234/apache2
```

**Provar des del navegador del client:**

```
http://IP_DEL_SERVIDOR
```

Hauria de mostrar la pàgina per defecte de l'Apache.

**Amb curl:**

```bash
# Al mateix servidor
curl http://localhost

# Al mateix servidor o des d'un client
curl http://IP_DEL_SERVIDOR

# Hauria de retornar l'HTML de la pàgina per defecte
```

### Estructura de Directoris

```
/etc/apache2/
├── apache2.conf              # Configuració principal
├── envvars                   # Variables d'entorn
├── ports.conf                # Ports d'escolta
├── mods-available/           # Mòduls disponibles
├── mods-enabled/             # Mòduls actius (symlinks)
├── conf-available/           # Configuracions disponibles
├── conf-enabled/             # Configuracions actives
├── sites-available/          # Virtual hosts disponibles
│   └── 000-default.conf      # VHost per defecte
├── sites-enabled/            # Virtual hosts actius
│   └── 000-default.conf -> ../sites-available/000-default.conf
└── magic                     # Tipus MIME

/var/www/
└── html/                     # Document root per defecte
    └── index.html

/var/log/apache2/
├── access.log                # Log d'accessos
└── error.log                 # Log d'errors
```
