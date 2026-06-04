🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 — Stratégies d'indexation

> **Chapitre 5 — Index et Performance** · Section 5.5  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Savoir *comment* créer un index ([section 5.4](04-creation-gestion-index.md)) ne dit pas *lesquels* créer. C'est pourtant là que se joue l'essentiel : un schéma bien indexé répond aux requêtes en quelques millisecondes ; le même schéma mal indexé peut s'effondrer sous la charge. Cette section pose le **cadre stratégique** de l'indexation, avant que les sous-sections suivantes n'examinent en détail les trois grandes cibles à privilégier.

---

## Le principe directeur : indexer la charge, pas le schéma

La plus grande erreur consiste à indexer « au cas où » — ajouter un index sur chaque colonne dans l'espoir d'être couvert. Cette approche est contre-productive : elle multiplie les coûts (voir plus bas) sans garantir le moindre gain, car un index n'est utile que s'il sert des **requêtes réellement exécutées**.

Le bon réflexe s'énonce simplement :

> **On n'indexe pas une table, on indexe des requêtes.**

Concrètement, les colonnes candidates à un index sont celles qui apparaissent dans :

- les clauses **`WHERE`** (conditions de filtrage) ;
- les conditions de **jointure** (`JOIN … ON`) ;
- les clauses **`ORDER BY`** et **`GROUP BY`** (tri et regroupement).

C'est en partant du **profil de requêtes** de l'application — et non de la seule structure des tables — que l'on décide quoi indexer.

---

## L'équilibre permanent : ni trop, ni trop peu

L'indexation est un **arbitrage**, rappelé dès la [section 5.1](01-fonctionnement-index.md) : un index accélère les lectures, mais il a un **coût**. Chaque index supplémentaire :

- **ralentit les écritures** : tout `INSERT`, `UPDATE` ou `DELETE` doit le mettre à jour ;
- **consomme de l'espace** disque et de la **mémoire** (Buffer Pool) ;
- **complexifie le travail de l'optimiseur**, qui doit évaluer davantage de chemins d'accès.

Le **sous-indexage** (requêtes lentes faute d'index) et le **sur-indexage** (écritures pénalisées, ressources gaspillées) sont donc deux écueils symétriques. L'objectif n'est jamais d'indexer *davantage*, mais d'indexer **juste** — le minimum d'index qui couvre efficacement la charge réelle.

---

## Les trois cibles à fort rendement

La majorité des gains s'obtient en traitant trois familles de besoins, qui font l'objet des sous-sections de cette section :

| Cible | Clause SQL concernée | Section |
|-------|----------------------|---------|
| **Colonnes fréquemment filtrées** | `WHERE` | [5.5.1](05.1-index-colonnes-filtrees.md) |
| **Clés étrangères** | `JOIN … ON`, intégrité référentielle | [5.5.2](05.2-index-cles-etrangeres.md) |
| **Tri et regroupement** | `ORDER BY`, `GROUP BY` | [5.5.3](05.3-index-order-group.md) |

Ces trois axes ne sont pas cloisonnés : une même requête combine souvent filtrage, jointure et tri, et un seul index bien conçu peut alors servir plusieurs de ces rôles à la fois — c'est tout l'intérêt des index composites, abordés juste après.

---

## Principes transversaux

Quelques principes s'appliquent quelle que soit la cible.

### La sélectivité d'abord

Un index n'est rentable que sur des colonnes **suffisamment sélectives**, c'est-à-dire à forte **cardinalité** (beaucoup de valeurs distinctes), comme vu en [section 5.2.1](02.1-btree.md). Indexer une colonne à deux ou trois valeurs apporte peu : l'optimiseur préférera souvent un balayage. Les colonnes les plus discriminantes sont les meilleures candidates.

### Mutualiser plutôt que multiplier

Avant d'ajouter un index, se demander si un index existant — ou un **index composite** bien ordonné — ne pourrait pas déjà répondre au besoin. Un index sur `(client_id, date_commande)` sert aussi les requêtes sur `client_id` seul (principe du *left-most prefix*, [section 5.6](06-index-composites.md)). Mieux : un index **couvrant** ([section 5.9](09-index-covering.md)) peut répondre à une requête sans même accéder à la table. Penser « mutualisation » réduit le nombre total d'index.

### Éviter la redondance

Dans le prolongement du point précédent, on traque les index **redondants** ou **en doublon** : un index sur `(a)` devient inutile s'il existe déjà un index sur `(a, b)` ; deux index identiques sur les mêmes colonnes dans le même ordre sont du pur gaspillage. Chaque index superflu pèse sur les écritures et le stockage, sans contrepartie.

### Traquer les index inutiles

Un index qui n'est **jamais utilisé** par aucune requête ne fait que coûter. MariaDB permet de les repérer via le **Performance Schema** et le **sys schema** ([section 15.8](../15-performance-tuning/08-performance-schema-sys.md)), qui recensent les index jamais sollicités. Avant de supprimer un index suspecté inutile, on peut d'abord le rendre **invisible** ([section 5.10](10-invisible-progressive-indexes.md)) pour valider sans risque l'absence d'impact.

---

## Une démarche en pratique

L'indexation efficace est **itérative** et fondée sur la **mesure**, jamais sur l'intuition seule :

1. **Identifier** les requêtes fréquentes et lentes, via le *slow query log* ([section 15.7](../15-performance-tuning/07-analyse-requetes-lentes.md)).
2. **Analyser** leur plan d'exécution avec `EXPLAIN` ([section 5.7](07-analyse-plans-execution.md)) pour repérer les balayages complets et les colonnes filtrées, jointes ou triées sans index adapté.
3. **Ajouter ou ajuster** l'index, en privilégiant la mutualisation (composite, couvrant) plutôt que l'empilement.
4. **Mesurer** l'effet sur les lectures *et* vérifier l'absence de régression sur les écritures.
5. **Maintenir** : tenir les statistiques à jour (`ANALYZE TABLE`, [chapitre 11](../11-administration-configuration/06.2-analyze-table.md)) et **supprimer** les index devenus inutiles.

---

> ### 📝 À retenir  
>  
> - **On indexe des requêtes, pas des tables** : les candidats sont les colonnes des clauses `WHERE`, des jointures, et des `ORDER BY` / `GROUP BY`.  
> - L'indexation est un **arbitrage** : le **sur-indexage** (écritures et ressources pénalisées) est aussi nuisible que le **sous-indexage**. Viser le **minimum efficace**.  
> - Trois cibles à fort rendement : **colonnes filtrées** (5.5.1), **clés étrangères** (5.5.2), **tri/regroupement** (5.5.3).  
> - Principes transversaux : privilégier la **sélectivité**, **mutualiser** (composites et couvrants), **éviter la redondance**, **traquer les index inutiles**.  
> - Procéder de façon **itérative et mesurée** : *slow query log* → `EXPLAIN` → ajustement → mesure → maintenance.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.4 Création et gestion des index](04-creation-gestion-index.md)
- ➡️ Section suivante : [5.5.1 Index sur colonnes fréquemment filtrées](05.1-index-colonnes-filtrees.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Index sur colonnes fréquemment filtrées](/05-index-et-performance/05.1-index-colonnes-filtrees.md)
