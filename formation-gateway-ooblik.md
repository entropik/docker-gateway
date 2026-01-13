# Formation : Infrastructure Gateway Sécurisée avec Docker

## Guide complet de mise en place d'un reverse proxy Traefik avec authentification 2FA et protection anti-intrusion

**Auteur :** OOBLIK  
**Date :** Janvier 2026  
**Durée estimée :** 4-6 heures  
**Niveau :** Intermédiaire

---

## 📋 Table des matières

1. [Introduction et objectifs](#1-introduction-et-objectifs)
2. [Prérequis](#2-prérequis)
3. [Architecture cible](#3-architecture-cible)
4. [Phase 1 : Installation de Traefik](#4-phase-1--installation-de-traefik)
5. [Phase 2 : Exposition du premier service (Immich)](#5-phase-2--exposition-du-premier-service-immich)
6. [Phase 3 : Sécurisation avec Authelia](#6-phase-3--sécurisation-avec-authelia)
7. [Phase 3 bis : Protection avec CrowdSec](#7-phase-3-bis--protection-avec-crowdsec)
8. [Phase 4 : Dashboard Heimdall](#8-phase-4--dashboard-heimdall)
9. [Phase 5 : Supervision avec Dockge et Diun](#9-phase-5--supervision-avec-dockge-et-diun)
10. [Phase 6 : Sauvegardes avec Duplicati](#10-phase-6--sauvegardes-avec-duplicati)
11. [Récapitulatif et fichiers de configuration](#11-récapitulatif-et-fichiers-de-configuration)
12. [Dépannage](#12-dépannage)
13. [Commandes utiles](#13-commandes-utiles)

---

## 1. Introduction et objectifs

### Qu'allons-nous construire ?

Une infrastructure complète permettant d'exposer des services auto-hébergés sur Internet de manière sécurisée :

- **Reverse proxy** : Point d'entrée unique pour tous les services
- **Certificats SSL automatiques** : HTTPS via Let's Encrypt
- **Authentification 2FA** : Protection par login + code TOTP
- **Protection anti-intrusion** : Blocage automatique des IP malveillantes
- **Dashboard centralisé** : Vue d'ensemble de tous les services
- **Gestion Docker visuelle** : Interface web pour gérer les conteneurs
- **Notifications de mises à jour** : Alertes quand des images Docker sont obsolètes
- **Sauvegardes** : Protection des configurations

### Pourquoi cette architecture ?

| Problème | Solution |
|----------|----------|
| Ports multiples à retenir (8080, 8443, 2283...) | Un seul point d'entrée sur ports 80/443 |
| Certificats SSL manuels | Génération et renouvellement automatique |
| Services exposés sans protection | Authentification centralisée 2FA |
| Attaques bruteforce | Détection et blocage automatique |
| Gestion Docker en ligne de commande | Interface web intuitive |

---

## 2. Prérequis

### Matériel

- **NAS Synology** (ou serveur Linux) avec Docker installé
- **IP publique fixe** (ou DynDNS)
- **Box internet** avec accès à la configuration des ports

### Logiciels

- Docker et Docker Compose
- Accès SSH au serveur
- Éditeur de texte (VSCode recommandé avec extension Remote-SSH)

### Réseau

- **Nom de domaine** avec accès à la configuration DNS
- **Ports 80 et 443** disponibles sur la box internet
- Connaissance de l'IP locale du serveur

### Notre environnement de référence

| Élément | Valeur |
|---------|--------|
| Serveur gateway | Synology Stock3 - 192.168.1.200 |
| Serveur Immich | Synology Stock2 - 192.168.1.120 |
| IP publique | 45.80.33.101 (Free Pro) |
| Domaine | ooblik.com (hébergé chez o2switch) |
| Interface réseau | bond0 (agrégation eth0+eth1) |

---

## 3. Architecture cible

### Schéma de flux

```
┌─────────────────────────────────────────────────────────────────────┐
│                           INTERNET                                   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FREEBOX (Routeur/Firewall)                        │
│                                                                       │
│   Port 80  ──────────────────────► 192.168.1.200:8080                │
│   Port 443 ──────────────────────► 192.168.1.200:8443                │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         STOCK3 (Gateway)                             │
│                         192.168.1.200                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      TRAEFIK                                 │    │
│  │                  (Reverse Proxy)                             │    │
│  │              Ports 8080:80 / 8443:443                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│           │              │              │              │             │
│           ▼              ▼              ▼              ▼             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │  AUTHELIA   │ │  CROWDSEC   │ │  HEIMDALL   │ │   DOCKGE    │   │
│  │    2FA      │ │  Anti-bot   │ │  Dashboard  │ │   Docker    │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                                       │
│  ┌─────────────┐ ┌─────────────┐                                    │
│  │    DIUN     │ │  DUPLICATI  │                                    │
│  │   Updates   │ │   Backup    │                                    │
│  └─────────────┘ └─────────────┘                                    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Réseau local
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         STOCK2 (Services)                            │
│                         192.168.1.120                                │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      IMMICH                                  │    │
│  │                  (Photos/Vidéos)                             │    │
│  │                    Port 2283                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Services et URLs finales

| Service | URL | Fonction | Protection |
|---------|-----|----------|------------|
| Immich | photos.ooblik.com | Galerie photos/vidéos | CrowdSec |
| Authelia | auth.ooblik.com | Portail d'authentification | — |
| Traefik | trafik.ooblik.com | Dashboard reverse proxy | Authelia + CrowdSec |
| Heimdall | kalon.ooblik.com | Dashboard applications | Authelia + CrowdSec |
| Dockge | dockge.ooblik.com | Gestion Docker | Authelia + CrowdSec |
| Duplicati | backup.ooblik.com | Sauvegardes | Authelia + CrowdSec |

### Structure des fichiers

```
/volume1/docker/gateway/
├── docker-compose.yml          # Configuration principale
├── letsencrypt/
│   └── acme.json              # Certificats SSL (chmod 600)
├── config/
│   └── dynamic/
│       ├── immich.yml         # Route vers Immich
│       ├── traefik-dashboard.yml  # Route dashboard Traefik
│       └── crowdsec.yml       # Middleware CrowdSec
├── authelia/
│   ├── configuration.yml      # Config Authelia
│   └── users.yml              # Utilisateurs
├── crowdsec/
│   ├── config/
│   └── data/
├── heimdall/
├── dockge/
├── duplicati/
├── diun/
└── logs/
    └── access.log             # Logs Traefik
```

---

## 4. Phase 1 : Installation de Traefik

### 4.1 Comprendre le problème des ports

Sur un NAS Synology, les ports 80 et 443 sont déjà utilisés par le système (Web Station, DSM). On ne peut pas les libérer sans casser des fonctionnalités.

**Solution** : Traefik écoute sur des ports alternatifs (8080/8443) et la box redirige les ports publics 80/443 vers ces ports.

```
Internet:80  ──► Freebox ──► 192.168.1.200:8080 (Traefik)
Internet:443 ──► Freebox ──► 192.168.1.200:8443 (Traefik)
```

### 4.2 Configuration DNS

Chez votre registrar DNS, créez un enregistrement A pour chaque sous-domaine :

| Type | Nom | Valeur | TTL |
|------|-----|--------|-----|
| A | test | 45.80.33.101 | 14400 |
| A | photos | 45.80.33.101 | 14400 |
| A | auth | 45.80.33.101 | 14400 |
| A | trafik | 45.80.33.101 | 14400 |
| A | kalon | 45.80.33.101 | 14400 |
| A | dockge | 45.80.33.101 | 14400 |
| A | backup | 45.80.33.101 | 14400 |

> **Note sur le TTL** : Time To Live en secondes. 14400 = 4 heures. Un TTL court (3600 = 1h) permet des changements DNS plus rapides mais génère plus de requêtes.

### 4.3 Configuration du routeur (Freebox Pro)

Dans l'interface de la Freebox, section **Gestion des ports** :

| Port externe | Protocole | IP destination | Port interne |
|--------------|-----------|----------------|--------------|
| 80 | TCP | 192.168.1.200 | 8080 |
| 443 | TCP | 192.168.1.200 | 8443 |

### 4.4 Création de la structure

```bash
# Connexion SSH au NAS
ssh utilisateur@192.168.1.200

# Création des dossiers
mkdir -p /volume1/docker/gateway
mkdir -p /volume1/docker/gateway/letsencrypt
mkdir -p /volume1/docker/gateway/config/dynamic
mkdir -p /volume1/docker/gateway/logs

# Création du fichier pour les certificats
touch /volume1/docker/gateway/letsencrypt/acme.json
chmod 600 /volume1/docker/gateway/letsencrypt/acme.json
```

### 4.5 Premier docker-compose.yml

Créez le fichier `/volume1/docker/gateway/docker-compose.yml` :

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    command:
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.file.directory=/etc/traefik/dynamic"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=votre@email.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--accesslog=true"
      - "--accesslog.filepath=/var/log/traefik/access.log"
      - "--log.level=INFO"
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
      - ./config:/etc/traefik
      - ./logs:/var/log/traefik
    networks:
      - proxy

networks:
  proxy:
    driver: bridge
```

### 4.6 Explication des paramètres Traefik

| Paramètre | Fonction |
|-----------|----------|
| `api.dashboard=true` | Active le tableau de bord web |
| `providers.docker=true` | Détecte automatiquement les conteneurs Docker |
| `providers.docker.exposedbydefault=false` | N'expose pas les conteneurs sauf demande explicite |
| `providers.file.directory` | Dossier pour les configurations dynamiques (services externes) |
| `entrypoints.web.address=:80` | Point d'entrée HTTP |
| `entrypoints.websecure.address=:443` | Point d'entrée HTTPS |
| `certificatesresolvers.letsencrypt...` | Configuration Let's Encrypt |
| `accesslog` | Active les logs d'accès |

### 4.7 Lancement et vérification

```bash
cd /volume1/docker/gateway
sudo docker-compose up -d

# Vérifier que le conteneur tourne
sudo docker ps

# Vérifier les logs
sudo docker logs traefik
```

**Résultat attendu** : Traefik démarre sans erreur.

---

## 5. Phase 2 : Exposition du premier service (Immich)

### 5.1 Concept des fichiers dynamiques

Traefik peut router vers des services qui ne sont pas dans le même docker-compose grâce aux **fichiers de configuration dynamiques**.

Pour Immich qui tourne sur Stock2 (192.168.1.120:2283), on crée un fichier YAML qui décrit comment y accéder.

### 5.2 Création du fichier de route

Créez `/volume1/docker/gateway/config/dynamic/immich.yml` :

```yaml
http:
  routers:
    immich:
      rule: "Host(`photos.ooblik.com`)"
      entryPoints:
        - websecure
      service: immich
      tls:
        certResolver: letsencrypt

  services:
    immich:
      loadBalancer:
        servers:
          - url: "http://192.168.1.120:2283"
```

### 5.3 Explication de la configuration

```yaml
routers:
  immich:                              # Nom du routeur (arbitraire)
    rule: "Host(`photos.ooblik.com`)"  # Condition : si le domaine correspond
    entryPoints:
      - websecure                      # Utilise HTTPS (port 443)
    service: immich                    # Vers quel service router
    tls:
      certResolver: letsencrypt        # Génère un certificat automatiquement

services:
  immich:                              # Nom du service (doit correspondre au routeur)
    loadBalancer:
      servers:
        - url: "http://192.168.1.120:2283"  # Adresse réelle du service
```

### 5.4 Test

Traefik recharge automatiquement les fichiers dynamiques. Testez dans votre navigateur :

**https://photos.ooblik.com**

Vous devriez voir :
- ✅ Connexion HTTPS sécurisée (cadenas vert)
- ✅ Page de login Immich

> **Note** : La première fois, Let's Encrypt doit valider le domaine. Attendez quelques secondes si vous voyez une erreur de certificat.

---

## 6. Phase 3 : Sécurisation avec Authelia

### 6.1 Qu'est-ce qu'Authelia ?

Authelia est un serveur d'authentification qui se place devant vos services. Il fournit :

- **Single Sign-On (SSO)** : Une seule connexion pour tous les services
- **Authentification 2FA** : Code TOTP (Google Authenticator, Authy...)
- **Contrôle d'accès** : Règles par domaine, utilisateur, groupe

### 6.2 Flux d'authentification

```
1. Utilisateur ──► https://dockge.ooblik.com
2. Traefik intercepte ──► Vérifie avec Authelia "Est-il connecté ?"
3. Authelia répond "Non" ──► Redirection vers https://auth.ooblik.com
4. Utilisateur entre login + mot de passe + code 2FA
5. Authelia valide ──► Redirection vers https://dockge.ooblik.com
6. Traefik vérifie ──► Authelia dit "OK" ──► Accès autorisé
```

### 6.3 Génération des secrets

Authelia nécessite plusieurs secrets cryptographiques :

```bash
# Générer 3 secrets
echo "JWT: $(openssl rand -base64 32)"
echo "Session: $(openssl rand -base64 32)"
echo "Storage: $(openssl rand -base64 32)"
```

**Notez ces valeurs !** Elles seront utilisées dans la configuration.

### 6.4 Création du fichier de configuration Authelia

Créez le dossier et le fichier :

```bash
mkdir -p /volume1/docker/gateway/authelia
```

Créez `/volume1/docker/gateway/authelia/configuration.yml` :

```yaml
server:
  address: 'tcp://0.0.0.0:9091'

log:
  level: info

jwt_secret: 'VOTRE_SECRET_JWT_ICI'

default_redirection_url: 'https://auth.ooblik.com'

totp:
  issuer: ooblik.com
  period: 30
  skew: 2

authentication_backend:
  file:
    path: /config/users.yml
    password:
      algorithm: bcrypt
      iterations: 12

access_control:
  default_policy: deny
  rules:
    - domain: photos.ooblik.com
      policy: bypass
    - domain: '*.ooblik.com'
      policy: two_factor

session:
  name: authelia_session
  secret: 'VOTRE_SECRET_SESSION_ICI'
  expiration: 86400
  inactivity: 3600
  domain: ooblik.com

regulation:
  max_retries: 3
  find_time: 120
  ban_time: 300

storage:
  encryption_key: 'VOTRE_SECRET_STORAGE_ICI'
  local:
    path: /config/db.sqlite3

notifier:
  filesystem:
    filename: /config/notification.txt
```

### 6.5 Explication des sections

| Section | Fonction |
|---------|----------|
| `server` | Port d'écoute interne |
| `jwt_secret` | Clé pour signer les tokens JWT |
| `totp` | Configuration de l'authentification 2FA |
| `authentication_backend` | Source des utilisateurs (fichier local) |
| `access_control` | Règles d'accès par domaine |
| `session` | Durée et paramètres des sessions |
| `regulation` | Protection anti-bruteforce |
| `storage` | Base de données (SQLite) |
| `notifier` | Mode de notification (fichier local) |

### 6.6 Politiques d'accès

| Policy | Signification |
|--------|---------------|
| `bypass` | Accès sans authentification |
| `one_factor` | Login + mot de passe uniquement |
| `two_factor` | Login + mot de passe + code 2FA |
| `deny` | Accès refusé |

### 6.7 Création des utilisateurs

Générez d'abord le hash du mot de passe :

```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'VotreMotDePasseSecurise'
```

Créez `/volume1/docker/gateway/authelia/users.yml` :

```yaml
users:
  marc:
    displayname: "Marc Tallec"
    password: "$argon2id$v=19$m=65536,t=3,p=4$HASH_GENERE_CI_DESSUS"
    email: votre@email.com
    groups:
      - admins
```

### 6.8 Ajout d'Authelia au docker-compose

Ajoutez ce service dans votre `docker-compose.yml` :

```yaml
  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    volumes:
      - ./authelia:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.authelia.rule=Host(`auth.ooblik.com`)"
      - "traefik.http.routers.authelia.entrypoints=websecure"
      - "traefik.http.routers.authelia.tls.certresolver=letsencrypt"
      - "traefik.http.services.authelia.loadbalancer.server.port=9091"
      - "traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.ooblik.com"
      - "traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true"
      - "traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email"
    networks:
      - proxy
```

### 6.9 Création du middleware pour les fichiers dynamiques

Le middleware défini dans les labels Docker s'appelle `authelia@docker`. Pour les services configurés via fichiers (comme le dashboard Traefik), on doit créer un middleware dans un fichier.

Modifiez `/volume1/docker/gateway/config/dynamic/traefik-dashboard.yml` :

```yaml
http:
  routers:
    traefik-dashboard:
      rule: "Host(`trafik.ooblik.com`)"
      entryPoints:
        - websecure
      service: api@internal
      tls:
        certResolver: letsencrypt
      middlewares:
        - authelia
        - crowdsec

  middlewares:
    authelia:
      forwardAuth:
        address: "http://authelia:9091/api/verify?rd=https://auth.ooblik.com"
        trustForwardHeader: true
        authResponseHeaders:
          - Remote-User
          - Remote-Groups
          - Remote-Name
          - Remote-Email
```

### 6.10 Test de l'authentification

```bash
cd /volume1/docker/gateway
sudo docker-compose down
sudo docker-compose up -d
```

1. Allez sur **https://auth.ooblik.com**
2. Connectez-vous avec vos identifiants
3. Configurez le 2FA en scannant le QR code
4. Testez l'accès à **https://trafik.ooblik.com**

---

## 7. Phase 3 bis : Protection avec CrowdSec

### 7.1 Qu'est-ce que CrowdSec ?

CrowdSec est un système de détection d'intrusion collaboratif :

- **Analyse les logs** pour détecter les comportements malveillants
- **Bloque automatiquement** les IP suspectes
- **Partage les menaces** avec la communauté (et bénéficie des signalements)
- **Gratuit et open-source**

### 7.2 Architecture CrowdSec

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│     TRAEFIK      │────►│    CROWDSEC      │────►│  CROWDSEC API    │
│   (génère logs)  │     │ (analyse logs)   │     │   (communauté)   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │    BOUNCER       │
                         │ (bloque les IP)  │
                         └──────────────────┘
```

### 7.3 Ajout de CrowdSec au docker-compose

```yaml
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - COLLECTIONS=crowdsecurity/traefik crowdsecurity/http-cve
    volumes:
      - ./crowdsec/config:/etc/crowdsec
      - ./crowdsec/data:/var/lib/crowdsec/data
      - ./logs:/var/log/traefik:ro
    networks:
      - proxy

  crowdsec-bouncer:
    image: fbonalair/traefik-crowdsec-bouncer:latest
    container_name: crowdsec-bouncer
    restart: unless-stopped
    environment:
      - CROWDSEC_BOUNCER_API_KEY=CLE_A_GENERER
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    networks:
      - proxy
```

### 7.4 Génération de la clé API pour le bouncer

```bash
# Démarrer d'abord CrowdSec seul
sudo docker-compose up -d crowdsec

# Attendre 30 secondes puis générer la clé
sudo docker exec crowdsec cscli bouncers add traefik-bouncer
```

Copiez la clé affichée et remplacez `CLE_A_GENERER` dans le docker-compose.

### 7.5 Création du middleware CrowdSec

Créez `/volume1/docker/gateway/config/dynamic/crowdsec.yml` :

```yaml
http:
  middlewares:
    crowdsec:
      forwardAuth:
        address: "http://crowdsec-bouncer:8080/api/v1/forwardAuth"
        trustForwardHeader: true
```

### 7.6 Application du middleware

Pour protéger un service avec CrowdSec, ajoutez le middleware dans sa configuration.

**Pour les services Docker (labels)** :
```yaml
labels:
  - "traefik.http.routers.monservice.middlewares=authelia@file,crowdsec@file"
```

**Pour les services externes (fichiers dynamiques)** :
```yaml
routers:
  monservice:
    middlewares:
      - crowdsec
```

### 7.7 Commandes utiles CrowdSec

```bash
# Voir les IP bannies
sudo docker exec crowdsec cscli decisions list

# Voir les alertes
sudo docker exec crowdsec cscli alerts list

# Bannir une IP manuellement
sudo docker exec crowdsec cscli decisions add --ip 1.2.3.4 --reason "Test manuel"

# Débannir une IP
sudo docker exec crowdsec cscli decisions delete --ip 1.2.3.4
```

---

## 8. Phase 4 : Dashboard Heimdall

### 8.1 Présentation

Heimdall est un dashboard permettant d'organiser et d'accéder rapidement à tous vos services. Il supporte :

- Tuiles personnalisables avec icônes
- Catégories
- Recherche
- API pour certains services (affichage de stats)

### 8.2 Ajout au docker-compose

```yaml
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    volumes:
      - ./heimdall:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.heimdall.rule=Host(`kalon.ooblik.com`)"
      - "traefik.http.routers.heimdall.entrypoints=websecure"
      - "traefik.http.routers.heimdall.tls.certresolver=letsencrypt"
      - "traefik.http.routers.heimdall.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.heimdall.loadbalancer.server.port=80"
    networks:
      - proxy
```

### 8.3 Configuration

1. Accédez à **https://kalon.ooblik.com** (après authentification Authelia)
2. Cliquez sur "Add an application here"
3. Remplissez :
   - **Application Type** : Immich (ou Generic)
   - **Application Name** : Immich
   - **URL** : https://photos.ooblik.com
4. Sauvegardez

Répétez pour chaque service.

---

## 9. Phase 5 : Supervision avec Dockge et Diun

### 9.1 Dockge - Gestion Docker visuelle

Dockge est une alternative légère à Portainer pour gérer vos stacks Docker :

```yaml
  dockge:
    image: louislam/dockge:latest
    container_name: dockge
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - DOCKGE_STACKS_DIR=/opt/stacks
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./dockge:/app/data
      - /volume1/docker:/opt/stacks
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dockge.rule=Host(`dockge.ooblik.com`)"
      - "traefik.http.routers.dockge.entrypoints=websecure"
      - "traefik.http.routers.dockge.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dockge.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.dockge.loadbalancer.server.port=5001"
    networks:
      - proxy
```

### 9.2 Diun - Notifications de mises à jour

Diun surveille vos images Docker et vous notifie quand une mise à jour est disponible :

```yaml
  diun:
    image: crazymax/diun:latest
    container_name: diun
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - DIUN_WATCH_WORKERS=20
      - DIUN_WATCH_SCHEDULE=0 */6 * * *
      - DIUN_PROVIDERS_DOCKER=true
      - DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT=true
      - DIUN_NOTIF_MAIL_HOST=smtp.votre-provider.com
      - DIUN_NOTIF_MAIL_PORT=587
      - DIUN_NOTIF_MAIL_SSL=false
      - DIUN_NOTIF_MAIL_USERNAME=votre_email
      - DIUN_NOTIF_MAIL_PASSWORD=votre_mot_de_passe
      - DIUN_NOTIF_MAIL_FROM=diun@ooblik.com
      - DIUN_NOTIF_MAIL_TO=votre@email.com
    volumes:
      - ./diun:/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy
```

---

## 10. Phase 6 : Sauvegardes avec Duplicati

### 10.1 Présentation

Duplicati permet de sauvegarder vos fichiers vers différentes destinations :
- Stockage local/NAS
- Cloud (S3, Backblaze, Google Drive...)
- SFTP/FTP

### 10.2 Ajout au docker-compose

```yaml
  duplicati:
    image: lscr.io/linuxserver/duplicati:latest
    container_name: duplicati
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - SETTINGS_ENCRYPTION_KEY=VotreCleDeChiffrementSecrete
      - CLI_ARGS=--webservice-password=VotreMotDePasseDuplicati
    volumes:
      - ./duplicati:/config
      - /volume1/docker:/source:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.duplicati.rule=Host(`backup.ooblik.com`)"
      - "traefik.http.routers.duplicati.entrypoints=websecure"
      - "traefik.http.routers.duplicati.tls.certresolver=letsencrypt"
      - "traefik.http.routers.duplicati.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.duplicati.loadbalancer.server.port=8200"
    networks:
      - proxy
```

### 10.3 Configuration recommandée

1. Créez un job de sauvegarde pour `/source` (contient `/volume1/docker`)
2. Destination suggérée : autre NAS, stockage cloud, ou disque externe
3. Planification : quotidienne ou hebdomadaire
4. Rétention : garder plusieurs versions

---

## 11. Récapitulatif et fichiers de configuration

### 11.1 docker-compose.yml complet

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    command:
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.file.directory=/etc/traefik/dynamic"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=atelier@ooblik.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--accesslog=true"
      - "--accesslog.filepath=/var/log/traefik/access.log"
      - "--log.level=INFO"
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
      - ./config:/etc/traefik
      - ./logs:/var/log/traefik
    networks:
      - proxy

  authelia:
    image: authelia/authelia:latest
    container_name: authelia
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    volumes:
      - ./authelia:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.authelia.rule=Host(`auth.ooblik.com`)"
      - "traefik.http.routers.authelia.entrypoints=websecure"
      - "traefik.http.routers.authelia.tls.certresolver=letsencrypt"
      - "traefik.http.services.authelia.loadbalancer.server.port=9091"
      - "traefik.http.middlewares.authelia.forwardauth.address=http://authelia:9091/api/verify?rd=https://auth.ooblik.com"
      - "traefik.http.middlewares.authelia.forwardauth.trustForwardHeader=true"
      - "traefik.http.middlewares.authelia.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Name,Remote-Email"
    networks:
      - proxy

  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - COLLECTIONS=crowdsecurity/traefik crowdsecurity/http-cve
    volumes:
      - ./crowdsec/config:/etc/crowdsec
      - ./crowdsec/data:/var/lib/crowdsec/data
      - ./logs:/var/log/traefik:ro
    networks:
      - proxy

  crowdsec-bouncer:
    image: fbonalair/traefik-crowdsec-bouncer:latest
    container_name: crowdsec-bouncer
    restart: unless-stopped
    environment:
      - CROWDSEC_BOUNCER_API_KEY=VOTRE_CLE_API
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    networks:
      - proxy

  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    volumes:
      - ./heimdall:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.heimdall.rule=Host(`kalon.ooblik.com`)"
      - "traefik.http.routers.heimdall.entrypoints=websecure"
      - "traefik.http.routers.heimdall.tls.certresolver=letsencrypt"
      - "traefik.http.routers.heimdall.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.heimdall.loadbalancer.server.port=80"
    networks:
      - proxy

  dockge:
    image: louislam/dockge:latest
    container_name: dockge
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - DOCKGE_STACKS_DIR=/opt/stacks
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./dockge:/app/data
      - /volume1/docker:/opt/stacks
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dockge.rule=Host(`dockge.ooblik.com`)"
      - "traefik.http.routers.dockge.entrypoints=websecure"
      - "traefik.http.routers.dockge.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dockge.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.dockge.loadbalancer.server.port=5001"
    networks:
      - proxy

  diun:
    image: crazymax/diun:latest
    container_name: diun
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - DIUN_WATCH_WORKERS=20
      - DIUN_WATCH_SCHEDULE=0 */6 * * *
      - DIUN_PROVIDERS_DOCKER=true
      - DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT=true
    volumes:
      - ./diun:/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy

  duplicati:
    image: lscr.io/linuxserver/duplicati:latest
    container_name: duplicati
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - SETTINGS_ENCRYPTION_KEY=VotreCleSecrete
      - CLI_ARGS=--webservice-password=VotreMotDePasse
    volumes:
      - ./duplicati:/config
      - /volume1/docker:/source:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.duplicati.rule=Host(`backup.ooblik.com`)"
      - "traefik.http.routers.duplicati.entrypoints=websecure"
      - "traefik.http.routers.duplicati.tls.certresolver=letsencrypt"
      - "traefik.http.routers.duplicati.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.duplicati.loadbalancer.server.port=8200"
    networks:
      - proxy

networks:
  proxy:
    driver: bridge
```

---

## 12. Dépannage

### Problème : 404 Page not found

**Causes possibles** :
1. DNS pas encore propagé → `nslookup domaine.com 8.8.8.8`
2. Middleware introuvable → Vérifier `@docker` vs `@file`
3. Conteneur pas démarré → `sudo docker ps`
4. Erreur de syntaxe YAML → Vérifier l'indentation

### Problème : Bad Gateway

**Causes possibles** :
1. Service cible non accessible → `curl http://IP:PORT`
2. Réseau Docker différent → Vérifier que tous sont sur `proxy`
3. Port incorrect dans la config

### Problème : Certificat invalide

**Causes possibles** :
1. Rate limit Let's Encrypt (trop de demandes)
2. Port 80 non accessible de l'extérieur
3. DNS pas encore propagé

**Vérification** :
```bash
sudo docker logs traefik 2>&1 | grep -i -E "certificate|acme|error"
```

### Problème : Boucle de redirection Authelia

**Cause** : Le domaine auth.ooblik.com est lui-même protégé par Authelia

**Solution** : Ne pas appliquer le middleware authelia au service authelia

---

## 13. Commandes utiles

### Docker

```bash
# Voir les conteneurs
sudo docker ps

# Voir les logs d'un conteneur
sudo docker logs <conteneur>
sudo docker logs <conteneur> -f  # Suivre en temps réel
sudo docker logs <conteneur> 2>&1 | tail -50  # Dernières 50 lignes

# Redémarrer un conteneur
sudo docker restart <conteneur>

# Relancer toute la stack
sudo docker-compose down
sudo docker-compose up -d

# Entrer dans un conteneur
sudo docker exec -it <conteneur> /bin/sh
```

### Traefik

```bash
# Voir les routers
curl -s http://localhost:8080/api/http/routers | jq

# Voir les services
curl -s http://localhost:8080/api/http/services | jq

# Voir les middlewares
curl -s http://localhost:8080/api/http/middlewares | jq
```

### CrowdSec

```bash
# IP bannies
sudo docker exec crowdsec cscli decisions list

# Alertes
sudo docker exec crowdsec cscli alerts list

# Statistiques
sudo docker exec crowdsec cscli metrics
```

### Réseau

```bash
# Vérifier la résolution DNS
nslookup domaine.com 8.8.8.8

# Tester la connectivité
curl -I https://domaine.com

# Vider le cache DNS Windows
ipconfig /flushdns
```

---

## Conclusion

Félicitations ! Vous disposez maintenant d'une infrastructure complète et sécurisée :

✅ **Reverse proxy** Traefik avec certificats SSL automatiques  
✅ **Authentification 2FA** avec Authelia  
✅ **Protection anti-intrusion** avec CrowdSec  
✅ **Dashboard centralisé** avec Heimdall  
✅ **Gestion Docker visuelle** avec Dockge  
✅ **Notifications de mises à jour** avec Diun  
✅ **Sauvegardes** avec Duplicati

### Prochaines étapes suggérées

1. **Configurer le SMTP** pour Diun et Authelia (récupération de mot de passe)
2. **Ajouter d'autres services** (Jellyfin, ResourceSpace, NextCloud...)
3. **Configurer les sauvegardes** dans Duplicati
4. **Documenter vos secrets** de manière sécurisée (gestionnaire de mots de passe)
5. **Monitorer** les logs CrowdSec régulièrement

---

*Document créé le 12 janvier 2026*  
*Dernière mise à jour : 12 janvier 2026*
