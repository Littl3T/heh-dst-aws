# Guide de Déploiement AWS

Ce document décrit l'architecture AWS cible pour déployer l'application URL Shortener. Les étudiants doivent rechercher et implémenter le déploiement en utilisant la documentation AWS et les meilleures pratiques.

## Table des Matières

1. [Prérequis](#prérequis)
2. [Architecture Cible](#architecture-cible)
3. [Artefacts de Déploiement](#artefacts-de-déploiement)
4. [Composants d'Infrastructure](#composants-dinfrastructure)
5. [Configuration de l'Application](#configuration-de-lapplication)
6. [Vérification](#vérification)
7. [Ressources](#ressources)

---

## Prérequis

Avant de déployer, vous devez avoir :

- Compte AWS avec les permissions appropriées
- AWS CLI configuré avec vos identifiants
- Compréhension de base des services AWS (VPC, EC2, RDS, S3, CloudFront)
- Familiarité avec l'administration système Linux
- Compréhension des concepts réseau (security groups, subnets, routing)

---

## Architecture Cible

L'application doit être déployée selon l'architecture AWS suivante :

```
Utilisateurs Internet
    ↓
CloudFront (CDN) - Distribution globale de contenu
    ↓
    ├─→ /* (défaut) → Bucket S3 (Fichiers statiques frontend)
    │
    └─→ /api/*, /url/* → Application Load Balancer
                           ↓
                       Instance(s) EC2 - API Backend
                           ↓
                       ┌───┴───┐
                       ↓       ↓
                   RDS       ElastiCache
                (PostgreSQL)   (Redis)
                       ↓
                   CloudWatch Logs
```

### Exigences Architecturales Clés

1. **Distribution Frontend** :
   - Fichiers statiques hébergés dans S3
   - Distribution globale via CloudFront CDN
   - Accès HTTPS uniquement
   - En-têtes de cache appropriés pour les assets

2. **Infrastructure Backend** :
   - Instances EC2 exécutant l'application Node.js
   - Application Load Balancer pour le routage des requêtes et health checks
   - Capacité de redémarrage automatique via systemd ou équivalent
   - Intégration CloudWatch Logs pour la journalisation centralisée

3. **Couche de Données** :
   - RDS PostgreSQL pour le stockage persistant des données
   - ElastiCache Redis pour la couche de cache
   - Isolation réseau appropriée via security groups

4. **Observabilité** :
   - Logs JSON structurés envoyés à CloudWatch
   - Point de terminaison health check pour monitoring ALB
   - Métriques d'application et système

---

## Artefacts de Déploiement

Des artefacts de déploiement pré-construits sont disponibles via GitHub Releases pour économiser du temps :

### Télécharger les Artefacts

```bash
# Dernière version
wget https://github.com/heh-dst/aws-url-shortener/releases/latest/download/backend-dist.tar.gz
wget https://github.com/heh-dst/aws-url-shortener/releases/latest/download/frontend-dist.tar.gz
wget https://github.com/heh-dst/aws-url-shortener/releases/latest/download/checksums.txt

# Vérifier l'intégrité
sha256sum -c checksums.txt
```

### Contenu des Artefacts

**backend-dist.tar.gz** :
- Application JavaScript compilée (répertoire `dist/`)
- Dépendances de production (`node_modules/`)
- Manifestes de package (`package.json`, `package-lock.json`)

**frontend-dist.tar.gz** :
- Fichiers statiques optimisés pour la production
- HTML, CSS et JavaScript minifiés
- Noms de fichiers avec hash de contenu pour le cache busting

### Alternative : Construire Localement

Si vous préférez construire les artefacts vous-même :

```bash
# Backend
cd backend
npm ci
npm run build

# Frontend
cd frontend
npm ci
npm run build
```

Voir le `Dockerfile` backend pour référence sur le processus de build de production.

---

## Composants d'Infrastructure

### Base de Données (RDS PostgreSQL)

**Exigences** :
- PostgreSQL 16 ou compatible
- Nom de base de données : `urlshortener`
- Initialiser avec `backend/src/db/init.sql`
- Accès réseau restreint aux instances EC2 backend

**Schéma** :
```sql
CREATE TABLE urls (
    id SERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    click_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_short_code ON urls(short_code);
```

### Couche de Cache (ElastiCache Redis)

**Exigences** :
- Redis 7 ou compatible
- Nœud unique acceptable pour développement/test
- Accès réseau restreint aux instances EC2 backend
- Pas de persistence requise (utilisation cache uniquement)

**Modèle de Cache** :
- Format de clé : `url:shortcode:{code}`
- TTL : 60 secondes (configurable via environnement)
- Modèle cache-aside (l'application gère le cache)

### Application Backend (EC2)

**Exigences** :
- Amazon Linux 2023 ou compatible
- Runtime Node.js 24.x
- L'application écoute sur le port 3000
- S'exécute comme service systemd pour redémarrage automatique
- Agent CloudWatch Logs installé et configuré
- Logs envoyés au groupe `/aws/ec2/url-shortener`

**Variables d'Environnement Requises** :
```bash
DB_HOST=<rds-endpoint>
DB_PORT=5432
DB_NAME=urlshortener
DB_USER=urluser
DB_PASSWORD=<mot-de-passe-sécurisé>

REDIS_HOST=<elasticache-endpoint>
REDIS_PORT=6379

PORT=3000
NODE_ENV=production
LOG_LEVEL=info
```

**Points de Terminaison de l'Application** :
- `GET /api/health` - Health check (retourne JSON avec statut db/redis)
- `POST /api/shorten` - Créer URL courte
- `GET /url/:code` - Rediriger vers l'URL originale
- `GET /api/stats/:code` - Obtenir statistiques URL

### Load Balancer (ALB)

**Exigences** :
- Application Load Balancer (pas Classic ou Network)
- Chemin health check : `/api/health`
- Port du groupe cible : 3000
- Intervalle health check : 30 secondes
- Au moins 2 zones de disponibilité
- Route le trafic vers les instances EC2

### Hébergement Frontend (S3 + CloudFront)

**Configuration S3** :
- Hébergement de site web statique activé
- Politique de bucket permet l'accès public en lecture
- En-têtes de cache appropriés :
  - `index.html` : `no-cache`
  - Assets : `max-age=31536000` (1 an)

**Configuration CloudFront** :
- Deux origines requises :
  1. Bucket S3 (pour fichiers statiques)
  2. ALB (pour trafic API/redirect)
- Comportements :
  - `/api/*` → origine ALB (pas de cache)
  - `/url/*` → origine ALB (pas de cache)
  - `/*` (défaut) → origine S3 (cache activé)
- Réponse d'erreur personnalisée : 404 → `/index.html` (pour routage SPA)
- Redirection HTTPS activée

### Journalisation (CloudWatch)

**Exigences** :
- Groupe de logs : `/aws/ec2/url-shortener`
- Logs d'application depuis le journal systemd
- Format JSON structuré (défaut Pino/Fastify)
- L'instance EC2 doit avoir un rôle IAM avec permissions CloudWatch Logs

**Permissions IAM Requises** :
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogStreams"
  ],
  "Resource": "arn:aws:logs:*:*:log-group:/aws/ec2/url-shortener*"
}
```

---

## Configuration de l'Application

### Security Groups

Vous devez configurer les security groups pour contrôler le trafic réseau :

**Security Group RDS** :
- Entrant : PostgreSQL (5432) depuis security group EC2

**Security Group ElastiCache** :
- Entrant : Redis (6379) depuis security group EC2

**Security Group EC2** :
- Entrant : SSH (22) depuis votre IP (pour gestion)
- Entrant : HTTP (3000) depuis security group ALB

**Security Group ALB** :
- Entrant : HTTP (80) depuis 0.0.0.0/0
- Entrant : HTTPS (443) depuis 0.0.0.0/0

### Architecture Réseau

Considérez la conception réseau suivante :
- Utiliser VPC avec sous-réseaux publics et privés
- EC2, RDS et ElastiCache dans sous-réseaux privés
- ALB dans sous-réseaux publics (au moins 2 AZ)
- NAT Gateway ou instance pour accès Internet sortant depuis sous-réseaux privés
- Configuration appropriée des tables de routage

### Script User Data EC2

**Script de déploiement automatique fourni** : Voir `ec2-user-data.sh`

Ce script :
- Installe le runtime Node.js
- Télécharge les artefacts de déploiement depuis GitHub Releases
- Configure les variables d'environnement
- Configure le service systemd
- Installe l'agent CloudWatch
- Démarre l'application

**⚠️ IMPORTANT** : Vous devez éditer le script pour :
1. Remplacer `DB_HOST` par votre endpoint RDS
2. Remplacer `DB_PASSWORD` par votre mot de passe sécurisé
3. Remplacer `REDIS_HOST` par votre endpoint ElastiCache
4. Optionnel : Modifier `VERSION` pour utiliser une version spécifique

---

## Vérification

### Liste de Vérification

Vérifiez que votre déploiement fonctionne correctement :

- [ ] La base de données RDS est accessible depuis EC2
- [ ] ElastiCache Redis est accessible depuis EC2
- [ ] L'application backend démarre avec succès
- [ ] Le point de terminaison health check retourne 200 OK avec JSON valide
- [ ] Les health checks ALB montrent des cibles saines
- [ ] Les fichiers frontend sont accessibles via le point de terminaison S3
- [ ] La distribution CloudFront est déployée (statut : Deployed)
- [ ] CloudFront route les requêtes `/api/*` vers ALB
- [ ] CloudFront route les requêtes `/*` vers S3
- [ ] Peut créer une URL courte via l'API
- [ ] Peut rediriger en utilisant le code court
- [ ] Les logs d'application apparaissent dans CloudWatch
- [ ] Le comportement cache hit/miss est journalisé

### Cas de Test

**Santé Backend** :
```bash
curl http://<alb-dns-name>/api/health
# Attendu : {"status":"ok","database":"ok","redis":"ok"}
```

**Raccourcissement d'URL** :
```bash
curl -X POST http://<cloudfront-domain>/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com"}'
# Attendu : {"short_code":"abc123","short_url":"https://<domain>/url/abc123"}
```

**Redirection URL** :
```bash
curl -I http://<cloudfront-domain>/url/github
# Attendu : redirection 302 (les données d'exemple incluent le code 'github')
```

**Accès Frontend** :
- Ouvrir le navigateur à l'URL CloudFront
- Vérifier que le frontend se charge
- Tester le formulaire de raccourcissement d'URL
- Voir la page de statistiques

### Vérification des Logs CloudWatch

Vérifiez que les logs sont envoyés à CloudWatch :
1. Naviguer vers CloudWatch → Groupes de logs
2. Trouver `/aws/ec2/url-shortener`
3. Vérifier les flux de logs avec les IDs d'instance
4. Vérifier les logs d'application au format JSON
5. Chercher les messages de démarrage, événements de création d'URL, cache hits/misses

### Problèmes Courants

**Échec de connexion à la base de données** : Vérifier les security groups, l'endpoint RDS, tester avec psql

**Échec de connexion Redis** : Vérifier les security groups, l'endpoint ElastiCache, tester avec redis-cli

**Échec du health check** : Vérifier les logs d'application dans CloudWatch, les variables d'environnement

**CloudFront ne route pas correctement** : Vérifier les priorités des comportements et paramètres d'origine

**Les logs n'apparaissent pas** : Vérifier les permissions du rôle IAM, le statut de l'agent CloudWatch

---

## Ressources

### Documentation AWS

- [Guide Utilisateur Amazon EC2](https://docs.aws.amazon.com/fr_fr/ec2/)
- [Guide Utilisateur Amazon RDS](https://docs.aws.amazon.com/fr_fr/rds/)
- [Amazon ElastiCache pour Redis](https://docs.aws.amazon.com/fr_fr/elasticache/)
- [Hébergement de Site Web Statique S3](https://docs.aws.amazon.com/fr_fr/s3/)
- [Guide Développeur Amazon CloudFront](https://docs.aws.amazon.com/fr_fr/cloudfront/)
- [Référence Agent CloudWatch Logs](https://docs.aws.amazon.com/fr_fr/AmazonCloudWatch/latest/logs/)
- [Guide Utilisateur VPC](https://docs.aws.amazon.com/fr_fr/vpc/)
- [Security Groups](https://docs.aws.amazon.com/fr_fr/vpc/latest/userguide/VPC_SecurityGroups.html)

### Références Application

- [Documentation Fastify](https://fastify.dev) - Framework backend
- [Pino Logger](https://getpino.io) - Journalisation structurée (utilisé par Fastify)
- [node-postgres](https://node-postgres.com) - Client PostgreSQL
- [ioredis](https://github.com/redis/ioredis) - Client Redis
- [Guide Build Vite](https://vitejs.dev/guide/build.html) - Outil de build frontend

### Modèles de Déploiement

- [AWS Well-Architected Framework](https://aws.amazon.com/fr/architecture/well-architected/)
- [Architecture Application Web Trois Tiers](https://docs.aws.amazon.com/whitepapers/latest/web-application-hosting-best-practices/an-aws-cloud-architecture-for-web-hosting.html)
- [Déployer Applications Node.js sur AWS](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/welcome.html)

### Fichiers de Référence dans ce Dépôt

- `backend/Dockerfile` - Montre le processus de build de production
- `backend/src/db/init.sql` - Schéma de base de données et données d'exemple
- `docker-compose.yml` - Stack de développement local (services similaires à AWS)
- `aws-architecture.mmd` - Source du diagramme d'architecture
- `ec2-user-data.sh` - **Script User Data complet pour déploiement EC2**

---

## Optimisation des Coûts

Pour les déploiements étudiants, considérez les options éligibles au niveau gratuit AWS :
- EC2 : t2.micro ou t3.micro
- RDS : db.t3.micro avec déploiement Single-AZ
- ElastiCache : cache.t3.micro
- CloudFront : 1 To de transfert de données par mois (niveau gratuit)
- S3 : 5 Go de stockage (niveau gratuit)
- CloudWatch : Métriques de base et 5 Go d'ingestion de logs (niveau gratuit)

**Important** : Arrêtez ou terminez les ressources quand vous ne testez pas activement pour minimiser les coûts.

---

## Meilleures Pratiques de Sécurité

1. **Ne jamais commiter les identifiants** dans le contrôle de version
2. Utiliser les **Security Groups** comme pare-feu - principe du moindre privilège
3. Activer le **chiffrement au repos** pour RDS et ElastiCache
4. Utiliser **HTTPS uniquement** via CloudFront
5. Stocker la configuration sensible dans **AWS Systems Manager Parameter Store** ou **Secrets Manager** (amélioration optionnelle)
6. **Faire tourner régulièrement les identifiants**
7. Activer **CloudTrail** pour la journalisation d'audit
8. Maintenir les **instances EC2 à jour** avec les mises à jour de sécurité
9. Utiliser les **rôles IAM** au lieu d'intégrer des identifiants sur EC2

---

## Prochaines Étapes

Après un déploiement réussi :
1. Configurer un domaine personnalisé avec Route 53 (optionnel)
2. Configurer des alarmes CloudWatch pour le monitoring
3. Configurer les sauvegardes automatiques RDS
4. Implémenter Auto Scaling pour les instances EC2
5. Ajouter des règles WAF à CloudFront pour la sécurité
6. Activer la compression CloudFront pour de meilleures performances

---

## Création de Nouvelles Versions

**Pour les instructeurs maintenant le dépôt :**

Pour publier de nouveaux artefacts de déploiement :

```bash
# Créer et pousser un tag de version
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

GitHub Actions construira et publiera automatiquement les artefacts sur la page Releases.

Pour plus de détails, voir `.github/workflows/README.md`.
