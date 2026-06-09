🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Niveaux d'isolation

L'**isolation** (le « I » d'ACID, voir [§6.1](01-concept-transaction-acid.md)) est la seule des quatre propriétés qui soit **réglable**. MariaDB permet de choisir, parmi quatre **niveaux d'isolation** normalisés, le degré de protection des transactions les unes vis-à-vis des autres. Cette section présente le **principe général** ; chaque niveau est ensuite détaillé dans sa propre sous-section (6.3.1 à 6.3.4).

## Le compromis : isolation contre concurrence

Plus une transaction est isolée des autres, plus son comportement est **prévisible** — mais plus le SGBD doit poser de **verrous** et restreindre le parallélisme, ce qui **réduit le débit** sous forte concurrence. À l'inverse, une isolation faible **maximise la concurrence**, mais expose la transaction à observer des données incohérentes.

Choisir un niveau d'isolation, c'est donc arbitrer entre **rigueur** et **performance**. Les niveaux standard se distinguent précisément par les **anomalies de lecture** qu'ils autorisent ou interdisent.

## Les anomalies de lecture

Trois anomalies, définies par le standard SQL, servent de référence pour classer les niveaux d'isolation.

### Lecture sale (*dirty read*)

Une transaction lit des données **modifiées par une autre transaction non encore validée**. Si cette dernière est finalement annulée (`ROLLBACK`), la première a travaillé sur des valeurs qui n'ont jamais réellement existé.

### Lecture non répétable (*non-repeatable read*)

Au sein d'une même transaction, une **seconde lecture** d'une ligne déjà lue renvoie une **valeur différente**, parce qu'une autre transaction a entre-temps **modifié (ou supprimé)** cette ligne et validé son changement.

### Lecture fantôme (*phantom read*)

Une transaction ré-exécute une requête portant sur un **ensemble de lignes** (par exemple `WHERE montant > 1000`) et constate l'**apparition** (ou la disparition) de lignes — des « fantômes » — insérées ou supprimées puis validées par une autre transaction. La nuance avec la lecture non répétable : celle-ci concerne la **valeur** d'une ligne existante, tandis que la lecture fantôme concerne la **population** de lignes satisfaisant un critère.

> Au-delà de ces trois anomalies « standard », d'autres existent, comme la **mise à jour perdue** (*lost update*), où deux transactions écrasent mutuellement leurs modifications. C'est ce type de problème que combat l'isolation par instantané activée par défaut (depuis MariaDB 11.6.2, donc en 12.3 ; voir [§6.9](09-snapshot-isolation.md)).

## Les quatre niveaux d'isolation standard

Le standard SQL définit quatre niveaux, du plus permissif au plus strict :

| Niveau d'isolation | Lecture sale | Lecture non répétable | Lecture fantôme |
|--------------------|--------------|-----------------------|-----------------|
| **READ UNCOMMITTED** | Possible | Possible | Possible |
| **READ COMMITTED** | Évitée | Possible | Possible |
| **REPEATABLE READ** | Évitée | Évitée | Possible *(selon le standard)* |
| **SERIALIZABLE** | Évitée | Évitée | Évitée |

Chaque niveau fait l'objet d'une sous-section dédiée :

- **[6.3.1 — READ UNCOMMITTED](03.1-read-uncommitted.md)** : le niveau le plus permissif ; autorise jusqu'aux lectures sales. Rarement employé.
- **[6.3.2 — READ COMMITTED](03.2-read-committed.md)** : ne lit que des données **validées** ; chaque requête voit le dernier état validé au moment où elle s'exécute.
- **[6.3.3 — REPEATABLE READ](03.3-repeatable-read.md)** : **niveau par défaut d'InnoDB** ; garantit des lectures stables tout au long de la transaction.
- **[6.3.4 — SERIALIZABLE](03.4-serializable.md)** : isolation **maximale** ; les transactions se comportent comme si elles s'exécutaient les unes après les autres.

## La spécificité d'InnoDB

Le niveau d'isolation **par défaut** d'InnoDB (donc de MariaDB) est **`REPEATABLE READ`**.

Un point important distingue InnoDB du strict minimum imposé par le standard : à `REPEATABLE READ`, InnoDB **évite en pratique les lectures fantômes** dans la plupart des cas. Les lectures simples (non verrouillantes) s'appuient sur un **instantané cohérent** fourni par le **MVCC** ([§6.6](06-mvcc.md)), tandis que les lectures verrouillantes (`SELECT … FOR UPDATE`) utilisent des **verrous d'intervalle** (*next-key locks*) qui empêchent l'insertion de lignes fantômes ([§6.4](04-verrous.md)). Le `REPEATABLE READ` d'InnoDB est donc **plus fort** que la ligne correspondante du tableau ci-dessus.

> **Isolation par instantané (par défaut).** La variable `innodb_snapshot_isolation` est **activée par défaut** (depuis MariaDB 11.6.2, donc en 11.8 LTS comme en 12.3), faisant de `REPEATABLE READ` une véritable **isolation par instantané** qui protège contre les mises à jour perdues. Le comportement détaillé est traité en [§6.9](09-snapshot-isolation.md).

## Définir et consulter le niveau d'isolation

Le niveau se définit avec `SET … TRANSACTION ISOLATION LEVEL`, selon **trois portées** :

```sql
-- 1) Pour la prochaine transaction UNIQUEMENT (à émettre avant START TRANSACTION)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 2) Pour toutes les transactions ULTÉRIEURES de la session courante
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 3) Valeur par défaut des NOUVELLES sessions (n'affecte pas les sessions déjà ouvertes)
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

Pour **consulter** le niveau courant :

```sql
SELECT @@transaction_isolation;          -- niveau de la session
SELECT @@global.transaction_isolation;   -- niveau global
```

Le niveau par défaut au démarrage du serveur peut aussi être fixé dans le fichier de configuration (noter la forme avec tirets) :

```ini
[mysqld]
transaction-isolation = READ-COMMITTED
```

> Rappel ([§6.2](02-gestion-transactions.md)) : le niveau d'isolation **ne** se définit **pas** dans `START TRANSACTION` ; il faut passer par `SET TRANSACTION ISOLATION LEVEL`.

## Comment choisir ?

- **`READ COMMITTED` et `REPEATABLE READ`** couvrent l'immense majorité des besoins transactionnels (OLTP). `REPEATABLE READ`, défaut d'InnoDB, est un excellent point de départ.
- **`READ UNCOMMITTED`** est à réserver à des cas très particuliers tolérant des lectures approximatives (certains reportings non critiques).
- **`SERIALIZABLE`** n'est justifié que lorsque la cohérence doit être absolue, au prix d'une concurrence fortement réduite.
- Règle générale : **monter en isolation accroît la contention** (davantage de verrous, plus d'attentes et de risques d'interblocage — voir [§6.5](05-deadlocks-resolution.md)). Choisissez le niveau le plus faible qui satisfait réellement vos besoins de cohérence.

---

> **Section suivante** : [6.3.1 — READ UNCOMMITTED : lectures sales possibles](03.1-read-uncommitted.md), premier des quatre niveaux d'isolation passés en revue en détail.

⏭️ [READ UNCOMMITTED : Dirty reads possibles](/06-transactions-et-concurrence/03.1-read-uncommitted.md)
