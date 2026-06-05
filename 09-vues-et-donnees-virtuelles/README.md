🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 9 · Vues et Données Virtuelles

> **Partie 4 — Moteurs de Stockage et Programmation Serveur** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Jusqu'ici, la formation a manipulé des **tables** : des structures physiques qui stockent réellement les données sur disque. Ce chapitre introduit une notion complémentaire et tout aussi fondamentale : les **vues** (*views*), c'est-à-dire des tables *virtuelles*.

Une vue n'est rien d'autre qu'une **requête `SELECT` nommée et enregistrée** dans le serveur. Du point de vue de l'utilisateur ou de l'application, elle se manipule presque comme une table ordinaire : on peut la lister, la requêter, parfois même y insérer ou y modifier des lignes. Mais contrairement à une table, une vue **ne contient aucune donnée propre**. Ce qui est stocké, c'est sa *définition* — la requête sous-jacente. Les données, elles, restent dans les tables d'origine et sont (re)calculées à la volée à chaque interrogation de la vue. C'est précisément ce que recouvre l'expression « données virtuelles » : une représentation des données qui existe logiquement sans être dupliquée physiquement.

Cette indirection, en apparence anodine, est l'un des outils les plus puissants du SGBD. Elle permet de séparer la *structure logique* perçue par les applications de la *structure physique* réelle des tables — une frontière essentielle pour la maintenabilité, la sécurité et l'évolutivité d'une base de données.

## À quoi servent les vues ?

Les vues répondent à plusieurs besoins récurrents en conception et en exploitation de bases de données :

- **Abstraction et simplification.** Une requête complexe — jointures multiples, sous-requêtes, agrégations — peut être encapsulée dans une vue. Les applications interrogent alors un objet simple plutôt que de réécrire (et maintenir) la même logique partout.
- **Sécurité et confidentialité.** Une vue peut n'exposer qu'un sous-ensemble de colonnes ou de lignes d'une table. On accorde aux utilisateurs un accès à la vue plutôt qu'à la table sous-jacente, ce qui permet de masquer des données sensibles (salaires, identifiants, informations personnelles) sans dupliquer ni restructurer les tables.
- **Cohérence et réutilisabilité.** En centralisant une logique métier dans une vue, on garantit que tous les consommateurs appliquent les mêmes règles (mêmes filtres, mêmes calculs), ce qui réduit les divergences et les erreurs.
- **Stabilité de l'interface.** Une vue agit comme un contrat entre la base et les applications. Le schéma physique peut évoluer (renommage de colonnes, refonte de tables) tout en préservant une interface stable, à condition d'adapter la définition de la vue en conséquence.

En contrepartie, les vues ne sont pas magiques : elles ajoutent une couche d'indirection qui peut avoir un **coût en performance**, et toutes ne sont pas modifiables. Comprendre *quand* et *comment* les utiliser — ainsi que leurs limites — est l'objet de ce chapitre.

## Ce que vous allez apprendre

À l'issue de ce chapitre, vous serez en mesure de :

- créer, modifier et gérer le cycle de vie des vues dans MariaDB ;
- comprendre pourquoi MariaDB ne propose pas de **vues matérialisées** natives, et mettre en œuvre les alternatives existantes ;
- déterminer si une vue est **modifiable** (*updatable*) et connaître les conditions à respecter ;
- contrôler l'intégrité des écritures via `WITH CHECK OPTION` ;
- exploiter les vues comme **mécanisme de sécurité** pour masquer ou restreindre l'accès aux données ;
- analyser et optimiser les performances des vues en maîtrisant les algorithmes **`MERGE`** et **`TEMPTABLE`** ;
- naviguer dans les **vues système** (`INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA`) et les tables système `mysql` pour inspecter et superviser le serveur.

## Plan du chapitre

| Section | Sujet | En bref |
|---------|-------|---------|
| **9.1** | Création et gestion des vues (`CREATE VIEW`, `ALTER VIEW`) | Syntaxe de base, modification et suppression des vues |
| **9.2** | Vues matérialisées : alternatives et *workarounds* | Pourquoi elles n'existent pas nativement, et comment les simuler |
| **9.3** | Vues *updatable* : conditions et limitations | Quand une vue accepte `INSERT`/`UPDATE`/`DELETE` |
| **9.4** | `WITH CHECK OPTION` | Garantir que les lignes écrites restent visibles via la vue |
| **9.5** | Sécurité et vues : masquage de données | Restreindre l'accès aux colonnes et lignes sensibles |
| **9.6** | Performance des vues : `MERGE` vs `TEMPTABLE` | Comprendre comment MariaDB exécute une vue |
| **9.7** | Vues système | `INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA`, tables système `mysql` |

## Prérequis

Ce chapitre suppose une bonne maîtrise des notions abordées précédemment dans la formation :

- les requêtes `SELECT`, les filtres `WHERE` et les **jointures** (chapitres 2 et 3) ;
- les concepts SQL avancés tels que les sous-requêtes, les CTE et les agrégations (chapitre 4) ;
- une connaissance du **système de privilèges** (`GRANT`/`REVOKE`), particulièrement utile pour la section sur la sécurité (le chapitre 10 y est entièrement consacré).

Une familiarité avec la **programmation côté serveur** (chapitre 8) constitue également un atout : vues, procédures stockées et déclencheurs forment ensemble la boîte à outils de l'abstraction logique côté serveur.

---

Place maintenant à la pratique, en commençant par les fondamentaux : la **création et la gestion des vues** avec `CREATE VIEW` et `ALTER VIEW` (§9.1).

⏭️ [Création et gestion des vues (CREATE VIEW, ALTER VIEW)](/09-vues-et-donnees-virtuelles/01-creation-gestion-vues.md)
