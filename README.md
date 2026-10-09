# TP Cloud : Docker & GitHub Pages

**Étudiant :** Yassine Benamara · **Niveau :** ING4 · **Enseignant :** Nader Belhadj

**Site en ligne (GitHub Pages) :** https://yba-sudo.github.io/tp-cloud-docker/

## Contenu du dépôt

| Fichier | Partie du TP |
|---|---|
| `monapp/index.html`, `monapp/Dockerfile` | Partie 2 : image Docker personnalisée (`monapp:v1`, `monapp:v2`) |
| `compose-tp/docker-compose.yml`, `compose-tp/site/` | Partie 3 : Nginx + PostgreSQL + Redis avec Docker Compose |
| `index.html` | Partie 4 : page publiée sur GitHub Pages |
| `.github/workflows/deploy.yml` | Partie 5 : pipeline CI/CD GitHub Actions |

## Commandes principales

```bash
# Partie 1
docker run hello-world
docker run -d -p 8080:80 --name monserveur nginx
docker ps

# Partie 2
cd monapp
docker build -t monapp:v1 .
docker run -d -p 8080:80 --name app monapp:v1
# v2 après modification du titre
docker build -t monapp:v2 .
docker stop app && docker rm app
docker run -d -p 8080:80 --name app monapp:v2

# Partie 3
cd compose-tp
docker compose up -d
docker compose ps
docker compose exec db psql -U admin -d monapp -c "SELECT version();"
docker compose down
```

## Déploiement

Chaque push sur `main` déclenche le workflow « Déploiement automatique » : il vérifie la présence de
`index.html` puis publie le site dans la branche `gh-pages`, qui est la source de GitHub Pages.
Le bloc `permissions: contents: write` a été ajouté au workflow du TP pour que le `GITHUB_TOKEN`
puisse pousser dans cette branche.

## Réponses aux questions du TP

**Question 1 : Différence entre une image et un conteneur Docker**

Une image est un modèle figé, en lecture seule : elle contient le système de fichiers, les dépendances et la commande à lancer (la recette de cuisine). Un conteneur est une instance en cours d'exécution de cette image, avec son propre processus, son réseau et une couche inscriptible (le plat cuisiné). À partir d'une seule image (nginx) on peut lancer autant de conteneurs que l'on veut ; supprimer un conteneur ne supprime pas l'image.

**Question 2 : Commandes utilisées pour passer en v2**

Après avoir modifié le titre dans index.html : docker build -t monapp:v2 . puis docker stop app, docker rm app, et enfin docker run -d -p 8080:80 --name app monapp:v2. Il faut arrêter et supprimer l'ancien conteneur car le nom « app » et le port 8080 sont déjà pris ; l'image v1 reste disponible, ce qui permet de revenir en arrière.

**Question 3 : Le volume pgdata**

pgdata est un volume nommé géré par Docker, monté sur /var/lib/postgresql/data, le répertoire où PostgreSQL écrit ses données. Il vit en dehors du conteneur : docker compose down supprime les conteneurs mais pas le volume. Sans lui, les données seraient écrites dans la couche inscriptible du conteneur et disparaîtraient à chaque suppression ou recréation du conteneur (mise à jour d'image, down/up). Je l'ai vérifié : une table créée avant docker compose down est toujours là après docker compose up -d.

**Question 4 : Le GITHUB_TOKEN**

C'est un jeton d'accès temporaire que GitHub crée automatiquement au début de chaque exécution du workflow et qui expire à la fin du job. Il est limité au dépôt et aux permissions déclarées (ici contents: write, pour pousser dans la branche gh-pages). On n'écrit jamais un mot de passe dans le fichier YAML parce que ce fichier est versionné dans un dépôt public : tout le monde pourrait le lire, il resterait dans l'historique Git même après suppression, et il faudrait le révoquer. Les secrets sont stockés chiffrés par GitHub, injectés à l'exécution et masqués dans les logs.

## Questions de réflexion

**1. IaaS, PaaS et SaaS avec des exemples du TP**

IaaS : on loue l'infrastructure (machine, réseau, stockage) et on gère soi-même le système et l'application. Dans le TP, c'est le conteneur Docker qui joue ce rôle (≈ une machine virtuelle), avec les réseaux et les volumes Docker. PaaS : on fournit seulement le code, la plateforme s'occupe du serveur, du déploiement et du HTTPS ; c'est GitHub Pages (≈ App Service). SaaS : un logiciel complet utilisé directement dans le navigateur, sans rien gérer ; c'est GitHub.com, GitHub Codespaces ou Docker Hub.

**2. Un conteneur s'arrête = données perdues : la solution de Docker**

Docker sépare les données du cycle de vie du conteneur grâce aux volumes. Un volume nommé (pgdata dans la Partie 3) est stocké par Docker sur l'hôte et remonté dans le nouveau conteneur ; un montage de dossier (./site vers /usr/share/nginx/html) fait la même chose avec un dossier de l'hôte. Les données ne sont supprimées que si on le demande explicitement (docker compose down -v ou docker volume rm).

**3. GitHub Pages face à un hébergeur classique (OVH, Hostinger)**

Avantages : gratuit, HTTPS automatique, aucun serveur à administrer, déploiement par simple git push, historique et retour arrière grâce à Git, intégration directe avec GitHub Actions. Limites : sites statiques uniquement (pas de PHP, pas de base de données, pas de code côté serveur), dépôt public obligatoire avec un compte gratuit, limites de taille et de bande passante (environ 1 Go et 100 Go par mois), pas d'usage commercial intensif, peu de configuration serveur possible. Un hébergeur classique est payant et demande plus d'administration, mais permet un backend, une base de données, des e-mails et une configuration libre.

**4. Importance du CI/CD dans un environnement cloud professionnel**

Le pipeline rend le déploiement automatique, identique à chaque fois et traçable : chaque commit est vérifié (ici la présence de index.html, en entreprise des tests et des analyses de sécurité) avant d'être mis en ligne. On supprime les manipulations manuelles et leurs erreurs, on livre plus souvent et en petits changements, un problème est détecté tôt et relié à un commit précis, et le retour arrière est simple. Les identifiants de déploiement restent dans le pipeline au lieu d'être sur les postes des développeurs.

**5. Passage à Azure en production**

Conteneur Docker → Azure Container Instances, Azure Container Apps ou AKS (ou une machine virtuelle Azure). Docker Hub → Azure Container Registry. Volume Docker → Azure Blob Storage, Azure Files ou Managed Disks. Réseau Docker → Azure Virtual Network. PostgreSQL en conteneur → Azure Database for PostgreSQL. Redis en conteneur → Azure Cache for Redis. GitHub Pages → Azure Static Web Apps ou App Service. GitHub Actions → Azure DevOps Pipelines (ou GitHub Actions déployant vers Azure). GITHUB_TOKEN et secrets → Azure Key Vault et identités managées.
