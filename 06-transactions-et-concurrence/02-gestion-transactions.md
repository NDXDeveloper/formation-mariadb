🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Gestion des transactions

La [section précédente](01-concept-transaction-acid.md) a défini la transaction et ses garanties (ACID). Place maintenant à la **pratique** : comment **délimiter** explicitement une transaction, la **valider** ou l'**annuler**, et comprendre le comportement par défaut de MariaDB — le mode *autocommit*.

## Le mode autocommit (comportement par défaut)

Par défaut, MariaDB fonctionne en mode **autocommit** : chaque instruction SQL est **automatiquement validée** dès qu'elle s'achève avec succès. Autrement dit, en l'absence de transaction explicite, **chaque instruction constitue à elle seule une transaction**.

```sql
SELECT @@autocommit;   -- renvoie 1 : autocommit actif (valeur par défaut)
```

Ce comportement est pratique pour les opérations isolées, mais il **ne convient pas** dès qu'on veut regrouper plusieurs instructions en un tout indivisible. Deux approches permettent de reprendre la main :

**1. Désactiver l'autocommit** pour la session :

```sql
SET autocommit = 0;    -- les instructions restent « en attente » jusqu'à COMMIT/ROLLBACK

UPDATE comptes SET solde = solde - 100 WHERE id = 1;
UPDATE comptes SET solde = solde + 100 WHERE id = 2;

COMMIT;                -- validation explicite désormais obligatoire
```

Tant que l'autocommit est à `0`, une nouvelle transaction est automatiquement ouverte après chaque `COMMIT` ou `ROLLBACK` : il faut donc valider (ou annuler) explicitement chaque groupe d'opérations.

**2. Démarrer une transaction explicite** (`START TRANSACTION` / `BEGIN`), qui suspend temporairement l'autocommit le temps de la transaction — c'est la méthode recommandée et l'objet de la suite.

> `autocommit` est une variable de **session** : la modifier n'affecte que la connexion courante.

## Démarrer une transaction : `START TRANSACTION` et `BEGIN`

`START TRANSACTION` ouvre une transaction explicite. Pendant toute sa durée, l'autocommit est **implicitement désactivé** : les instructions ne sont validées qu'au `COMMIT`.

```sql
START TRANSACTION;
  UPDATE comptes SET solde = solde - 100 WHERE id = 1;
  UPDATE comptes SET solde = solde + 100 WHERE id = 2;
COMMIT;   -- ou ROLLBACK pour tout annuler
```

`BEGIN` et `BEGIN WORK` sont des **alias** de `START TRANSACTION` :

```sql
BEGIN;
  -- … opérations …
COMMIT;
```

> ⚠️ **Attention dans les routines stockées** : à l'intérieur d'une procédure, d'une fonction ou d'un *trigger*, le mot-clé `BEGIN` introduit un **bloc** `BEGIN … END` et **ne démarre pas** de transaction. Pour ouvrir une transaction dans ce contexte, il faut impérativement utiliser `START TRANSACTION`.

### Options de `START TRANSACTION`

`START TRANSACTION` accepte une ou plusieurs caractéristiques, séparées par des virgules :

- **`READ WRITE`** (par défaut) / **`READ ONLY`** — fixe le mode d'accès. Une transaction `READ ONLY` interdit les écritures et autorise le moteur à certaines optimisations.

```sql
START TRANSACTION READ ONLY;        -- consultation seule
```

- **`WITH CONSISTENT SNAPSHOT`** — établit immédiatement un **instantané cohérent** des données. Toutes les lectures de la transaction verront alors la base telle qu'elle était à cet instant précis. Cette option n'a d'effet réel qu'avec InnoDB au niveau d'isolation `REPEATABLE READ`.

```sql
START TRANSACTION WITH CONSISTENT SNAPSHOT;
```

Les caractéristiques peuvent se combiner :

```sql
START TRANSACTION WITH CONSISTENT SNAPSHOT, READ ONLY;
```

> Le **niveau d'isolation** d'une transaction ne se définit pas ici mais via `SET TRANSACTION ISOLATION LEVEL …` (voir [§6.3](03-niveaux-isolation.md)). Le fonctionnement de l'instantané cohérent est lié au **MVCC** ([§6.6](06-mvcc.md)).

## Valider : `COMMIT`

`COMMIT` rend **définitives** toutes les modifications de la transaction et y met fin. À partir de ce moment, les changements sont durables (propriété de **durabilité**) et deviennent visibles par les autres transactions.

```sql
START TRANSACTION;
  INSERT INTO commandes (client_id, montant) VALUES (42, 250);
  UPDATE stocks SET quantite = quantite - 1 WHERE produit_id = 7;
COMMIT;   -- les deux opérations sont validées ensemble
```

## Annuler : `ROLLBACK`

`ROLLBACK` **annule** toutes les modifications de la transaction en cours et ramène la base à l'état où elle se trouvait au début de la transaction.

```sql
START TRANSACTION;
  DELETE FROM commandes WHERE statut = 'brouillon';
  -- contrôle : le nombre de lignes affectées est anormal
ROLLBACK;   -- aucune suppression n'est conservée
```

