🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3. Requêtes SQL Intermédiaires

> **Partie 2 : Requêtes SQL Intermédiaires et Avancées** · Niveau : *Intermédiaire*  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La [Partie 1](../partie-01-introduction-fondamentaux.md) vous a permis de créer des bases et des tables, d'y insérer des données et d'écrire vos premières requêtes de lecture avec `SELECT`, `WHERE`, `ORDER BY` et `LIMIT`. Vous savez désormais *stocker* et *retrouver* des lignes. Ce chapitre marque une étape : passer de la simple consultation à la **véritable interrogation des données**, c'est-à-dire produire de l'information à partir de tables brutes.

Les requêtes intermédiaires constituent le cœur du SQL que l'on écrit au quotidien, que l'on soit développeur ou DBA. On y apprend à **résumer** des données (agrégation), à les **regrouper** selon des critères, à **combiner** plusieurs tables entre elles (jointures), à **imbriquer** des requêtes les unes dans les autres (sous-requêtes), à **réunir** des ensembles de résultats, et enfin à **transformer** les valeurs à la volée — textes, dates et expressions conditionnelles. Ces techniques se combinent ensuite librement : une requête réelle mêle souvent jointures, agrégats et conditions dans une même instruction.

Ce chapitre prépare directement le terrain pour la [Partie 3](../partie-03-index-transactions-performance.md), où l'on s'intéressera aux index et à la performance : bien comprendre *comment* une requête lit les données est le préalable indispensable pour, plus tard, la rendre rapide.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- **Calculer des valeurs synthétiques** sur un ensemble de lignes avec les fonctions d'agrégation (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) et comprendre leur comportement face aux valeurs `NULL`.
- **Regrouper les données** avec `GROUP BY` et **filtrer ces groupes** avec `HAVING`, en distinguant clairement le rôle de `WHERE` (filtre des lignes) de celui de `HAVING` (filtre des groupes).
- **Combiner plusieurs tables** à l'aide des différents types de jointures et choisir la jointure adaptée à chaque besoin.
- **Écrire des sous-requêtes** scalaires, ensemblistes et corrélées, et les placer dans les clauses `SELECT`, `FROM` ou `WHERE`.
- **Réunir des résultats** issus de plusieurs requêtes avec les opérateurs ensemblistes (`UNION`, `INTERSECT`, `EXCEPT`).
- **Manipuler les chaînes de caractères et les valeurs temporelles** grâce aux fonctions intégrées.
- **Exprimer une logique conditionnelle** directement en SQL (`CASE`, `IF`, `IFNULL`, `COALESCE`, `NULLIF`).
- **Identifier les apports de compatibilité Oracle** introduits dans la série 12.x et savoir dans quel contexte ils s'activent.

---

## Prérequis

Ce chapitre suppose la maîtrise des notions de la [Partie 1](../partie-01-introduction-fondamentaux.md), en particulier :

- les [types de données MariaDB](../02-bases-du-sql/02-types-de-donnees.md) ;
- la [création et la modification de tables](../02-bases-du-sql/04-creation-modification-tables.md) ainsi que les [contraintes](../02-bases-du-sql/05-contraintes.md) (notamment les clés étrangères, centrales pour les jointures) ;
- les [requêtes de sélection simples](../02-bases-du-sql/07-requetes-selection-simples.md) (`SELECT`, `WHERE`, `ORDER BY`, `LIMIT`).

Une première familiarité avec les valeurs `NULL` est utile. Leur traitement complet relève de la logique ternaire, abordée plus loin au chapitre [4.6 — Gestion des valeurs NULL](../04-concepts-avances-sql/06-gestion-valeurs-null.md).

---

## Plan du chapitre

| Section | Sujet | Points clés |
|---------|-------|-------------|
| [3.1](01-fonctions-agregation.md) | Fonctions d'agrégation | `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `DISTINCT`, comportement avec `NULL` |
| [3.2](02-regroupement-donnees.md) | Regroupement de données | `GROUP BY`, `HAVING`, distinction `WHERE`/`HAVING` |
| [3.3](03-jointures.md) | Jointures | `INNER`, `LEFT`/`RIGHT`, `CROSS`, self-join, syntaxe `( + )` en mode Oracle 🆕 |
| [3.4](04-sous-requetes.md) | Sous-requêtes et requêtes imbriquées | sous-requêtes scalaires, `IN`, `EXISTS`, corrélées |
| [3.5](05-operateurs-ensemblistes.md) | Opérateurs ensemblistes | `UNION` / `UNION ALL`, `INTERSECT`, `EXCEPT` |
| [3.6](06-fonctions-chaines.md) | Fonctions de chaînes de caractères | concaténation, extraction, recherche, transformation |
| [3.7](07-fonctions-dates-heures.md) | Fonctions de dates et heures | calcul et formatage de dates, fonctions Oracle `TO_DATE`/`TRUNC`/`TO_NUMBER`/`TO_CHAR` 🆕 |
| [3.8](08-expressions-conditionnelles.md) | Expressions conditionnelles | `CASE`, `IF`, `IFNULL`, `COALESCE`, `NULLIF` |

---

## À propos de la version et des conventions

Sauf mention contraire, les exemples de ce chapitre s'appliquent à **MariaDB 12.3 LTS** avec un `sql_mode` par défaut. Deux nouveautés 🆕 de la série 12.x relèvent de la **compatibilité Oracle** et n'entrent en jeu que lorsque le mode Oracle est actif (`SET sql_mode = 'ORACLE'`) :

- la syntaxe historique `( + )` pour les jointures externes ([3.3.5](03.5-oracle-outer-join.md)) ;
- les fonctions de conversion `TO_DATE`, `TRUNC`, `TO_NUMBER` et `TO_CHAR` (avec le modificateur de format `FM`) ([3.7.1](07.1-fonctions-oracle.md)).

Ces apports s'adressent avant tout aux équipes en cours de migration : ils sont remis en contexte dans la section [19.2.1 — Migration depuis Oracle](../19-migration-compatibilite/02.1-depuis-oracle.md). Dans l'écriture courante de requêtes MariaDB, on privilégiera les fonctions natives présentées dans le reste du chapitre.

Pour situer ces nouveautés dans l'ensemble des évolutions de la version, voir l'[Annexe F — Nouveautés MariaDB 12.3 LTS](../annexes/f-nouveautes-12-3/README.md).

---

## Navigation

- ⬅️ Partie précédente : [Partie 1 — Introduction et Fondamentaux](../partie-01-introduction-fondamentaux.md)
- ➡️ Section suivante : [3.1 — Fonctions d'agrégation](01-fonctions-agregation.md)
- ⬆️ Retour au [Sommaire](../SOMMAIRE.md)

⏭️ [Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)](/03-requetes-sql-intermediaires/01-fonctions-agregation.md)
