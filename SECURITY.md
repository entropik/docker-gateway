# Politique de Sécurité

## 🔒 Gestion des Secrets

### Principes Fondamentaux

Ce projet suit une approche hybride en 3 niveaux pour la gestion des secrets :

1. **Fichier .env central** : Variables d'environnement pour docker-compose.yml
2. **Fichiers secrets individuels** : Secrets Authelia chargés via `_FILE` variables
3. **.gitignore complet** : Protection de tous les fichiers sensibles

### Secrets Requis

#### 1. Variables d'Environnement (.env)

```bash
# Fichier: .env (JAMAIS versionné)
TZ=Europe/Paris
DOMAIN=example.com
ADMIN_EMAIL=admin@example.com
ACME_EMAIL=${ADMIN_EMAIL}
CROWDSEC_BOUNCER_API_KEY=<généré par CrowdSec>
DUPLICATI_SETTINGS_ENCRYPTION_KEY=<32 bytes base64>
DUPLICATI_WEBSERVICE_PASSWORD=<mot de passe fort>
HEIMDALL_APP_KEY=base64:<32 bytes base64>
```

**Génération des clés** :
```bash
# Clé de chiffrement (32 bytes)
openssl rand -base64 32

# Mot de passe sécurisé
openssl rand -base64 24
```

#### 2. Secrets Authelia (authelia/secrets/)

```bash
# Fichiers: authelia/secrets/*.txt (JAMAIS versionnés)
authelia/secrets/jwt_secret.txt
authelia/secrets/session_secret.txt
authelia/secrets/storage_encryption_key.txt
```

**Génération** :
```bash
mkdir -p authelia/secrets
chmod 700 authelia/secrets
openssl rand -base64 32 > authelia/secrets/jwt_secret.txt
openssl rand -base64 32 > authelia/secrets/session_secret.txt
openssl rand -base64 32 > authelia/secrets/storage_encryption_key.txt
chmod 600 authelia/secrets/*.txt
```

#### 3. Fichier Utilisateurs Authelia (authelia/users.yml)

```bash
# Fichier: authelia/users.yml (JAMAIS versionné)
```

**Génération du hash de mot de passe** :
```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'VotreMotDePasseFort123!'
```

**IMPORTANT** : Utilisez des mots de passe forts (minimum 16 caractères, avec majuscules, minuscules, chiffres et symboles).

### Fichiers Protégés par .gitignore

Le fichier `.gitignore` protège automatiquement :

```
✅ Secrets
- .env, .env.local, .env.*.local
- authelia/secrets/
- authelia/users.yml
- heimdall/www/.env
- crowdsec/config/*_credentials.yaml

✅ Certificats et clés privées
- letsencrypt/
- heimdall/keys/*.key
- *.key, *.pem, *.p12, *.pfx

✅ Bases de données
- *.sqlite, *.sqlite3, *.db
- authelia/db.sqlite3

✅ Données runtime
- logs/
- crowdsec/data/
- diun/, dockge/, duplicati/
```

## 🛡️ Bonnes Pratiques de Sécurité

### 1. Mots de Passe

- ✅ **Minimum 16 caractères** pour les comptes Authelia
- ✅ **Unique par service** (ne jamais réutiliser)
- ✅ **Stockage sécurisé** (gestionnaire de mots de passe)
- ✅ **Rotation régulière** (tous les 90 jours recommandé)

### 2. Authentification 2FA

- ✅ **Obligatoire** pour tous les services exposés
- ✅ **Application TOTP** recommandée (Google Authenticator, Authy, etc.)
- ✅ **Codes de secours** sauvegardés en lieu sûr
- ✅ **Période de validité** : 30 secondes (configuration par défaut)

### 3. Certificats SSL/TLS

- ✅ **Let's Encrypt** pour certificats automatiques
- ✅ **Renouvellement automatique** via Traefik
- ✅ **Permissions strictes** : `chmod 600 letsencrypt/acme.json`
- ✅ **TLS 1.2+** uniquement

### 4. Réseau Docker

- ✅ **Isolation réseau** : réseau bridge dédié `proxy`
- ✅ **Exposition minimale** : seuls Traefik expose des ports (80, 443)
- ✅ **Communication interne** : via noms de services Docker
- ✅ **Pas de host network** sauf cas exceptionnels

### 5. CrowdSec et Protection Anti-Intrusions

- ✅ **Bouncer actif** sur tous les services exposés
- ✅ **Collections installées** : traefik, http-cve
- ✅ **Logs analysés** : Traefik access.log
- ✅ **Décisions automatiques** : ban après 3 tentatives ratées

**Commandes utiles** :
```bash
# Voir les IPs bannies
docker exec crowdsec cscli decisions list

# Débannir une IP légitime
docker exec crowdsec cscli decisions delete --ip 1.2.3.4

# Whitelister une IP de confiance
docker exec crowdsec cscli decisions add --ip 1.2.3.4 \
  --duration 999999h --type whitelist
```

### 6. Règles d'Accès Authelia

Configurer des règles granulaires dans `authelia/configuration.yml` :

