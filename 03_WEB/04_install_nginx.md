## 04. Instal·lació de Nginx

### Instal·lació a Debian/Ubuntu

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar Nginx
sudo apt install nginx

# Verificar instal·lació
nginx -v

# Sortida:
# nginx version: nginx/1.22.1 (Ubuntu)
```

### Gestió del Servei

```bash
# Aturar l'Apache (no tenir conflicte de serveis al mateix port)
sudo systemctl stop apache2

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

# Provar amb curl al mateix servidor
curl http://localhost

# Navegador del client
http://IP_DEL_SERVIDOR
```
