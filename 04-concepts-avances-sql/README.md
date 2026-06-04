🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4. Concepts Avancés SQL

> **Partie 2 — Requêtes SQL Intermédiaires et Avancées** · Niveau : Intermédiaire → Avancé  
> Version de référence : **MariaDB 12.3 LTS**

Une fois les fondamentaux du langage SQL maîtrisés (chapitres 2 et 3), il devient possible de résoudre dans une seule requête déclarative des problèmes qui, autrement, imposeraient plusieurs allers-retours avec l'application ou du code procédural complexe. Ce chapitre rassemble les **constructions SQL avancées** de MariaDB : traitements analytiques, hiérarchiques et semi-structurés.

L'idée directrice est simple : *déplacer la logique vers la base de données lorsque c'est pertinent*. Calculer un classement, parcourir une arborescence, produire un tableau croisé ou interroger un document JSON sont autant d'opérations que MariaDB sait exprimer nativement. Bien utilisées, ces fonctionnalités réduisent le volume de données transférées, simplifient le code applicatif et laissent l'optimiseur du moteur faire son travail.

La plupart de ces concepts (CTE, fonctions de fenêtrage, JSON, expressions régulières) sont stables et disponibles depuis plusieurs versions LTS. La 12.3 y apporte surtout des **raffinements alignés sur le standard SQL**, signalés au fil du chapitre par le marqueur 🆕.

## 🎯 Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- Écrire des requêtes **récursives** pour traiter des structures hiérarchiques (organigrammes, nomenclatures, graphes) et pour générer des séries de valeurs.
- Exploiter les **fonctions de fenêtrage** (*window functions*) afin de produire classements, cumuls, moyennes mobiles et comparaisons entre lignes, sans pour autant agréger les données.
- Restructurer des résultats à l'aide de **requêtes pivotées** et de transformations lignes ↔ colonnes.
- Factoriser et clarifier des requêtes complexes grâce aux **CTE** (*Common Table Expressions*), y compris au sein d'instructions `UPDATE` et `DELETE`.
- Maîtriser la **logique ternaire** associée aux valeurs `NULL` et éviter les pièges classiques.
- Stocker, interroger, valider et indexer des données **JSON** dans un modèle relationnel.
- Filtrer et transformer du texte au moyen des **expressions régulières** (moteur PCRE2).

## 📋 Prérequis

Ce chapitre suppose la maîtrise des notions abordées précédemment :

- **Chapitre 2 — Bases du SQL** : types de données, création de tables, `SELECT` / `WHERE` / `ORDER BY` / `LIMIT`.
- **Chapitre 3 — Requêtes SQL Intermédiaires** : fonctions d'agrégation, `GROUP BY` / `HAVING`, jointures, sous-requêtes et expressions conditionnelles (`CASE`, `IFNULL`, `COALESCE`…).

Une familiarité avec les index et l'optimiseur (chapitre 5) n'est pas indispensable, mais elle facilitera la compréhension des remarques de performance évoquées au fil des sections, notamment pour le JSON et son indexation.

## 🗺️ Structure du chapitre

Le chapitre s'organise en quatre grandes familles de concepts.

### Requêtes hiérarchiques et analytiques

- **4.1 — Requêtes récursives (`WITH RECURSIVE`)** : parcours d'arborescences et de graphes, génération de séquences.
- **4.2 — Window Functions** : fonctions analytiques calculées sur une *fenêtre* de lignes.
    - 4.2.1 Fonctions de rang (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`)
    - 4.2.2 Fonctions de valeur (`LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`)
    - 4.2.3 Frames de fenêtre (`ROWS`, `RANGE`, `GROUPS`)
    - 4.2.4 Cas d'usage : Top N, moyenne mobile, cumuls
- **4.3 — Requêtes pivotées et transformations** : produire des tableaux croisés. MariaDB ne dispose pas d'opérateur `PIVOT` natif ; le résultat s'obtient par combinaison de `CASE` et d'agrégation.

### Lisibilité et requêtes complexes

- **4.4 — Expressions de table communes (CTE)** : la clause `WITH` pour structurer et factoriser les requêtes.
    - 4.4.1 `UPDATE` / `DELETE` lisant une CTE 🆕
- **4.5 — Requêtes complexes multi-tables** : composer jointures, sous-requêtes et CTE.
- **4.6 — Gestion des valeurs `NULL`** : logique ternaire (`TRUE` / `FALSE` / `UNKNOWN`) et ses conséquences sur les filtres, jointures et agrégations.

### Données semi-structurées : JSON

- **4.7 — JSON dans MariaDB** : stockage (le type `JSON` est un *alias* de `LONGTEXT`), fonctions principales, opérateur raccourci et validation.
    - 4.7.1 Stockage et type de données JSON
    - 4.7.2 Fonctions JSON (`JSON_EXTRACT`, `JSON_SET`, `JSON_ARRAY`, etc.)
    - 4.7.3 Opérateur raccourci (`->>`)
    - 4.7.4 Prédicat `IS JSON` (standard SQL) et suppression de la limite de profondeur 🆕
- **4.8 — JSON Path Expressions avancées** : chemins, filtres et caractères génériques.
- **4.9 — JSON Schema Validation** : valider la structure d'un document avant insertion.
- **4.10 — Indexation de colonnes virtuelles extraites du JSON** : concilier la souplesse du JSON et la performance des index.

### Manipulation de texte

- **4.11 — Expressions régulières** : `REGEXP`, `REGEXP_REPLACE`, `REGEXP_SUBSTR` (moteur PCRE2).

## 🆕 Nouveautés MariaDB 12.3 abordées dans ce chapitre

Deux raffinements de la série 12.x, alignés sur le standard SQL, sont traités ici :

- **`UPDATE` / `DELETE` lisant une CTE** (§ 4.4.1) : une CTE déclarée en tête d'une instruction de modification peut désormais être référencée par celle-ci, ce qui simplifie certaines mises à jour conditionnelles auparavant écrites sous forme de sous-requêtes.
- **Prédicat `IS JSON`** et **levée de la limite de profondeur** des documents (§ 4.7.4) : `IS JSON` offre une vérification de validité conforme au standard, et les documents profondément imbriqués ne sont plus contraints par l'ancienne limite de niveaux.

## 💡 Conseil d'approche

Ces concepts gagnent à être abordés dans l'ordre : les CTE et les fonctions de fenêtrage constituent un socle réutilisé ensuite dans les requêtes multi-tables et dans le traitement du JSON. La logique ternaire (§ 4.6), bien que présentée à mi-parcours, sous-tend en réalité l'ensemble du chapitre : un filtre, une jointure ou une agrégation peut produire des résultats inattendus dès qu'une valeur `NULL` entre en jeu. Mieux vaut donc s'y reporter en cas de doute, quelle que soit la section en cours.

---

**Section suivante :** [4.1 — Requêtes récursives (`WITH RECURSIVE`)](01-requetes-recursives.md)

⏭️ [Requêtes récursives (WITH RECURSIVE)](/04-concepts-avances-sql/01-requetes-recursives.md)
