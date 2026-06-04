🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Verrous

Les **verrous** (*locks*) sont le mécanisme par lequel MariaDB **arbitre les accès concurrents** aux données et fait respecter les niveaux d'isolation ([§6.3](03-niveaux-isolation.md)). Lorsqu'une transaction doit garantir qu'une donnée ne change pas sous ses pieds, ou réserver des lignes en vue de les modifier, elle pose un verrou ; les autres transactions qui veulent un accès incompatible doivent alors **attendre**.

Il faut d'emblée distinguer deux modes d'accès en lecture :

- les **lectures non verrouillantes** (`SELECT` simple), qui sous InnoDB s'appuient sur le **MVCC** ([§6.6](06-mvcc.md)) et ne posent **aucun verrou** ;
- les **lectures verrouillantes** (`SELECT … FOR UPDATE`, `LOCK IN SHARE MODE`) et les **écritures** (`UPDATE`, `DELETE`), qui posent des verrous.

Cette section décrit les verrous explicites de table (`LOCK TABLES`) et, surtout, les verrous de ligne d'InnoDB sur lesquels repose le travail transactionnel.

## Deux granularités : table et ligne

| Granularité | Mécanisme | Moteur | Concurrence |
|-------------|-----------|--------|-------------|
| **Table** | `LOCK TABLES` (explicite), verrous implicites | tous (inhérent à MyISAM) | faible (toute la table bloquée) |
| **Ligne** | verrous S/X sur les enregistrements d'index | **InnoDB** | élevée (seules les lignes concernées sont verrouillées) |

En pratique, avec InnoDB on privilégie **presque toujours** les verrous de ligne, bien plus fins, et l'on réserve `LOCK TABLES` à des cas particuliers.

## Verrous de table : `LOCK TABLES` / `UNLOCK TABLES`

`LOCK TABLES` pose un verrou **explicite au niveau de la table entière**, pour la session courante.

```sql
LOCK TABLES comptes WRITE, journal READ;
  -- … opérations sur comptes (lecture/écriture) et journal (lecture seule) …
UNLOCK TABLES;
```

- **`READ`** (partagé) : la session **et** les autres peuvent **lire** la table ; **personne ne peut l'écrire** (y compris la session détentrice).
- **`WRITE`** (exclusif) : seule la session détentrice peut lire **et écrire** ; toutes les autres sont **bloquées**.

Le verrou est conservé jusqu'à `UNLOCK TABLES`, la fin de la session, ou un nouveau `LOCK TABLES`. Deux points de vigilance :

- pendant qu'un `LOCK TABLES` est actif, la session **ne peut accéder qu'aux tables explicitement verrouillées** ;
- `LOCK TABLES` provoque une **validation implicite** de la transaction en cours ([§6.2](02-gestion-transactions.md)). Mêler `LOCK TABLES` et transactions est donc délicat, et généralement déconseillé avec InnoDB.