> **Comportement en cas d'erreur** : dans MariaDB, l'échec d'**une** instruction n'annule pas automatiquement toute la transaction — seule l'instruction fautive reste sans effet, et la transaction demeure ouverte. C'est à l'application de décider d'émettre un `ROLLBACK`. (Les cas particuliers comme les interblocages, qui peuvent provoquer une annulation complète, sont traités en [§6.5](05-deadlocks-resolution.md).)

## Les validations implicites (*implicit commit*)

Certaines instructions provoquent une **validation implicite** de la transaction en cours : MariaDB exécute automatiquement un `COMMIT` **avant** de les exécuter, ce qui rend toute annulation ultérieure impossible. C'est le cas notamment :

- des instructions **DDL** : `CREATE`, `ALTER`, `DROP` (tables, bases, index, vues, procédures…), ainsi que `TRUNCATE TABLE` et `RENAME TABLE` ;
- des instructions de **gestion des comptes et privilèges** : `CREATE USER`, `DROP USER`, `GRANT`, `REVOKE`, `SET PASSWORD`… ;
- de `LOCK TABLES` / `UNLOCK TABLES` ;
- du démarrage d'une **nouvelle** transaction (`START TRANSACTION` / `BEGIN`) alors qu'une transaction est déjà ouverte, ou de `SET autocommit = 1`.

> ⚠️ **Point essentiel** : MariaDB **ne dispose pas de DDL transactionnel**. Une commande comme `ALTER TABLE` ou `DROP TABLE` **ne peut pas être annulée** par un `ROLLBACK`, et elle valide au passage toute modification de données en attente. Il faut donc éviter de mêler DDL et transactions de données.

## Chaînage et libération : `AND CHAIN`, `RELEASE`

`COMMIT` et `ROLLBACK` acceptent des clauses optionnelles :

```sql
COMMIT   [WORK] [AND [NO] CHAIN] [[NO] RELEASE];
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE];
```

- **`AND CHAIN`** — clôt la transaction puis en **ouvre aussitôt une nouvelle**, avec le même niveau d'isolation et le même mode d'accès. Pratique pour enchaîner des transactions sans réémettre `START TRANSACTION`.
- **`RELEASE`** — **déconnecte** la session cliente une fois la transaction terminée.

```sql
COMMIT AND CHAIN;   -- valide, puis démarre immédiatement une nouvelle transaction
```

Le comportement par défaut peut être fixé globalement par la variable `completion_type` (`0` = `NO CHAIN`, valeur par défaut ; `1` = `CHAIN` ; `2` = `RELEASE`).

## Transaction, session et absence d'imbrication

Une transaction est **propre à une connexion** (session) :

- il ne peut y avoir **qu'une seule transaction active** par session : il **n'existe pas de transactions imbriquées** au sens strict. Démarrer une transaction alors qu'une autre est ouverte **valide implicitement** la précédente ;
- si la connexion est **interrompue** avant le `COMMIT` (déconnexion, crash du client), la transaction est **automatiquement annulée** (*rollback*).

Pour annuler **partiellement** une transaction — revenir à un point intermédiaire sans tout perdre — MariaDB offre les **savepoints**, présentés en [§6.7](07-savepoints.md).

## Bonnes pratiques

- **Soyez explicite** : encadrez clairement les opérations liées par `START TRANSACTION` … `COMMIT`/`ROLLBACK` plutôt que de vous reposer sur l'autocommit pour des traitements multi-instructions.
- **Gardez les transactions courtes** : une transaction qui reste ouverte longtemps conserve des verrous et de l'historique MVCC, ce qui nuit à la concurrence (voir [§6.4](04-verrous.md) et [§6.6](06-mvcc.md)).
- **Gérez les erreurs** : prévoyez systématiquement un `ROLLBACK` dans les chemins d'erreur de votre application (ou de vos routines, via `DECLARE … HANDLER`, cf. §8.6).
- **Ne mêlez pas DDL et données** : les instructions DDL provoquent une validation implicite et ne sont pas annulables.
- **Utilisez un moteur transactionnel** (InnoDB) pour toutes les tables impliquées (rappel de la [§6.1](01-concept-transaction-acid.md)).

## Synthèse des instructions

| Instruction | Rôle |
|-------------|------|
| `START TRANSACTION [READ WRITE\|READ ONLY] [, WITH CONSISTENT SNAPSHOT]` | démarre une transaction explicite (suspend l'autocommit) |
| `BEGIN` / `BEGIN WORK` | alias de `START TRANSACTION` (sauf dans les routines stockées) |
| `COMMIT [AND [NO] CHAIN] [[NO] RELEASE]` | valide définitivement la transaction |
| `ROLLBACK [AND [NO] CHAIN] [[NO] RELEASE]` | annule la transaction en cours |
| `SET autocommit = 0 \| 1` | désactive / réactive la validation automatique (session) |

---

> **Section suivante** : [6.3 — Niveaux d'isolation](03-niveaux-isolation.md), pour contrôler ce que chaque transaction « voit » des autres et arbitrer entre rigueur et performance.

⏭️ [Niveaux d'isolation](/06-transactions-et-concurrence/03-niveaux-isolation.md)
