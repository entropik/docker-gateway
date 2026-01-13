# Infrastructure Gateway Sécurisée - OOBLIK

## Contexte du projet

Ce projet est une **infrastructure Gateway complète** déployée sur un NAS Synology (Stock3) qui expose de manière sécurisée des services auto-hébergés sur Internet. L'infrastructure utilise Docker Compose et combine plusieurs technologies pour créer une gateway robuste avec authentification 2FA et protection anti-intrusion.

## Architecture

### Services principaux

- **Traefik v3.3** : Reverse proxy avec certificats SSL automatiques (Let's Encrypt)
- **Authelia** : Serveur d'authentification avec 2FA (TOTP)
- **CrowdSec** : Système de détection et blocage d'intrusions
- **Heimdall** : Dashboard centralisé pour accéder aux services
- **Dockge** : Interface de gestion Docker
- **Diun** : Notifications de mises à jour des images Docker
- **Duplicati** : Solution de sauvegarde

### Environnement

| Élément | Valeur |
|---------|--------|
| Serveur Gateway | Synology Stock3 - 192.168.1.200 |
| Serveur Services | Synology Stock2 - 192.168.1.120 (ex: Immich) |
| IP Publique | 45.80.33.101 |
| Domaine | ooblik.com |
| Ports publics | 80 → 8080, 443 → 8443 (NAT sur Freebox) |
| Timezone | Europe/Paris |

### Domaines et accès

| Service | URL | Protection |
|---------|-----|------------|
| Authelia | auth.ooblik.com | Public (portail de connexion) |
| Traefik Dashboard | trafik.ooblik.com | Authelia 2FA + CrowdSec |
| Heimdall | kalon.ooblik.com | Authelia 2FA + CrowdSec |
| Dockge | dockge.ooblik.com | Authelia 2FA + CrowdSec |
| Duplicati | backup.ooblik.com | Authelia 2FA + CrowdSec |
| Immich (externe) | photos.ooblik.com | CrowdSec uniquement |

## Structure des fichiers

```
/volume1/docker/gateway/
├── docker-compose.yml           # Configuration principale de tous les services
├── letsencrypt/
│   └── acme.json               # Certificats SSL (chmod 600)
├── config/
│   └── dynamic/                # Configurations Traefik dynamiques
│       ├── immich.yml          # Route vers service externe (Stock2)
│       ├── traefik-dashboard.yml  # Dashboard Traefik avec auth
│       └── crowdsec.yml        # Middleware CrowdSec
├── authelia/
│   ├── configuration.yml       # Config principale Authelia
│   ├── users.yml              # Base utilisateurs (hashed passwords)
│   └── db.sqlite3             # Base de données sessions (auto-créé)
├── crowdsec/
│   ├── config/                # Configuration CrowdSec
│   └── data/                  # Base de données des décisions
├── heimdall/                  # Config Heimdall
├── dockge/                    # Config Dockge
├── diun/                      # Config Diun
├── duplicati/                 # Config Duplicati
└── logs/
    └── access.log             # Logs Traefik analysés par CrowdSec
```

## Conventions et patterns du projet

### Configuration Traefik

#### Services Docker (dans docker-compose.yml)
Les services Docker utilisent des **labels Traefik** pour leur configuration :

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.SERVICE.rule=Host(`SUBDOMAIN.ooblik.com`)"
  - "traefik.http.routers.SERVICE.entrypoints=websecure"
  - "traefik.http.routers.SERVICE.tls.certresolver=letsencrypt"
  - "traefik.http.routers.SERVICE.middlewares=authelia@file,crowdsec@file"
  - "traefik.http.services.SERVICE.loadbalancer.server.port=PORT"
```

**Important** : Les middlewares utilisent `@file` car ils sont définis dans les fichiers dynamiques.

#### Services externes (fichiers dynamiques)
Les services hors du docker-compose Gateway (ex: Immich sur Stock2) utilisent des **fichiers YAML** dans `config/dynamic/` :

```yaml
http:
  routers:
    service-name:
      rule: "Host(`subdomain.ooblik.com`)"
      entryPoints:
        - websecure
      service: service-name
      tls:
        certResolver: letsencrypt
      middlewares:
        - crowdsec  # ou - authelia + crowdsec

  services:
    service-name:
      loadBalancer:
        servers:
          - url: "http://IP:PORT"
```

### Middlewares

Les middlewares sont **toujours définis dans des fichiers** (`config/dynamic/`), jamais dans les labels Docker :

- **authelia** : Défini dans `traefik-dashboard.yml`
- **crowdsec** : Défini dans `crowdsec.yml`

Ces middlewares sont ensuite référencés via `@file` dans les labels Docker.

### Politique de sécurité Authelia

Dans `authelia/configuration.yml` :
- `bypass` : Accès sans authentification (ex: Immich public)
- `one_factor` : Login + mot de passe uniquement
- `two_factor` : Login + mot de passe + code TOTP (par défaut pour `*.ooblik.com`)
- `deny` : Accès refusé

**Règle par défaut** : `deny` pour tout, puis exceptions explicites.

## Aide à l'assistant IA

### Quand l'utilisateur demande d'ajouter un nouveau service

1. **Service dans le même docker-compose** :
   - Ajouter le service avec les labels Traefik appropriés
   - Utiliser `authelia@file,crowdsec@file` comme middlewares si protection nécessaire
   - Ajouter au réseau `proxy`
   - Créer l'enregistrement DNS A pointant vers 45.80.33.101

2. **Service externe (autre serveur)** :
   - Créer un fichier YAML dans `config/dynamic/`
   - Définir le router et le service avec l'URL complète
   - Appliquer les middlewares nécessaires
   - Créer l'enregistrement DNS A

### Debugging commun

#### Erreur 404 / Service introuvable
- Vérifier que le conteneur est bien démarré : `docker ps`
- Vérifier les logs Traefik : `docker logs traefik`
- Vérifier la syntaxe YAML (indentation)
- Vérifier que le service est sur le réseau `proxy`
- Vérifier DNS : `nslookup subdomain.ooblik.com 8.8.8.8`

#### Bad Gateway (502)
- Le service cible n'est pas accessible
- Vérifier l'IP et le port dans la configuration
- Tester : `curl http://IP:PORT` depuis le serveur
- Vérifier que tous les services sont sur le même réseau Docker

#### Boucle de redirection Authelia
- Ne jamais appliquer le middleware `authelia` sur le service `authelia` lui-même
- Le domaine `auth.ooblik.com` doit être accessible sans middleware d'authentification

#### Certificat SSL invalide
- Vérifier que le port 80 est accessible depuis Internet (HTTP challenge)
- Vérifier les logs : `docker logs traefik 2>&1 | grep -i certificate`
- Attendre la propagation DNS (peut prendre quelques minutes)
- Vérifier le rate limit Let's Encrypt (max 5 certificats/semaine pour le même domaine)

### Commandes utiles

#### Docker
```bash
docker ps                           # Voir tous les conteneurs
docker logs <conteneur>             # Voir les logs
docker logs <conteneur> -f          # Suivre les logs en temps réel
docker restart <conteneur>          # Redémarrer un conteneur
docker-compose down && docker-compose up -d  # Relancer toute la stack
```

#### CrowdSec
```bash
docker exec crowdsec cscli decisions list    # Voir les IP bannies
docker exec crowdsec cscli alerts list       # Voir les alertes
docker exec crowdsec cscli metrics           # Voir les statistiques
docker exec crowdsec cscli decisions add --ip X.X.X.X  # Bannir une IP
docker exec crowdsec cscli decisions delete --ip X.X.X.X  # Débannir une IP
```

#### Traefik API (depuis le serveur)
```bash
curl -s http://localhost:8080/api/http/routers | jq
curl -s http://localhost:8080/api/http/services | jq
curl -s http://localhost:8080/api/http/middlewares | jq
```

## Considérations de sécurité

### Secrets exposés actuellement
Le `docker-compose.yml` contient des secrets en clair :
- `CROWDSEC_BOUNCER_API_KEY` (ligne 75)
- `SETTINGS_ENCRYPTION_KEY` pour Duplicati (ligne 141)
- `CLI_ARGS` avec mot de passe Duplicati (ligne 142)

**Recommandation** : Migrer vers Docker secrets ou variables d'environnement dans un fichier `.env` non versionné.

### Fichier acme.json
Le fichier `letsencrypt/acme.json` DOIT avoir les permissions `chmod 600` (lecture/écriture propriétaire uniquement). Traefik refusera de démarrer sinon.

### Base utilisateurs Authelia
Le fichier `authelia/users.yml` contient les hashes de mots de passe. Utiliser `argon2id` pour hasher les mots de passe (déjà configuré).

### Logs
Les logs Traefik dans `logs/access.log` peuvent contenir des informations sensibles. Configurer une rotation des logs.

## Points d'attention spécifiques

### Synology et ports
Les ports 80 et 443 sont utilisés par DSM. Traefik écoute donc sur 8080 et 8443, avec NAT configuré sur la Freebox :
- Internet:80 → 192.168.1.200:8080
- Internet:443 → 192.168.1.200:8443

Ne JAMAIS essayer de libérer les ports 80/443 sur Synology (casse DSM).

### Niveau de log Traefik
Actuellement en `DEBUG` (ligne 24 du docker-compose). Passer à `INFO` ou `WARN` en production pour réduire la verbosité.

### Collections CrowdSec
Collections installées : `crowdsecurity/traefik` et `crowdsecurity/http-cve`
Pour ajouter d'autres collections : modifier la variable `COLLECTIONS` dans le service `crowdsec`.

### Réseau Docker
Tous les services utilisent le réseau `proxy` en mode bridge. Ceci permet la communication inter-conteneurs via les noms de services (ex: `http://authelia:9091`).

## Documentation de référence

Le fichier `formation-gateway-ooblik.md` contient le guide complet de formation avec toutes les explications détaillées de l'architecture, de la configuration et du dépannage.

## Workflow de travail

1. **Modification de configuration** : Éditer les fichiers appropriés
2. **Redémarrage** : `docker-compose down && docker-compose up -d`
3. **Vérification** : Consulter les logs avec `docker logs <service>`
4. **Test** : Accéder à l'URL dans le navigateur
5. **Debugging** : Utiliser les commandes de la section "Commandes utiles"

---

**Note pour l'assistant** : Ce projet suit les bonnes pratiques Docker et Traefik v3. Toujours vérifier la compatibilité des configurations avec Traefik v3 (syntaxe différente de v2). Privilégier la simplicité et la sécurité dans les suggestions.
