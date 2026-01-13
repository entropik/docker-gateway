# Guide de Déploiement Complet

Ce guide détaille le déploiement de l'infrastructure Gateway depuis zéro, avec toutes les configurations et vérifications nécessaires.

## 📋 Table des Matières

- [Prérequis](#prérequis)
- [Architecture](#architecture)
- [Étape 1 : Préparation du Serveur](#étape-1--préparation-du-serveur)
- [Étape 2 : Configuration DNS](#étape-2--configuration-dns)
- [Étape 3 : Installation Docker](#étape-3--installation-docker)
- [Étape 4 : Clonage et Configuration](#étape-4--clonage-et-configuration)
- [Étape 5 : Génération des Secrets](#étape-5--génération-des-secrets)
- [Étape 6 : Configuration des Services](#étape-6--configuration-des-services)
- [Étape 7 : Premier Démarrage](#étape-7--premier-démarrage)
- [Étape 8 : Configuration Post-Déploiement](#étape-8--configuration-post-déploiement)
- [Étape 9 : Tests et Validation](#étape-9--tests-et-validation)
- [Dépannage](#dépannage)
- [Maintenance](#maintenance)

## Prérequis

### Matériel Recommandé

- **CPU** : 2 cores minimum (4 cores recommandé)
- **RAM** : 4 GB minimum (8 GB recommandé)
- **Stockage** : 20 GB minimum (SSD recommandé)
- **Réseau** : Connexion stable, bande passante suffisante

### Logiciels Requis

- **OS** : Linux (Ubuntu 22.04 LTS recommandé) ou Synology DSM 7+
- **Docker Engine** : 20.10+ ([Installation](https://docs.docker.com/engine/install/))
- **Docker Compose** : v2.0+ (inclus dans Docker Desktop)
- **curl, wget** : Pour les tests
- **openssl** : Pour générer les secrets

### Réseau

- **Domaine** : Un domaine enregistré avec accès aux DNS
- **IP Publique** : IP fixe recommandée (ou DynDNS)
- **Ports ouverts** : 80 (HTTP) et 443 (HTTPS) depuis Internet
- **Firewall** : Ports 80/443 autorisés en entrée

### Accès

- **SSH** : Accès root ou sudo sur le serveur
- **Registrar DNS** : Accès pour créer/modifier les enregistrements DNS

## Architecture

### Topologie Réseau

```
Internet
   │
   ├─── Port 80 (HTTP)
   └─── Port 443 (HTTPS)
          │
    [Routeur/Firewall]
       NAT: 80 → 8080
       NAT: 443 → 8453
          │
    [Serveur Gateway]
          │
     [Docker Bridge: proxy]
          │
    ┌─────┴─────┬──────┬───────┬─────────┐
    │           │      │       │         │
 Traefik   Authelia  CrowdSec  Heimdall  ...
  :8080      :9091    :8080     :80
```

### Flux d'Authentification

```
1. Client → Traefik (HTTPS)
2. Traefik → CrowdSec Bouncer (vérification IP)
3. Si IP OK → Traefik → Authelia (vérification auth)
4. Si Auth OK → Traefik → Service final
5. Service → Réponse au client
```

## Étape 1 : Préparation du Serveur

### 1.1 Mise à Jour du Système

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git openssl jq

# CentOS/RHEL
sudo yum update -y
sudo yum install -y curl wget git openssl jq
```

### 1.2 Configuration du Firewall

```bash
# UFW (Ubuntu)
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# firewalld (CentOS)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### 1.3 Synology DSM (cas particulier)

Sur Synology, les ports 80/443 sont utilisés par DSM. Configuration NAT :

```
Freebox/Routeur:
- Internet:80 → NAS:8080
- Internet:443 → NAS:8453

docker-compose.yml:
  traefik:
    ports:
      - "8080:80"    # HTTP
      - "8453:443"   # HTTPS
```

## Étape 2 : Configuration DNS

### 2.1 Créer les Enregistrements DNS

Chez votre registrar (OVH, Cloudflare, etc.), créer des enregistrements A :

| Sous-domaine | Type | Valeur | TTL |
|--------------|------|--------|-----|
| auth | A | Votre_IP_Publique | 300 |
| trafik | A | Votre_IP_Publique | 300 |
| kalon | A | Votre_IP_Publique | 300 |
| dockge | A | Votre_IP_Publique | 300 |
| backup | A | Votre_IP_Publique | 300 |
| photos | A | Votre_IP_Publique | 300 |

### 2.2 Vérifier la Propagation DNS

```bash
# Attendre quelques minutes puis vérifier
nslookup auth.example.com 8.8.8.8
dig auth.example.com +short

# Vérifier tous les sous-domaines
for sub in auth trafik kalon dockge backup photos; do
  echo "$sub.example.com: $(dig +short $sub.example.com @8.8.8.8)"
done
```

**IMPORTANT** : Ne pas continuer tant que tous les DNS ne répondent pas.

## Étape 3 : Installation Docker

### 3.1 Installation Docker Engine

```bash
# Ubuntu/Debian - Installation via script officiel
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER
newgrp docker

# Vérifier l'installation
docker --version
docker compose version
```

### 3.2 Configuration Docker (optionnel)

```bash
# Créer le fichier de configuration
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

Contenu recommandé :

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

```bash
# Redémarrer Docker
sudo systemctl restart docker
```

### 3.3 Synology DSM

Sur Synology :
1. Ouvrir le Package Center
2. Installer **Container Manager** (anciennement Docker)
3. Activer SSH dans Panneau de configuration → Terminal & SNMP
4. Se connecter en SSH : `ssh admin@nas-ip`

## Étape 4 : Clonage et Configuration

### 4.1 Cloner le Repository

```bash
# Choisir un emplacement (ex: /opt pour serveur, /volume1/docker pour Synology)
cd /opt  # ou cd /volume1/docker pour Synology
git clone https://github.com/entropik/docker-gateway.git gateway
cd gateway
```

### 4.2 Créer l'Arborescence

```bash
# Créer les dossiers nécessaires
mkdir -p authelia/secrets letsencrypt logs

# Définir les permissions
chmod 700 authelia/secrets
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

## Étape 5 : Génération des Secrets

### 5.1 Fichier .env Principal

```bash
# Copier le template
cp .env.example .env

# Éditer le fichier
nano .env
```

Remplacer les valeurs :

```bash
TZ=Europe/Paris                           # Votre timezone
DOMAIN=example.com                        # VOTRE domaine
ADMIN_EMAIL=admin@example.com             # VOTRE email
ACME_EMAIL=${ADMIN_EMAIL}

# Générer les clés
CROWDSEC_BOUNCER_API_KEY=                 # Sera généré à l'étape 8
DUPLICATI_SETTINGS_ENCRYPTION_KEY=$(openssl rand -base64 32)
DUPLICATI_WEBSERVICE_PASSWORD=$(openssl rand -base64 24)
HEIMDALL_APP_KEY=base64:$(openssl rand -base64 32)
```

**Script pour générer automatiquement** :

```bash
# Remplacer DOMAIN et EMAIL puis exécuter
cat > .env <<EOF
TZ=Europe/Paris
DOMAIN=example.com
ADMIN_EMAIL=admin@example.com
ACME_EMAIL=\${ADMIN_EMAIL}
CROWDSEC_BOUNCER_API_KEY=TEMPORARY
DUPLICATI_SETTINGS_ENCRYPTION_KEY=$(openssl rand -base64 32)
DUPLICATI_WEBSERVICE_PASSWORD=$(openssl rand -base64 24)
HEIMDALL_APP_KEY=base64:$(openssl rand -base64 32)
EOF

chmod 600 .env
```

### 5.2 Secrets Authelia

```bash
# Générer les 3 secrets Authelia
openssl rand -base64 32 > authelia/secrets/jwt_secret.txt
openssl rand -base64 32 > authelia/secrets/session_secret.txt
openssl rand -base64 32 > authelia/secrets/storage_encryption_key.txt

# Sécuriser les permissions
chmod 600 authelia/secrets/*.txt

# Vérifier
ls -la authelia/secrets/
```

### 5.3 Utilisateur Authelia

```bash
# Copier le template
cp authelia/users.yml.example authelia/users.yml

# Générer un hash de mot de passe (choisir un mot de passe FORT)
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'MonMotDePasseTresSecurise123!'

# Copier le hash généré et éditer users.yml
nano authelia/users.yml
```

Remplacer :

```yaml
users:
  admin:  # Votre nom d'utilisateur
    displayname: "Administrateur"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."  # VOTRE hash
    email: admin@example.com  # VOTRE email
    groups:
      - admins
```

```bash
# Sécuriser
chmod 600 authelia/users.yml
```

## Étape 6 : Configuration des Services

### 6.1 Authelia

```bash
# Copier le fichier de configuration
cp authelia/configuration.yml.example authelia/configuration.yml

# Éditer pour remplacer example.com par votre domaine
sed -i 's/example.com/VOTRE-DOMAINE.com/g' authelia/configuration.yml

# Vérifier
grep -n "VOTRE-DOMAINE.com" authelia/configuration.yml
```

### 6.2 Heimdall (optionnel)

```bash
# Si le dossier heimdall/www n'existe pas, le créer
mkdir -p heimdall/www

# Copier le template (si accessible)
cp heimdall/www/.env.example heimdall/www/.env 2>/dev/null || echo "APP_KEY=base64:$(openssl rand -base64 32)" > heimdall/www/.env

# Éditer
nano heimdall/www/.env
```

Minimum requis :

```bash
APP_KEY=base64:VotreClefGeneree
APP_URL=https://kalon.example.com
```

### 6.3 docker-compose.yml

Vérifier que les domaines correspondent :

```bash
# Rechercher example.com dans docker-compose.yml
grep -n "example.com" docker-compose.yml

# Si trouvé, remplacer (le fichier doit déjà utiliser ${DOMAIN})
# Normalement rien à faire si le fichier utilise les variables
```

## Étape 7 : Premier Démarrage

### 7.1 Vérifier la Configuration

```bash
# Valider la syntaxe Docker Compose
docker compose config

# Vérifier que toutes les variables sont définies
docker compose config | grep -i "changeme\|example.com" && echo "⚠️ Variables non remplacées !" || echo "✓ Configuration OK"
```

### 7.2 Démarrer les Services

```bash
# Démarrer en mode détaché
docker compose up -d

# Vérifier que tous les conteneurs démarrent
docker compose ps

# Suivre les logs en temps réel
docker compose logs -f
```

**Statut attendu** :
```
NAME                 IMAGE                   STATUS
traefik              traefik:v3.3            Up X seconds
authelia             authelia/authelia       Up X seconds
crowdsec             crowdsecurity/crowdsec  Up X seconds
crowdsec-bouncer     fbonalair/...           Up X seconds
heimdall             lscr.io/.../heimdall    Up X seconds
dockge               louislam/dockge         Up X seconds
diun                 crazymax/diun           Up X seconds
duplicati            lscr.io/.../duplicati   Up X seconds
```

### 7.3 Surveiller les Logs

```bash
# Authelia (doit montrer "Startup complete")
docker compose logs authelia | tail -20

# Traefik (doit montrer les certificats en cours de génération)
docker compose logs traefik | tail -20

# CrowdSec
docker compose logs crowdsec | tail -20
```

## Étape 8 : Configuration Post-Déploiement

### 8.1 Générer la Clé CrowdSec Bouncer

```bash
# Attendre que CrowdSec soit complètement démarré (30 secondes)
sleep 30

# Générer la clé bouncer
docker exec crowdsec cscli bouncers add traefik-bouncer

# Copier la clé affichée (format: longue chaîne alphanumérique)
# Exemple: YpfwtZVI8i3fHapoHAtwvLSNrGoCuMdq5o84NkrvwQY
```

Éditer `.env` :

```bash
nano .env
# Remplacer CROWDSEC_BOUNCER_API_KEY=TEMPORARY par la vraie clé
```

Redémarrer le bouncer :

```bash
docker compose restart crowdsec-bouncer

# Vérifier qu'il n'y a plus d'erreur 403
docker compose logs crowdsec-bouncer --tail=20
```

### 8.2 Attendre les Certificats SSL

```bash
# Surveiller l'obtention des certificats (peut prendre 5-10 minutes)
docker compose logs -f traefik | grep -i certificate

# Vérifier acme.json
sudo ls -lh letsencrypt/acme.json
# Doit contenir des données (plusieurs Ko)
```

**Problèmes courants** :
- Port 80 non accessible → Vérifier firewall et NAT
- Rate limit Let's Encrypt → Attendre 1 heure
- DNS non propagés → Attendre ou vérifier DNS

### 8.3 Configurer l'Authentification 2FA

1. Accéder à `https://auth.VOTRE-DOMAINE.com`
2. Se connecter avec le compte créé (users.yml)
3. Scanner le QR code avec Google Authenticator / Authy
4. Entrer le code TOTP pour valider
5. **IMPORTANT** : Sauvegarder les codes de secours !

## Étape 9 : Tests et Validation

### 9.1 Tests de Connectivité

```bash
# Test HTTP → HTTPS redirect
curl -I http://auth.example.com
# Attendu: 301 ou 308 vers https://

# Test HTTPS
curl -I https://auth.example.com
# Attendu: 200 OK

# Test tous les services
for service in auth trafik kalon dockge backup; do
  echo "Testing $service.example.com:"
  curl -I https://$service.example.com 2>&1 | head -1
done
```

### 9.2 Tests d'Authentification

1. **Sans authentification** :
   ```bash
   curl -I https://kalon.example.com
   # Attendu: 302 redirect vers https://auth.example.com
   ```

2. **Avec navigateur** :
   - Accéder à https://kalon.example.com
   - Devrait rediriger vers page de login Authelia
   - Login + code TOTP
   - Redirection vers Heimdall

### 9.3 Tests CrowdSec

```bash
# Vérifier que le bouncer communique avec CrowdSec
docker compose logs crowdsec-bouncer | grep -i "bouncer"

# Simuler une attaque (10 requêtes rapides sur login)
for i in {1..10}; do
  curl -X POST https://auth.example.com/api/firstfactor \
    -d '{"username":"fake","password":"fake"}' &
done

# Attendre 30 secondes puis vérifier les décisions
docker exec crowdsec cscli decisions list
# Votre IP devrait être bannie temporairement

# Débannir votre IP
docker exec crowdsec cscli decisions delete --ip VOTRE_IP
```

### 9.4 Tests de Certificats

```bash
# Vérifier la validité du certificat
echo | openssl s_client -connect auth.example.com:443 \
  -servername auth.example.com 2>/dev/null | \
  openssl x509 -noout -dates

# Tester avec SSL Labs (en ligne)
# https://www.ssllabs.com/ssltest/analyze.html?d=auth.example.com
```

## Dépannage

### Traefik ne démarre pas

```bash
# Vérifier les logs
docker compose logs traefik

# Problèmes courants :
# 1. acme.json permissions
chmod 600 letsencrypt/acme.json

# 2. Port déjà utilisé
sudo netstat -tulpn | grep :80
sudo netstat -tulpn | grep :443

# 3. Configuration invalide
docker compose config
```

### Authelia erreur "secret already defined"

```bash
# Vérifier authelia/configuration.yml
# Les clés jwt_secret, session.secret, storage.encryption_key
# NE DOIVENT PAS être présentes (chargées via _FILE)

grep -E "jwt_secret:|session.*secret:|encryption_key:" authelia/configuration.yml
# Ne doit rien retourner ou seulement des commentaires
```

### Certificat SSL non généré

```bash
# 1. Vérifier que le port 80 est accessible depuis Internet
curl -I http://auth.example.com

# 2. Vérifier les logs ACME
docker compose logs traefik | grep -i acme

# 3. Forcer le renouvellement
rm letsencrypt/acme.json
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
docker compose restart traefik
```

### Service 502 Bad Gateway

```bash
# 1. Vérifier que le service cible est UP
docker compose ps

# 2. Vérifier le réseau
docker compose exec traefik ping nom-du-service

# 3. Vérifier les routes Traefik
docker compose exec traefik wget -qO- http://localhost:8080/api/http/routers | jq

# 4. Vérifier les logs du service cible
docker compose logs nom-du-service
```

## Maintenance

### Sauvegardes

```bash
# Script de backup complet
#!/bin/bash
BACKUP_DIR="/backup/gateway"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

# Backup configuration
tar -czf $BACKUP_DIR/gateway-config-$DATE.tar.gz \
  .env \
  docker-compose.yml \
  authelia/ \
  config/ \
  --exclude='authelia/db.sqlite3'

# Backup secrets (à stocker séparément !)
tar -czf $BACKUP_DIR/gateway-secrets-$DATE.tar.gz \
  .env \
  authelia/secrets/ \
  authelia/users.yml

# Backup base de données Authelia
cp authelia/db.sqlite3 $BACKUP_DIR/authelia-db-$DATE.sqlite3

# Backup certificats
tar -czf $BACKUP_DIR/letsencrypt-$DATE.tar.gz letsencrypt/

echo "Backup terminé: $BACKUP_DIR"
```

### Mises à Jour

```bash
# Vérifier les nouvelles versions
docker compose pull

# Mettre à jour un service spécifique
docker compose pull authelia
docker compose up -d authelia

# Mettre à jour tous les services
docker compose up -d

# Nettoyer les anciennes images
docker image prune -a
```

### Monitoring

```bash
# Script de monitoring
#!/bin/bash
echo "=== État des services ==="
docker compose ps

echo -e "\n=== Utilisation ressources ==="
docker stats --no-stream

echo -e "\n=== IPs bannies CrowdSec ==="
docker exec crowdsec cscli decisions list

echo -e "\n=== Dernières alertes ==="
docker exec crowdsec cscli alerts list --limit 5

echo -e "\n=== Certificats SSL ==="
sudo ls -lh letsencrypt/acme.json
```

## Checklist Post-Déploiement

- [ ] Tous les conteneurs sont UP
- [ ] Authelia accessible et 2FA configuré
- [ ] Certificats SSL valides pour tous les sous-domaines
- [ ] CrowdSec bouncer opérationnel (pas d'erreur 403)
- [ ] Services protégés nécessitent authentification
- [ ] Tests de connectivité réussis
- [ ] Backups configurés et testés
- [ ] Documentation mise à jour avec vos spécificités
- [ ] Contacts d'urgence définis
- [ ] Plan de rollback documenté

## Ressources Supplémentaires

- [Documentation Traefik](https://doc.traefik.io/traefik/)
- [Documentation Authelia](https://www.authelia.com/)
- [Documentation CrowdSec](https://docs.crowdsec.net/)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)

---

**Support** : Pour toute question, ouvrir une [issue GitHub](https://github.com/entropik/docker-gateway/issues)
