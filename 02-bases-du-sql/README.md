🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 2 — Bases du SQL

> **Partie 1 : Introduction et Fondamentaux** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

## Introduction

Le chapitre 1 a posé le décor : ce qu'est MariaDB, son histoire, son écosystème, sa politique de versions et l'installation d'une instance fonctionnelle. Ce deuxième chapitre marque le véritable point de départ pratique de la formation, car c'est ici que l'on commence à écrire du SQL.

SQL (*Structured Query Language*) est le langage commun à la quasi-totalité des bases de données relationnelles. En maîtriser les fondamentaux — déclarer des structures, y insérer des données, puis les interroger et les modifier — constitue le socle indispensable avant d'aborder les requêtes intermédiaires et avancées (chapitres 3 et 4), l'indexation (chapitre 5) ou l'administration.

Ce chapitre couvre l'essentiel du langage tel qu'il s'utilise au quotidien dans MariaDB 12.3. On y traite la création et la gestion des bases et des tables (le volet DDL, *Data Definition Language*), la définition de contraintes pour garantir l'intégrité, ainsi que les opérations de base sur les données (le volet DML, *Data Manipulation Language* : `INSERT`, `SELECT`, `UPDATE`, `DELETE`). Une attention particulière est portée aux **types de données** de MariaDB, depuis les types standard (numériques, texte, temporels, binaires) jusqu'aux types plus spécifiques (`JSON`, `UUID`, `INET6`, `VECTOR`), sans oublier le **type XML basique** `XMLTYPE` introduit au titre de la compatibilité avec Oracle.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez en mesure de :

- situer le SQL et comprendre la distinction entre ses sous-langages (DDL, DML, DQL, DCL, TCL) ;
- choisir le type de données le mieux adapté à chaque colonne, en tenant compte de la performance, de l'espace de stockage et de l'intégrité ;
- créer, modifier et supprimer des bases de données et des tables ;
- définir des contraintes pour garantir la cohérence et l'intégrité référentielle des données ;
- insérer des données, du cas unitaire au chargement en masse ;
- écrire des requêtes `SELECT` simples pour extraire, filtrer, trier et paginer les données ;
- mettre à jour et supprimer des données en appliquant les précautions d'usage.

## Prérequis

- Avoir suivi le chapitre 1 (notions générales et installation).
- Disposer d'une instance MariaDB 12.3 fonctionnelle et d'un outil de connexion (client `mariadb` en ligne de commande, HeidiSQL, DBeaver ou phpMyAdmin).
- Aucune connaissance préalable du SQL n'est requise : ce chapitre part de zéro.

## Plan du chapitre

- **2.1 [Introduction au langage SQL](01-introduction-langage-sql.md)** — Origines et rôle du SQL, distinction entre ses sous-ensembles (DDL, DML, DQL, DCL, TCL) et conventions de syntaxe propres à MariaDB.

- **2.2 [Types de données MariaDB](02-types-de-donnees.md)** — Panorama complet des types disponibles et critères de choix : un bon choix de types conditionne la performance, le stockage et l'intégrité des données.
    - **2.2.1 [Numériques](02.1-types-numeriques.md)** — `INT`, `BIGINT`, `DECIMAL`, `FLOAT`, `DOUBLE` : entiers, décimaux exacts et flottants, et quand préférer l'un à l'autre.
    - **2.2.2 [Texte](02.2-types-texte.md)** — `VARCHAR`, `TEXT`, `CHAR`, `ENUM`, `SET` : chaînes de longueur fixe ou variable et listes de valeurs énumérées.
    - **2.2.3 [Temporels](02.3-types-temporels.md)** — `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR`, en tenant compte de l'extension de la plage `TIMESTAMP` au-delà de 2038.
    - **2.2.4 [Binaires](02.4-types-binaires.md)** — `BLOB`, `BINARY`, `VARBINARY` pour les données non textuelles.
    - **2.2.5 [Spécifiques MariaDB](02.5-types-specifiques-mariadb.md)** — `JSON`, `UUID`, `INET6` et `VECTOR`, ce dernier ouvrant la voie aux usages IA et à la recherche vectorielle.
    - **2.2.6 [Type XML basique](02.6-type-xml.md)** 🆕 — Prise en charge du type `XMLTYPE` pour la compatibilité avec Oracle.

- **2.3 [Création et gestion des bases de données](03-creation-gestion-bases.md)** — `CREATE DATABASE`, `USE`, `DROP DATABASE`, et le choix du jeu de caractères et de la collation (`utf8mb4` par défaut).

- **2.4 [Création et modification de tables](04-creation-modification-tables.md)** — `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` : définir le schéma d'une table et le faire évoluer.

- **2.5 [Contraintes](05-contraintes.md)** — `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `DEFAULT` : garantir la cohérence et l'intégrité référentielle des données.

- **2.6 [Insertion de données](06-insertion-donnees.md)** — `INSERT`, `INSERT ... SELECT` et `LOAD DATA` pour alimenter les tables, du cas unitaire au chargement en masse.

- **2.7 [Requêtes de sélection simples](07-requetes-selection-simples.md)** — `SELECT`, `WHERE`, `ORDER BY`, `LIMIT` : extraire, filtrer, trier et paginer les données.

- **2.8 [Mise à jour et suppression de données](08-mise-a-jour-suppression.md)** — `UPDATE`, `DELETE` et `TRUNCATE`, avec les précautions à prendre (clause `WHERE`, différences entre `DELETE` et `TRUNCATE`).

## Note de version

La majorité des notions de ce chapitre relèvent du tronc commun du SQL et sont stables depuis de nombreuses versions. Deux points méritent toutefois d'être signalés au regard de la 12.3 :

- Le **type XML basique** `XMLTYPE` (§2.2.6), marqué 🆕, fait partie des nouveautés de la série 12.x et vise la **compatibilité avec Oracle**.
- Le jeu de caractères `utf8mb4` par défaut et l'extension de la plage `TIMESTAMP` au-delà de 2038 (évoqués respectivement en §2.3 et §2.2.3) sont désormais du contenu standard, hérités de la 11.8.

---

← Chapitre précédent : [1. Introduction et Fondamentaux](../01-introduction-fondamentaux/README.md) · [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Introduction au langage SQL](/02-bases-du-sql/01-introduction-langage-sql.md)