> `LOCK TABLES` reste surtout utile pour MyISAM (qui n'a pas de verrous de ligne) ou pour certaines opérations de maintenance. Pour le transactionnel, on s'appuie sur les verrous de ligne ci-dessous.

## Verrous de ligne InnoDB : partagés (S) et exclusifs (X)

InnoDB pose des verrous **au niveau de la ligne** (plus précisément, des **enregistrements d'index**), de deux types :

- **verrou partagé (S, *shared*)** : posé par les lectures verrouillantes `LOCK IN SHARE MODE`. Plusieurs transactions peuvent détenir simultanément un verrou S sur la même ligne. Il autorise la lecture, **interdit la modification**.
- **verrou exclusif (X, *exclusive*)** : posé par les écritures (`UPDATE`, `DELETE`) et par `SELECT … FOR UPDATE`. **Un seul** détenteur ; incompatible avec tout autre verrou (S ou X).

**Table de compatibilité** (verrou demandé vs verrou déjà détenu sur la même ligne) :

| | S détenu | X détenu |
|:---:|:---:|:---:|
| **Demande S** | ✅ Compatible | ⏳ Attente |
| **Demande X** | ⏳ Attente | ⏳ Attente |

InnoDB pose aussi automatiquement des **verrous d'intention** (IS, IX) au niveau de la table : ils signalent qu'une transaction détient (ou veut acquérir) des verrous de ligne S ou X, sans bloquer les autres verrous de ligne. Leur gestion est transparente.

> Les verrous de ligne sont **conservés jusqu'à la fin de la transaction** (`COMMIT`/`ROLLBACK`) — et non à la fin de chaque instruction (à l'exception, à `READ COMMITTED`, des lignes ne correspondant pas au `WHERE` : voir la lecture semi-cohérente en [§6.3.2](03.2-read-committed.md)).

## Lectures verrouillantes : `FOR UPDATE` et `LOCK IN SHARE MODE`

Une **lecture verrouillante** permet de lire des lignes **en posant un verrou**, typiquement pour un schéma « lire puis modifier » (*read-modify-write*) à l'abri des accès concurrents.

### `SELECT … FOR UPDATE`

Pose un verrou **exclusif (X)** sur les lignes lues : aucune autre transaction ne peut les modifier ni les verrouiller jusqu'au `COMMIT`.

```sql
START TRANSACTION;
  SELECT solde FROM comptes WHERE id = 1 FOR UPDATE;    -- réserve la ligne
  UPDATE comptes SET solde = solde - 100 WHERE id = 1;  -- modification sûre
COMMIT;
```

### `SELECT … LOCK IN SHARE MODE`

Pose un verrou **partagé (S)** : les autres transactions peuvent encore **lire** la ligne (y compris en `LOCK IN SHARE MODE`), mais **pas la modifier** jusqu'au `COMMIT`.

```sql
SELECT solde FROM comptes WHERE id = 1 LOCK IN SHARE MODE;
```

> ⚠️ **Spécificité MariaDB** : le verrou de lecture partagée s'écrit **`LOCK IN SHARE MODE`**. La syntaxe `SELECT … FOR SHARE` introduite par **MySQL 8.0** n'est **pas** reconnue par MariaDB (elle provoque une erreur de syntaxe `1064`). Les options `NOWAIT` et `SKIP LOCKED` restent néanmoins utilisables après `LOCK IN SHARE MODE`.

### Démonstration : `FOR UPDATE` sérialise un *read-modify-write*

| Étape | Session A | Session B |
|:---:|---|---|
| ① | `START TRANSACTION;` | |
| ② | `SELECT … id = 1 FOR UPDATE;` → **1000** *(verrou X)* | |
| ③ | | `UPDATE comptes SET solde = 900 WHERE id = 1;` → **bloquée** |
| ④ | `UPDATE … ; COMMIT;` *(libère le verrou)* | |
| ⑤ | | l'`UPDATE` de B se débloque et s'exécute |

Tant que A n'a pas validé, B attend : les deux transactions ne peuvent pas modifier la ligne « en même temps ».

## Options `NOWAIT` et `SKIP LOCKED`

Par défaut, une lecture verrouillante **attend** la libération des verrous (jusqu'à `innodb_lock_wait_timeout`, 50 s par défaut). Deux options modifient ce comportement :

- **`NOWAIT`** — renvoie **immédiatement une erreur** si les lignes sont déjà verrouillées, au lieu d'attendre :

```sql
SELECT … FROM comptes WHERE id = 1 FOR UPDATE NOWAIT;
```

- **`SKIP LOCKED`** — **ignore** les lignes verrouillées et ne renvoie que celles immédiatement disponibles. Idéal pour distribuer des tâches depuis une file (*job queue*) sans contention :

```sql
SELECT id FROM taches
 WHERE statut = 'à traiter'
 ORDER BY id
 LIMIT 1
 FOR UPDATE SKIP LOCKED;
```

## Granularité fine : record, gap et next-key locks

Au sein des verrous de ligne, InnoDB distingue trois variantes — clés pour comprendre la prévention des fantômes ([§6.3.3](03.3-repeatable-read.md)) :

- **Record lock** : verrou sur un **enregistrement d'index** précis.
- **Gap lock** : verrou sur l'**intervalle** (le « trou ») entre deux enregistrements d'index ; il **empêche l'insertion** de nouvelles lignes dans cet intervalle, sans verrouiller d'enregistrement existant.
- **Next-key lock** : combinaison d'un *record lock* et du *gap lock* qui le précède. C'est le **verrou par défaut à `REPEATABLE READ`** : en verrouillant à la fois la ligne et l'espace qui la précède, il **bloque les insertions fantômes**.

> ⚠️ **Les verrous portent sur les enregistrements d'index.** Si une requête de modification ou de lecture verrouillante **ne peut pas utiliser d'index** pour cibler ses lignes, InnoDB doit **verrouiller tous les enregistrements parcourus** — potentiellement **toute la table**. **Indexer les colonnes du `WHERE`** est donc essentiel non seulement pour la performance, mais aussi pour **limiter l'étendue des verrous** (voir [Chapitre 5 — Index et Performance](../05-index-et-performance/README.md)).

## Verrous de métadonnées (MDL)

Indépendamment des verrous de données, MariaDB pose des **verrous de métadonnées** (*metadata locks*) sur les tables qu'une transaction utilise. Une transaction conserve ces MDL **jusqu'à sa fin** : tant qu'elle est ouverte, une opération **DDL** (`ALTER TABLE`, `DROP TABLE`…) sur les mêmes tables sera **bloquée**.

> C'est l'une des raisons pour lesquelles une transaction laissée ouverte trop longtemps peut **bloquer une modification de schéma**. Pour des changements de schéma sans interruption, voir [§18.11 — Online Schema Change](../18-fonctionnalites-avancees/11-online-schema-change.md).

## Observer les verrous

Pour diagnostiquer attentes et contentions :

- **`SHOW ENGINE INNODB STATUS`** — sa section *TRANSACTIONS* liste les verrous détenus et les attentes en cours ;
- **`information_schema.INNODB_TRX`** — les transactions InnoDB actives (utile pour repérer une transaction qui retient des verrous) ;
- les **tables de verrous** du **Performance Schema** / **`INFORMATION_SCHEMA`**, pour le détail des verrous et des relations d'attente.

> Des requêtes de diagnostic prêtes à l'emploi (et adaptées à votre version) figurent en [Annexe C.1](../annexes/c-requetes-sql-reference/01-requetes-administration.md) et [C.2](../annexes/c-requetes-sql-reference/02-requetes-monitoring.md).

## Bonnes pratiques

- **Transactions courtes** : plus une transaction reste ouverte, plus longtemps ses verrous (et ses MDL) pénalisent les autres.
- **Indexer les colonnes filtrées** : pour que les verrous restent **ciblés** et n'escaladent pas vers un verrouillage massif.
- **Accéder aux ressources dans un ordre constant** entre transactions : c'est la première parade aux **interblocages** ([§6.5](05-deadlocks-resolution.md)).
- **`FOR UPDATE`** pour les schémas *read-modify-write* ; **`SKIP LOCKED`** pour les files de tâches concurrentes.
- **Préférer les verrous de ligne** (InnoDB) à `LOCK TABLES` pour le transactionnel.

## À retenir

- Les verrous font respecter l'isolation ; les `SELECT` simples (MVCC) **ne verrouillent pas**, les lectures verrouillantes et les écritures **oui**.
- **`LOCK TABLES`** = verrou de table (grossier, valide implicitement la transaction) ; à éviter avec InnoDB.
- InnoDB : verrous de ligne **partagés (S)** et **exclusifs (X)** ; `FOR UPDATE` → X, `LOCK IN SHARE MODE` → S.
- **`NOWAIT`** (erreur immédiate) et **`SKIP LOCKED`** (ignorer les lignes verrouillées) affinent le comportement d'attente.
- Variantes **record / gap / next-key** ; le *next-key lock* (défaut à `REPEATABLE READ`) **bloque les fantômes**.
- Les verrous portent sur les **index** : sans index adapté, le verrouillage peut s'étendre à toute la table.
- Verrous **libérés au `COMMIT`/`ROLLBACK`** ; les **MDL** peuvent bloquer le DDL tant qu'une transaction est ouverte.

---

> **Section suivante** : [6.5 — Deadlocks : détection et résolution](05-deadlocks-resolution.md), qui traite des interblocages auxquels mène inévitablement le verrouillage concurrent.

⏭️ [Deadlocks : Détection et résolution](/06-transactions-et-concurrence/05-deadlocks-resolution.md)
