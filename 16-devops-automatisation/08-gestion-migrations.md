🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.8 — Gestion des migrations

> Chapitre 16 — DevOps et Automatisation · Vue d'ensemble de la section, avant l'étude des outils de migration.

---

## Introduction

La section précédente a établi que la **migration** est l'unité de changement d'une base dans une chaîne CI/CD (§16.7), et elle a renvoyé à la présente section le soin de présenter les **outils** qui l'implémentent. C'est l'objet de cette page d'introduction : poser ce qu'est la gestion des migrations et distinguer les **familles d'outils**, avant que les sous-sections n'entrent dans le détail de chacun — Flyway (§16.8.1), Liquibase (§16.8.2), puis les outils de modification en ligne gh-ost et pt-online-schema-change (§16.8.3).

## Qu'est-ce que la gestion des migrations

La **gestion des migrations** est la discipline — et l'outillage — qui permet d'**écrire, versionner, suivre et appliquer** les changements de schéma de façon fiable et reproductible. Le problème qu'elle résout est celui de la **cohérence** : garder le schéma de **tous les environnements** — production, préproduction, mais aussi le poste de chaque développeur — identique et reproductible, et appliquer les changements de manière **contrôlée, répétable et auditable**.

Sans elle, le schéma dérive : du SQL appliqué à la main, ici et là, sans trace de ce qui a été exécuté où — un « schéma de compagnie » irréproductible, écho du « serveur de compagnie » évoqué en §16.1. La gestion des migrations rétablit la discipline en faisant du schéma un artefact **versionné et géré**, partie intégrante du cycle de vie logiciel.

## Le mécanisme central : un historique des migrations

Les cadres de migration reposent sur un mécanisme commun. L'outil maintient, **dans la base elle-même**, une **table d'historique** consignant les migrations déjà appliquées. À chaque exécution, il compare les migrations disponibles à cet historique et applique, **dans l'ordre**, celles qui sont **en attente**.

De ce principe découlent deux propriétés essentielles. D'abord, l'application est **idempotente au niveau de la migration** : chaque migration ne s'exécute **qu'une fois**, ce qui rend l'opération rejouable sans risque. Ensuite, elle est **reproductible** : un même ensemble de migrations produit le même schéma. Pour garantir l'intégrité, chaque migration porte un **identifiant** et une **empreinte** (*checksum*), qui permet de détecter toute altération d'une migration déjà appliquée. C'est cette mécanique que partagent Flyway et Liquibase.

## Migrations versionnées et répétables

Un raffinement utile distingue deux natures de migration. Les migrations **versionnées** sont appliquées **une seule fois, dans l'ordre**, et deviennent immuables une fois exécutées — typiquement l'ajout d'une colonne ou la création d'une table. Les migrations **répétables** sont **réappliquées dès que leur contenu change** — utiles pour des objets que l'on redéfinit, comme une vue, une fonction ou une procédure stockée. La plupart des cadres prennent en charge les deux.

## Deux familles d'outils

C'est le point de cadrage de toute la section. La gestion des migrations mobilise **deux familles d'outils**, complémentaires et non concurrentes.

| Famille | Outils (sous-sections) | Rôle | Question répondue |
|---|---|---|---|
| **Cadres de migration** | Flyway (§16.8.1), Liquibase (§16.8.2) | Versionner, suivre et appliquer les changements | *Quels* changements, dans *quel ordre*, et appliquer ceux en attente |
| **Modification de schéma en ligne** | gh-ost, pt-online-schema-change (§16.8.3) | Rendre un `ALTER` lourd **non bloquant** | *Comment* appliquer un gros changement sans interruption |

La distinction est essentielle. Un **cadre de migration** répond au *quoi* et au *quand* : il décide quelles migrations appliquer, les ordonne et en tient l'historique. Un **outil de modification en ligne** répond au *comment* pour un changement précis : exécuter un `ALTER` volumineux **sans verrouiller la table** le temps de l'opération. Loin de s'opposer, ils se **complètent** : dans une chaîne mature, un cadre de migration peut **coordonner** ou déléguer à un outil de modification en ligne pour les `ALTER` lourds.

## Cadres de migration : Flyway et Liquibase

