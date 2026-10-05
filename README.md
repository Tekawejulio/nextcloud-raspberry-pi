# Nextcloud sur Raspberry Pi (Docker, architecture modulaire)

Déploiement d'un serveur [Nextcloud](https://nextcloud.com/) personnel sur Raspberry Pi 5, avec une architecture multi-conteneurs (application + base de données + reverse proxy HTTPS), plutôt que la version "tout-en-un" (AIO).

## Stack

- **Matériel** : Raspberry Pi 5 (Debian/Raspberry Pi OS, arm64)
- **Conteneurisation** : Docker + Docker Compose
- **Application** : Nextcloud (image officielle `nextcloud:apache`)
- **Base de données** : MariaDB 11
- **Reverse proxy / HTTPS** : Nginx (certificat auto-signé)

## Pourquoi une architecture modulaire plutôt que Nextcloud AIO ?

Une première tentative avec **Nextcloud AIO** (le pack tout-en-un officiel) a été abandonnée : sur un Raspberry Pi avec seulement une carte microSD (pas de stockage externe), l'empreinte de plusieurs conteneurs lancés automatiquement par AIO était disproportionnée par rapport au besoin réel.

Le choix s'est porté sur une **architecture modulaire** : un conteneur Nextcloud, un conteneur MariaDB séparé, et un reverse proxy Nginx pour le HTTPS — plus légère, et surtout plus formatrice : chaque brique est comprise et configurée individuellement plutôt que masquée derrière un installeur automatique.

## Éviter les conflits de ports

Le Raspberry Pi héberge déjà plusieurs services ([Pi-hole](https://github.com/Tekawejulio/pihole-raspberry-pi) en `network_mode: host` sur les ports 53/80/443, Jellyfin sur 8096, Netdata sur 19999). Avant tout déploiement, les ports libres ont été vérifiés avec :

```bash
sudo ss -tulnp | grep LISTEN
```

Nextcloud a été configuré sur le port **8081** (HTTP interne) et **8443** (HTTPS via le proxy), en évitant tout chevauchement.

## Déploiement

Fichier `docker-compose.yml` :

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'CHANGE_MOI_ROOT'
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: 'CHANGE_MOI_DB'
    volumes:
      - ./db-data:/var/lib/mysql
      - ./mariadb-custom.cnf:/etc/mysql/conf.d/custom.cnf:ro

  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: 'CHANGE_MOI_DB'
    volumes:
      - ./nc-data:/var/www/html
    depends_on:
      - db

  proxy:
    image: nginx:alpine
    container_name: nextcloud-proxy
    restart: unless-stopped
    ports:
      - "8443:443"
    volumes:
      - ./nginx-proxy.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
```

Lancement :
```bash
sudo docker compose up -d
```

## Optimisation de la base de données

Après l'installation, la page **Paramètres d'administration → Vue d'ensemble** de Nextcloud a signalé plusieurs avertissements de performance sur MariaDB :

| Vérification | Valeur initiale | Valeur recommandée |
|---|---|---|
| `slow_query_log` | désactivé | activé |
| `long_query_time` | 10s | 2s |
| `innodb_log_file_size` | 96 Mo | 256 Mo |
| `table_open_cache` | insuffisant | 4000 |

Ces réglages ont été appliqués via un fichier de configuration MariaDB personnalisé, monté dans le conteneur :

```ini
# mariadb-custom.cnf
[mariadb]
slow_query_log = 1
long_query_time = 2
innodb_log_file_size = 256M
table_open_cache = 4000
```

### ⚠️ Point d'attention : `docker compose up -d` ne recharge pas un fichier monté

Modifier `mariadb-custom.cnf` puis relancer `docker compose up -d` ne suffit pas : Compose ne redémarre un conteneur que si sa définition dans `docker-compose.yml` change, pas le contenu d'un fichier qu'il monte. Un `docker compose restart db` explicite est nécessaire pour que MariaDB relise sa configuration.

Résultat : les vérifications de base de données sont passées de **4 échecs à 1 seul** — le dernier (« Replica lag ») est un faux positif connu de Nextcloud, qui teste un système de réplication non configuré et non nécessaire ici.

## Mise en place du HTTPS (reverse proxy)

Un domaine public et un certificat Let's Encrypt n'étant pas pertinents pour un accès purement local, un **certificat auto-signé** a été généré :

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nextcloud.key -out nextcloud.crt \
  -subj "/CN=192.168.129.200"
```

Un conteneur **Nginx** fait office de reverse proxy, terminant le TLS et redirigeant vers le conteneur Nextcloud :

```nginx
server {
    listen 443 ssl;
    server_name 192.168.129.200;

    ssl_certificate     /etc/nginx/certs/nextcloud.crt;
    ssl_certificate_key /etc/nginx/certs/nextcloud.key;

    client_max_body_size 512M;

    location / {
        proxy_pass http://app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### ⚠️ Problème rencontré : boucle de redirection vers le mauvais port

**Symptôme** : après connexion via `https://192.168.129.200:8443`, Nextcloud redirigeait systématiquement vers `http://localhost/login` — perdant à la fois le HTTPS et le port.

**Diagnostic** : un test direct avec `curl -Ik https://localhost:8443` a révélé l'en-tête `Location: http://localhost/login` dans la réponse — Nextcloud générait ses URLs sans savoir qu'il était derrière un proxy, se basant uniquement sur le port interne (80) qu'il voit depuis son propre conteneur.

**Solution** : indiquer explicitement à Nextcloud son adresse et son protocole publics, et faire confiance au réseau Docker interne comme proxy :
```bash
php occ config:system:set overwriteprotocol --value="https"
php occ config:system:set overwritehost --value="192.168.129.200:8443"
php occ config:system:set overwrite.cli.url --value="https://192.168.129.200:8443"
php occ config:system:set trusted_proxies 0 --value="172.20.0.0/16"
```

Un second blocage — **"Accès à partir d'un domaine non approuvé"** — a nécessité l'ajout du domaine à la liste blanche de Nextcloud :
```bash
php occ config:system:set trusted_domains 1 --value="192.168.129.200"
```

Après ces deux correctifs, l'accès HTTPS complet fonctionne, tableau de bord compris.

## Applications complémentaires

Les applications Calendrier, Contacts et Mail ont échoué au téléchargement lors de l'assistant d'installation web (souci ponctuel, probablement lié au moment du démarrage). Un test en ligne de commande a confirmé qu'il n'y avait pas de blocage réseau réel, et l'installation a réussi sans difficulté :
```bash
php occ app:install calendar
php occ app:install contacts
php occ app:install mail
```

## Compétences mises en œuvre

- Architecture multi-conteneurs (application, base de données, reverse proxy)
- Administration MariaDB (réglages de performance, diagnostic via les outils d'administration Nextcloud)
- Mise en place de HTTPS avec un reverse proxy Nginx et un certificat auto-signé
- Diagnostic de redirections HTTP incorrectes (`curl -I`, en-têtes `Location`)
- Gestion de la configuration Nextcloud en ligne de commande (`occ`)
- Cohabitation de plusieurs services Docker sur un même hôte sans conflit de ports


## Aperçu

![Tableau de bord Nextcloud](Capture%20d'écran%202026-10-05%20174504.png)

## Auteur

Julio — [GitHub](https://github.com/Tekawejulio)
