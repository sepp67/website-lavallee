---
title: "Ajouter le MFA sans casser l'existant : une réflexion d'architecte"
date: 2026-06-22
description: "La mise en place de l'authentification MFA constitue la première étape concrète d'un projet d'infrastructure d'identité"
---

## Le problème qui ne se résout pas avec un simple plugin

Vous reconnaissez peut-être cette situation. Votre système d'authentification repose sur un **OpenLDAP** en place depuis des années. Il fait tourner votre **Nextcloud**, peut-être d'autres applications internes. Tout fonctionne. Personne n'a envie d'y toucher.

Et puis arrive la question qui revient souvent : *"Pourquoi le MFA n'est-il pas encore activé partout ?"*

La réponse honnête, c'est souvent : parce que personne ne sait comment le faire sans tout casser.

Les attaques par vol d'identifiants sont devenues la porte d'entrée numéro un dans la majorité des incidents de sécurité. Un mot de passe seul, aussi complexe soit-il, ne suffit plus à protéger un accès. Les assureurs cyber le savent, les auditeurs le savent, et de plus en plus de clients et partenaires le demandent explicitement avant de signer un contrat. Le MFA (authentification à plusieurs facteurs) n'est donc plus un confort de sécurité — c'est devenu une attente de base.

Le problème, ce n'est pas de savoir **s'il faut** le faire. C'est de savoir **comment** le faire sans tout remettre en cause.

## Trois chemins, un seul qui prépare l'avenir

Quand on regarde les options disponibles pour ajouter le MFA à une infrastructure existante, on tombe généralement sur trois chemins.

**Migrer vers une solution d'identité tout-en-un.** C'est la promesse d'une plateforme moderne et intégrée qui gère tout. Mais en pratique, c'est un projet long, coûteux et risqué : il faut migrer l'annuaire, reconfigurer chaque application, former les équipes, et accepter une fenêtre de transition pendant laquelle tout peut casser.

**Ajouter un plugin MFA spécifique à une seule application.** Dans notre cas, PrivacyIDEA propose justement un plugin natif pour Nextcloud. C'est rapide, peu coûteux, et ça fonctionne. Mais ça résout le problème pour une seule application. Si demain il faut protéger un VPN, une autre application web, ou l'accès à des serveurs, il faut repartir de zéro avec un autre outil, une autre logique, une autre maintenance.

**Ajouter une couche d'identité centrale, sans toucher à l'existant.** C'est l'option la moins intuitive au premier abord, parce qu'elle demande d'introduire de nouveaux composants. Mais c'est la seule qui ne touche ni à l'annuaire ni aux applications, et qui pose les bases pour sécuriser tout le reste de l'infrastructure plus tard.

Nous avons choisi la troisième voie. C'est le cœur de cet article : pourquoi ce choix, comment il se construit techniquement, et ce qu'il rend possible une fois en place.

## L'architecture retenue : deux rôles, clairement séparés

La décision la plus importante de ce projet n'a pas été un choix d'outil, mais un choix de **séparation des responsabilités**. Plutôt que de chercher un produit unique qui ferait tout, nous avons introduit deux composants entre l'annuaire et les applications, chacun avec un rôle précis.

### Keycloak, comme point de passage unique

