# Guide complet — Inception (42)

> **Login utilisé dans ce guide :** `login` → remplace-le partout par ton propre login 42.

---

## Table des matières

1. [Comprendre le projet](#1-comprendre-le-projet)
2. [Préparer la machine virtuelle](#2-préparer-la-machine-virtuelle)
3. [Installer Docker et Docker Compose](#3-installer-docker-et-docker-compose)
4. [Structure du projet](#4-structure-du-projet)
5. [Les secrets Docker](#5-les-secrets-docker)
6. [Le fichier .env](#6-le-fichier-env)
7. [Container MariaDB](#7-container-mariadb)
8. [Container WordPress + php-fpm](#8-container-wordpress--php-fpm)
9. [Container NGINX](#9-container-nginx)
10. [Le fichier docker-compose.yml](#10-le-fichier-docker-composeyml)
11. [Le Makefile](#11-le-makefile)
12. [Configurer le domaine local](#12-configurer-le-domaine-local)
13. [Fichiers de documentation requis](#13-fichiers-de-documentation-requis)
14. [BONUS — Redis cache](#14-bonus--redis-cache)
15. [BONUS — Serveur FTP](#15-bonus--serveur-ftp)
16. [BONUS — Site statique](#16-bonus--site-statique)
17. [BONUS — Adminer](#17-bonus--adminer)
18. [BONUS — Service au choix](#18-bonus--service-au-choix)
19. [Checklist finale avant soutenance](#19-checklist-finale-avant-soutenance)

---

## 1. Comprendre le projet

### Ce que tu vas construire

Tu dois mettre en place une **infrastructure Docker** composée de plusieurs services qui communiquent entre eux via un réseau Docker privé :

```
Internet (port 443)
       │
       ▼
  [NGINX] ──(TLS 1.2/1.3)
       │
       │ port 9000 (FastCGI)
       ▼
[WordPress + php-fpm]
       │
       │ port 3306 (MySQL)
       ▼
   [MariaDB]
       │
  [Volume DB]    [Volume WordPress files]
```

### Règles importantes à ne pas oublier

- **Tout doit tourner dans une VM** (pas directement sur ton OS).
- **Chaque service = un container = un Dockerfile** (pas de `docker pull` d'images toutes faites sauf Alpine/Debian).
- Utilise **Alpine ou Debian dans leur avant-dernière version stable** (ex: Debian 12 Bookworm ou Alpine 3.19).
- **Pas de `tail -f`, `sleep infinity`, `while true`, `bash`** comme commande de démarrage → apprends le fonctionnement des daemons (PID 1).
- **Pas de mot de passe en clair dans les Dockerfiles** → utilise les variables d'environnement et les Docker secrets.
- **Pas le tag `latest`** dans tes Dockerfiles.
- Les containers doivent **redémarrer automatiquement** en cas de crash (`restart: unless-stopped` ou `restart: always`).
- Le seul point d'entrée externe est **NGINX sur le port 443**.
- `network: host`, `--link` et `links:` sont **interdits**.

---

## 2. Préparer la machine virtuelle

### Installer VirtualBox (sur ton PC portable)

Télécharge VirtualBox depuis le site officiel et installe-le.

### Créer la VM

- **OS :** Debian 12 (Bookworm) — image ISO officielle Debian
- **RAM :** minimum 4 Go (8 Go recommandé)
- **Disque :** minimum 20 Go (30 Go recommandé pour les volumes)
- **CPU :** 2 cœurs minimum

### Configuration réseau recommandée

Dans VirtualBox, configure l'adaptateur réseau de ta VM en **NAT** avec une redirection de port :
- Port hôte `443` → Port VM `443`

Ou utilise le mode **Bridge** pour que ta VM soit directement sur ton réseau local.

### Installation de Debian dans la VM

1. Lance l'ISO, suis l'installateur standard.
2. Installe uniquement le minimum (pas d'interface graphique nécessaire, mais c'est plus confortable pour débuter).
3. Crée un utilisateur et note le mot de passe root.

### Mise à jour du système

```bash
su -
apt update && apt upgrade -y
apt install -y sudo curl git make
# Ajoute ton utilisateur au groupe sudo
usermod -aG sudo login
```

Reconnecte-toi pour que le groupe prenne effet.

---

## 3. Installer Docker et Docker Compose

### Installer Docker Engine

```bash
# Dépendances
sudo apt install -y ca-certificates curl gnupg lsb-release

# Clé GPG officielle Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajoute le dépôt Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installe Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Permettre à ton utilisateur d'utiliser Docker sans sudo

```bash
sudo usermod -aG docker $USER
# Reconnecte-toi ou tape :
newgrp docker
```

### Vérifier l'installation

```bash
docker --version
docker compose version
```

---

## 4. Structure du projet

Voici la structure complète du dépôt Git à respecter :

```
.
├── Makefile
├── README.md
├── USER_DOC.md
├── DEV_DOC.md
├── secrets/
│   ├── credentials.txt          # login WordPress admin
│   ├── db_password.txt          # mot de passe user MariaDB
│   └── db_root_password.txt     # mot de passe root MariaDB
└── srcs/
    ├── .env
    ├── docker-compose.yml
    └── requirements/
        ├── mariadb/
        │   ├── Dockerfile
        │   ├── .dockerignore
        │   ├── conf/
        │   │   └── 50-server.cnf
        │   └── tools/
        │       └── init_db.sh
        ├── nginx/
        │   ├── Dockerfile
        │   ├── .dockerignore
        │   ├── conf/
        │   │   └── nginx.conf
        │   └── tools/
        │       └── generate_ssl.sh
        ├── wordpress/
        │   ├── Dockerfile
        │   ├── .dockerignore
        │   ├── conf/
        │   │   └── www.conf
        │   └── tools/
        │       └── init_wp.sh
        └── bonus/                # uniquement si tu fais les bonus
            ├── redis/
            │   └── Dockerfile
            ├── ftp/
            │   └── Dockerfile
            ├── static/
            │   ├── Dockerfile
            │   └── index.html
            └── adminer/
                └── Dockerfile
```

### Créer la structure

```bash
mkdir -p srcs/requirements/{mariadb,nginx,wordpress}/{conf,tools}
mkdir -p srcs/requirements/bonus/{redis,ftp,static,adminer}
mkdir -p secrets
touch Makefile README.md USER_DOC.md DEV_DOC.md
touch srcs/.env srcs/docker-compose.yml
```

### Créer le .gitignore

```bash
cat > .gitignore << 'EOF'
secrets/
srcs/.env
*.swp
*.swo
EOF
```

> **Important :** les fichiers `secrets/` et `.env` ne doivent **jamais** être commités sur Git. Toute credential dans ton repo = **échec du projet**.

---

## 5. Les secrets Docker

Les Docker secrets permettent de passer des informations sensibles aux containers sans les exposer comme variables d'environnement classiques. Ils sont montés comme fichiers dans `/run/secrets/`.

### Créer les fichiers de secrets

```bash
# Mot de passe de l'utilisateur MariaDB
echo "monMotDePasseDB" > secrets/db_password.txt

# Mot de passe root MariaDB
echo "monMotDePasseRoot" > secrets/db_root_password.txt

# Identifiants WordPress admin (format user:password)
echo "wp_admin_user" > secrets/credentials.txt
# ou un fichier dédié par information selon ta préférence
```

> **Règle :** le login de l'admin WordPress ne doit **pas** contenir `admin`, `Admin`, `administrator` ou `Administrator`.

---

## 6. Le fichier .env

Le fichier `.env` contient les variables d'environnement non sensibles utilisées par `docker-compose.yml`.

```bash
# srcs/.env

DOMAIN_NAME=login.42.fr

# MySQL / MariaDB
MYSQL_DATABASE=wordpress
MYSQL_USER=wp_user

# WordPress
WP_TITLE=Mon Site Inception
WP_ADMIN_LOGIN=wp_superadmin
WP_ADMIN_EMAIL=admin@login.42.fr
WP_USER_LOGIN=wp_editor
WP_USER_EMAIL=editor@login.42.fr
```

> Les mots de passe ne sont **pas** ici — ils viennent des secrets.

---

## 7. Container MariaDB

### Pourquoi MariaDB ?

MariaDB est le serveur de base de données qui stocke toutes les données WordPress (articles, utilisateurs, configuration). Il tourne dans un container isolé, accessible uniquement depuis le réseau Docker interne.

### Dockerfile MariaDB

```dockerfile
# srcs/requirements/mariadb/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    mariadb-server \
    && rm -rf /var/lib/apt/lists/*

COPY conf/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
COPY tools/init_db.sh /usr/local/bin/init_db.sh
RUN chmod +x /usr/local/bin/init_db.sh

EXPOSE 3306

ENTRYPOINT ["/usr/local/bin/init_db.sh"]
```

### Configuration MariaDB

```ini
# srcs/requirements/mariadb/conf/50-server.cnf
[mysqld]
user            = mysql
pid-file        = /run/mysqld/mysqld.pid
socket          = /run/mysqld/mysqld.sock
port            = 3306
# Écoute sur toutes les interfaces (nécessaire pour que WordPress s'y connecte)
bind-address    = 0.0.0.0
datadir         = /var/lib/mysql
```

### Script d'initialisation

```bash
#!/bin/bash
# srcs/requirements/mariadb/tools/init_db.sh

# Lecture des secrets
DB_PASSWORD=$(cat /run/secrets/db_password)
DB_ROOT_PASSWORD=$(cat /run/secrets/db_root_password)

# Initialise le datadir si vide (premier démarrage)
if [ ! -d "/var/lib/mysql/mysql" ]; then
    mysql_install_db --user=mysql --datadir=/var/lib/mysql > /dev/null

    # Démarre MariaDB temporairement pour la configuration initiale
    mysqld --user=mysql --bootstrap << EOF
USE mysql;
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '${DB_ROOT_PASSWORD}';
CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
CREATE USER IF NOT EXISTS '${MYSQL_USER}'@'%' IDENTIFIED BY '${DB_PASSWORD}';
GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '${MYSQL_USER}'@'%';
FLUSH PRIVILEGES;
EOF
fi

# Lance MariaDB en avant-plan (PID 1)
exec mysqld --user=mysql
```

> **PID 1 expliqué :** Dans un container Docker, le processus lancé par `CMD`/`ENTRYPOINT` est le PID 1. Si ce processus s'arrête, le container s'arrête. C'est pourquoi on utilise `exec mysqld` (et non `service mysql start`) : `exec` remplace le processus shell par `mysqld` directement, qui devient le PID 1 et reste en avant-plan.

---

## 8. Container WordPress + php-fpm

### Pourquoi php-fpm ?

WordPress est un CMS écrit en PHP. `php-fpm` (FastCGI Process Manager) est le processus qui interprète le PHP. NGINX lui envoie les requêtes PHP via le protocole FastCGI sur le port 9000. Les deux sont dans des **containers séparés**.

### Dockerfile WordPress

```dockerfile
# srcs/requirements/wordpress/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    php8.2-fpm \
    php8.2-mysql \
    php8.2-curl \
    php8.2-gd \
    php8.2-xml \
    php8.2-mbstring \
    php8.2-zip \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Télécharge wp-cli pour configurer WordPress
RUN wget -q https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    -O /usr/local/bin/wp \
    && chmod +x /usr/local/bin/wp

COPY conf/www.conf /etc/php/8.2/fpm/pool.d/www.conf
COPY tools/init_wp.sh /usr/local/bin/init_wp.sh
RUN chmod +x /usr/local/bin/init_wp.sh

RUN mkdir -p /run/php

EXPOSE 9000

ENTRYPOINT ["/usr/local/bin/init_wp.sh"]
```

### Configuration php-fpm

```ini
; srcs/requirements/wordpress/conf/www.conf
[www]
user = www-data
group = www-data
listen = 0.0.0.0:9000
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

> On écoute sur `0.0.0.0:9000` et non sur un socket Unix, car NGINX est dans un container **différent** et communique via le réseau Docker.

### Script d'initialisation WordPress

```bash
#!/bin/bash
# srcs/requirements/wordpress/tools/init_wp.sh

# Lecture des secrets
DB_PASSWORD=$(cat /run/secrets/db_password)
WP_ADMIN_PASSWORD=$(cat /run/secrets/db_root_password)  # ou un secret dédié
WP_USER_PASSWORD="user_password_from_secret"

WORDPRESS_PATH=/var/www/html

mkdir -p ${WORDPRESS_PATH}

# Installe WordPress si pas déjà fait (volume vide = premier démarrage)
if [ ! -f "${WORDPRESS_PATH}/wp-config.php" ]; then

    # Télécharge WordPress
    wp core download --path=${WORDPRESS_PATH} --allow-root

    # Crée wp-config.php
    wp config create \
        --path=${WORDPRESS_PATH} \
        --dbname=${MYSQL_DATABASE} \
        --dbuser=${MYSQL_USER} \
        --dbpass=${DB_PASSWORD} \
        --dbhost=mariadb:3306 \
        --allow-root

    # Attends que MariaDB soit prêt
    until wp db check --path=${WORDPRESS_PATH} --allow-root 2>/dev/null; do
        echo "En attente de MariaDB..."
        sleep 2
    done

    # Installe WordPress
    wp core install \
        --path=${WORDPRESS_PATH} \
        --url=https://${DOMAIN_NAME} \
        --title="${WP_TITLE}" \
        --admin_user=${WP_ADMIN_LOGIN} \
        --admin_password=${WP_ADMIN_PASSWORD} \
        --admin_email=${WP_ADMIN_EMAIL} \
        --allow-root

    # Crée un second utilisateur (non admin)
    wp user create \
        --path=${WORDPRESS_PATH} \
        ${WP_USER_LOGIN} ${WP_USER_EMAIL} \
        --role=editor \
        --user_pass=${WP_USER_PASSWORD} \
        --allow-root
fi

# Lance php-fpm en avant-plan (PID 1)
exec php-fpm8.2 -F
```

> Le flag `-F` de `php-fpm` signifie "foreground" : le processus reste en avant-plan et ne se met pas en daemon. C'est ce qu'on veut pour Docker.

---

## 9. Container NGINX

### Rôle de NGINX

NGINX est le seul point d'entrée depuis l'extérieur. Il :
1. Termine la connexion TLS (déchiffre HTTPS).
2. Sert les fichiers statiques WordPress directement.
3. Redirige les requêtes PHP vers php-fpm via FastCGI (port 9000).

### Générer un certificat SSL auto-signé

Pour le développement local, on génère un certificat auto-signé avec OpenSSL. En production, on utiliserait Let's Encrypt.

```bash
# Exemple de commande pour générer manuellement (pour comprendre)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/login.key \
    -out /etc/nginx/ssl/login.crt \
    -subj "/C=FR/ST=Paris/L=Paris/O=42/CN=login.42.fr"
```

### Dockerfile NGINX

```dockerfile
# srcs/requirements/nginx/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    nginx \
    openssl \
    && rm -rf /var/lib/apt/lists/*

COPY conf/nginx.conf /etc/nginx/nginx.conf
COPY tools/generate_ssl.sh /usr/local/bin/generate_ssl.sh
RUN chmod +x /usr/local/bin/generate_ssl.sh

RUN mkdir -p /etc/nginx/ssl

EXPOSE 443

ENTRYPOINT ["/usr/local/bin/generate_ssl.sh"]
```

### Script de génération SSL

```bash
#!/bin/bash
# srcs/requirements/nginx/tools/generate_ssl.sh

# Génère le certificat auto-signé au premier démarrage
if [ ! -f /etc/nginx/ssl/${DOMAIN_NAME}.crt ]; then
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/nginx/ssl/${DOMAIN_NAME}.key \
        -out /etc/nginx/ssl/${DOMAIN_NAME}.crt \
        -subj "/C=FR/ST=IDF/L=Paris/O=42/CN=${DOMAIN_NAME}"
fi

# Lance NGINX en avant-plan (PID 1)
exec nginx -g "daemon off;"
```

> `daemon off;` empêche NGINX de se mettre en arrière-plan. Sans ça, le container s'arrêterait immédiatement après le lancement.

### Configuration NGINX

```nginx
# srcs/requirements/nginx/conf/nginx.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name login.42.fr;  # Remplace login par ton login

        # Certificats TLS
        ssl_certificate     /etc/nginx/ssl/login.42.fr.crt;
        ssl_certificate_key /etc/nginx/ssl/login.42.fr.key;

        # TLS 1.2 et 1.3 uniquement (obligatoire selon le sujet)
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers   HIGH:!aNULL:!MD5;

        root /var/www/html;
        index index.php index.html;

        # Fichiers statiques servis directement
        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        # Requêtes PHP redirigées vers php-fpm
        location ~ \.php$ {
            fastcgi_pass   wordpress:9000;
            fastcgi_index  index.php;
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO $fastcgi_path_info;
        }

        # Interdit l'accès aux fichiers cachés (.htaccess, etc.)
        location ~ /\. {
            deny all;
        }
    }
}
```

> `wordpress:9000` : NGINX résout le hostname `wordpress` via le réseau Docker interne — pas besoin d'adresse IP manuelle.

---

## 10. Le fichier docker-compose.yml

Le `docker-compose.yml` orchestre tous les containers : il définit les services, les volumes, le réseau et les secrets.

```yaml
# srcs/docker-compose.yml
version: '3.8'

services:

  mariadb:
    build: ./requirements/mariadb
    image: mariadb
    container_name: mariadb
    restart: unless-stopped
    networks:
      - inception
    volumes:
      - db_data:/var/lib/mysql
    env_file:
      - .env
    secrets:
      - db_password
      - db_root_password

  wordpress:
    build: ./requirements/wordpress
    image: wordpress
    container_name: wordpress
    restart: unless-stopped
    networks:
      - inception
    volumes:
      - wp_data:/var/www/html
    env_file:
      - .env
    secrets:
      - db_password
    depends_on:
      - mariadb

  nginx:
    build: ./requirements/nginx
    image: nginx
    container_name: nginx
    restart: unless-stopped
    networks:
      - inception
    volumes:
      - wp_data:/var/www/html:ro
    ports:
      - "443:443"
    env_file:
      - .env
    depends_on:
      - wordpress

networks:
  inception:
    driver: bridge
    name: inception

volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/login/data/db

  wp_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/login/data/wordpress

secrets:
  db_password:
    file: ../secrets/db_password.txt
  db_root_password:
    file: ../secrets/db_root_password.txt
```

### Explication des volumes nommés

Le sujet exige des **volumes nommés** (pas de bind mounts directs dans les services), mais leurs données doivent se trouver dans `/home/login/data/`. La syntaxe avec `driver_opts: type: none, o: bind, device: ...` crée un volume nommé Docker dont le répertoire de stockage est fixé sur le système hôte — c'est la technique acceptée.

```bash
# Crée les dossiers sur la VM avant le premier lancement
mkdir -p /home/login/data/db
mkdir -p /home/login/data/wordpress
```

---

## 11. Le Makefile

Le Makefile est à la racine du projet et automatise les commandes courantes.

```makefile
# Makefile

COMPOSE_FILE = srcs/docker-compose.yml
DATA_DIR     = /home/login/data

all: setup
	docker compose -f $(COMPOSE_FILE) up -d --build

setup:
	mkdir -p $(DATA_DIR)/db
	mkdir -p $(DATA_DIR)/wordpress

down:
	docker compose -f $(COMPOSE_FILE) down

stop:
	docker compose -f $(COMPOSE_FILE) stop

start:
	docker compose -f $(COMPOSE_FILE) start

re: clean all

clean: down
	docker system prune -f

fclean: clean
	docker volume rm $$(docker volume ls -q) 2>/dev/null || true
	sudo rm -rf $(DATA_DIR)/db/*
	sudo rm -rf $(DATA_DIR)/wordpress/*

logs:
	docker compose -f $(COMPOSE_FILE) logs -f

ps:
	docker compose -f $(COMPOSE_FILE) ps

.PHONY: all setup down stop start re clean fclean logs ps
```

> **Attention :** dans un Makefile, l'indentation se fait avec des **tabulations** (`Tab`), pas des espaces.

---

## 12. Configurer le domaine local

Le sujet impose que `login.42.fr` pointe vers l'adresse IP de ta VM.

### Trouver l'IP de ta VM

```bash
# Dans ta VM
ip addr show | grep inet
```

Repère l'IP locale, par exemple `192.168.56.101`.

### Modifier le fichier `/etc/hosts` de la VM

```bash
sudo nano /etc/hosts
```

Ajoute la ligne :

```
127.0.0.1   login.42.fr
```

> On utilise `127.0.0.1` car NGINX écoute sur la VM elle-même.

### Pour accéder depuis ton PC hôte (optionnel)

Sur ton **PC hôte** (pas la VM), modifie aussi `/etc/hosts` :

```
# Linux/Mac : /etc/hosts
# Windows : C:\Windows\System32\drivers\etc\hosts
192.168.56.101   login.42.fr
```

Remplace `192.168.56.101` par l'IP réelle de ta VM.

---

## 13. Fichiers de documentation requis

Le sujet v5.4 exige trois fichiers de documentation à la racine du repo.

### README.md

```markdown
*This project has been created as part of the 42 curriculum by login.*

## Description

Inception is a system administration project that sets up a small
infrastructure using Docker and Docker Compose. The stack includes
NGINX (TLS only), WordPress with php-fpm, and MariaDB, all running
in isolated containers communicating over a private Docker network.

**Design choices:**
- Virtual Machines vs Docker: Docker provides lighter isolation...
- Secrets vs Environment Variables: secrets are mounted as files...
- Docker Network vs Host Network: bridge network is used for isolation...
- Docker Volumes vs Bind Mounts: named volumes with local driver...

## Instructions

See DEV_DOC.md for setup and DEV_DOC.md for user instructions.

## Resources

- Docker documentation: https://docs.docker.com
- MariaDB docs: https://mariadb.com/kb/en/
- WordPress CLI: https://wp-cli.org
- AI usage: used to understand php-fpm FastCGI configuration and
  TLS certificate generation syntax.
```

### USER_DOC.md

```markdown
# User Documentation — Inception

## Services provided

- **WordPress website** accessible at https://login.42.fr
- **WordPress admin panel** at https://login.42.fr/wp-admin

## Start and stop the project

```bash
# Start
make

# Stop (keeps data)
make stop

# Remove containers (keeps data)
make down
```

## Access the website

Open your browser and go to: https://login.42.fr
(Accept the self-signed certificate warning)

## Admin panel

URL: https://login.42.fr/wp-admin
Credentials are stored in secrets/credentials.txt

## Check services are running

```bash
docker compose -f srcs/docker-compose.yml ps
```

All three services (nginx, wordpress, mariadb) should show status "Up".
```

### DEV_DOC.md

```markdown
# Developer Documentation — Inception

## Prerequisites

- Debian 12 VM with Docker and Docker Compose installed
- make, git

## Setup from scratch

1. Clone the repository
2. Create the secrets files:
   - secrets/db_password.txt
   - secrets/db_root_password.txt
   - secrets/credentials.txt
3. Create the .env file (see srcs/.env.example)
4. Add `127.0.0.1 login.42.fr` to /etc/hosts

## Build and launch

```bash
make
```

This creates data directories, builds all Docker images, and starts containers.

## Useful commands

```bash
make logs        # Follow container logs
make ps          # Container status
make re          # Full rebuild
make fclean      # Remove everything including data
docker exec -it mariadb mariadb -u root -p  # Access DB
docker exec -it wordpress bash              # Shell into WP container
```

## Data persistence

- WordPress database: /home/login/data/db (Docker named volume: db_data)
- WordPress files:    /home/login/data/wordpress (Docker named volume: wp_data)
```

---

## 14. BONUS — Redis cache

Redis est un cache en mémoire. Il accélère WordPress en mettant en cache les résultats des requêtes DB.

### Dockerfile Redis

```dockerfile
# srcs/requirements/bonus/redis/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    redis-server \
    && rm -rf /var/lib/apt/lists/*

COPY conf/redis.conf /etc/redis/redis.conf

EXPOSE 6379

CMD ["redis-server", "/etc/redis/redis.conf"]
```

### Configuration Redis

```
# srcs/requirements/bonus/redis/conf/redis.conf
bind 0.0.0.0
port 6379
daemonize no
maxmemory 256mb
maxmemory-policy allkeys-lru
```

> `daemonize no` : comme pour NGINX, Redis doit rester en avant-plan.

### Connecter WordPress à Redis

Dans le script d'init WordPress, après l'installation, ajoute :

```bash
# Active le plugin Redis Object Cache
wp plugin install redis-cache --activate --path=${WORDPRESS_PATH} --allow-root
wp config set WP_REDIS_HOST redis --path=${WORDPRESS_PATH} --allow-root
wp config set WP_REDIS_PORT 6379 --raw --path=${WORDPRESS_PATH} --allow-root
wp redis enable --path=${WORDPRESS_PATH} --allow-root
```

### Ajout dans docker-compose.yml

```yaml
  redis:
    build: ./requirements/bonus/redis
    image: redis
    container_name: redis
    restart: unless-stopped
    networks:
      - inception
    volumes:
      - redis_data:/data

# Dans la section volumes :
  redis_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/login/data/redis
```

---

## 15. BONUS — Serveur FTP

Le container FTP permet de naviguer et transférer des fichiers vers le volume WordPress via le protocole FTP.

### Dockerfile FTP (vsftpd)

```dockerfile
# srcs/requirements/bonus/ftp/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    vsftpd \
    && rm -rf /var/lib/apt/lists/*

COPY conf/vsftpd.conf /etc/vsftpd.conf
COPY tools/init_ftp.sh /usr/local/bin/init_ftp.sh
RUN chmod +x /usr/local/bin/init_ftp.sh

EXPOSE 21 21100-21110

ENTRYPOINT ["/usr/local/bin/init_ftp.sh"]
```

### Configuration vsftpd

```
# srcs/requirements/bonus/ftp/conf/vsftpd.conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=21100
pasv_max_port=21110
user_sub_token=$USER
local_root=/var/www/html
```

### Script init FTP

```bash
#!/bin/bash
# srcs/requirements/bonus/ftp/tools/init_ftp.sh

FTP_PASSWORD=$(cat /run/secrets/db_password)  # ou un secret dédié au FTP

# Crée l'utilisateur FTP
useradd -m -d /var/www/html -s /bin/bash ftpuser 2>/dev/null || true
echo "ftpuser:${FTP_PASSWORD}" | chpasswd

exec vsftpd /etc/vsftpd.conf
```

### Ajout dans docker-compose.yml

```yaml
  ftp:
    build: ./requirements/bonus/ftp
    image: ftp
    container_name: ftp
    restart: unless-stopped
    networks:
      - inception
    volumes:
      - wp_data:/var/www/html
    ports:
      - "21:21"
      - "21100-21110:21100-21110"
    secrets:
      - db_password
```

---

## 16. BONUS — Site statique

Un site statique simple (HTML/CSS/JS — **pas de PHP**). Idéalement un portfolio ou CV.

### Dockerfile site statique (avec NGINX)

```dockerfile
# srcs/requirements/bonus/static/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    nginx \
    && rm -rf /var/lib/apt/lists/*

COPY site/ /var/www/static/
COPY conf/static.conf /etc/nginx/sites-available/default

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Configuration NGINX statique

```nginx
# srcs/requirements/bonus/static/conf/static.conf
server {
    listen 80;
    root /var/www/static;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Page HTML exemple

```html
<!-- srcs/requirements/bonus/static/site/index.html -->
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Portfolio — login</title>
</head>
<body>
    <h1>Bonjour, je suis login</h1>
    <p>Étudiant à 42.</p>
</body>
</html>
```

### Ajout dans docker-compose.yml

```yaml
  static:
    build: ./requirements/bonus/static
    image: static
    container_name: static
    restart: unless-stopped
    networks:
      - inception
    # Pas de port exposé directement : NGINX principal peut proxy vers ce service
```

Et dans la config NGINX principale, ajoute un bloc `location` :

```nginx
location /static/ {
    proxy_pass http://static:80/;
}
```

---

## 17. BONUS — Adminer

Adminer est une interface web légère pour gérer MariaDB (alternative à phpMyAdmin).

### Dockerfile Adminer

```dockerfile
# srcs/requirements/bonus/adminer/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y \
    php8.2-fpm \
    php8.2-mysql \
    wget \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/www/adminer && \
    wget -q https://www.adminer.org/latest.php -O /var/www/adminer/index.php

COPY conf/www.conf /etc/php/8.2/fpm/pool.d/www.conf

RUN mkdir -p /run/php

EXPOSE 8080

CMD ["php-fpm8.2", "-F"]
```

> Tu dois ensuite ajouter un bloc server dans NGINX pour servir Adminer (via FastCGI comme WordPress), ou utiliser PHP built-in server.

### Ajout dans docker-compose.yml

```yaml
  adminer:
    build: ./requirements/bonus/adminer
    image: adminer
    container_name: adminer
    restart: unless-stopped
    networks:
      - inception
    depends_on:
      - mariadb
```

Dans la config NGINX principale, ajoute :

```nginx
server {
    listen 443 ssl;
    server_name login.42.fr;
    # ...SSL config identique...

    location /adminer/ {
        fastcgi_pass adminer:9000;
        fastcgi_param SCRIPT_FILENAME /var/www/adminer/index.php;
        include fastcgi_params;
    }
}
```

---

## 18. BONUS — Service au choix

Le sujet demande un **service supplémentaire utile** que tu devras justifier à l'oral. Quelques idées :

| Service | Utilité | Complexité |
|---|---|---|
| **Portainer** | Interface web de gestion Docker | Facile |
| **Fail2ban** | Protection contre les attaques brute-force | Moyen |
| **Netdata** | Monitoring système en temps réel | Facile |
| **Prometheus + Grafana** | Monitoring avancé avec graphiques | Difficile |
| **Mailhog** | Intercepte les emails WordPress en dev | Facile |

### Exemple avec Portainer

```dockerfile
# srcs/requirements/bonus/portainer/Dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y wget && \
    wget -q https://github.com/portainer/portainer/releases/download/2.19.4/portainer-2.19.4-linux-amd64.tar.gz \
    -O /tmp/portainer.tar.gz && \
    tar xzf /tmp/portainer.tar.gz -C /opt && \
    rm /tmp/portainer.tar.gz && \
    apt-get remove -y wget && rm -rf /var/lib/apt/lists/*

EXPOSE 9443

CMD ["/opt/portainer/portainer", "--https-addr", ":9443"]
```

---

## 19. Checklist finale avant soutenance

### Structure

- [ ] `Makefile` à la racine
- [ ] `README.md` à la racine (en anglais)
- [ ] `USER_DOC.md` à la racine
- [ ] `DEV_DOC.md` à la racine
- [ ] Tous les fichiers de config dans `srcs/`
- [ ] `secrets/` **non commité** dans Git

### Containers obligatoires

- [ ] NGINX écoute uniquement sur le port 443 en TLSv1.2 ou 1.3
- [ ] WordPress + php-fpm (sans NGINX dans le même container)
- [ ] MariaDB (sans NGINX dans le même container)
- [ ] 2 volumes nommés (db_data, wp_data)
- [ ] 1 réseau Docker bridge
- [ ] Tous les containers redémarrent automatiquement

### WordPress

- [ ] 2 utilisateurs dans la DB WordPress
- [ ] L'admin **ne s'appelle pas** `admin`, `Admin`, `administrator`, `Administrator`, etc.
- [ ] Le site est accessible via `https://login.42.fr`

### Sécurité

- [ ] Aucun mot de passe dans les Dockerfiles
- [ ] Aucun mot de passe dans le repo Git
- [ ] Variables d'environnement utilisées
- [ ] Docker secrets utilisés pour les credentials
- [ ] `.env` dans le `.gitignore`

### Interdictions

- [ ] Pas de `tail -f`, `sleep infinity`, `while true`, `bash` comme entrypoint
- [ ] Pas de `network: host` ni `--link` ni `links:`
- [ ] Pas d'images toutes faites (sauf Alpine/Debian de base)
- [ ] Pas du tag `latest`
- [ ] Pas de bind mounts directs pour les deux volumes principaux

### Test final

```bash
# Lance tout depuis zéro
make fclean
make

# Vérifie que les containers tournent
docker compose -f srcs/docker-compose.yml ps

# Vérifie les logs
docker compose -f srcs/docker-compose.yml logs

# Teste NGINX
curl -k https://login.42.fr

# Vérifie que TLS 1.0 et 1.1 sont refusés
openssl s_client -connect login.42.fr:443 -tls1   # doit échouer
openssl s_client -connect login.42.fr:443 -tls1_2 # doit réussir

# Vérifie la DB
docker exec -it mariadb mariadb -u root -p -e "SHOW DATABASES;"
```

---

*Ce guide couvre la version 5.4 du sujet Inception (42). Remplace `login` par ton propre identifiant 42 partout.*
