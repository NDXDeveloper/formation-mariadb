🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 MVCC (Multi-Version Concurrency Control)

Le **MVCC** (*Multi-Version Concurrency Control*, contrôle de concurrence multi-versions) est le mécanisme qui permet à InnoDB d'offrir des **lectures cohérentes sans verrou**. C'est lui qui explique la plupart des comportements vus jusqu'ici : pourquoi un `SELECT` simple ne bloque pas les écrivains, pourquoi `REPEATABLE READ` fournit un instantané stable, ou pourquoi `READ COMMITTED` voit à chaque requête les dernières données validées.

Sa devise tient en une phrase : **les lecteurs ne bloquent pas les écrivains, et les écrivains ne bloquent pas les lecteurs.**

## L'idée : faire coexister plusieurs versions

Plutôt que de verrouiller une donnée pour empêcher tout accès concurrent pendant qu'elle est lue ou modifiée, le MVCC **conserve plusieurs versions** de chaque ligne. Quand une transaction modifie une ligne, l'ancienne version **n'est pas immédiatement détruite** : une nouvelle version est créée, et l'ancienne reste disponible pour les transactions qui doivent encore la voir.

Chaque transaction lit alors la version qui correspond à **son** instantané — pas nécessairement la plus récente. Lectures et écritures peuvent ainsi progresser **en parallèle**.

## Comment InnoDB implémente le MVCC

InnoDB s'appuie sur trois éléments.

### Colonnes système cachées

À chaque ligne, InnoDB ajoute des colonnes internes invisibles, dont deux sont essentielles au MVCC :

- **`DB_TRX_ID`** (6 octets) — l'identifiant de la **dernière transaction** ayant inséré ou modifié la ligne ;
- **`DB_ROLL_PTR`** (7 octets) — un **pointeur de retour** (*roll pointer*) vers l'enregistrement d'undo log permettant de reconstituer la version **précédente** de la ligne.