Les deux cadres de référence, tous deux matures et compatibles avec MariaDB, diffèrent surtout par leur **approche**. **Flyway** privilégie une logique **orientée SQL** : on écrit ses migrations en SQL, dans des fichiers nommés par version, selon une convention simple et légère. **Liquibase** repose sur un **journal de changements** (*changelog*) dans un format pouvant être **indépendant du moteur** (XML, YAML, JSON, ou SQL), offrant une abstraction plus riche et un support de retour arrière plus développé, au prix d'une courbe d'apprentissage plus marquée.

Le choix entre les deux relève donc d'un arbitrage entre la **simplicité orientée SQL** et l'**abstraction agnostique** — arbitrage que détaillent les sous-sections §16.8.1 et §16.8.2.

## Changements de schéma en ligne : pourquoi et quand

La seconde famille répond à un problème opérationnel précis, déjà soulevé en §16.7 : un `ALTER` naïf **verrouille la table**, ce qui, sur une grande table, signifie une interruption inacceptable. Les outils de **modification en ligne** (gh-ost, pt-online-schema-change) contournent ce verrou en procédant par étapes : création d'une **table fantôme** à la structure cible, copie progressive des données, synchronisation des écritures concurrentes, puis **basculement** — sans verrou prolongé.

Une alternative native existe : le **changement de schéma en ligne** de MariaDB (§18.11) traite déjà de nombreux `ALTER` en ligne, sans outil externe. Le choix se résume alors ainsi : on s'appuie sur le **DDL en ligne natif** lorsqu'il suffit, et sur un **outil externe** pour les cas que le natif ne sait pas traiter en ligne, ou pour un contrôle plus fin (limitation de débit, observation). Le détail relève de §16.8.3.

## Migrations et topologies

Une particularité propre aux bases s'ajoute sur les architectures de haute disponibilité. En **réplication**, un changement de schéma se propage aux réplicas et peut y induire du **retard** ; l'**`ALTER TABLE` optimiste** (§13.10) en atténue l'effet. En **Galera**, le DDL est traité de façon spécifique (chapitre 14). Dans tous les cas, l'outil de migration s'applique au **primaire**, la propagation relevant ensuite du mécanisme de réplication de la base (chapitres 13 et 14).

## Migration ou état désiré

Un mot sur le paysage plus large. L'approche présentée ici est dite **fondée sur les migrations** : on écrit la **séquence** des changements. Une approche alternative, **fondée sur l'état désiré**, consiste à **déclarer le schéma voulu** et à laisser un outil en calculer la différence avec l'existant. Les cadres traités dans cette section relèvent de la première approche, qui s'accorde naturellement avec la logique de migrations versionnées du CI/CD (§16.7).

## Place dans le chapitre et la formation

La gestion des migrations s'insère dans tout ce qui précède. Les migrations sont **versionnées dans Git** et circulent dans la **chaîne CI/CD** (§16.7) ; sur Kubernetes, elles peuvent s'exécuter comme des **Jobs** ou via la ressource **`SqlJob`** de l'opérateur (§16.5.1) ; et elles s'inscrivent dans une démarche **GitOps** (§16.12). Elles incarnent enfin la **couche schéma** du modèle d'Infrastructure as Code posé en §16.1. Les pratiques — migrations orientées vers l'avant, motif « étendre puis contracter », changements non bloquants — ont été établies en §16.7 ; les outils de cette section en sont la **mise en œuvre**.

## Spécificités MariaDB 12.3 LTS

Deux fonctionnalités de MariaDB servent directement la gestion des migrations : le **changement de schéma en ligne** (§18.11), qui rend de nombreux `ALTER` non bloquants sans outil externe, et l'**`ALTER TABLE` optimiste** en réplication (§13.10), qui limite le retard induit sur les réplicas. Les cadres de migration eux-mêmes ne dépendent pas de la version ; ils pilotent MariaDB 12.3 comme les versions antérieures.

## Ce que couvrent les sous-sections

À partir de cette vue d'ensemble, les sous-sections entrent dans le concret. La première (§16.8.1) détaille **Flyway** et son approche orientée SQL. La deuxième (§16.8.2) présente **Liquibase** et son journal de changements. La troisième (§16.8.3) traite des outils de **modification en ligne**, gh-ost et pt-online-schema-change.

---

↩️ [Section précédente : 16.7 — CI/CD pour bases de données](07-cicd-bases-donnees.md)  
➡️ **Sous-section suivante :** [16.8.1 — Flyway](08.1-flyway.md)

⏭️ [Flyway](/16-devops-automatisation/08.1-flyway.md)
