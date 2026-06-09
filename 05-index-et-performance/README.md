🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 📌 Chapitre 5 — Index et Performance

> **Partie 3 : Index, Transactions et Performance** · Niveau : Intermédiaire  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Un index est à une base de données ce qu'un index alphabétique est à un livre : un dispositif qui permet de **retrouver une information sans parcourir l'intégralité des pages**. Sans index, MariaDB n'a d'autre choix que de lire chaque ligne d'une table pour répondre à une requête — c'est ce qu'on appelle un *full table scan*. Sur une table de quelques centaines de lignes, l'opération est imperceptible ; sur une table de plusieurs dizaines de millions de lignes, elle peut transformer une requête de quelques millisecondes en plusieurs secondes, voire minutes.

L'indexation est, de loin, **le levier le plus rentable pour améliorer les performances** d'une base relationnelle. C'est aussi l'un des plus subtils. Un index bien conçu accélère spectaculairement les lectures ; un index mal choisi, redondant ou inutile, ralentit les écritures, consomme de l'espace disque et complexifie le travail de l'optimiseur. Indexer n'est donc pas une case à cocher mais un **problème d'optimisation** : il s'agit de trouver le bon équilibre entre vitesse de lecture, coût d'écriture et empreinte de stockage, en fonction des requêtes réellement exécutées par l'application.

Ce chapitre constitue le socle de toute démarche de performance. Il couvre à la fois le **fonctionnement interne** des index (la structure B-Tree qui en est le cœur), le **panorama des types disponibles** dans MariaDB — du classique B-Tree aux index VECTOR (HNSW) destinés aux applications d'IA — et les **stratégies concrètes** pour les utiliser efficacement. Il introduit également l'outil de diagnostic indispensable du développeur et du DBA : la commande `EXPLAIN`, qui révèle comment MariaDB exécute réellement une requête. Les techniques d'*ajustement fin* (tuning serveur, partitionnement, benchmarking) seront approfondies plus tard, au [chapitre 15](../15-performance-tuning/README.md) ; ce chapitre 5 en pose les fondations.

---

## 🎯 Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- **Expliquer** la structure interne d'un index B-Tree et comprendre pourquoi elle rend les recherches efficaces.
- **Distinguer** les différents types d'index (B-Tree, Hash, Full-Text, Spatial, VECTOR) et choisir le type adapté à chaque besoin.
- **Créer, modifier et supprimer** des index, et appliquer des stratégies d'indexation cohérentes (colonnes filtrées, clés étrangères, tri et regroupement).
- **Concevoir** des index composites en maîtrisant l'importance de l'ordre des colonnes.
- **Lire et interpréter** un plan d'exécution avec `EXPLAIN` et `ANALYZE`.
- **Identifier et corriger** les requêtes mal optimisées, en tirant parti des optimisations récentes de l'optimiseur.
- **Exploiter** des fonctionnalités avancées comme les index *covering*, les *index-only scans* et les index invisibles pour tester et déployer des changements sans risque.

---

## 📋 Prérequis

Pour suivre ce chapitre dans de bonnes conditions, il est recommandé d'avoir assimilé :

- Les **bases du SQL** et la création de tables et de contraintes ([chapitre 2](../02-bases-du-sql/README.md)) — notamment les clés primaires, qui sont elles-mêmes des index.
- Idéalement, les **requêtes intermédiaires et avancées** ([chapitres 3](../03-requetes-sql-intermediaires/README.md) et [4](../04-concepts-avances-sql/README.md)), qui fournissent les jointures, regroupements et sous-requêtes servant d'exemples tout au long du chapitre.

Une familiarité avec la notion de **complexité algorithmique** (la différence entre un parcours linéaire et une recherche logarithmique) est utile pour saisir l'intérêt des index, mais elle n'est pas indispensable : les concepts seront expliqués au fur et à mesure.

---

## 🗺️ Plan du chapitre

Le chapitre progresse du fonctionnement interne vers les usages avancés, selon le découpage suivant :

