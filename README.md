# website-lavallee

Site portfolio/blog multilingue (FR/EN/DE) de [lavallee.tech](https://lavallee.tech),
généré avec [Hugo](https://gohugo.io/) et servi par Nginx dans un conteneur Docker.

## Structure

```
website-lavallee/
├── content/          # Articles et pages Hugo (FR / EN / DE)
├── data/             # Données YAML des pages d'accueil (home.fr/en/de.yaml)
├── layouts/          # Templates HTML Hugo
├── static/           # Assets statiques (CSS, images, logo)
├── archetypes/       # Archétype Hugo pour les nouveaux contenus
├── docker/
│   └── nginx.conf    # Configuration Nginx du conteneur
├── config.yaml       # Configuration Hugo (langues, menus, params)
└── Dockerfile        # Build multi-stage Hugo → Nginx
```

## Lancer en local

### Via Docker (recommandé)

```bash
docker build -t website-lavallee .
docker run --rm -p 8080:80 website-lavallee
```

Accès : [http://localhost:8080](http://localhost:8080)
(redirige automatiquement vers `/en/`)

### Via Hugo natif

```bash
# Prérequis : Hugo extended installé localement
# https://gohugo.io/installation/

hugo server --buildDrafts
```

Accès : [http://localhost:1313](http://localhost:1313)

## Publication de l'image

L'image Docker est publiée automatiquement sur GHCR :

- à chaque push sur `main`
- à chaque tag `v*`
- via `.github/workflows/publish-ghcr.yml`

**Image publiée :** `ghcr.io/sepp67/website-lavallee:latest`

## Déploiement

Le déploiement en staging et production est géré exclusivement par le dépôt
[devops_staging_prod_infra](https://github.com/sepp67/devops_staging_prod_infra).

Ce dépôt ne contient pas de logique de déploiement.