*(Une troisième colonne, `DB_ROW_ID`, n'est ajoutée qu'en l'absence de clé primaire définie par l'utilisateur ; elle n'intervient pas directement dans le MVCC.)*

### Undo log et chaîne de versions

Lorsqu'une ligne est modifiée, sa version antérieure est écrite dans l'**undo log** (le même journal qui sert à l'annulation, et donc à l'atomicité — cf. [§6.1](01-concept-transaction-acid.md) et [§7.2.3](../07-moteurs-de-stockage/02.3-innodb-redo-undo-log.md)). En suivant les `DB_ROLL_PTR` de proche en proche, InnoDB peut **remonter la chaîne des versions** successives d'une ligne :

```
[ version courante : solde=1500, TRX_ID=W ]
        │  DB_ROLL_PTR
        ▼
[ undo : solde=1000, TRX_ID=100 (version antérieure) ]
```

### Vue de lecture (*read view*)

Au moment où une transaction effectue une lecture cohérente, InnoDB lui associe une **vue de lecture** : un instantané de l'état des transactions (lesquelles étaient validées, lesquelles encore actives) à cet instant. Pour chaque ligne, la vue de lecture détermine **quelle version est visible** :

- une version est **visible** si elle a été créée par une transaction **déjà validée** au moment où la vue a été constituée (ou par la transaction elle-même) ;
- une version créée par une transaction **encore active**, ou **démarrée après** la vue, n'est **pas** visible : InnoDB suit alors le `DB_ROLL_PTR` pour reconstituer une version antérieure visible.

## Reconstruction d'une version : exemple

Soit `id = 1`, `solde = 1000`. La transaction **R** (`REPEATABLE READ`) prend sa vue de lecture, puis la transaction **W** modifie et valide.

| Moment | Événement |
|--------|-----------|
| t₁ | R fait son premier `SELECT` → **vue de lecture constituée** ; lit `solde = 1000` |
| t₂ | W : `UPDATE … solde = 1500 WHERE id = 1; COMMIT;` → nouvelle version `1500`, ancienne `1000` poussée dans l'undo |
| t₃ | R refait son `SELECT` : la version courante `1500` a été validée par W **après** la vue de R → **non visible**. InnoDB remonte la chaîne et renvoie `1000` |

R lit toujours `1000` : c'est, mécaniquement, la **répétabilité** décrite en [§6.3.3](03.3-repeatable-read.md).

## Quand la vue de lecture est-elle créée ?

C'est précisément ce qui différencie les niveaux d'isolation sous InnoDB :

- **`REPEATABLE READ`** — une **seule** vue de lecture, constituée à la **première** lecture et conservée jusqu'à la fin de la transaction → instantané **stable** ([§6.3.3](03.3-repeatable-read.md)) ;
- **`READ COMMITTED`** — une **nouvelle** vue de lecture à **chaque** lecture cohérente → chaque requête voit les dernières données validées, d'où les lectures non répétables ([§6.3.2](03.2-read-committed.md)).

## Lectures cohérentes vs lectures courantes

Le MVCC ne concerne que les **lectures cohérentes** (les `SELECT` simples), qui lisent l'instantané **sans poser de verrou**. En revanche :

- les **lectures verrouillantes** (`SELECT … FOR UPDATE` / `LOCK IN SHARE MODE`) et les **écritures** (`UPDATE`, `DELETE`) effectuent une **lecture courante** (*current read*) : elles lisent la **dernière version validée** — pas l'instantané — et **posent un verrou** ([§6.4](04-verrous.md)).

C'est l'explication mécanique de la distinction « lecture d'instantané vs lecture courante » introduite en [§6.3.3](03.3-repeatable-read.md), et la raison pour laquelle un `UPDATE` peut agir sur une valeur que le `SELECT` simple précédent n'affichait pas.

## Le revers : purge et croissance de l'historique

Conserver d'anciennes versions a un coût. Une version ne peut être supprimée que lorsque **plus aucune vue de lecture** n'est susceptible d'en avoir besoin. Un processus d'arrière-plan, la **purge**, se charge d'éliminer les versions obsolètes de l'undo log.

> ⚠️ **Le piège des transactions longues.** La purge **ne peut pas dépasser la plus ancienne vue de lecture encore active**. Une transaction laissée ouverte longtemps (en particulier à `REPEATABLE READ`) **empêche la purge de progresser** : l'undo log gonfle, l'**historique** (*history list*) s'allonge, et les lectures deviennent plus coûteuses (chaînes de versions plus longues à remonter). C'est l'une des raisons majeures pour lesquelles il faut **garder les transactions courtes**.

On peut surveiller la taille de l'historique via la ligne **`History list length`** de `SHOW ENGINE INNODB STATUS`. Le dimensionnement de l'undo et de la purge (variables `innodb_purge_threads`, `innodb_undo_log_truncate`…) relève du tuning (Partie 7).

## Lien avec l'isolation par instantané

Le MVCC **fournit** l'instantané ; l'**isolation par instantané** activée par défaut (depuis MariaDB 11.6, donc en 11.8 LTS comme en 12.3) vient s'**appuyer** dessus. Avec `innodb_snapshot_isolation = ON`, lorsqu'une transaction tente une lecture courante (verrou/écriture) sur une ligne dont la version a **changé depuis sa vue de lecture**, InnoDB **lève une erreur** au lieu d'effectuer silencieusement la lecture courante — fermant ainsi la porte aux mises à jour perdues. Voir [§6.9](09-snapshot-isolation.md).

## À retenir

- Le **MVCC** fait coexister **plusieurs versions** de chaque ligne : **lecteurs et écrivains ne se bloquent pas**.
- Implémentation InnoDB : colonnes cachées **`DB_TRX_ID`** / **`DB_ROLL_PTR`**, **undo log** (chaîne de versions) et **vue de lecture** (visibilité).
- La **date de création de la vue** distingue les niveaux : **une fois** par transaction à `REPEATABLE READ`, **à chaque requête** à `READ COMMITTED`.
- Seules les **lectures simples** profitent du MVCC ; les **lectures verrouillantes et écritures** font une **lecture courante** (dernière version validée + verrou).
- Conserver les versions a un coût : une **transaction longue bloque la purge** → croissance de l'undo / de l'historique. **Transactions courtes !**
- L'**isolation par instantané** (activée par défaut) s'appuie sur le MVCC pour transformer un conflit de mise à jour en **erreur** ([§6.9](09-snapshot-isolation.md)).

---

> **Section suivante** : [6.7 — Savepoints : points de sauvegarde](07-savepoints.md), pour annuler partiellement une transaction sans tout perdre.

⏭️ [Savepoints : Points de sauvegarde](/06-transactions-et-concurrence/07-savepoints.md)
