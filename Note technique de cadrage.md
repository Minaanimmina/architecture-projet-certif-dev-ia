---
created: 2026-06-23
updated: 2026-06-24
version: v2
tags:
  - cadrage
  - technique
---
# Note technique de cadrage

## 1. Vision d'ensemble du projet

### 1.1 Le problème résolu

Une bibliothèque publique produit des données dans trois logiciels séparés : Nanook (le SIGB, qui gère les prêts et le catalogue), Bokeh (l'OPAC, le portail public) et AFI-Multimédia (la gestion des espaces multimédia, ou EPN). Ces trois sources ne se parlent pas. Pour répondre à une question simple comme « combien de prêts de documents jeunesse au premier trimestre sur tel site ? », un bibliothécaire doit aujourd'hui solliciter un service technique ou manipuler des exports.

Le projet supprime cet obstacle : il consolide les trois sources dans une base analytique unique et permet d'interroger cette base en langage naturel. Le bibliothécaire pose sa question en français, le système l'interprète, va chercher l'information et répond.

### 1.2 Le principe de fonctionnement

Une question suit un trajet en quatre niveaux : l'interface (où la question est posée), l'orchestration (où elle est comprise, triée et sécurisée), l'accès aux données (où l'information est récupérée) et le stockage analytique (où les données consolidées résident). Une réponse remonte ensuite le même chemin en sens inverse, reformulée en langage naturel.

En amont de tout cela, un flux d'alimentation (l'ETL, éventuellement précédé du datalake) remplit la base analytique à partir des sources de production. Ce flux d'alimentation et le flux de question sont deux choses distinctes : le premier tourne périodiquement en arrière-plan, le second se déclenche à chaque question.

---

## 2. Architecture détaillée

### 2.1 Schéma d'architecture

![[Schéma architecture.png]]
 
> Ce schéma montre les sept dépôts comme composants, regroupés par zones fonctionnelles (sources, datalake conditionnel, collecte et stockage, intégration IA, application, orchestration). Il fait apparaître la frontière stricte entre le serveur MCP et l'API IA, ainsi que le caractère conditionnel du datalake (en pointillés orange).

### 2.2 Le découpage en sept dépôts

Le projet est découpé en sept dépôts Git indépendants plutôt qu'en un seul. Ce choix n'est pas cosmétique : chaque dépôt a son propre cycle de vie, ses propres dépendances et son propre rythme de déploiement. Les regrouper créerait des couplages artificiels (un changement mineur dans l'interface forcerait à redéployer l'ETL, par exemple). Les séparer permet de faire évoluer chaque brique sans toucher aux autres. Ils sont reliés au moment de l'exécution par le réseau, et orchestrés ensemble par Docker Compose.

| Dépôt                              | Rôle                                                                                          | Compétences portées     |
| ---------------------------------- | --------------------------------------------------------------------------------------------- | ----------------------- |
| 1. ETL et données                  | Extraction des sources, nettoyage, agrégation, chargement de la base analytique, modélisation | C1, C2, C3, C4          |
| 2. Serveur MCP                     | Expose 9 outils d'accès aux données. Frontière : accès données seulement                      | C8                      |
| 3. API REST données                | Met les données à disposition des autres composants via REST. Consommée par l'application pour ses vues directes (chiffres-clés, listes), sans LLM | C5                      |
| 4. API REST IA                     | Orchestre le modèle de langage, sécurise, route. Frontière : orchestration seulement          | C9, C10                 |
| 5. Classifieur d'intention         | Modèle entraîné qui trie les questions en amont du routage                                    | C12, C13                |
| 6. Application Chainlit            | Interface conversationnelle accessible pour le bibliothécaire                                 | C14 à C19               |
| 7. Orchestration et infrastructure | Docker Compose, monitoring, déploiement                                                       | C15, C18, C19, C20, C21 |

### 2.3 La frontière stricte MCP / API IA

C'est le point d'architecture le plus important à tenir, et un argument de soutenance. Deux composants pourraient sembler redondants : le serveur MCP et l'API IA. Ils ne le sont pas, et leur séparation est délibérée.

- **Le serveur MCP a un seul rôle : accéder aux données.** Il expose neuf outils analytiques (par exemple « prêts par période » ou « sessions EPN »). Il ne décide rien, il ne sécurise pas la logique métier, il ne parle pas au modèle de langage. Il lit la base et renvoie des chiffres.
- **L'API IA a un seul rôle : orchestrer et sécuriser.** Elle reçoit la question, la fait trier par le classifieur, appelle le modèle de langage (Claude), applique l'authentification et les règles de sécurité, et décide quel outil MCP mobiliser. Elle ne touche jamais directement à la base de données.