```yaml
access_control:
  default_policy: deny  # Deny par défaut = sécurité maximale
  rules:
    # Service public (attention !)
    - domain: public.example.com
      policy: bypass

    # Service avec 2FA obligatoire
    - domain: '*.example.com'
      policy: two_factor

    # Service accessible seulement d'un réseau spécifique
    - domain: admin.example.com
      policy: two_factor
      networks:
        - 192.168.1.0/24
```

### 7. Mises à Jour de Sécurité

- ✅ **Diun activé** : notifications automatiques des mises à jour
- ✅ **Images officielles** : toujours utiliser les images officielles
- ✅ **Tags spécifiques** : éviter `:latest` en production
- ✅ **Mise à jour régulière** : minimum mensuelle

**Procédure de mise à jour** :
```bash
# 1. Vérifier les images disponibles
docker compose pull

# 2. Vérifier les changelogs (breaking changes)
# https://github.com/<service>/releases

# 3. Backup avant mise à jour
tar -czf backup-$(date +%Y%m%d).tar.gz .env authelia/ config/

# 4. Mettre à jour
docker compose up -d

# 5. Vérifier les logs
docker compose logs -f
```

### 8. Logs et Monitoring

- ✅ **Logs centralisés** : `logs/access.log` pour Traefik
- ✅ **Rotation automatique** : configurer logrotate
- ✅ **Analyse régulière** : vérifier les tentatives d'accès
- ✅ **Alertes CrowdSec** : consulter `docker exec crowdsec cscli alerts list`

### 9. Sauvegardes

- ✅ **Automatisation** : Duplicati configuré pour backups réguliers
- ✅ **Chiffrement** : `DUPLICATI_SETTINGS_ENCRYPTION_KEY` obligatoire
- ✅ **Stockage externe** : ne pas sauvegarder sur le même serveur
- ✅ **Test de restauration** : vérifier régulièrement les backups

**Fichiers critiques à sauvegarder** :
```
- .env
- authelia/secrets/
- authelia/users.yml
- authelia/db.sqlite3
- config/
- letsencrypt/acme.json
```

### 10. Permissions des Fichiers

```bash
# Secrets Authelia
chmod 700 authelia/secrets/
chmod 600 authelia/secrets/*.txt

# Fichier .env
chmod 600 .env

# Base de données Authelia
chmod 600 authelia/db.sqlite3

# Certificats Let's Encrypt
chmod 600 letsencrypt/acme.json
```

## 🚨 Signalement de Vulnérabilités

### Procédure

Si vous découvrez une vulnérabilité de sécurité dans ce projet :

1. **NE PAS** créer d'issue publique
2. **Envoyer un email** à : atelier@ooblik.com
3. **Inclure** :
   - Description détaillée de la vulnérabilité
   - Étapes pour reproduire
   - Impact potentiel
   - Version affectée

### Délai de Réponse

- **Accusé de réception** : 48 heures
- **Analyse initiale** : 7 jours
- **Correctif** : selon la criticité (0-90 jours)

### Divulgation Responsable

Nous suivons le principe de **divulgation coordonnée** :
- Notification privée au mainteneur
- Correction développée et testée
- Publication du correctif
- Divulgation publique (CVE si applicable)

## 🔍 Audit de Sécurité

### Auto-Audit

Utilisez ce checklist pour vérifier votre installation :

```bash
# 1. Vérifier que .env n'est pas versionné
git check-ignore .env  # Doit afficher ".env"

# 2. Vérifier les permissions
stat -c "%a %n" .env authelia/secrets/*.txt letsencrypt/acme.json

# 3. Vérifier qu'aucun secret n'est présent dans Git
git log --all --full-history --source -- '*.env' 'authelia/secrets/*'

# 4. Scanner les ports exposés
docker compose ps

# 5. Vérifier les certificats SSL
echo | openssl s_client -connect auth.example.com:443 -servername auth.example.com 2>/dev/null | openssl x509 -noout -dates

# 6. Tester l'authentification 2FA
curl -I https://auth.example.com

# 7. Vérifier les décisions CrowdSec
docker exec crowdsec cscli decisions list
```

### Outils de Scan

```bash
# Scanner les vulnérabilités des images Docker
docker scan traefik:v3.3
docker scan authelia/authelia:latest

# Analyser les dépendances
trivy image traefik:v3.3
```

## 📋 Conformité

### RGPD / GDPR

- Données personnelles stockées : email, nom, hash de mot de passe
- Localisation : base de données SQLite locale (authelia/db.sqlite3)
- Chiffrement : hash Argon2id pour mots de passe
- Droit à l'oubli : supprimer l'utilisateur de users.yml et de la base

### Journalisation

- Logs Traefik : adresses IP, user-agents, URLs accédées
- Logs Authelia : tentatives de connexion, actions d'authentification
- Logs CrowdSec : IPs bannies, alertes de sécurité
- **Retention** : configurer rotation selon besoins légaux

## 🔗 Ressources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Authelia Security](https://www.authelia.com/overview/security/introduction/)
- [Traefik Security](https://doc.traefik.io/traefik/https/acme/)
- [CrowdSec Security](https://docs.crowdsec.net/)

## 📅 Historique des Versions

| Version | Date | Changements |
|---------|------|-------------|
| 1.0.0 | 2026-01-13 | Publication initiale |

---

**Note** : Ce document est mis à jour régulièrement. Dernière révision : 2026-01-13