Le premier composant est **Keycloak**, utilisé comme *Identity Provider* (fournisseur d'identité). Son rôle : devenir le point unique par lequel transitent toutes les authentifications de l'organisation. Keycloak vient se connecter à l'annuaire OpenLDAP existant — sans le modifier — et c'est désormais lui qui reçoit les demandes de connexion, à la place de chaque application individuellement.

Concrètement, Nextcloud ne vérifie plus lui-même un mot de passe dans l'annuaire. Il délègue cette vérification à Keycloak. Pour l'utilisateur, rien ne change dans les apparences. Pour l'organisation, tout change dans la structure : il existe désormais un seul endroit où décider des règles d'authentification, au lieu d'une règle par application.

### PrivacyIDEA, spécialiste du second facteur

Le second composant est **PrivacyIDEA**. Il faut être précis sur ce qu'il est, car c'est souvent une source de confusion : PrivacyIDEA n'est **pas** un fournisseur d'identité. C'est un **serveur de tokens**. Son rôle se limite à une chose, mais il la fait très bien : enregistrer un second facteur (token OTP, application mobile, etc.) pour chaque utilisateur de l'annuaire, et vérifier la valeur saisie au moment de la connexion.

Keycloak propose lui-même, nativement, un mécanisme d'enrôlement de token. Pourquoi ne pas s'en contenter ? Parce que ses options restent limitées à des scénarios simples. PrivacyIDEA, lui, est conçu pour gérer des **workflows d'authentification** — et c'est précisément ce qui justifie sa présence dans l'architecture plutôt qu'une dépendance unique à Keycloak.

## L'authentification comme processus, pas comme case à cocher

Une authentification n'est pas une opération binaire. C'est un enchaînement de vérifications, qui peut rester très simple ou devenir très riche selon les besoins de l'organisation. C'est un point que l'on néglige souvent en abordant le MFA comme un simple interrupteur à activer — alors que c'est un workflow à concevoir.

Dans le scénario le plus simple que nous avons mis en place :
- Keycloak vérifie le mot de passe directement auprès de l'annuaire OpenLDAP.
- PrivacyIDEA, en parallèle, vérifie uniquement la valeur du code à usage unique (OTP).

Chaque composant fait une seule chose, et la fait bien. Mais l'architecture permet d'aller plus loin : il est possible de transmettre également le mot de passe à PrivacyIDEA, qui peut alors s'en servir comme déclencheur pour des scénarios plus élaborés — exiger un facteur différent selon le profil de l'utilisateur, adapter la politique selon le contexte de connexion, ou enchaîner plusieurs vérifications. C'est cette flexibilité que ne propose pas, à ce stade, le mécanisme natif de Keycloak seul.

Un point souvent négligé dans les déploiements MFA : que se passe-t-il pour un utilisateur qui se connecte pour la première fois, sans second facteur encore enregistré ? C'est un cas que l'architecture doit traiter explicitement — le workflow doit pouvoir détecter l'absence de token et déclencher un parcours d'enrôlement, plutôt que de bloquer purement et simplement l'utilisateur. C'est un détail, mais c'est souvent celui qui décide si un déploiement MFA se passe sans friction ou génère des tickets de support en série.

À ce stade, aucune des applications existantes n'a eu besoin d'être profondément reconfigurée. OpenLDAP reste l'annuaire de référence, inchangé. Nextcloud continue de fonctionner normalement, simplement reconnecté à Keycloak plutôt qu'à l'annuaire directement.

## Pourquoi cette architecture, "plus lourde" en apparence, est en réalité le bon calcul

C'est ici que la réflexion d'architecte prend tout son sens. Un plugin MFA spécifique à Nextcloud aurait résolu le problème immédiat plus vite. Mais il aurait aussi fermé la porte à tout ce qui suit naturellement une fois le MFA en place : sécuriser d'autres applications, d'autres types d'accès, avec la même cohérence.

En plaçant Keycloak au centre, nous n'avons pas seulement protégé Nextcloud. Nous avons créé un point de passage unique capable d'absorber, sans reconstruction, les besoins de sécurité des prochaines années. C'est cette différence qui sépare un correctif ponctuel d'un investissement dans l'infrastructure — et c'est exactement le type d'arbitrage qu'un architecte doit rendre explicite avant de poser la première brique, plutôt que de le découvrir au moment où un second besoin se présente.

### OpenID Connect : ouvrir la porte aux applications web

Une fois Keycloak en place, une nouvelle possibilité s'ouvre : connecter n'importe quelle application web compatible avec **OpenID Connect** (OIDC).

OpenID Connect est un protocole ouvert — au même titre que HTTP l'est pour le web — qui définit comment une application délègue l'authentification de ses utilisateurs à un fournisseur d'identité externe. Le fonctionnement repose sur trois éléments :

- L'**OpenID Provider (OP)** : c'est Keycloak. Il authentifie l'utilisateur et délivre un jeton.
- L'**ID Token** : un jeton au format JWT, qui atteste que l'utilisateur a bien été authentifié, et transporte certaines de ses informations.
- La **Relying Party (RP)** : c'est l'application cliente — celle que l'utilisateur veut réellement utiliser. Elle fait confiance au jeton délivré par Keycloak plutôt que de gérer elle-même l'authentification.

Ce qui rend ce protocole précieux pour une organisation, c'est ce qu'il ne change pas. Une application connectée en OpenID Connect garde sa propre configuration, ses propres groupes d'utilisateurs, ses propres droits internes. Seule l'authentification est déléguée. Là où l'intégration d'un nouveau système d'authentification demande habituellement des semaines de développement spécifique, une application compatible OIDC peut généralement être connectée à Keycloak en quelques heures.

### RADIUS : la même logique pour les VPN et les serveurs

L'authentification web n'est pas le seul terrain à sécuriser. Les accès VPN et les connexions à des machines virtuelles reposent généralement sur un protocole différent : **RADIUS**.

La même architecture d'identité, construite pour le MFA web, s'étend naturellement à ces usages. Le principe reste identique : un point central d'authentification, capable de vérifier un mot de passe et un second facteur, quel que soit le canal — une page web, une connexion VPN, ou l'accès à un serveur.

C'est là que l'investissement initial prend tout son sens. L'organisation ne déploie pas un outil MFA pour Nextcloud, un autre pour le VPN, un troisième pour les serveurs. Elle déploie une seule politique d'authentification, appliquée de façon cohérente partout où elle est nécessaire.

## Ce qu'il faut retenir

Le MFA n’es jamais un projet isolé. C’est toujoiurs la première brique d’un projet d'infrastructure d'identité.

La différence est importante au moment de l'évaluer : un projet MFA se termine quand le second facteur fonctionne. Un projet d'infrastructure d'identité continue à produire de la valeur à chaque nouvelle application connectée, à chaque nouvel accès sécurisé, sans qu'il soit nécessaire de repartir d'une feuille blanche.

C'est exactement le type d'architecture que nje sais concevoir et mettre en place : partir d'un système existant, sans le casser, et construire dessus une couche d'identité capable d'absorber les besoins de sécurité des années à venir.

Si votre entreprise est dans cette situation (un OpenLDAP, un Nextcloud, ou tout autre système existant) et un besoin de MFA qui s'annonce comme le premier d'une longue série de besoins de sécurité, parlons-en. C'est précisément le type de projet que j’accompagne.
