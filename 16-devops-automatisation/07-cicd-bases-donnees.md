🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.7 — CI/CD pour bases de données

> Chapitre 16 — DevOps et Automatisation · Appliquer l'intégration et la livraison continues aux changements de base de données.

---

## Introduction

Après l'orchestration et les opérateurs, ce chapitre aborde une autre facette de l'automatisation : la **chaîne d'intégration et de livraison continues** (CI/CD) appliquée à la base de données. L'enjeu n'est plus de provisionner ou d'exploiter MariaDB, mais de faire **évoluer son schéma** de façon automatisée, testée et fiable, au rythme des évolutions applicatives.

Cette section traite de la **discipline et du pipeline** ; les **outils** qui les mettent en œuvre — Flyway, Liquibase, et les outils de modification en ligne — font l'objet de la section suivante (§16.8). On verra que, comme partout dans ce chapitre, la **nature stateful** de la base impose une approche spécifique.

## Pourquoi le CI/CD d'une base est difficile

Pour du code applicatif, la livraison continue est relativement aisée : un déploiement est **sans état** et **réversible** — pour revenir en arrière, on redéploie la version précédente. Une base de données contredit ces deux propriétés. Un changement de schéma **transforme des données existantes**, il est souvent **irréversible**, et l'on ne peut pas simplement « redéployer l'ancienne version » pour l'annuler. C'est le point déjà soulevé en §16.1 à propos de la couche schéma : elle ne se gère pas comme du code applicatif.

Le CI/CD d'une base est donc une **discipline à part entière**, qui exige des pratiques propres. C'est l'objet de cette section.

## Le principe : des migrations versionnées et progressives

La pierre angulaire est de capturer chaque changement de schéma sous forme de **migration** : un script versionné, **appliqué une seule fois**, dans un ordre déterminé, et dont l'application est **suivie**. Les migrations sont **progressives** (chacune fait évoluer le schéma d'un état au suivant) et, par défaut, **orientées vers l'avant** (*forward-only*).

C'est la migration — et non le schéma entier, ni les données — qui constitue l'**unité circulant dans le pipeline**. Cette approche prolonge le modèle en couches de §16.1 : le schéma relève des migrations, et les outils de la section §16.8 (Flyway, Liquibase) en implémentent précisément la mécanique.

## Les changements de base dans le contrôle de version

Les scripts de migration **vivent dans le dépôt**, versionnés aux côtés du code applicatif. On en retire les mêmes bénéfices que pour le code : **historique** des changements de schéma, **relecture** par les pairs (*pull requests*), **traçabilité** et **collaboration**. Le schéma cesse d'être un artefact géré à la main, séparé du cycle de vie logiciel, pour en devenir partie intégrante.

## L'intégration continue d'une base

À chaque changement touchant les migrations, la chaîne d'**intégration continue** valide ces migrations avant qu'elles n'atteignent la production. Le pipeline enchaîne typiquement les étapes suivantes.

- **Provisionner une base jetable** : un conteneur MariaDB éphémère, rapide et reproductible — exactement l'usage prévu en §16.3.2 et §16.3.3.
- **Appliquer les migrations** : depuis un schéma vierge, puis — point souvent négligé — **par-dessus le schéma de production courant**, afin de vérifier qu'elles s'appliquent réellement dans le contexte cible.
- **Analyser les migrations** (*linting*) : détecter les opérations dangereuses (par exemple un `ALTER` bloquant ou une suppression de colonne risquée).
- **Exécuter les tests applicatifs** contre le schéma migré, pour s'assurer que l'application reste compatible.

La base jetable est l'élément clé qui rend cette validation possible : on reconstruit un environnement propre à volonté, et l'on **détecte les problèmes avant la production**.

## La livraison continue d'une base

La **livraison continue** promeut ensuite les migrations à travers les environnements — développement, préproduction, production — en appliquant **les mêmes scripts versionnés, dans le même ordre**. L'application des migrations devient une **étape automatisée** du pipeline (l'outil de migration exécute les migrations en attente), encadrée par des **conditions** : succès des tests, et **approbations** pour la production. L'ordre et le suivi des migrations, assurés par l'outil, garantissent qu'aucune n'est appliquée deux fois ni dans le désordre.

## Le problème difficile : coordonner schéma et application

Voici la difficulté la plus structurante, et la plus souvent sous-estimée. Le schéma et l'application sont déployés par des **mécanismes distincts**, mais doivent rester **compatibles** à tout instant. Lors d'un déploiement progressif, l'ancienne et la nouvelle version de l'application s'exécutent **simultanément** sur un même schéma — qui doit donc convenir aux deux.

La technique de référence pour résoudre cela est le motif **« étendre puis contracter »** (*expand-contract*, ou changement parallèle), en trois temps.

1. **Étendre** : on ajoute le nouvel élément de schéma de façon **additive et rétrocompatible** — par exemple une colonne nullable ou une nouvelle table. L'ancienne comme la nouvelle version de l'application continuent de fonctionner.
2. **Migrer** : on déploie les versions de l'application qui **utilisent le nouvel élément tout en tolérant l'ancien**, en rapatriant au besoin les données.
3. **Contracter** : une fois qu'**aucune version de l'application** ne dépend plus de l'ancien élément, on le **supprime** dans une migration ultérieure.

