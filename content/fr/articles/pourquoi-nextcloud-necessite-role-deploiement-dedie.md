---

title: "Pourquoi Nextcloud nécessite un rôle de déploiement dédié plutôt qu'un rôle Docker générique"
date: 2026-06-08
author: "Sébastien Lavallée"
description: "Retour d'expérience sur le déploiement de Nextcloud dans un environnement de staging géré par Ansible et Docker."
---

Dans le cadre de la restructuration de mon infrastructure, j'ai récemment mis en place un environnement de staging séparé de la production. L'objectif était simple : valider les nouvelles versions des applications avant leur déploiement en production.

La plupart de mes applications suivent le même modèle :

* Récupérer une image Docker depuis GitHub Container Registry.
* Exposer un port HTTP.
* Configurer le proxy inverse.
* Vérifier le bon fonctionnement de l'application.

Cette approche fonctionne parfaitement pour :

* Grav CMS
* Webcam Stream
* Facturier Demo
* Sites web statiques

Pour cela, j'ai créé un rôle Ansible générique nommé :

```text
staging_docker_app
```

Ce rôle effectue simplement :

1. Le téléchargement de l'image.
2. La création du conteneur.
3. L'exposition du port.
4. Le démarrage du service.

Pour les applications simples, cette approche est efficace, réutilisable et facile à maintenir.

Cependant, Nextcloud a rapidement démontré les limites de cette méthode.

## L'hypothèse initiale

Au départ, j'ai supposé que Nextcloud pouvait être déployé comme n'importe quelle autre application.

La configuration ressemblait à ceci :

```yaml
app_name: nextcloud
app_image: ghcr.io/sepp67/ansible-role-nextcloud-stack:latest
app_container_name: nextcloud-staging
app_host_port: 18080
app_container_port: 80
```

Le déploiement semblait réussir.

Le conteneur démarrait.

Le proxy fonctionnait.

Mais l'application n'était pas réellement opérationnelle.

## Comprendre le problème

Contrairement à une application web classique, Nextcloud n'est pas constitué d'un seul service.

Mon déploiement se compose de :

```text
Nextcloud Stack
├── Nextcloud
├── PostgreSQL
├── Redis
└── Cron
```

Le fichier Docker Compose l'illustre clairement :

```yaml
services:
  db:
    image: postgres

  redis:
    image: redis

  app:
    image: nextcloud

  cron:
    image: nextcloud
```

Le conteneur principal dépend :

* de PostgreSQL pour le stockage des données ;
* de Redis pour le cache et le verrouillage des fichiers ;
* des tâches Cron pour les traitements en arrière-plan.

Démarrer uniquement le conteneur Nextcloud conduit donc à un déploiement incomplet.

## Une différence d'architecture fondamentale

Mon rôle générique reposait sur l'hypothèse :

```text
Une application = Un conteneur
```

Nextcloud suit un modèle différent :

```text
Une application = Plusieurs services coopérants
```

Il s'agit d'un schéma de déploiement totalement différent.

Essayer de gérer ces deux modèles dans un même rôle conduit rapidement à :

* de nombreuses conditions particulières ;
* une logique difficile à suivre ;
* une maintenance plus complexe ;
* une perte de lisibilité.

Le rôle générique cesse alors d'être réellement générique.

## La bonne solution

Plutôt que de compliquer le rôle existant, j'ai créé un rôle dédié :

```text
staging_nextcloud_stack
```

Les responsabilités sont désormais clairement séparées.

### Rôle générique

```text
staging_docker_app
```

Utilisé pour :

* Grav
* Webcam Stream
* Facturier Demo
* Sites web statiques

Responsabilités :

* télécharger l'image ;
* créer le conteneur ;
* exposer le port ;
* démarrer le service.

### Rôle dédié Nextcloud

```text
staging_nextcloud_stack
```

Responsabilités :

* créer les répertoires persistants ;
* générer les variables d'environnement ;
* générer le fichier Docker Compose ;
* déployer PostgreSQL ;
* déployer Redis ;
* déployer Nextcloud ;
* déployer Cron ;
* démarrer l'ensemble de la pile.

## Réutiliser les composants existants

Une autre leçon importante concerne la réutilisation.

Mon infrastructure disposait déjà d'un rôle :

```text
docker_host
```

chargé d'installer :

* Docker CE ;
* Docker Compose Plugin ;
* Buildx ;
* Containerd.

Au départ, j'avais tenté d'installer Docker directement depuis le rôle Nextcloud.

Cela créait des responsabilités redondantes.

L'architecture finale est devenue :

```text
docker_host
        │
        ▼
staging_nextcloud_stack
```

Le rôle Docker gère Docker.

Le rôle Nextcloud gère Nextcloud.

## Architecture finale

L'environnement de staging se présente désormais ainsi :

```text
vm-proxy-staging
│
├── lavallee.staging.local
├── facturier.staging.local
├── grav.staging.local
├── webcam.staging.local
└── nextcloud.staging.local
```

Les applications sont déployées sur des machines virtuelles dédiées.

La VM Nextcloud contient :

```text
vm-nextcloud-staging
│
├── PostgreSQL
├── Redis
├── Nextcloud
└── Cron
```

L'ensemble est déployé automatiquement avec Ansible et Docker Compose.

## Conclusion

L'objectif de l'automatisation n'est pas de rendre tous les déploiements identiques.

L'objectif est de rendre chaque déploiement prévisible, maintenable et compréhensible.

Pour les applications simples, un rôle générique est parfaitement adapté.

Pour les applications composées de plusieurs services, comme Nextcloud, un rôle dédié permet d'obtenir une architecture plus propre et plus durable.

La leçon est simple :

> Les rôles génériques doivent rester génériques. Les applications complexes méritent leur propre logique de déploiement.
