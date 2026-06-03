🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.8 Mise à jour et suppression de données (UPDATE, DELETE, TRUNCATE)

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

Après avoir inséré et interrogé des données, il reste à les **modifier** et à les **supprimer**. Trois instructions s'en chargent : `UPDATE` modifie des lignes existantes, `DELETE` en supprime, et `TRUNCATE` vide entièrement une table. Un fil rouge traverse toute cette section : la clause **`WHERE`** est ici cruciale. L'oublier ne provoque pas une erreur, mais applique l'opération à **toutes les lignes** de la table — une cause classique d'incidents en production.

## `UPDATE` : modifier des lignes

`UPDATE` change la valeur d'une ou plusieurs colonnes pour les lignes désignées par `WHERE`. La nouvelle valeur peut être une constante ou une **expression** :

```sql
UPDATE client
SET pays = 'BE'
WHERE id = 42;

UPDATE produit
SET prix = prix * 1.10                 -- expression : +10 % sur le prix
WHERE categorie = 'electronique';
```

Le point de vigilance majeur tient en une ligne : **sans `WHERE`, toutes les lignes sont modifiées**.

```sql
UPDATE client SET actif = FALSE;       -- ⚠ SANS WHERE : met à jour TOUTES les lignes
```

MariaDB autorise `ORDER BY` et `LIMIT` sur un `UPDATE` mono-table, utile pour traiter les lignes par lots :

```sql
UPDATE commande SET statut = 'archivee'
WHERE statut = 'expediee'
ORDER BY cree_le
LIMIT 1000;
```

Des formes plus avancées existent : la mise à jour **multi-tables** (avec jointures) est abordée au chapitre 3, et la mise à jour s'appuyant sur une **CTE** (`WITH`) en section 4.4.1.

## `DELETE` : supprimer des lignes

`DELETE` supprime les lignes correspondant à `WHERE` :

```sql
DELETE FROM client WHERE id = 42;
DELETE FROM log    WHERE cree_le < '2025-01-01';
```

La même mise en garde s'applique, plus tranchante encore : **sans `WHERE`, toutes les lignes sont supprimées**.

```sql
DELETE FROM client;                    -- ⚠ SANS WHERE : supprime TOUTES les lignes
```

Comme `UPDATE`, `DELETE` accepte `ORDER BY` et `LIMIT`, ce qui permet de **supprimer par lots** — pratique pour purger une grande table sans saturer les ressources :

```sql
DELETE FROM log WHERE cree_le < '2025-01-01'
ORDER BY cree_le
LIMIT 5000;
```

MariaDB propose en outre la clause `RETURNING`, qui renvoie les lignes effectivement supprimées :

```sql
DELETE FROM commande WHERE statut = 'annulee'
RETURNING id, reference;
```

La suppression multi-tables (jointures) relève également du chapitre 3.

## `TRUNCATE` : vider une table

`TRUNCATE TABLE` **vide intégralement** une table — toutes les lignes — tout en conservant sa structure :

```sql
TRUNCATE TABLE log_temporaire;
```

Sous MariaDB, `TRUNCATE` n'est pas une variante de `DELETE` : c'est une opération **DDL**. Elle entraîne une **validation implicite** (elle ne peut donc pas être annulée par `ROLLBACK`), **réinitialise le compteur `AUTO_INCREMENT`**, ne déclenche **pas** les triggers `DELETE`, et est **bien plus rapide** que `DELETE` pour vider une grande table, car elle reconstruit le stockage plutôt que d'effacer ligne à ligne. À noter : `TRUNCATE` échoue si la table est référencée par une **clé étrangère** depuis une autre table (sous InnoDB).

## `DELETE` ou `TRUNCATE` ?

Le choix entre les deux dépend du besoin : suppression **sélective** ou vidage **complet**.

| Critère | `DELETE` | `TRUNCATE TABLE` |
|---|---|---|
| Catégorie | DML | DDL |
| Sélection (`WHERE`) | oui | non (toutes les lignes) |
| Transactionnel / `ROLLBACK` | oui | non (validation implicite) |
| Compteur `AUTO_INCREMENT` | inchangé | réinitialisé |
| Triggers `DELETE` | déclenchés | non |
| Vitesse (vidage complet) | plus lente (ligne à ligne) | très rapide |
| Nombre de lignes affectées | renvoyé | non |

En résumé, on utilise `DELETE` dès qu'une condition entre en jeu ou qu'on souhaite pouvoir annuler l'opération, et `TRUNCATE` pour réinitialiser rapidement et complètement une table (table temporaire, jeu de test, etc.).

## Travailler en sécurité

Compte tenu du caractère potentiellement destructeur d'`UPDATE` et de `DELETE`, quelques réflexes évitent bien des regrets. D'abord, **vérifier d'abord avec un `SELECT`** utilisant exactement le même `WHERE`, pour visualiser les lignes qui seront touchées. Ensuite, **opérer dans une transaction**, ce qui permet d'annuler en cas d'erreur :

```sql
-- 1. Vérifier les lignes concernées
SELECT * FROM client WHERE pays = 'XX';

-- 2. Modifier dans une transaction réversible
START TRANSACTION;
UPDATE client SET actif = FALSE WHERE pays = 'XX';
-- contrôler le résultat...
COMMIT;        -- ou ROLLBACK pour tout annuler
```

MariaDB propose aussi un garde-fou : le mode `sql_safe_updates`. Lorsqu'il est actif, un `UPDATE` ou un `DELETE` sans `WHERE` exploitant une clé (ou sans `LIMIT`) est **refusé**, ce qui protège contre les modifications massives accidentelles.

```sql
SET sql_safe_updates = 1;
```

Enfin, lorsque l'historique des données importe, on peut préférer une suppression « logique » (un indicateur `supprime` plutôt qu'un effacement physique) ou recourir aux **tables temporelles** qui conservent l'historique (§18.2), plutôt qu'à un `DELETE` définitif.

## En résumé

`UPDATE` modifie des lignes (avec des expressions possibles dans `SET`) et `DELETE` les supprime — dans les deux cas, **la clause `WHERE` détermine la portée**, et son oubli affecte toute la table. Tous deux acceptent `ORDER BY`/`LIMIT` pour un traitement par lots, et `DELETE` offre `RETURNING`. `TRUNCATE`, opération DDL, vide intégralement et rapidement une table en réinitialisant l'`AUTO_INCREMENT`, mais sans condition ni possibilité d'annulation. La prudence — `SELECT` préalable, transactions, `sql_safe_updates` — est la meilleure alliée face à ces commandes. Ce chapitre a ainsi parcouru tout le cycle de vie des données : création des structures, insertion, interrogation, modification et suppression. Le chapitre 3 ouvre la voie aux requêtes plus riches : agrégation, regroupement et jointures.

---

← Section précédente : [2.7 Requêtes de sélection simples](07-requetes-selection-simples.md) · [Sommaire du chapitre](README.md) · Chapitre suivant : [3. Requêtes SQL Intermédiaires](../03-requetes-sql-intermediaires/README.md) →

⏭️ [Partie 2 : Requêtes SQL Intermédiaires et Avancées (Intermédiaire)](/partie-02-requetes-sql-intermediaires-avancees.md)