Cette frontière doit être nette dans le code, pas seulement dans le discours. Si l'API IA venait à accéder aux données en direct, l'argument s'effondrerait et la démonstration des compétences C5/C9 deviendrait confuse.

### 2.4 Les deux API distinctes (C5 et C9)

Dans le prolongement de la frontière ci-dessus, il y a deux API REST : l'API données (dépôt 3, compétence C5) et l'API IA (dépôt 4, compétence C9). Elles se ressemblent en surface (deux API REST en FastAPI) mais diffèrent par leur profil de dépendances et leur cycle de déploiement. L'API données évolue avec le schéma de la base ; l'API IA évolue avec le modèle et les règles d'orchestration.

L'API données n'est pas un composant théorique : elle est consommée au présent par l'application Chainlit, qui l'appelle pour afficher des vues directes (chiffres-clés à l'ouverture, listes de référence) sans solliciter le modèle de langage. Cet usage démontre concrètement l'exploitation exigée par C5. Elle est par ailleurs conçue de façon découplée et documentée (OpenAPI) pour qu'un futur tableau de bord analytique plus complet, porté par AFI, puisse s'y brancher.

L'API données applique une règle d'exposition RGPD en sortie. Elle ne renvoie jamais la dimension abonné au grain individuel : seuls des agrégats (comptes, moyennes) et des listes de référence non personnelles (sites, réseaux, documents, périodes) sont exposés. Tout agrégat ventilé par un attribut sociodémographique (tranche d'age, sexe, CSP) applique un seuil de petit effectif : aucune cellule représentant moins de cinq individus n'est renvoyée, afin d'éviter toute ré-identification par recoupement dans les petites bibliothèques. Cette règle ferme la chaine RGPD côté sortie, en complément de la pseudonymisation appliquée à l'entrée par l'ETL. Elle s'applique de la même façon aux outils du serveur MCP, qui constituent la seconde porte de sortie des données vers l'utilisateur.

### 2.5 Le classifieur d'intention

Le classifieur est un modèle léger entraîné (type CamemBERT), entraînable sur un GPU local. Il intervient en amont du routage : avant d'appeler le modèle de langage, il trie la question en trois classes.

- **Analytique** : une vraie question sur les données. Elle est transmise au modèle de langage.
- **Hors périmètre** : une question à laquelle le système n'est pas censé répondre. Elle reçoit une réponse cadrée, sans appeler le modèle de langage.
- **Méta** : une question sur le système lui-même (« que sais-tu faire ? »). Réponse directe, sans appel coûteux.

Ce composant a une double valeur. D'un côté, il est utile au produit : il évite d'envoyer au modèle de langage des questions inutiles, ce qui réduit les coûts d'appel et cadre les réponses hors sujet. De l'autre, il couvre nativement les compétences C12 (tests automatisés d'un modèle d'IA) et C13 (chaîne de livraison continue d'un modèle d'IA), parce que c'est un modèle entraîné qui produit des métriques (accuracy, matrice de confusion). Le suivi des entraînements est assuré par MLflow (via la compatibilité GitLab), à ne pas confondre avec le monitoring du modèle en production : MLflow trace les expériences d'entraînement (C12, C13), tandis que Langfuse observe le LLM une fois déployé (C11). Le classifieur est un modèle léger entraîné (type CamemBERT), entraînable sur un GPU local. Il intervient en amont du routage : avant d'appeler le modèle de langage, il trie la question en trois classes.

### 2.6 Le stockage analytique : PostgreSQL et DuckDB

La base analytique repose sur deux briques complémentaires. 

1. **PostgreSQL** est la base relationnelle qui stocke les données consolidées selon un schéma en constellation (des tables de faits comme les prêts, entourées de dimensions comme l'abonné, le site, le document, le temps). 
2. **DuckDB** est une couche de lecture analytique rapide, posée au-dessus, optimisée pour les requêtes d'agrégation et capable de lire directement des fichiers Parquet (ce qui sera utile si le datalake est construit).

DuckDB joue aussi un rôle dans la stratégie de certification : il sert d'équivalent « système big data » pour la compétence C1, sous réserve de validation par le formateur. C'est l'un des trois éléments du double filet C1 (voir section 6).

---

## 3. Les technologies : quoi, où, pourquoi

Le tableau ci-dessous recense les technologies retenues, l'endroit du projet où chacune intervient, la raison du choix et la ou les compétences qu'elle sert à démontrer.

| Technologie               | Où elle intervient                             | Pourquoi ce choix                                                                                   | Compétences         |     |
| ------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------- | --- |
| Python                    | Partout (ETL, API, classifieur, scripts)       | Langage pivot du projet, écosystème data et IA mature                                               | C1, C2, C3, C9, C17 |     |
| Polars                    | Pipeline ETL (transformation)                  | Traitement de données rapide et économe en mémoire, adapté aux volumes (fait_prets ~2,3M lignes)    | C3                  |     |
| dbt                       | Pipeline ETL (transformations SQL versionnées) | Structure et documente les transformations analytiques, versionnable                                | C3                  |     |
| SQLAlchemy                | Pipeline ETL (accès bases)                     | Abstraction d'accès aux bases, facilite la migration vers PostgreSQL                                | C1, C4              |     |
| PostgreSQL                | Base analytique (stockage)                     | Base relationnelle robuste, adaptée à un produit destiné à des clients                              | C4                  |     |
| DuckDB                    | Couche de lecture analytique                   | Lecture analytique rapide, lit le Parquet, sert d'équivalent big data pour C1                       | C1                  |     |
| RustFS + Parquet           | Datalake (conditionnel)                        | Stockage objet open source compatible S3 + format colonnaire ; système big data crédible pour C1    | C1, C4              |     |
| FastMCP                   | Serveur MCP                                    | Expose les outils d'accès aux données en transport HTTP, avec authentification JWT                  | C8                  |     |
| FastAPI                   | API données et API IA                          | Framework REST moderne, documentation OpenAPI native, performant                                    | C5, C9              |     |
| Claude (API)              | Appelé par l'API IA                            | Modèle de langage pour interpréter les questions et formuler les réponses                           | C8, C9              |     |
| CamemBERT (ou équivalent) | Classifieur d'intention                        | Modèle léger francophone, entraînable sur GPU local, produit des métriques réelles                  | C12, C13            |     |
| Chainlit                  | Application (interface)                        | Interface conversationnelle en Python pur (pas de JavaScript requis), accessible                    | C17                 |     |
| JWT                       | API données, API IA, MCP                       | Authentification standard par jetons, gestion de l'expiration                                       | C5, C9, C10         |     |
| Langfuse                  | Monitoring de l'API IA                         | Observe le comportement du modèle de langage (monitoring LLM spécialisé)                            | C11                 |     |
| Prometheus + Grafana      | Monitoring de l'application                    | Collecte de métriques et tableaux de bord pour la surveillance applicative                          | C20                 |     |
| OpenTelemetry (optionnel) | Instrumentation fine                           | Traçabilité détaillée ; passé en optionnel pour alléger la charge                                   | C20                 |     |
| GitLab CI/CD              | Tous les dépôts (template mutualisé)           | Intégration et livraison continues, template partagé entre les 7 dépôts : lint, types, tests, build, scan de sécurité (bandit/safety), hooks pre-commit. Conventions de commit (Conventional Commits) et stratégie de branches main/develop | C18, C19, C13       |     |
| Docker Compose            | Orchestration                                  | Assemble et lance les composants ensemble en local et en déploiement                                | C15                 |     |
| Hetzner ou serveurs AFI   | Déploiement                                    | Hébergement du produit déployé (Hetzner/EasyPanel ou environnement AFI). Le choix d'hébergeur ne change pas la démonstration : ce qui compte est une procédure de déploiement reproductible et documentée | C15, C19            |     |
| Redmine                   | Pilotage de projet                             | Suivi agile horodaté ; l'historique des tâches est une preuve pour C16                              | C16                 |     |
| ruff, mypy, pytest-cov    | CI (qualité de code)                           | Linting, vérification de types, couverture de tests, intégrés à la CI                               | C18                 |     |
| Model Registry GitLab     | Classifieur (stockage du modèle)               | Fonctionnalité native dédiée : versions, artefacts et métriques attachés, liaison au pipeline CI/CD | C12, C13            |     |
| MLflow                    | Classifieur (suivi d'expériences)              | Journalise les runs d'entraînement (paramètres, métriques)                                          | C12, C13            |     |
### Technologies écartées et raisons (à savoir justifier à l'oral)

| Écartée               | Remplacée par | Raison                                                  |
| --------------------- | ------------- | ------------------------------------------------------- |
| MariaDB               | PostgreSQL    | Montée en compétence, robustesse pour un produit client |
| Streamlit             | Chainlit      | Mieux adapté à une interface conversationnelle          |
| Evidently             | Langfuse      | Monitoring spécialisé pour les modèles de langage       |
| DVC                   | GitLab natif  | Simplification du versionnement des modèles             |
| MkDocs / GitLab Pages | à définir     | GitLab Pages indisponible sur l'instance AFI            |

---

## 4. Les flux : comment circulent les données et les questions

### 4.1 Schéma des flux

**A INSÉRER UNE FOIS TERMINÉ**

> Ce schéma se lit en deux temps : en haut, le flux d'ingestion (conditionnel, nocturne) qui alimente la base analytique via le datalake ; en bas, le flux nominal d'une question du bibliothécaire à travers les quatre niveaux, avec le tri du classifieur et le court-circuit des questions hors périmètre.

### 4.2 Le flux d'alimentation (en arrière-plan)

C'est le flux qui remplit la base analytique. Deux variantes selon la décision datalake :

- **Sans datalake :** le pipeline ETL lit directement les bases de production (ou des sources de repli), transforme les données et charge PostgreSQL.
- **Avec datalake :** chaque nuit, un script Python extrait les données des bases de production, les anonymise (RGPD), et les dépose au format Parquet dans le datalake (RustFS). Le pipeline ETL lit ensuite ce datalake au lieu des bases de production. La cible (PostgreSQL) est identique dans les deux cas.

Le point clé : la partie aval (PostgreSQL et tout ce qui suit) ne change pas selon la variante. Seule la source d'entrée de l'ETL change. C'est ce qui permet de construire le datalake comme une greffe en amont, sans refondre l'existant.

Dans les deux variantes, deux sources externes complètent les bases métier : l'API REST du Ministère de la Culture (indicateurs nationaux des bibliothèques, data.culture.gouv.fr) et un fichier CSV de prêts annuels (réseau BMVR Toulouse, data.toulouse-metropole.fr). Ces sources apportent respectivement la nature "service web API REST" et la nature "fichier de données" exigées par le mix de sources de C1. La comparaison avec la BMVR est limitée aux bibliothèques d'envergure comparable, ce point est documenté dans la doc ETL.
### 4.3 Le flux de question (temps réel)

Quand un bibliothécaire pose une question, elle traverse les quatre niveaux :

1. **Interface (Chainlit).** La question est saisie en langage naturel. L'utilisateur est authentifié (JWT) et ses droits d'accès déterminent ce qu'il peut voir.
2. **Orchestration (API IA).** Le classifieur trie la question. Si elle est hors périmètre ou méta, une réponse cadrée est renvoyée immédiatement, sans appel au modèle de langage. Si elle est analytique, l'API IA appelle Claude, qui interprète la question et choisit les outils à mobiliser. Toute cette étape est observée par Langfuse.
3. **Accès aux données (serveur MCP).** Le serveur MCP exécute l'outil analytique demandé et lit la base analytique en SQL direct (couche DuckDB / PostgreSQL), en filtrant par client (multi-tenant). Il n'utilise pas l'API données : celle-ci est réservée aux vues directes de l'application.
4. **Stockage analytique.** PostgreSQL et DuckDB fournissent les données consolidées.

La réponse remonte ensuite le chemin inverse et est reformulée en langage naturel pour le bibliothécaire.

---

## 5. Séquencement et points de couture

### 5.1 La règle anti-double-migration

Dans le sprint 1, attaquer d'abord ce qui ne dépend pas de la source : schéma cible PostgreSQL, couche DuckDB, modélisation Merise complète. Le schéma analytique en bout de chaîne est identique que les données viennent des bases de prod ou du datalake. Le **branchement final de l'ETL** (tâche TD04) est le point de couture unique qui absorbe l'une ou l'autre source. On ne migre qu'une fois.

### 5.2 La règle du jalon pour C1

C1 ne bascule officiellement sur le datalake qu'au **jalon « datalake opérationnel »** (TD06), c'est-à-dire quand la brique existe et tourne, pas à la décision tuteur. Tant que ce jalon n'est pas franchi, le double filet reste actif : DuckDB (sous réserve formateur) et scraping conditionnel. **Ne jamais débrancher un filet avant que la preuve principale soit réellement en main.**

### 5.3 Le double filet C1, expliqué

La compétence C1 exige une extraction depuis un système big data. Trois éléments peuvent porter cette preuve, par ordre de préférence :

1. **Le datalake** (RustFS + Parquet) : la preuve la plus robuste, mais conditionnelle à la décision tuteur et à sa construction effective.
2. **La couche DuckDB** comme équivalent big data : disponible dès la migration, sous réserve de validation du formateur.
3. **Une quatrième source par scraping** : filet de dernier recours, conservé au backlog en conditionnel, activé seulement si le datalake n'est pas acté et que DuckDB est refusé.

L'idée du double (voire triple) filet est de ne jamais se retrouver sans preuve pour C1, quelle que soit l'issue des décisions externes.

### 5.4 Rédaction des dossiers au fil de l'eau

Chaque dossier d'épreuve est rédigé dans le sprint où son bloc se termine, pas groupé en décembre. C'est ce qui résout structurellement la surcharge de fin de parcours.

### 5.5 Figeage des versions

Par tags Git au moment de rédiger chaque dossier d'épreuve, pas comme contrainte posée en amont. Plus souple et plus fidèle à l'état réel.

---

## 6. Capacité et marges , le point de tension

Capacité disponible estimée : environ 486h.

- **Scénario sans datalake :** 467h core+jalon, soit environ 19h de marge brute (≈4 %). Le filet C1 est porté par DuckDB + scraping conditionnel (6h). En intégrant la veille récurrente (24h), la capacité est en réalité pleinement engagée.
- **Scénario avec datalake acté :** 467h + 34h = 501h, soit un dépassement d'environ 15h sur la capacité. Le retrait du scraping (le datalake porte alors C1) ne rend que 6h et ne suffit pas à revenir à l'équilibre. **Ce scénario n'est tenable que si le temps entreprise augmente sensiblement et/ou si une part de l'optionnel est abandonnée.**

Ce qui peut élargir la marge : une augmentation du temps entreprise disponible si le tuteur adhère au projet (probable mais non acquis, donc non intégré dans les chiffres). Ce qui peut la détruire : tout dérapage en sprint 1 ou 2, là où se concentrent migration et datalake.

**Vigilance :** la capacité est désormais pleinement engagée même sans datalake. Si le datalake est acté, le plan passe en dépassement et l'arbitrage de périmètre devient obligatoire, pas optionnel. Surveiller de près la charge réelle de S1 et S2 dès les premières semaines, et être prêt à sortir des tâches optionnelles (extension classifieur par domaine, OpenTelemetry) sans état d'âme.

---

## 7. Risques identifiés

|Risque|Impact|Parade|
|---|---|---|
|Datalake acté tard ou jamais, sur une brique jamais construite|C1 fragilisé si on a débranché le filet trop tôt|Jalon TD06 = seul moment de bascule. Filet maintenu jusque-là.|
|Migration PostgreSQL : régression sur les 9 outils MCP|Périmètre certifiant cassé|Tests de non-régression (TD05, T037) avant toute bascule.|
|Capacité pleinement engagée sans datalake, dépassement avec datalake|Aucun imprévu absorbable ; périmètre à arbitrer si datalake acté|Tâches optionnelles et compression de la veille comme variables d'ajustement. Arbitrage explicite à la décision datalake.|
|DuckDB refusé comme big data par le formateur, datalake non acté|C1 sans preuve big data solide|Scraping conditionnel (T013b) comme dernier filet.|
|Sur-engineering du datalake (feuille blanche)|Explosion de la charge|Cible bornée : RustFS + Parquet, rien de plus lourd (pas de Delta/Iceberg).|
|Accès jury au GitLab AFI non confirmé|Blocage administratif à la soutenance|À confirmer tôt avec le tuteur, sans urgence mais sans oubli.|

---

## 8. Points en attente de décision externe

- **DuckDB comme big data pour C1** : validation formateur.
- **Construction du datalake** : décision tuteur ≤ 31 juillet.
- **Contraintes infra datalake** côté AFI : à recueillir si datalake acté.
- **Accès jury au dépôt** : modalités à confirmer.

---

## 9. Ce qu'il ne faut pas perdre de vue

- Le classifieur n'est pas qu'un coche-case pour C12/C13 : son court-circuit « hors périmètre » est un vrai argument de pertinence et de coût à l'oral.
- La frontière MCP / API IA doit rester nette dans le code, pas seulement dans le discours. Toute fuite (l'API IA qui accède aux données en direct) ruinerait l'argument.
- Les dossiers se rédigent au fil de l'eau. Tout report vers décembre recrée la surcharge qu'on a éliminée.
- En cas de doute sur la capacité, les optionnels sautent avant le core. Jamais l'inverse.
- MLflow trace l'entraînement (C12/C13), Langfuse surveille la production (C11)
- L'API données a un consommateur réel (les vues directes de l'app), ce n'est pas un composant pour cocher C5. Le MCP, lui, accède à la base en SQL direct, jamais via l'API données.

---

_Document de cadrage technique personnel. À relire au début de chaque sprint et à mettre à jour quand une décision déterminante tombe._