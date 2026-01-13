# Docker Gateway - Infrastructure Sécurisée

Infrastructure Gateway complète avec reverse proxy, authentification 2FA, et protection anti-intrusions pour services auto-hébergés.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## 🎯 Vue d'ensemble

Cette infrastructure fournit une gateway sécurisée pour exposer des services Docker sur Internet avec :

- **Reverse Proxy** : Traefik v3.3 avec certificats SSL automatiques (Let's Encrypt)
- **Authentification 2FA** : Authelia avec TOTP (Google Authenticator, etc.)
- **Protection Anti-Intrusions** : CrowdSec avec détection comportementale
- **Gestion Centralisée** : Dashboard, gestion Docker, monitoring des mises à jour
- **Sauvegardes** : Solution de backup automatisée

## 📦 Services Inclus

| Service | Version | Description | Port |
|---------|---------|-------------|------|
| **Traefik** | 3.3 | Reverse proxy avec SSL automatique | 80, 443 |
| **Authelia** | Latest | Serveur d'authentification 2FA | 9091 |
| **CrowdSec** | Latest | Détection d'intrusions collaborative | 8080 |
| **Heimdall** | Latest | Dashboard d'applications | 80 |
| **Dockge** | Latest | Interface de gestion Docker | 5001 |
| **Diun** | Latest | Notifications de mises à jour Docker | - |
| **Duplicati** | Latest | Sauvegarde avec chiffrement | 8200 |

## ✨ Fonctionnalités

### Sécurité
- ✅ Authentification 2FA obligatoire (TOTP)
- ✅ Détection et blocage automatique des IPs malveillantes
- ✅ Certificats SSL/TLS automatiques et renouvelés
- ✅ Protection contre les attaques par force brute
- ✅ Détection de CVE connues (Log4Shell, etc.)

### Facilité d'utilisation
- ✅ Configuration déclarative (Docker Compose)
- ✅ Dashboard centralisé pour tous les services
- ✅ Interface web pour gérer les stacks Docker
- ✅ Notifications automatiques des mises à jour

### Fiabilité
- ✅ Redémarrage automatique des services
- ✅ Sauvegardes automatisées et chiffrées
- ✅ Logs centralisés pour le debugging
- ✅ Isolation réseau entre services

## 🚀 Installation Rapide

### Prérequis

- Docker Engine 20.10+
- Docker Compose v2.0+
- Un domaine avec accès aux enregistrements DNS
- Ports 80 et 443 accessibles depuis Internet

### 1. Cloner le repository

```bash
git clone https://github.com/entropik/docker-gateway.git
cd docker-gateway
```

### 2. Configurer les secrets

Créer le fichier `.env` depuis le template :

```bash
cp .env.example .env
nano .env  # Éditer et remplacer tous les CHANGEME
```

Variables obligatoires à définir :
- `DOMAIN` : Votre domaine (ex: example.com)
- `ADMIN_EMAIL` : Email pour Let's Encrypt
- `CROWDSEC_BOUNCER_API_KEY` : Clé API CrowdSec (générée à l'étape 4)
- `DUPLICATI_SETTINGS_ENCRYPTION_KEY` : Clé de chiffrement (32 bytes base64)
- `DUPLICATI_WEBSERVICE_PASSWORD` : Mot de passe admin Duplicati

### 3. Configurer les secrets Authelia

Créer le dossier et les fichiers secrets :

```bash
mkdir -p authelia/secrets
chmod 700 authelia/secrets

# Générer les secrets (32 bytes en base64)
openssl rand -base64 32 > authelia/secrets/jwt_secret.txt
openssl rand -base64 32 > authelia/secrets/session_secret.txt
openssl rand -base64 32 > authelia/secrets/storage_encryption_key.txt

chmod 600 authelia/secrets/*.txt
```

### 4. Configurer les utilisateurs Authelia

Créer le fichier des utilisateurs depuis le template :

```bash
cp authelia/users.yml.example authelia/users.yml

# Générer un hash de mot de passe
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'VotreMotDePasse'

# Éditer users.yml et remplacer le hash
nano authelia/users.yml
```

### 5. Configurer Heimdall (optionnel)

Si vous utilisez Heimdall :

```bash
cp heimdall/www/.env.example heimdall/www/.env

# Générer une clé d'application
docker run --rm php:cli php -r "echo 'base64:' . base64_encode(random_bytes(32)) . PHP_EOL;"

# Ajouter la clé dans heimdall/www/.env
nano heimdall/www/.env
```

### 6. Configurer les enregistrements DNS

Créer des enregistrements DNS A pour tous vos sous-domaines :

```
auth.example.com      → Votre IP publique
dockge.example.com    → Votre IP publique
backup.example.com    → Votre IP publique
photos.example.com    → Votre IP publique (si service externe)
```

### 7. Démarrer les services

```bash
# Démarrer l'infrastructure
docker compose up -d

# Vérifier que tous les services sont démarrés
docker compose ps

# Vérifier les logs
docker compose logs -f
```

### 8. Générer la clé API CrowdSec

Une fois CrowdSec démarré :

```bash
# Générer la clé bouncer
docker exec crowdsec cscli bouncers add traefik-bouncer

# Copier la clé affichée dans .env
nano .env  # Remplacer CROWDSEC_BOUNCER_API_KEY

# Redémarrer le bouncer
docker compose restart crowdsec-bouncer
```

### 9. Configurer l'authentification 2FA

1. Accéder à `https://auth.example.com`
2. Se connecter avec le compte créé à l'étape 4
3. Scanner le QR code avec Google Authenticator ou équivalent
4. Valider le code TOTP

## 📖 Configuration Avancée

### Ajouter un nouveau service protégé

Pour exposer un nouveau service Docker :

```yaml
# Dans docker-compose.yml
  mon-service:
    image: mon/image:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mon-service.rule=Host(`service.${DOMAIN}`)"
      - "traefik.http.routers.mon-service.entrypoints=websecure"
      - "traefik.http.routers.mon-service.tls.certresolver=letsencrypt"
      - "traefik.http.routers.mon-service.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.mon-service.loadbalancer.server.port=80"
    networks:
      - proxy
```

### Exposer un service externe

Pour un service sur un autre serveur, créer un fichier dans `config/dynamic/` :

```yaml
# config/dynamic/mon-service-externe.yml
http:
  routers:
    mon-service-externe:
      rule: "Host(`service.example.com`)"
      entryPoints:
        - websecure
      service: mon-service-externe
      tls:
        certResolver: letsencrypt
      middlewares:
        - authelia
        - crowdsec

  services:
    mon-service-externe:
      loadBalancer:
        servers:
          - url: "http://192.168.1.100:8080"
```

### Personnaliser les règles d'accès Authelia

Éditer `authelia/configuration.yml` :

```yaml
access_control:
  default_policy: deny
  rules:
    # Service public (sans auth)
    - domain: public.example.com
      policy: bypass

    # Service avec 2FA
    - domain: '*.example.com'
      policy: two_factor
```

## 🛠️ Maintenance

### Mettre à jour les images

```bash
# Voir les images disponibles
docker compose images

# Mettre à jour toutes les images
docker compose pull

# Redémarrer avec les nouvelles images
docker compose up -d

# Supprimer les anciennes images
docker image prune -a
```

### Consulter les logs

```bash
# Tous les services
docker compose logs -f

# Un service spécifique
docker compose logs -f authelia

# Dernières 100 lignes
docker compose logs --tail=100
```

### Sauvegarder la configuration

```bash
# Créer un backup complet (hors données Docker)
tar -czf gateway-backup-$(date +%Y%m%d).tar.gz \
  docker-compose.yml \
  .env \
  authelia/ \
  config/ \
  --exclude='authelia/db.sqlite3' \
  --exclude='authelia/notification.txt'

# Sauvegarder les secrets (à stocker en lieu sûr !)
tar -czf gateway-secrets-$(date +%Y%m%d).tar.gz .env authelia/secrets/
```

### Gérer CrowdSec

```bash
# Voir les décisions (IPs bannies)
docker exec crowdsec cscli decisions list

# Voir les alertes
docker exec crowdsec cscli alerts list

# Débannir une IP
docker exec crowdsec cscli decisions delete --ip 1.2.3.4

# Ajouter une IP à la whitelist
docker exec crowdsec cscli decisions add --ip 1.2.3.4 --duration 999999h --type whitelist
```

## 🐛 Dépannage

### Traefik ne démarre pas

```bash
# Vérifier les permissions acme.json
chmod 600 letsencrypt/acme.json

# Vérifier les logs
docker compose logs traefik
```

### Authelia refuse les connexions

```bash
# Vérifier que les secrets sont chargés
docker compose logs authelia | grep -i secret

# Vérifier la base de données
docker exec -it authelia ls -la /config/db.sqlite3
```

### Certificat SSL invalide

```bash
# Vérifier les logs ACME
docker compose logs traefik | grep -i certificate

# Vérifier que le port 80 est accessible
curl -I http://example.com

# Forcer le renouvellement (supprimer acme.json et redémarrer)
rm letsencrypt/acme.json
docker compose restart traefik
```

### Service inaccessible (502 Bad Gateway)

```bash
# Vérifier que le service cible est démarré
docker compose ps

# Vérifier les routes Traefik
docker exec traefik wget -qO- http://localhost:8080/api/http/routers | jq

# Vérifier la connectivité réseau
docker exec traefik ping nom-du-service
```

## 📚 Documentation

- [DEPLOYMENT.md](DEPLOYMENT.md) - Guide de déploiement complet
- [SECURITY.md](SECURITY.md) - Politique de sécurité et bonnes pratiques
- [formation-gateway-ooblik.md](formation-gateway-ooblik.md) - Guide de formation détaillé
- [CLAUDE.md](CLAUDE.md) - Documentation pour assistant IA

## 🤝 Contribution

Les contributions sont bienvenues ! Pour proposer des améliorations :

1. Fork le projet
2. Créer une branche (`git checkout -b feature/amelioration`)
3. Commit les changements (`git commit -m 'Ajout de fonctionnalité'`)
4. Push vers la branche (`git push origin feature/amelioration`)
5. Ouvrir une Pull Request

## 📝 Licence

Ce projet est sous licence MIT - voir le fichier [LICENSE](LICENSE) pour plus de détails.

## 🙏 Remerciements

- [Traefik](https://traefik.io/) - The Cloud Native Application Proxy
- [Authelia](https://www.authelia.com/) - The Single Sign-On Multi-Factor portal
- [CrowdSec](https://www.crowdsec.net/) - Collaborative IPS
- [Heimdall](https://heimdall.site/) - Application Dashboard
- [Dockge](https://dockge.kuma.pet/) - Docker Compose Stack Manager

## 📞 Support

Pour toute question ou problème :
- Ouvrir une [issue](https://github.com/entropik/docker-gateway/issues)
- Consulter la [documentation Traefik](https://doc.traefik.io/traefik/)
- Consulter la [documentation Authelia](https://www.authelia.com/overview/prologue/introduction/)

---

⭐ Si ce projet vous est utile, n'hésitez pas à lui donner une étoile !