Ce découplage des changements de schéma et des déploiements applicatifs est ce qui rend possible le **zéro interruption** (sujet approfondi en §19.8). C'est sans doute la pratique la plus importante de tout le CI/CD de base de données.

## Les changements de schéma non bloquants

Une seconde difficulté opérationnelle se présente : un `ALTER` naïf peut **verrouiller une table** durablement, provoquant une interruption inacceptable en livraison continue. La réponse tient aux **modifications de schéma non bloquantes** — assurées par les outils de modification en ligne (gh-ost, pt-online-schema-change, §16.8.3) et par le support natif de MariaDB pour le **changement de schéma en ligne** (§18.11). C'est ce qui autorise un déploiement **continu** de changements de schéma **sans interruption de service**. On veillera donc, dans le pipeline, à privilégier ces mécanismes pour toute opération potentiellement bloquante.

## Tester les changements de base

Le test est au cœur du CI/CD de base, et porte sur plusieurs dimensions. On vérifie que la migration **s'applique proprement**, tant depuis un schéma vierge que **par-dessus le schéma de production**. On contrôle, lorsqu'une migration inverse est définie, sa **réversibilité** — ou l'on assume explicitement le caractère *forward-only*. On exécute les **tests applicatifs** contre le nouveau schéma, on guette les **régressions de performance** (un index manquant, une requête dégradée — chapitre 15), et l'on **analyse** les migrations pour repérer les opérations dangereuses. Ces tests rejoignent le chapitre 17 (§17.6, tests de bases de données).

## Rollback et contingence

Une réalité doit être assumée : à la différence du code, on **ne revient pas en arrière** par un simple redéploiement, car une migration peut avoir transformé des données de façon irréversible. Les stratégies sont donc d'un autre ordre : la **correction par l'avant** (écrire une nouvelle migration corrective), les **migrations inverses** lorsqu'elles sont possibles, et — filet de sécurité ultime — les **sauvegardes et la PITR** (§16.5.4, chapitre 12). Le message essentiel est que la **prévention** (tests, *expand-contract*, changements non bloquants) prime sur le rollback, et que les sauvegardes constituent le dernier recours, jamais la stratégie de première intention. Ces questions sont approfondies en §19.7.

## Secrets dans le pipeline

Le pipeline a besoin d'**identifiants** pour appliquer les migrations à la base cible. On les confie à la **gestion de secrets de la plateforme CI/CD**, sans jamais les inscrire en clair, dans le prolongement direct de la discipline posée en §16.1 et des principes de sécurité du chapitre 10.

## CI/CD, opérateur et GitOps

Sur Kubernetes, les briques de ce chapitre se combinent. Les migrations peuvent s'exécuter comme des **Jobs** dans le cadre d'un déploiement — ou, avec l'opérateur, au moyen de la ressource **`SqlJob`** (§16.5.1) qui orchestre des scripts SQL. Le tout s'inscrit naturellement dans une démarche **GitOps** (§16.12), où l'état désiré du schéma (les migrations) réside dans Git. Pipeline CI/CD, opérateur et GitOps sont ainsi **complémentaires**, chacun couvrant une part de l'automatisation.

## Spécificités MariaDB 12.3 LTS

Deux fonctionnalités de MariaDB servent directement le CI/CD. Le **changement de schéma en ligne** (§18.11) rend de nombreux `ALTER` **non bloquants**, condition d'un déploiement continu sans interruption. Et l'**`ALTER TABLE` optimiste** en réplication (§13.10) réduit le retard induit par un changement de schéma sur les réplicas. Ces mécanismes facilitent l'application continue de migrations sur une topologie de production.

## Bonnes pratiques

On capture les changements de schéma en **migrations versionnées et orientées vers l'avant**, conservées dans Git et relues. On **valide en intégration continue** sur des bases jetables, en appliquant les migrations **par-dessus le schéma de production** et en **analysant** les opérations dangereuses. On adopte le motif **« étendre puis contracter »** pour découpler schéma et application et viser le zéro interruption, en privilégiant les **changements non bloquants**. On considère la **prévention** avant le rollback, en gardant **sauvegardes et PITR** comme filet de sécurité. On **gère les secrets** par la plateforme, et l'on **coordonne** les déploiements de schéma et d'application.

## Ce que couvre la section suivante

Forte de cette discipline, la section §16.8 présente les **outils de gestion des migrations** qui l'implémentent : Flyway (§16.8.1), Liquibase (§16.8.2), et les outils de modification en ligne gh-ost et pt-online-schema-change (§16.8.3).

---

↩️ [Section précédente : 16.6 — MariaDB Enterprise Operator](06-mariadb-enterprise-operator.md)  
➡️ **Section suivante :** [16.8 — Gestion des migrations](08-gestion-migrations.md)

⏭️ [Gestion des migrations](/16-devops-automatisation/08-gestion-migrations.md)