| Section | Sujet | En bref |
|---------|-------|---------|
| **5.1** | [Fonctionnement des index : structure B-Tree](01-fonctionnement-index.md) | Le mécanisme fondamental qui permet une recherche en temps logarithmique. |
| **5.2** | [Types d'index](02-types-index.md) | Panorama : B-Tree, Hash, Full-Text, Spatial — et le bon usage de chacun. |
| **5.3** | [Index VECTOR (HNSW)](03-index-vector-hnsw.md) | Recherche par similarité vectorielle pour les cas d'usage IA et RAG. |
| **5.4** | [Création et gestion des index](04-creation-gestion-index.md) | Syntaxe `CREATE INDEX`, `ALTER TABLE`, suppression et maintenance. |
| **5.5** | [Stratégies d'indexation](05-strategies-indexation.md) | Où placer les index : colonnes filtrées, clés étrangères, `ORDER BY` / `GROUP BY`. |
| **5.6** | [Index composites et ordre des colonnes](06-index-composites.md) | Le principe du *left-most prefix* et son impact décisif. |
| **5.7** | [Analyse des plans d'exécution](07-analyse-plans-execution.md) | `EXPLAIN` et `ANALYZE`, l'outil de diagnostic central. |
| **5.8** | [Optimisation des requêtes](08-optimisation-requetes.md) | Techniques d'optimisation, incluant les nouveautés sur les scans inversés. |
| **5.9** | [Index *covering* et *index-only scans*](09-index-covering.md) | Répondre à une requête sans jamais accéder à la table. |
| **5.10** | [Invisible indexes et Progressive indexes](10-invisible-progressive-indexes.md) | Tester l'impact d'un index avant de l'adopter ou de le retirer. |

---

## 🆕 Nouveautés 12.x abordées dans ce chapitre

MariaDB 12.3 LTS consolide plusieurs améliorations notables de l'indexation et de l'optimiseur introduites au fil de la série 12.x (depuis la 11.8) :

- **Optimisations sur les scans inversés** (section [5.8.1](08.1-optimisations-scans-inverses.md)) : le *Rowid Filtering*, l'*Index Condition Pushdown* et le *Loose Index Scan* bénéficient désormais d'optimisations sur les clés en ordre descendant (`DESC`), améliorant les requêtes qui exploitent un parcours inversé.
- **Recherche vectorielle mature** (section [5.3](03-index-vector-hnsw.md)) : l'index VECTOR fondé sur l'algorithme **HNSW** (*Hierarchical Navigable Small Worlds*) est pleinement intégré, ouvrant la voie aux applications de recherche sémantique et de RAG. Les optimisations de calcul de distance propres à la 12.3 sont détaillées au [chapitre 15](../15-performance-tuning/README.md).

Le travail de l'optimiseur s'appuie par ailleurs sur un modèle de coûts **conscient des SSD**, lui aussi développé plus en détail au chapitre 15.

---

## 💡 Idée directrice

Si vous ne deviez retenir qu'un principe de ce chapitre, ce serait celui de l'**arbitrage** :

> **Un index accélère les lectures, mais ralentit les écritures et occupe de l'espace.**

Chaque `INSERT`, `UPDATE` ou `DELETE` doit non seulement modifier les données, mais aussi mettre à jour tous les index concernés. Multiplier les index sans discernement dégrade donc les performances en écriture et peut même nuire aux lectures en surchargeant l'optimiseur de choix. L'objectif n'est jamais d'indexer *davantage*, mais d'indexer *juste* : créer les index que les requêtes utilisent réellement, et savoir mesurer cet usage. C'est précisément ce que ce chapitre vous apprendra à faire.

---

## 🧭 Navigation

- ⬅️ Partie précédente : [Partie 2 — Requêtes SQL Intermédiaires et Avancées](../partie-02-requetes-sql-intermediaires-avancees.md)
- 📂 Partie courante : [Partie 3 — Index, Transactions et Performance](../partie-03-index-transactions-performance.md)
- ➡️ Section suivante : [5.1 Fonctionnement des index : structure B-Tree](01-fonctionnement-index.md)
- 📖 Chapitre lié : [Chapitre 6 — Transactions et Concurrence](../06-transactions-et-concurrence/README.md) · [Chapitre 15 — Performance et Tuning](../15-performance-tuning/README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Fonctionnement des index : Structure B-Tree](/05-index-et-performance/01-fonctionnement-index.md)
