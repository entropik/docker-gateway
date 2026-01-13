# Tutoriel : Ajouter un nouveau service à votre Gateway Docker

> **Guide pratique pour exposer des applications web sur Internet avec SSL automatique et authentification 2FA**

**Durée estimée :** 30-45 minutes
**Niveau :** Débutant
**Auteur :** Communauté Docker Gateway
**Date :** Janvier 2026
**Version :** 1.0
**Licence :** CC BY-SA 4.0

---

## Table des Matières

1. [Introduction et Concepts](#1-introduction-et-concepts)
2. [Prérequis et Préparation](#2-prérequis-et-préparation)
3. [Méthode A - Service Interne via Dockge](#3-méthode-a---service-interne-via-dockge)
4. [Méthode B - Service Externe](#4-méthode-b---service-externe-via-fichiers-dynamiques)
5. [Dépannage](#5-dépannage)
6. [Concepts Avancés](#6-concepts-avancés)
7. [Bonnes Pratiques](#7-bonnes-pratiques)
8. [Exemples Autres Services](#8-exemples-autres-services)
9. [Ressources](#9-ressources)
10. [Conclusion](#10-conclusion)
11. [Annexes](#annexes)

---

# 1. Introduction et Concepts

## 1.1 Objectif du tutoriel

Ce tutoriel vous guide pas à pas pour **ajouter n'importe quel service Docker** à votre infrastructure Gateway, de manière sécurisée et accessible sur Internet.

**Ce que nous allons construire :**
- Service **Planka** (outil de gestion de projet type Trello/Kanban)
- Accessible sur **`projets.example.com`**
- Protégé par **authentification 2FA** (Authelia)
- Sécurisé avec **certificat SSL automatique** (Let's Encrypt)
- Protégé contre les **attaques** (CrowdSec)

**Compétences acquises :**
- Ajouter des services Docker via interface web (sans terminal)
- Configurer DNS et SSL automatique
- Appliquer sécurité 2FA sur vos services
- Diagnostiquer et résoudre les problèmes courants

---

## 1.2 Deux méthodes d'ajout de services

Il existe **deux façons** d'ajouter un service à votre Gateway :

| Critère | ✅ Méthode A: Service interne | ✅ Méthode B: Service externe |
|---------|--------------------------------|--------------------------------|
| **Localisation** | Même serveur (NAS1) | Autre serveur (NAS2, VM, etc.) |
| **Configuration** | Via Dockge (interface web) | Fichier YAML dans config/dynamic/ |
| **Gestion** | Labels Traefik dans docker-compose | Configuration statique |
| **Cas d'usage** | Nouveaux services à déployer | Services existants ailleurs |
| **Exemple** | Planka, Jellyfin, NextCloud | Immich sur NAS2, VM externe |

**Choix recommandé :** Méthode A si le service doit tourner sur votre serveur Gateway (le plus courant).

---

## 1.3 Comment ça marche ? (Schéma du flux)

Voici ce qui se passe quand quelqu'un accède à votre service :

```
Internet
   ↓
DNS (projets.example.com) → Résout vers 203.0.113.10 (IP publique)
   ↓
Freebox (NAT) → Redirige vers 192.168.1.100:8443
   ↓
Traefik (Reverse Proxy) → Lit le nom de domaine, route la requête
   ↓
CrowdSec → Vérifie si l'IP est malveillante
   ↓
Authelia → Vérifie si l'utilisateur est authentifié (2FA)
   ↓
Service (Planka) → Affiche l'application
```

---

## 1.4 Explication des composants

| Composant | Rôle | Pourquoi c'est important |
|-----------|------|--------------------------|
| **DNS** | Traduit `projets.example.com` en IP `203.0.113.10` | Sans DNS, personne ne peut trouver votre service |
| **NAT (Freebox)** | Redirige le trafic Internet vers votre serveur local | Permet d'exposer votre serveur privé sur Internet |
| **Traefik** | Reverse proxy qui route les requêtes vers le bon service | Un seul point d'entrée pour tous vos services |
| **Let's Encrypt** | Fournit des certificats SSL gratuits | Chiffre les communications (HTTPS) |
| **Authelia** | Serveur d'authentification avec 2FA | Protège vos services avec login + code 2FA |
| **CrowdSec** | Système de détection d'intrusions | Bloque automatiquement les IPs malveillantes |

---

# 2. Prérequis et Préparation

## 2.1 Checklist avant de commencer

Avant de démarrer, assurez-vous d'avoir accès à :

- [ ] **Interface Dockge** : `https://dockge.example.com`
- [ ] **Panneau DNS** de votre registrar (o2switch, OVH, Cloudflare, etc.)
- [ ] **File Station** du NAS Synology (pour Méthode B uniquement)
- [ ] **IP publique** de votre box Internet (ex: 203.0.113.10)
- [ ] **Domaine** configuré (ex: example.com)

---

## 2.2 Informations à préparer

Complétez ce tableau avant de commencer :

| Information | Exemple | Votre valeur |
|-------------|---------|--------------|
| **Nom du service** | planka | ___________ |
| **Sous-domaine souhaité** | projets | ___________ |
| **Domaine complet** | projets.example.com | ___________ |
| **IP publique** | 203.0.113.10 | ___________ |
| **Serveur de destination** | 192.168.1.100 (NAS1) | ___________ |
| **Port interne du service** | 1337 | ___________ |
| **Base de données nécessaire ?** | Oui (PostgreSQL) | ___________ |

---

## 2.3 Comprendre l'exemple : Planka

**Pourquoi Planka ?**
- Application web complète (nécessite une base de données PostgreSQL)
- Démontre la gestion des dépendances Docker
- Cas d'usage réel : gestion de projets en équipe (comme Trello)

**Architecture de Planka :**

```
Service Planka (port 1337)
    ↓ depends_on
Service PostgreSQL (port 5432)
    ↓ utilise
Volumes de données persistantes
```

**Caractéristiques :**
- Image Docker : `ghcr.io/plankanban/planka:latest`
- Port interne : 1337
- Base de données : PostgreSQL 16
- Volumes : uploads d'images, pièces jointes

---

# 3. Méthode A - Service Interne via Dockge

Cette méthode est **recommandée** pour ajouter un nouveau service qui tournera sur votre serveur Gateway (NAS1).

**Durée totale estimée :** 35 minutes

---

## 3.1 Étape 1: Configuration DNS (5 minutes)

### Pourquoi commencer par le DNS ?

Le DNS peut prendre **5 à 30 minutes** pour se propager globalement. En commençant par cette étape, le temps de propagation se fait pendant que vous configurez le service.

### Instructions détaillées

1. **Connectez-vous** à l'interface de votre registrar DNS (o2switch, OVH, Cloudflare, etc.)
2. **Accédez** à la section "Gestion DNS" ou "Zone DNS"
3. **Ajoutez** un nouvel enregistrement avec ces valeurs :

| Champ | Valeur | Explication |
|-------|--------|-------------|
| **Type** | A | Enregistrement d'adresse IPv4 |
| **Nom / Sous-domaine** | projets | Le préfixe avant votre domaine |
| **Valeur / Cible** | 203.0.113.10 | Votre IP publique (celle de la Freebox) |
| **TTL** | 3600 (1 heure) | Temps de mise en cache (plus court = changements plus rapides) |

4. **Cliquez** sur "Ajouter" ou "Sauvegarder"
5. **Attendez** quelques minutes

### Vérification de la propagation DNS

**Option 1 : En ligne (recommandé)**
- Allez sur [https://dnschecker.org](https://dnschecker.org)
- Entrez `projets.example.com`
- Vérifiez que l'IP affichée est bien `203.0.113.10`

**Option 2 : Navigateur**
- Ouvrez `https://projets.example.com` dans votre navigateur
- **Si vous voyez** "Ce site est inaccessible" ou "Connexion refusée" → DNS fonctionne ✅
- **Si vous voyez** "Nom de domaine introuvable" → DNS pas encore propagé, attendez encore

### Pourquoi cette étape est cruciale ?

Sans DNS fonctionnel, Traefik ne pourra pas générer de certificat SSL. Let's Encrypt a besoin de vérifier que vous contrôlez le domaine en interrogeant le DNS public.

---

## 3.2 Étape 2: Accéder à Dockge (5 minutes)

1. Ouvrez votre navigateur web
2. Allez sur `https://dockge.example.com`
3. **Authentification Authelia** :
   - Entrez votre login
   - Entrez votre mot de passe
   - Entrez le code 2FA depuis votre application (Google Authenticator, etc.)
4. Dans la liste des stacks, trouvez **"gateway"**
5. Cliquez sur le bouton **"Edit"** (icône crayon)

**Vous devriez voir :**
- En haut : `version: "3.9"`
- Section `services:` avec tous les conteneurs existants (traefik, authelia, heimdall, etc.)
- En bas : Section `networks:` avec le réseau `proxy`

---

## 3.3 Étape 3: Ajouter PostgreSQL (10 minutes)

### Pourquoi PostgreSQL d'abord ?

Planka dépend de la base de données. Docker doit démarrer PostgreSQL **avant** Planka grâce à l'instruction `depends_on`.

### Localisation dans le fichier

- Descendez jusqu'au **dernier service** (par exemple, après `duplicati:`)
- Placez-vous **avant** la section `networks:` (tout en bas)

### Code à copier-coller

```yaml
  planka-db:
    image: postgres:16-alpine
    container_name: planka-db
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - POSTGRES_DB=planka
      - POSTGRES_USER=planka
      - POSTGRES_PASSWORD=VOTRE_MOT_DE_PASSE_SECURISE_ICI
    volumes:
      - ./planka/db:/var/lib/postgresql/data
    networks:
      - proxy
```

### Explication ligne par ligne

| Ligne | Ce qu'elle fait | Pourquoi c'est important |
|-------|-----------------|--------------------------|
| `planka-db:` | Nom du service dans Docker | Planka se connectera via ce nom d'hôte |
| `image: postgres:16-alpine` | Version de PostgreSQL | Alpine = image légère, 16 = version stable LTS |
| `container_name: planka-db` | Nom du conteneur | Facilite l'identification dans `docker ps` |
| `restart: unless-stopped` | Politique de redémarrage | Redémarre automatiquement même après reboot du serveur |
| `TZ=Europe/Paris` | Timezone | Pour les logs et timestamps corrects |
| `POSTGRES_DB=planka` | Nom de la base de données | Planka s'y connectera |
| `POSTGRES_USER=planka` | Utilisateur PostgreSQL | Identifiant de connexion |
| `POSTGRES_PASSWORD=...` | Mot de passe | **DOIT** correspondre à la config Planka |
| `volumes: ./planka/db:...` | Persistance des données | Sans ça, toutes les données sont perdues au redémarrage |
| `networks: - proxy` | Réseau Docker | Permet communication avec Traefik et Planka |

### Points d'attention critiques

⚠️ **Remplacer `VOTRE_MOT_DE_PASSE_SECURISE_ICI`** par un vrai mot de passe fort
- Générez-le avec un gestionnaire de mots de passe
- Minimum 16 caractères avec majuscules, minuscules, chiffres et symboles
- Exemple : `Kp9$mL2#vR8!nQ4@wX6&`

⚠️ **Copiez ce mot de passe** quelque part (vous en aurez besoin pour Planka)

⚠️ **L'indentation est cruciale** en YAML :
- Utilisez 2 espaces par niveau (jamais de tabulations)
- `planka-db:` doit être au même niveau que les autres services
- Les propriétés sous `planka-db:` sont indentées de 2 espaces

---

## 3.4 Étape 4: Ajouter Planka avec labels Traefik (15 minutes)

### Localisation

Juste **après** la section PostgreSQL que vous venez d'ajouter.

### Code à copier-coller

```yaml
  planka:
    image: ghcr.io/plankanban/planka:latest
    container_name: planka
    restart: unless-stopped
    depends_on:
      - planka-db
    environment:
      - TZ=Europe/Paris
      - BASE_URL=https://projets.example.com
      - DATABASE_URL=postgresql://planka:VOTRE_MOT_DE_PASSE@planka-db:5432/planka
      - SECRET_KEY=GENERER_UNE_CLE_SECRETE_LONGUE_ICI
    volumes:
      - ./planka/uploads:/app/public/user-avatars
      - ./planka/attachments:/app/public/project-background-images
      - ./planka/attachments:/app/private/attachments
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.planka.rule=Host(`projets.example.com`)"
      - "traefik.http.routers.planka.entrypoints=websecure"
      - "traefik.http.routers.planka.tls.certresolver=letsencrypt"
      - "traefik.http.routers.planka.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.planka.loadbalancer.server.port=1337"
    networks:
      - proxy
```

### Explication des variables d'environnement

| Variable | Valeur | Explication | Personnaliser |
|----------|--------|-------------|---------------|
| `TZ` | Europe/Paris | Timezone | Selon votre localisation |
| `BASE_URL` | https://projets.example.com | URL publique | **Obligatoire** : votre domaine complet |
| `DATABASE_URL` | postgresql://planka:PASS@planka-db:5432/planka | Connexion PostgreSQL | Remplacer PASS par le mot de passe PostgreSQL |
| `SECRET_KEY` | Clé aléatoire longue | Signature des sessions | Générer une clé de 64 caractères |

### Comment générer SECRET_KEY ?

**Option 1 : UUID Generator (recommandé)**
1. Allez sur [https://www.uuidgenerator.net/version4](https://www.uuidgenerator.net/version4)
2. Générez 2-3 UUIDs
3. Concaténez-les bout à bout
4. Exemple : `d4f7e8c9-2a1b-4c3d-9e8f-7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0`

**Option 2 : Gestionnaire de mots de passe**
- Générez un mot de passe de 64 caractères (lettres + chiffres)

### Explication des labels Traefik (la magie de l'exposition)

Les labels Traefik indiquent à Traefik comment router le trafic vers votre service.

| Label | Fonction | Pourquoi c'est important |
|-------|----------|--------------------------|
| `traefik.enable=true` | Active Traefik pour ce conteneur | Par défaut, Traefik ignore tous les conteneurs |
| `rule=Host(projets.example.com)` | Condition de routage | "Si quelqu'un demande projets.example.com, c'est ici qu'il faut router" |
| `entrypoints=websecure` | Utilise HTTPS (port 443) | Sécurise la connexion avec SSL/TLS |
| `tls.certresolver=letsencrypt` | Active SSL automatique | Génère et renouvelle automatiquement le certificat |
| `middlewares=authelia@file,crowdsec@file` | Active la protection | Authentification 2FA + protection anti-bot |
| `loadbalancer.server.port=1337` | Port interne de Planka | Traefik sait vers quel port router les requêtes |

### Pourquoi `@file` dans les middlewares ?

Les middlewares `authelia` et `crowdsec` sont **définis dans des fichiers YAML** dans `config/dynamic/`, pas dans les labels Docker. Le suffixe `@file` indique à Traefik de les chercher dans les fichiers dynamiques.

### Explication des volumes (persistance des données)

| Volume | Ce qui est stocké | Pourquoi persistant |
|--------|-------------------|---------------------|
| `./planka/uploads:...` | Avatars des utilisateurs | Gardés après redémarrage |
| `./planka/attachments:...` | Images et fichiers des cartes | Gardés après redémarrage |

Tous ces dossiers seront créés automatiquement sur le NAS dans `/volume1/docker/gateway/planka/`.

### Explication de `depends_on`

```yaml
depends_on:
  - planka-db
```

Cette ligne indique à Docker : **"Ne démarre pas Planka avant que planka-db soit démarré"**. C'est crucial pour éviter les erreurs de connexion à la base de données.

---

## 3.5 Étape 5: Vérifier la configuration (5 minutes)

Avant de lancer le déploiement, **vérifiez attentivement** :

### Checklist de vérification visuelle

**1. Indentation correcte**
- [ ] Chaque niveau est indenté de **2 espaces** (pas de tabulations)
- [ ] Les tirets des labels sont alignés verticalement
- [ ] Les sections `services:` et `networks:` sont au même niveau d'indentation

**2. Pas de fautes de frappe**
- [ ] `projets.example.com` est correct (pas de `.com` manquant)
- [ ] `authelia@file` et `crowdsec@file` (pas `@docker`)
- [ ] Réseau `proxy` écrit partout de la même façon

**3. Secrets remplacés**
- [ ] `POSTGRES_PASSWORD` contient un **vrai** mot de passe (pas le placeholder)
- [ ] `DATABASE_URL` contient le **même** mot de passe que PostgreSQL
- [ ] `SECRET_KEY` est définie (pas de texte "GENERER...")

**4. Syntaxe YAML valide**
- [ ] Dockge n'affiche pas d'erreur de syntaxe (ligne rouge)
- [ ] Pas d'avertissement dans l'interface

### Astuce : Validateur YAML en ligne

Si vous avez un doute sur la syntaxe :
1. Copiez tout le contenu de votre docker-compose.yml
2. Allez sur [https://www.yamllint.com/](https://www.yamllint.com/)
3. Collez le contenu
4. Vérifiez qu'il n'y a pas d'erreurs

---

## 3.6 Étape 6: Déployer la stack (5 minutes)

### Lancement du déploiement

1. Dans Dockge, cliquez sur le bouton **"Save"** (en haut de l'éditeur)
2. Une fois sauvegardé, cliquez sur **"Restart"** ou **"Up"**
3. **Observez les logs** en temps réel dans Dockge

### Logs attendus pour un démarrage réussi

**Pour `planka-db` :**
```
PostgreSQL init process complete; ready for start up.
database system is ready to accept connections
```

**Pour `planka` :**
```
Server listening on port 1337
Connected to database successfully
```

**Pour `traefik` (si vous regardez ses logs) :**
```
Creating certificate for projets.example.com
Certificate obtained for projets.example.com
Router planka@docker registered
```

### Temps d'attente estimés

| Composant | Temps de démarrage | Remarque |
|-----------|--------------------|-----------|
| PostgreSQL | 5-15 secondes | Doit démarrer en premier |
| Planka | 10-30 secondes | Attend que PostgreSQL soit prêt |
| Certificat SSL | 30-60 secondes | **Uniquement** la première fois |

### Si vous voyez des erreurs

**Ne paniquez pas !** C'est normal lors de la première configuration. Passez à la **Section 5 : Dépannage** pour identifier et résoudre le problème.

---

## 3.7 Étape 7: Tester l'accès (5 minutes)

### Test d'accès complet

1. Ouvrez votre navigateur web
2. Allez sur `https://projets.example.com`

### Premier écran attendu : Authelia

Vous devriez être **automatiquement redirigé** vers :
```
https://auth.example.com
```

**Page de connexion Authelia :**
- Entrez votre nom d'utilisateur
- Entrez votre mot de passe
- Entrez le code 2FA depuis votre application (Google Authenticator, Authy, etc.)

### Redirection automatique

Après authentification réussie, vous êtes **automatiquement redirigé** vers :
```
https://projets.example.com
```

### Page Planka attendue

Vous devriez voir :
- **Interface de création de compte** Planka
- Design minimaliste avec formulaire d'inscription
- Logo Planka en haut

### Vérifications de sécurité

Vérifiez ces éléments dans votre navigateur :

- [ ] **Cadenas vert** dans la barre d'adresse (HTTPS valide)
- [ ] **Certificat émis par "Let's Encrypt"** (cliquez sur le cadenas)
- [ ] **Authelia a demandé l'authentification 2FA** (login + mot de passe + code)
- [ ] **Pas de port visible** dans l'URL (pas de `:8443` ou autre)

### Configuration initiale de Planka

1. **Créez un compte administrateur**
   - Nom d'utilisateur
   - Email
   - Mot de passe
2. **Créez votre premier projet**
   - Donnez-lui un nom (ex: "Test")
3. **Ajoutez une carte de test**
   - Vérifiez que tout fonctionne

---

## 🎉 Félicitations !

Votre service Planka est maintenant :
- ✅ Accessible sur Internet via `https://projets.example.com`
- ✅ Protégé par authentification 2FA (Authelia)
- ✅ Sécurisé avec un certificat SSL valide (Let's Encrypt)
- ✅ Protégé contre les attaques (CrowdSec)
- ✅ Fonctionnel avec base de données PostgreSQL

Vous pouvez maintenant utiliser Planka pour gérer vos projets en équipe !

---

# 4. Méthode B - Service Externe via Fichiers Dynamiques

Cette méthode est utilisée pour **exposer un service qui tourne sur un autre serveur** (NAS2, VM, serveur distant, etc.) via votre Gateway.

**Durée totale estimée :** 25 minutes

---

## 4.1 Quand utiliser cette méthode ?

### Cas d'usage courants

| Situation | Exemple concret |
|-----------|-----------------|
| **Service sur autre serveur** | Immich qui tourne sur NAS2 (192.168.1.50) |
| **VM ou conteneur ailleurs** | Service dans une VM Proxmox |
| **Service legacy existant** | Serveur web Apache déjà en production |
| **Application non-Docker** | Nginx traditionnel, serveur Node.js |
| **Service sur autre stack Docker** | Service dans un docker-compose séparé |

### Exemple concret pour ce tutoriel

**Scénario :** Planka tourne déjà sur NAS2 à l'adresse `192.168.1.50:1337`, et vous voulez l'exposer via votre Gateway NAS1 sur `https://projets.example.com`.

---

## 4.2 Étape 1: Configuration DNS

**Identique à la Méthode A** (voir Section 3.1)

Créez un enregistrement DNS A :
- Type : A
- Nom : projets
- Valeur : 203.0.113.10
- TTL : 3600

Vérifiez sur [dnschecker.org](https://dnschecker.org) que `projets.example.com` pointe bien vers votre IP publique.

---

## 4.3 Étape 2: Vérifier l'accessibilité du service cible (5 minutes)

### Pourquoi cette étape ?

Avant de configurer Traefik, il faut s'assurer que le service distant **est accessible** depuis le serveur Gateway (NAS1).

### Test depuis NAS1

**Option 1 : Navigateur sur le NAS**
1. Ouvrez un navigateur sur votre NAS (via DSM)
2. Allez sur `http://192.168.1.50:1337`
3. Vous devriez voir l'interface Planka

**Option 2 : Via Dockge terminal (console conteneur Traefik)**
1. Dans Dockge, sélectionnez le conteneur "traefik"
2. Ouvrez le terminal/console
3. Tapez : `curl http://192.168.1.50:1337`
4. Vous devriez voir du HTML (code de la page Planka)

### Si le service n'est pas accessible

**Vérifications :**
1. **Service tourne sur NAS2 ?**
   - Connectez-vous en SSH sur NAS2
   - Tapez `docker ps` et vérifiez que Planka est en cours d'exécution

2. **Pare-feu sur NAS2 ?**
   - Vérifiez que NAS2 autorise les connexions depuis `192.168.1.100` (NAS1)
   - Sur Synology : Panneau de configuration → Sécurité → Pare-feu

3. **Port correct ?**
   - Vérifiez que Planka écoute bien sur le port 1337
   - Consultez la documentation ou les logs de Planka

---

## 4.4 Étape 3: Créer le fichier de configuration dynamique (10 minutes)

### Structure des fichiers dynamiques

Les fichiers dynamiques Traefik sont des **fichiers YAML** stockés dans :
```
/volume1/docker/gateway/config/dynamic/
```

Traefik surveille automatiquement ce dossier et **recharge les configurations** dès qu'un fichier est modifié (sans redémarrage).

### Accéder au dossier

**Option 1 : File Station (Synology DSM)**
1. Ouvrez DSM dans votre navigateur
2. Lancez l'application "File Station"
3. Naviguez vers `/docker/gateway/config/dynamic/`

**Option 2 : Montage réseau Windows**
1. Explorateur de fichiers Windows
2. Tapez dans la barre d'adresse : `\\stock3\docker\gateway\config\dynamic\`
3. Entrez vos identifiants Synology

### Créer le fichier

1. Dans le dossier `dynamic/`, créez un **nouveau fichier** nommé `planka.yml`
2. Ouvrez-le avec l'éditeur intégré de File Station (ou Notepad++)

### Contenu du fichier planka.yml

Copiez-collez ce contenu :

```yaml
http:
  routers:
    planka:
      rule: "Host(`projets.example.com`)"
      entryPoints:
        - websecure
      service: planka
      tls:
        certResolver: letsencrypt
      middlewares:
        - authelia
        - crowdsec

  services:
    planka:
      loadBalancer:
        servers:
          - url: "http://192.168.1.50:1337"
```

### Explication de la structure

#### Section `routers:` (routage des requêtes)

| Champ | Valeur | Explication |
|-------|--------|-------------|
| `planka:` | Nom arbitraire du routeur | Utilisé dans les logs Traefik pour l'identification |
| `rule:` | `Host(projets.example.com)` | Condition de routage : matcher le domaine exact |
| `entryPoints: [websecure]` | Port 443 HTTPS | Point d'entrée HTTPS de Traefik |
| `service: planka` | Référence au service backend (défini plus bas) | Indique vers quel backend router le trafic |
| `tls: certResolver: letsencrypt` | SSL automatique | Génère et renouvelle le certificat SSL |
| `middlewares: [authelia, crowdsec]` | Protection | Authentification 2FA + anti-intrusions |

#### Section `services:` (backends)

| Champ | Valeur | Explication |
|-------|--------|-------------|
| `planka:` | Nom du service (doit correspondre au router) | Définit le backend de destination |
| `loadBalancer: servers:` | Liste des serveurs backend | Peut contenir plusieurs URLs (load balancing) |
| `- url: "http://IP:PORT"` | Adresse réelle du service | Traefik fait un proxy inverse vers cette URL |

### Différences avec les labels Docker

| Aspect | Labels Docker | Fichiers dynamiques |
|--------|---------------|---------------------|
| **Middlewares** | `authelia@file,crowdsec@file` | `authelia, crowdsec` (sans @file) |
| **Structure** | Labels plats (clé-valeur) | YAML hiérarchique |
| **Rechargement** | Nécessite restart du conteneur | **Automatique** (surveillé par Traefik) |
| **Localisation** | Dans docker-compose.yml | Fichier séparé dans config/dynamic/ |
| **Cas d'usage** | Services dans le même docker-compose | Services externes ou réutilisables |

---

## 4.5 Étape 4: Enregistrer et vérifier le chargement (5 minutes)

### Sauvegarde du fichier

1. **Sauvegardez** le fichier `planka.yml` dans File Station
2. Traefik **détecte automatiquement** le nouveau fichier (surveillance du dossier)

### Vérifier que Traefik a chargé la configuration

**Option 1 : Logs Traefik dans Dockge**
1. Allez dans Dockge
2. Sélectionnez le conteneur "traefik"
3. Regardez les logs en temps réel
4. Cherchez une ligne contenant :
   ```
   Configuration loaded from file
   Router planka@file added
   ```

**Option 2 : Dashboard Traefik (recommandé)**
1. Ouvrez `https://trafik.example.com`
2. Authentifiez-vous avec Authelia (2FA)
3. Allez dans la section **"HTTP" → "Routers"**
4. Cherchez le routeur nommé **`planka@file`**
5. Vérifiez qu'il a le statut **OK** (vert)

### Résultat attendu

Vous devriez voir dans le dashboard Traefik :
- **Nom :** `planka@file` (le suffixe `@file` indique qu'il vient d'un fichier dynamique)
- **Status :** ✅ OK (vert)
- **Rule :** Host(`projets.example.com`)
- **EntryPoints :** websecure
- **Middlewares :** authelia@file, crowdsec@file
- **Service :** planka@file

---

## 4.6 Étape 5: Tester l'accès

**Le processus de test est identique à la Méthode A** (voir Section 3.7).

1. Ouvrez `https://projets.example.com`
2. Authentifiez-vous via Authelia (2FA)
3. Vérifiez que vous accédez bien à Planka

### Particularité

Dans les logs Traefik, vous verrez :
- `planka@file` au lieu de `planka@docker`

Le suffixe `@file` indique que la configuration vient d'un fichier dynamique.

---

## 4.7 Quand utiliser quelle méthode ? (Tableau décisionnel)

| Situation | Méthode recommandée | Raison |
|-----------|---------------------|--------|
| **Nouveau service à déployer** | A (Dockge) | Tout géré au même endroit, facilité de gestion |
| **Service sur NAS1 mais hors docker-compose gateway** | B (Fichier dynamique) | Service dans sa propre stack Docker |
| **Service sur autre serveur (NAS2, VM, etc.)** | B (Fichier dynamique) | Pas d'autre choix, service distant |
| **Service non-Docker (Apache, Nginx, etc.)** | B (Fichier dynamique) | Ne peut pas être dans docker-compose |
| **Besoin de load balancing (plusieurs backends)** | B (Fichier dynamique) | Peut lister plusieurs serveurs |
| **Service temporaire / test** | A (Dockge) | Facile à ajouter et supprimer |

---

# 5. Dépannage

Cette section vous guide pour **diagnostiquer et résoudre** les problèmes courants lors de l'ajout d'un service.

---

## 5.1 Erreur : 404 Page Not Found

### Symptôme

Page blanche affichant :
```
404 page not found
```
(Message de Traefik)

### Causes possibles et solutions

| Cause | Comment vérifier | Solution |
|-------|------------------|----------|
| **DNS pas encore propagé** | [dnschecker.org](https://dnschecker.org) | Attendre 5-30 minutes |
| **Middleware introuvable** | Logs Traefik : "middleware not found" | Vérifier `@file` vs `@docker` |
| **Routeur mal configuré** | Dashboard Traefik : routeur absent | Corriger la syntaxe YAML |
| **Conteneur pas démarré** | Dockge : conteneur rouge/arrêté | Lire les logs, corriger l'erreur, restart |

### Procédure de diagnostic étape par étape

**1. Vérifier la résolution DNS**
```
- Allez sur https://dnschecker.org
- Entrez le domaine complet (projets.example.com)
- Vérifiez que tous les serveurs affichent 203.0.113.10
```

**2. Vérifier que Traefik a chargé le routeur**
```
- Allez sur https://trafik.example.com
- Section "HTTP" → "Routers"
- Cherchez "planka@docker" ou "planka@file"
- Si absent : problème de configuration
```

**3. Vérifier que le conteneur est bien démarré**
```
- Dans Dockge, vérifiez que les conteneurs planka et planka-db sont verts
- Si rouges : cliquez dessus, lisez les logs d'erreur
```

**4. Vérifier les logs Traefik**
```
- Dans Dockge, sélectionnez le conteneur "traefik"
- Cherchez des erreurs mentionnant votre service
- Messages courants :
  - "middleware authelia@file not found" → Vérifier l'orthographe et la définition du middleware
  - "service planka not found" → Problème dans la section services
```

---

## 5.2 Erreur : 502 Bad Gateway

### Symptôme

Page Traefik affichant :
```
502 Bad Gateway
```

### Signification

Traefik reçoit bien la requête mais **ne peut pas joindre le service backend** (le conteneur/serveur cible).

### Causes possibles et solutions

| Cause | Comment vérifier | Solution |
|-------|------------------|----------|
| **Service pas démarré** | Dockge : conteneur arrêté | Vérifier les logs, corriger l'erreur, restart |
| **Mauvais port configuré** | Logs du service : quel port écoute-t-il ? | Corriger le port dans labels/config |
| **Service pas sur le réseau `proxy`** | `docker inspect planka` | Ajouter `networks: - proxy` |
| **IP incorrecte (Méthode B)** | Tester `curl http://IP:PORT` depuis traefik | Corriger l'IP dans le fichier YAML |
| **Service en cours de démarrage** | Attendre 30 secondes | Recharger la page |

### Procédure de diagnostic

**1. Vérifier que le conteneur cible est bien démarré**
```
- Dans Dockge, vérifiez que le conteneur est vert
- Vérifiez les logs pour "Server listening" ou "Ready"
```

**2. Vérifier le port interne**
```
- Les logs du service indiquent généralement "Listening on port XXXX"
- Ce port doit correspondre à `loadbalancer.server.port` dans la config
```

**3. Vérifier le réseau Docker (services internes)**
```
- Dans docker-compose.yml, vérifiez que `networks: - proxy` est présent
- Les deux conteneurs doivent être sur le même réseau
```

**4. Tester la connectivité directe (Méthode B)**
```
- Depuis un autre conteneur ou le serveur : curl http://IP:PORT
- Si timeout : problème réseau ou pare-feu
- Si "Connection refused" : service pas démarré ou mauvais port
```

---

## 5.3 Erreur : Certificat SSL invalide

### Symptôme

Page d'avertissement du navigateur :
```
Votre connexion n'est pas privée
NET::ERR_CERT_AUTHORITY_INVALID
```

### Causes possibles et solutions

| Cause | Comment vérifier | Solution |
|-------|------------------|----------|
| **Certificat pas encore généré** | Premier accès (< 1 minute) | Attendre 30-60 secondes, recharger la page |
| **Port 80 bloqué** | Logs Traefik : "acme challenge failed" | Vérifier redirection port 80 sur la box |
| **DNS pas encore propagé** | `nslookup` depuis Internet | Attendre la propagation complète |
| **Rate limit Let's Encrypt** | Logs : "too many certificates" | Attendre 1 semaine (max 5 cert/domaine/semaine) |
| **Domaine en .local ou .test** | Impossible d'obtenir certificat public | Utiliser un vrai domaine public |

### Procédure de diagnostic

**1. Vérifier les logs Traefik pour le processus ACME**
```
Logs Traefik → Chercher "certificate" ou "acme"
Messages attendus :
- "Requesting certificate for projets.example.com"
- "Certificate obtained for projets.example.com"
```

**2. Vérifier que le port 80 est accessible depuis Internet**
```
Let's Encrypt a besoin d'accéder à :
http://projets.example.com/.well-known/acme-challenge/...

Vérifier la redirection NAT sur la Freebox :
Internet:80 → 192.168.1.100:8080
```

**3. Vérifier le rate limit Let's Encrypt**
```
Let's Encrypt limite à 5 certificats par domaine par semaine
Si dépassé : attendre ou utiliser un sous-domaine différent
```

**4. Attendre et réessayer**
```
La première génération peut prendre 1-2 minutes
Rafraîchir la page toutes les 30 secondes
```

---

## 5.4 Erreur : Boucle de redirection Authelia

### Symptôme

Le navigateur affiche :
```
Cette page ne fonctionne pas
Trop de redirections
ERR_TOO_MANY_REDIRECTS
```

### Cause unique

Le domaine `auth.example.com` est lui-même protégé par le middleware Authelia, créant une **boucle infinie**.

### Explication du problème

```
User → trafik.example.com
  → Traefik vérifie avec Authelia → Utilisateur pas connecté
  → Redirige vers auth.example.com
  → auth.example.com vérifie avec Authelia → Pas connecté
  → Redirige vers auth.example.com (BOUCLE INFINIE)
```

### Solution

Le service `authelia` dans docker-compose.yml **NE DOIT PAS** avoir le middleware `authelia@file` dans ses labels.

**MAUVAIS (cause la boucle) :**
```yaml
labels:
  - "traefik.http.routers.authelia.middlewares=authelia@file,crowdsec@file"
```

**CORRECT :**
```yaml
labels:
  - "traefik.http.routers.authelia.middlewares=crowdsec@file"
  # OU sans middleware du tout (juste crowdsec pour anti-bot)
```

---

## 5.5 Erreur : Base de données inaccessible

### Symptôme

Logs du service affichant :
```
Connection refused
Database connection failed
ECONNREFUSED 127.0.0.1:5432
```

### Causes possibles et solutions

| Cause | Comment vérifier | Solution |
|-------|------------------|----------|
| **DB pas démarrée** | Dockge : conteneur DB arrêté | Vérifier les logs de la DB, corriger l'erreur |
| **Mot de passe incorrect** | Comparer `POSTGRES_PASSWORD` et `DATABASE_URL` | Corriger pour qu'ils correspondent |
| **Service démarre avant la DB** | `depends_on` absent | Ajouter `depends_on: - planka-db` |
| **Mauvais hostname** | `DATABASE_URL` contient "localhost" | Utiliser le nom du service Docker (planka-db) |
| **Pas sur le même réseau** | `networks` différents | Ajouter `networks: - proxy` aux deux services |

### Procédure de diagnostic

**1. Vérifier que la base de données est bien démarrée**
```
- Dans Dockge, vérifiez que le conteneur de la DB est vert
- Regarder les logs : "database system is ready to accept connections"
```

**2. Vérifier la chaîne de connexion DATABASE_URL**
```yaml
DATABASE_URL=postgresql://USER:PASSWORD@HOST:PORT/DATABASE

Exemple correct :
DATABASE_URL=postgresql://planka:monmotdepasse@planka-db:5432/planka

Erreurs courantes :
- localhost au lieu de planka-db (dans Docker, utiliser le nom du service)
- Mot de passe différent entre POSTGRES_PASSWORD et DATABASE_URL
- Port incorrect (5432 pour PostgreSQL, 3306 pour MySQL)
```

**3. Vérifier le `depends_on`**
```yaml
planka:
  depends_on:
    - planka-db  # Force Docker à démarrer la DB avant l'appli
```

**4. Attendre le démarrage complet**
```
PostgreSQL peut prendre 10-30 secondes pour être vraiment prêt
Le service applicatif peut nécessiter plusieurs tentatives de connexion
```

---

## 5.6 Checklist de diagnostic générale

Quand un service ne fonctionne pas, suivez cet ordre :

1. [ ] **DNS résout-il vers la bonne IP ?** (dnschecker.org)
2. [ ] **La redirection NAT est-elle configurée ?** (ports 80/443 sur la box)
3. [ ] **Le conteneur est-il démarré ?** (Dockge : vert)
4. [ ] **Y a-t-il des erreurs dans les logs ?** (Dockge : logs)
5. [ ] **Le routeur Traefik existe-t-il ?** (Dashboard Traefik)
6. [ ] **Les middlewares sont-ils correctement référencés ?** (`@file` vs `@docker`)
7. [ ] **Le service est-il sur le réseau `proxy` ?**
8. [ ] **Le port dans `loadbalancer.server.port` est-il correct ?**
9. [ ] **Le certificat SSL a-t-il été généré ?** (Logs Traefik ACME)
10. [ ] **Votre IP n'est-elle pas bannie par CrowdSec ?** (cscli decisions list)

---

# 6. Concepts Avancés

Cette section couvre des configurations plus avancées pour personnaliser vos services.

---

## 6.1 Modifier les niveaux de protection

### Options de protection disponibles

| Configuration | Cas d'usage | Middlewares |
|---------------|-------------|-------------|
| **Public (sans auth)** | Blog public, site vitrine | `crowdsec@file` uniquement |
| **Mot de passe simple** | Service interne non critique | `one_factor` dans Authelia + crowdsec |
| **2FA complet (RECOMMANDÉ)** | Services sensibles (admin, données) | `authelia@file,crowdsec@file` |
| **Complètement ouvert (DÉCONSEILLÉ)** | Debug temporaire uniquement | Aucun middleware |

### Comment rendre un service public (sans Authelia)

**Méthode 1 : Modifier les labels Docker (services internes)**
```yaml
labels:
  - "traefik.http.routers.planka.middlewares=crowdsec@file"
  # Supprimer authelia@file
```

**Méthode 2 : Modifier le fichier dynamique (services externes)**
```yaml
middlewares:
  - crowdsec  # Seulement anti-bot, pas d'auth
```

**Méthode 3 : Configuration Authelia (bypass pour domaine spécifique)**
```yaml
# Dans authelia/configuration.yml
access_control:
  rules:
    - domain: projets.example.com
      policy: bypass  # Pas d'authentification requise
```

⚠️ **Attention :** Un service public est accessible par **n'importe qui sur Internet**. Assurez-vous qu'il dispose de sa propre authentification interne ou que c'est bien voulu.

---

## 6.2 Ajouter un service sans base de données

### Template simplifié

Pour les services **standalone** (sans dépendances) comme Heimdall, Uptime Kuma, dashboard statique, etc.

```yaml
  mon-service:
    image: organisation/nom-image:latest
    container_name: mon-service
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - PUID=1000
      - PGID=1000
    volumes:
      - ./mon-service:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mon-service.rule=Host(`subdomain.example.com`)"
      - "traefik.http.routers.mon-service.entrypoints=websecure"
      - "traefik.http.routers.mon-service.tls.certresolver=letsencrypt"
      - "traefik.http.routers.mon-service.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.mon-service.loadbalancer.server.port=XXXX"
    networks:
      - proxy
```

### Éléments à personnaliser

1. **`mon-service`** → Nom de votre service (3 occurrences : service, container_name, router)
2. **`organisation/nom-image`** → Image Docker officielle (ex: `linuxserver/heimdall`)
3. **`subdomain.example.com`** → Votre sous-domaine souhaité
4. **`XXXX`** → Port interne de l'application (voir documentation de l'image)
5. **`./mon-service:/config`** → Chemin de persistance des données

### Comment trouver le port interne ?

- **Documentation officielle** de l'image Docker
- **Docker Hub** : page de l'image, section "Ports" ou "Expose"
- **Commande EXPOSE** dans le Dockerfile

---

## 6.3 Gérer plusieurs domaines

### Cas d'usage

Vous avez plusieurs domaines (ex: `example.com` et `exemple.fr`) tous deux pointant vers le même serveur, et vous voulez qu'un service soit accessible sur les deux.

### Solution : Règle OR dans Traefik

**Pour les labels Docker :**
```yaml
- "traefik.http.routers.planka.rule=Host(`projets.example.com`) || Host(`projets.exemple.fr`)"
```

**Pour les fichiers dynamiques :**
```yaml
rule: "Host(`projets.example.com`) || Host(`projets.exemple.fr`)"
```

### Certificat SSL

Let's Encrypt générera automatiquement un **certificat SAN** (Subject Alternative Name) couvrant les deux domaines.

---

## 6.4 Load balancing entre plusieurs backends

### Cas d'usage

Vous avez le même service déployé sur **plusieurs serveurs** (haute disponibilité, répartition de charge).

### Configuration dans fichier dynamique

```yaml
services:
  planka:
    loadBalancer:
      servers:
        - url: "http://192.168.1.50:1337"
        - url: "http://192.168.1.121:1337"
        - url: "http://192.168.1.122:1337"
      healthCheck:
        path: /health
        interval: 10s
        timeout: 3s
```

### Explications

- **Traefik distribue** les requêtes entre les 3 serveurs (round-robin par défaut)
- **healthCheck** : Si un serveur ne répond pas, il est temporairement retiré de la rotation
- **Répartition automatique** de la charge entre les backends disponibles

---

## 6.5 Redirection automatique HTTP → HTTPS

### Problème actuel

Si quelqu'un tape `http://projets.example.com` (sans le "s"), ça ne fonctionne pas.

### Solution : Middleware de redirection global

Créez le fichier `config/dynamic/http-to-https.yml` :

```yaml
http:
  middlewares:
    redirect-to-https:
      redirectScheme:
        scheme: https
        permanent: true

  routers:
    http-catchall:
      rule: "HostRegexp(`{host:.+}`)"
      entryPoints:
        - web
      middlewares:
        - redirect-to-https
      service: noop@internal
```

### Effet

- Toute requête HTTP (port 80) est automatiquement redirigée en HTTPS (port 443)
- Status 301 (permanent redirect) pour les moteurs de recherche

---

# 7. Bonnes Pratiques

Cette section couvre les **meilleures pratiques** pour gérer vos services de manière sécurisée et maintenable.

---

## 7.1 Gestion des secrets

### Problème actuel

Actuellement, les mots de passe sont en **clair dans docker-compose.yml**.

### Solution recommandée : Fichier .env

**Créer le fichier `/volume1/docker/gateway/.env` :**

```env
DOMAIN=example.com
ACME_EMAIL=votre@email.com
POSTGRES_PASSWORD_PLANKA=motdepasse_securise_ici
SECRET_KEY_PLANKA=cle_secrete_longue_ici
CROWDSEC_BOUNCER_API_KEY=cle_api_crowdsec
```

**Modifier docker-compose.yml :**

```yaml
  planka-db:
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD_PLANKA}

  planka:
    environment:
      - DATABASE_URL=postgresql://planka:${POSTGRES_PASSWORD_PLANKA}@planka-db:5432/planka
      - SECRET_KEY=${SECRET_KEY_PLANKA}
```

### Avantages

- ✅ Secrets **hors du docker-compose**
- ✅ Fichier `.env` peut être **ignoré par Git**
- ✅ Facile à gérer et à sauvegarder séparément
- ✅ Pas besoin d'outils externes

---

## 7.2 Sauvegarde des configurations

### Que sauvegarder ?

| Dossier / Fichier | Importance | Fréquence recommandée |
|-------------------|------------|------------------------|
| `config/` | ⭐⭐⭐ Critique | Après chaque modification |
| `authelia/` | ⭐⭐⭐ Critique | Hebdomadaire |
| `letsencrypt/` | ⭐⭐ Important | Quotidienne |
| `planka/db/` | ⭐⭐⭐ Critique | Quotidienne |
| `docker-compose.yml` | ⭐⭐⭐ Critique | Après chaque modification |
| `.env` | ⭐⭐⭐ Critique | Après chaque modification |

### Méthode simple via Duplicati

Duplicati est déjà installé dans votre Gateway (`https://backup.example.com`).

**Configuration recommandée :**
1. Créer un job de sauvegarde
2. **Source** : `/volume1/docker/gateway/`
3. **Destination** : Stockage externe (autre NAS, cloud, disque USB)
4. **Planification** : Quotidienne à 3h du matin
5. **Rétention** : Garder 30 versions

---

## 7.3 Nomenclature et conventions

### Pour la cohérence de l'infrastructure

**Noms des services Docker :**
- ✅ Tout en **minuscules**
- ✅ Traits d'union pour séparer les mots : `planka-db`
- ✅ Suffixe `-db` pour les bases de données
- ✅ Suffixe `-cache` pour Redis/Memcached
- ✅ Suffixe `-worker` pour les workers/queues

**Sous-domaines :**
- ✅ Court et descriptif : `projets` plutôt que `gestion-de-projets`
- ✅ Éviter les caractères spéciaux
- ✅ Préférer l'anglais pour la cohérence (mais français OK)

**Fichiers dynamiques :**
- ✅ Nom du service `.yml` : `planka.yml`
- ✅ Un fichier par service (sauf middlewares globaux)
- ✅ Noms en minuscules

**Volumes :**
- ✅ Structure claire : `./nom-du-service/sous-dossier:/chemin/dans/conteneur`
- ✅ Exemple : `./planka/db:/var/lib/postgresql/data`

---

## 7.4 Surveillance et maintenance

### Actions régulières recommandées

**Hebdomadaire :**
- [ ] Vérifier les logs CrowdSec : `docker exec crowdsec cscli alerts list`
- [ ] Vérifier les notifications Diun (mises à jour disponibles)
- [ ] Tester l'accès aux services critiques

**Mensuel :**
- [ ] Mettre à jour les images Docker (via Dockge : Pull → Restart)
- [ ] Vérifier l'espace disque utilisé : `du -sh /volume1/docker/gateway/*`
- [ ] Vérifier les certificats SSL (renouvellement automatique avant expiration)
- [ ] Tester les sauvegardes (restauration sur environnement de test)

**Commandes utiles :**

```bash
# Voir les IPs bannies par CrowdSec
docker exec crowdsec cscli decisions list

# Voir l'utilisation disque
du -sh /volume1/docker/gateway/*

# Vérifier les certificats (dans logs Traefik)
docker compose logs traefik | grep -i certificate
```

---

# 8. Exemples d'Autres Services

Cette section fournit des exemples **prêts à l'emploi** pour d'autres services populaires.

---

## 8.1 Jellyfin (Serveur média)

### Description

Jellyfin est un serveur multimédia open-source pour films, séries TV, musique, etc.

### Configuration docker-compose

```yaml
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    volumes:
      - ./jellyfin/config:/config
      - /volume1/medias/movies:/data/movies:ro
      - /volume1/medias/tvshows:/data/tvshows:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`media.example.com`)"
      - "traefik.http.routers.jellyfin.entrypoints=websecure"
      - "traefik.http.routers.jellyfin.tls.certresolver=letsencrypt"
      - "traefik.http.routers.jellyfin.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
    networks:
      - proxy
```

### Particularités

- **Volumes en lecture seule (`:ro`)** pour les médias (sécurité)
- **Port interne** : 8096
- **Espace disque** : Nécessite beaucoup d'espace pour les médias
- **Transcoding** : Peut être gourmand en CPU/GPU

---

## 8.2 NextCloud (Cloud personnel)

### Description

NextCloud est une plateforme de cloud personnel (stockage, calendrier, contacts, etc.).

### Configuration docker-compose (avec PostgreSQL)

```yaml
  nextcloud-db:
    image: postgres:16-alpine
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD}
    volumes:
      - ./nextcloud/db:/var/lib/postgresql/data
    networks:
      - proxy

  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - nextcloud-db
    environment:
      - POSTGRES_HOST=nextcloud-db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=${NEXTCLOUD_DB_PASSWORD}
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.example.com
    volumes:
      - ./nextcloud/html:/var/www/html
      - ./nextcloud/data:/var/www/html/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(`cloud.example.com`)"
      - "traefik.http.routers.nextcloud.entrypoints=websecure"
      - "traefik.http.routers.nextcloud.tls.certresolver=letsencrypt"
      - "traefik.http.routers.nextcloud.middlewares=crowdsec@file"
      - "traefik.http.services.nextcloud.loadbalancer.server.port=80"
    networks:
      - proxy
```

### Particularités

- **Authentification intégrée** (pas d'Authelia nécessaire)
- **Variable `NEXTCLOUD_TRUSTED_DOMAINS`** importante pour éviter les erreurs "Untrusted Domain"
- **Port interne** : 80
- **Configuration initiale** : Via l'interface web au premier accès

---

## 8.3 Uptime Kuma (Monitoring)

### Description

Uptime Kuma est un outil de monitoring pour surveiller la disponibilité de vos services.

### Configuration docker-compose

```yaml
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
    volumes:
      - ./uptime-kuma:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.uptime-kuma.rule=Host(`status.example.com`)"
      - "traefik.http.routers.uptime-kuma.entrypoints=websecure"
      - "traefik.http.routers.uptime-kuma.tls.certresolver=letsencrypt"
      - "traefik.http.routers.uptime-kuma.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.uptime-kuma.loadbalancer.server.port=3001"
    networks:
      - proxy
```

### Particularités

- **Pas de base de données externe** (SQLite intégré)
- **Port interne** : 3001
- **Interface intuitive** de monitoring
- **Notifications** : Email, Telegram, Discord, etc.

---

## 8.4 Vaultwarden (Gestionnaire de mots de passe)

### Description

Vaultwarden est une implémentation open-source du serveur Bitwarden (gestionnaire de mots de passe).

### Configuration docker-compose (avec PostgreSQL)

```yaml
  vaultwarden-db:
    image: postgres:16-alpine
    container_name: vaultwarden-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=vaultwarden
      - POSTGRES_USER=vaultwarden
      - POSTGRES_PASSWORD=${VAULTWARDEN_DB_PASSWORD}
    volumes:
      - ./vaultwarden/db:/var/lib/postgresql/data
    networks:
      - proxy

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    depends_on:
      - vaultwarden-db
    environment:
      - DATABASE_URL=postgresql://vaultwarden:${VAULTWARDEN_DB_PASSWORD}@vaultwarden-db:5432/vaultwarden
      - DOMAIN=https://passwords.example.com
    volumes:
      - ./vaultwarden/data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.rule=Host(`passwords.example.com`)"
      - "traefik.http.routers.vaultwarden.entrypoints=websecure"
      - "traefik.http.routers.vaultwarden.tls.certresolver=letsencrypt"
      - "traefik.http.routers.vaultwarden.middlewares=crowdsec@file"
      - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"
    networks:
      - proxy
```

### Particularités

- **Authentification intégrée** (pas d'Authelia)
- **Port interne** : 80
- **Très léger** comparé à Bitwarden officiel
- **Compatible** avec les apps Bitwarden (mobile, extension navigateur)

---

# 9. Ressources

## 9.1 Documentation officielle

### Services de la Gateway

- **Traefik v3** : [https://doc.traefik.io/traefik/](https://doc.traefik.io/traefik/)
- **Authelia** : [https://www.authelia.com/](https://www.authelia.com/)
- **CrowdSec** : [https://docs.crowdsec.net/](https://docs.crowdsec.net/)
- **Docker Compose** : [https://docs.docker.com/compose/](https://docs.docker.com/compose/)
- **Dockge** : [https://github.com/louislam/dockge](https://github.com/louislam/dockge)

### Services d'exemple

- **Planka** : [https://planka.app/](https://planka.app/)
- **Jellyfin** : [https://jellyfin.org/docs/](https://jellyfin.org/docs/)
- **NextCloud** : [https://docs.nextcloud.com/](https://docs.nextcloud.com/)
- **Uptime Kuma** : [https://github.com/louislam/uptime-kuma](https://github.com/louislam/uptime-kuma)

---

## 9.2 Recherche d'images Docker

- **Docker Hub** : [https://hub.docker.com/](https://hub.docker.com/)
  - Plus grand registre d'images Docker
  - Cherchez par nom de service

- **LinuxServer.io** : [https://fleet.linuxserver.io/](https://fleet.linuxserver.io/)
  - Images optimisées et maintenues
  - Support PUID/PGID pour permissions

- **GitHub Container Registry** : [https://ghcr.io](https://ghcr.io)
  - Images hébergées sur GitHub
  - Exemple : Planka utilise `ghcr.io/plankanban/planka`

---

## 9.3 Outils utiles

### Validation et test

- **YAML Validator** : [https://www.yamllint.com/](https://www.yamllint.com/)
- **DNS Checker** : [https://dnschecker.org/](https://dnschecker.org/)
- **SSL Checker** : [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

### Génération de secrets

- **Password Generator** : [https://www.uuidgenerator.net/version4](https://www.uuidgenerator.net/version4)
- **Authelia Password Hash** :
  ```bash
  docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'VotreMotDePasse'
  ```

---

## 9.4 Communautés et support

- **Reddit** :
  - [r/selfhosted](https://www.reddit.com/r/selfhosted/) - Communauté self-hosting
  - [r/docker](https://www.reddit.com/r/docker/) - Tout sur Docker
  - [r/traefik](https://www.reddit.com/r/Traefik/) - Communauté Traefik

- **Discord** :
  - Traefik Labs Discord
  - LinuxServer.io Discord

- **Forum Synology** : [https://community.synology.com/](https://community.synology.com/)

---

# 10. Conclusion

## 10.1 Récapitulatif des compétences acquises

Félicitations ! Vous savez maintenant :

- [ ] **Créer des enregistrements DNS A** pour vos services
- [ ] **Ajouter un service Docker avec base de données** (Méthode A)
- [ ] **Exposer un service externe** via fichiers dynamiques (Méthode B)
- [ ] **Appliquer l'authentification 2FA** avec Authelia
- [ ] **Protéger vos services** avec CrowdSec
- [ ] **Générer automatiquement des certificats SSL** avec Let's Encrypt
- [ ] **Diagnostiquer et résoudre** les problèmes courants
- [ ] **Appliquer les bonnes pratiques** de sécurité et de maintenance

---

## 10.2 Checklist finale pour chaque nouveau service

Avant de considérer un service comme "en production" :

1. [ ] DNS propagé et résolvant correctement (dnschecker.org)
2. [ ] Certificat SSL valide (cadenas vert dans le navigateur)
3. [ ] Authentification fonctionnelle (Authelia ou interne au service)
4. [ ] CrowdSec activé (protection anti-bot)
5. [ ] Volumes configurés (données persistantes)
6. [ ] Service accessible depuis Internet
7. [ ] Logs sans erreurs critiques
8. [ ] Sauvegarde configurée (Duplicati)
9. [ ] Documentation personnelle mise à jour
10. [ ] Test de redémarrage (vérifier `restart: unless-stopped`)

---

## 10.3 Services suggérés à ajouter ensuite

**Par ordre de difficulté (facile → avancé) :**

1. **Uptime Kuma** - Monitoring
   - ✅ Aucune dépendance
   - ✅ Configuration simple
   - ✅ Interface intuitive

2. **Jellyfin** - Serveur média
   - ⚠️ Nécessite de l'espace disque
   - ⚠️ Volumes à configurer
   - ✅ Pas de base de données externe

3. **Vaultwarden** - Gestionnaire de mots de passe
   - ⚠️ Nécessite PostgreSQL
   - ✅ Très léger
   - ✅ Compatible Bitwarden

4. **NextCloud** - Cloud personnel
   - ⚠️ Configuration complexe
   - ⚠️ Nécessite PostgreSQL
   - ⚠️ Beaucoup de dépendances

5. **GitLab** - Git + CI/CD
   - ⚠️ Très avancé
   - ⚠️ Très gourmand en ressources (CPU, RAM, disque)
   - ⚠️ Configuration complexe

---

## 10.4 Pour aller plus loin

### Sujets avancés (hors scope de ce tutoriel)

- Configuration d'un **serveur SMTP** pour les notifications (Authelia, Uptime Kuma, etc.)
- **Monitoring avancé** avec Prometheus + Grafana
- **Sauvegardes automatisées off-site** (cloud, serveur distant)
- **Haute disponibilité** et failover (plusieurs serveurs Gateway)
- **Intégration LDAP** avec Authelia pour la gestion centralisée des utilisateurs
- Utilisation de **Docker Secrets** pour une sécurité accrue

### Lectures recommandées

- [formation-gateway.md](formation-gateway.md) - Formation complète dans ce dépôt
- Documentation Traefik sur les middlewares avancés
- Best practices Authelia pour la sécurité

---

## 10.5 Contribution et feedback

Ce tutoriel est open-source. Si vous avez des suggestions d'amélioration :

1. **Issues** : [https://github.com/entropik/docker-gateway/issues](https://github.com/entropik/docker-gateway/issues)
2. **Pull Requests** : Proposez vos améliorations
3. **Discussions** : Partagez vos retours d'expérience

---

# Annexes

## Annexe A : Template vierge docker-compose (service simple)

Template prêt à copier-coller pour un service sans base de données :

```yaml
  NOM_SERVICE:
    image: ORGANISATION/IMAGE:TAG
    container_name: NOM_SERVICE
    restart: unless-stopped
    environment:
      - TZ=Europe/Paris
      - PUID=1000
      - PGID=1000
    volumes:
      - ./NOM_SERVICE:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.NOM_SERVICE.rule=Host(`SOUS_DOMAINE.example.com`)"
      - "traefik.http.routers.NOM_SERVICE.entrypoints=websecure"
      - "traefik.http.routers.NOM_SERVICE.tls.certresolver=letsencrypt"
      - "traefik.http.routers.NOM_SERVICE.middlewares=authelia@file,crowdsec@file"
      - "traefik.http.services.NOM_SERVICE.loadbalancer.server.port=PORT_INTERNE"
    networks:
      - proxy
```

**À remplacer :**
- `NOM_SERVICE` : Nom de votre service (3 occurrences)
- `ORGANISATION/IMAGE:TAG` : Image Docker officielle
- `SOUS_DOMAINE` : Sous-domaine souhaité
- `PORT_INTERNE` : Port interne du service

---

## Annexe B : Template fichier dynamique (service externe)

Template pour `config/dynamic/NOM_SERVICE.yml` :

```yaml
http:
  routers:
    nom-service:
      rule: "Host(`sous-domaine.example.com`)"
      entryPoints:
        - websecure
      service: nom-service
      tls:
        certResolver: letsencrypt
      middlewares:
        - authelia
        - crowdsec

  services:
    nom-service:
      loadBalancer:
        servers:
          - url: "http://IP_SERVEUR:PORT"
```

**À remplacer :**
- `nom-service` : Nom de votre service (2 occurrences)
- `sous-domaine.example.com` : Domaine complet
- `IP_SERVEUR:PORT` : Adresse du serveur distant

---

## Annexe C : Variables d'environnement courantes

| Variable | Utilisation | Exemple |
|----------|-------------|---------|
| `TZ` | Timezone | `Europe/Paris` |
| `PUID` | User ID Linux | `1000` |
| `PGID` | Group ID Linux | `1000` |
| `BASE_URL` | URL publique du service | `https://projets.example.com` |
| `DATABASE_URL` | Connexion base de données | `postgresql://user:pass@host:port/db` |
| `SECRET_KEY` | Clé de chiffrement | UUID long aléatoire (64 chars) |
| `DOMAIN` | Domaine sans protocole | `example.com` |

---

## Annexe D : Ports courants des services

| Service | Port par défaut | Protocole |
|---------|-----------------|-----------|
| **Traefik Dashboard** | 8080 | HTTP |
| **Authelia** | 9091 | HTTP |
| **Heimdall** | 80 | HTTP |
| **Dockge** | 5001 | HTTP |
| **Jellyfin** | 8096 | HTTP |
| **NextCloud** | 80 | HTTP |
| **Planka** | 1337 | HTTP |
| **Uptime Kuma** | 3001 | HTTP |
| **Vaultwarden** | 80 | HTTP |
| **PostgreSQL** | 5432 | TCP |
| **MySQL/MariaDB** | 3306 | TCP |
| **Redis** | 6379 | TCP |

---

## Annexe E : Commandes de dépannage rapide

### Vérifier l'état des services

```bash
# Voir tous les conteneurs
docker ps

# Voir les logs d'un service
docker logs planka

# Suivre les logs en temps réel
docker logs -f planka

# Redémarrer un service
docker restart planka

# Redémarrer toute la stack
cd /volume1/docker/gateway
docker compose restart
```

### CrowdSec

```bash
# Voir les IPs bannies
docker exec crowdsec cscli decisions list

# Voir les alertes
docker exec crowdsec cscli alerts list

# Débannir une IP
docker exec crowdsec cscli decisions delete --ip 1.2.3.4

# Whitelister une IP de confiance
docker exec crowdsec cscli decisions add --ip 1.2.3.4 --duration 999999h --type whitelist
```

### Traefik

```bash
# Voir les routeurs chargés
docker exec traefik wget -qO- http://localhost:8080/api/http/routers | jq

# Voir les services backend
docker exec traefik wget -qO- http://localhost:8080/api/http/services | jq

# Voir les middlewares
docker exec traefik wget -qO- http://localhost:8080/api/http/middlewares | jq
```

---

**Document créé en janvier 2026**
**Version 1.0**
**Licence: CC BY-SA 4.0**

⭐ Si ce tutoriel vous a été utile, n'hésitez pas à donner une étoile au projet sur [GitHub](https://github.com/entropik/docker-gateway) !
