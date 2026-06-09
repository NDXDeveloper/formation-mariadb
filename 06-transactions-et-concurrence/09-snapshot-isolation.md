🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.9 Snapshot Isolation

> **Comportement par défaut d'InnoDB en MariaDB 12.3** — `innodb_snapshot_isolation` est **activé par défaut**. ⚠️ Ce n'est **pas** une nouveauté de la 12.3 : ce défaut est en place **depuis MariaDB 11.6.2**, donc déjà actif en **11.8 LTS** (voir la précision de version plus bas).

C'est l'aboutissement de plusieurs points laissés ouverts dans ce chapitre. Nous avons vu en [§6.3.3](03.3-repeatable-read.md) que, à `REPEATABLE READ`, un `SELECT` simple lit l'**instantané** tandis qu'une écriture effectue une **lecture courante** sur la dernière version validée — un écart mécaniquement expliqué par le MVCC ([§6.6](06-mvcc.md)) et susceptible de provoquer une **mise à jour perdue**. La variable `innodb_snapshot_isolation`, **activée par défaut** (depuis MariaDB 11.6.2, donc en 11.8 LTS comme en 12.3), ferme précisément cette faille en transformant `REPEATABLE READ` en une véritable **isolation par instantané** (*snapshot isolation*).

## Le problème : la mise à jour perdue à `REPEATABLE READ`

Rappelons la situation décrite en [§6.3.3](03.3-repeatable-read.md). À `REPEATABLE READ`, une transaction fige un **instantané** à sa première lecture, mais ses **écritures** s'appliquent sur la **dernière version validée** des lignes ([§6.6](06-mvcc.md)).

Une application peut alors : (1) **lire** une valeur dans son instantané, (2) **calculer** une nouvelle valeur à partir de cette lecture, puis (3) l'**écrire** — sans détecter qu'une autre transaction a, entre-temps, modifié et validé cette même ligne. L'écriture **écrase silencieusement** le changement intermédiaire : c'est la **mise à jour perdue** (*lost update*).

Tant que la variable est à `OFF` — son défaut jusqu'à **MariaDB 11.4 LTS** incluse, et dans les versions **10.x** —, MariaDB ne signale rien.

## Ce que change `innodb_snapshot_isolation = ON`

Avec l'isolation par instantané activée, InnoDB ajoute une **vérification** : lorsqu'une transaction tente de **verrouiller ou de modifier** une ligne qui a été **modifiée et validée par une autre transaction depuis la création de sa vue de lecture**, il **refuse l'opération et lève une erreur** au lieu d'effectuer la lecture courante :

```
ERROR 1020 (ER_CHECKREAD): Record has changed since last read in table 'comptes'
```

La transaction est ainsi informée du **conflit** et doit le traiter — typiquement en **rejouant** la transaction, exactement comme pour un interblocage ([§6.5](05-deadlocks-resolution.md)). Résultat net : **plus de mise à jour perdue silencieuse**.

## Démonstration : avant / après

Compte `id = 1`, solde `1000`. La session A lit à `REPEATABLE READ` et **prévoit de retirer 100** (donc d'amener le solde à `900`, d'après sa lecture). La session B crédite `500` et valide entre-temps.

| Étape | Session A — `REPEATABLE READ` | Session B |
|:---:|---|---|
| ① | `START TRANSACTION;` `SELECT solde … id = 1;` → **1000** *(instantané ; cible calculée : 900)* | |
| ② | | `UPDATE comptes SET solde = solde + 500 WHERE id = 1;` `COMMIT;` *(validé : 1500)* |
| ③ | `UPDATE comptes SET solde = 900 WHERE id = 1;` *(valeur déduite de la lecture de 1000)* | |

Le comportement à l'étape ③ **diffère** selon la valeur de la variable :

| `innodb_snapshot_isolation` | Comportement à l'étape ③ |
|---|---|
| **`OFF`** (défaut ≤ 11.4 LTS et 10.x) | l'`UPDATE` écrit `900`, **écrasant** le `1500` validé par B → le crédit de B est **perdu** (on attendrait `1400`) : **mise à jour perdue** |
| **`ON`** (défaut depuis 11.6.2, donc en 11.8 LTS et 12.3) | **erreur** « *Record has changed since last read* » : la ligne a changé depuis l'instantané de A → A doit **rejouer** |

Au rejeu (cas `ON`), la session A reconstitue un instantané frais (lisant `1500`), recalcule sa cible (`1400`) et écrit la valeur correcte.

## Portée et limites

- **Niveaux concernés** : l'isolation par instantané s'applique à `REPEATABLE READ` (et `SERIALIZABLE`), qui reposent sur une vue de lecture valable pour toute la transaction. À `READ COMMITTED`, où la vue est rafraîchie à chaque requête ([§6.6](06-mvcc.md)), la question ne se pose pas de la même façon.
- ⚠️ **Ce n'est pas de la sérialisabilité.** L'isolation par instantané écarte les **mises à jour perdues** (conflits écriture-écriture sur une **même** ligne), mais **pas l'écriture biaisée** (*write skew*) — où deux transactions écrivent sur des lignes **distinctes** et violent ensemble un invariant ([§6.3.4](03.4-serializable.md), exemple des médecins de garde). Pour s'en prémunir, seul `SERIALIZABLE` convient.

## Un comportement familier pour qui vient d'ailleurs

Ce modèle n'a rien d'exotique : il **rapproche** MariaDB d'autres SGBD bien connus.

- Le niveau `REPEATABLE READ` de **PostgreSQL** *est* une isolation par instantané, qui renvoie une erreur de sérialisation sur conflit ([§19.2.3](../19-migration-compatibilite/02.3-depuis-postgresql.md)).
- Le niveau `SERIALIZABLE` d'**Oracle** repose lui aussi sur le snapshot isolation ([§19.2.1](../19-migration-compatibilite/02.1-depuis-oracle.md)).

Une application portée depuis ces moteurs y retrouvera donc un comportement attendu — et la nécessité, déjà familière, de gérer les **erreurs de conflit** par un rejeu.

## Configuration

`innodb_snapshot_isolation` est une variable système, modifiable au niveau **global** comme **session** :

```sql
SELECT @@innodb_snapshot_isolation;            -- 1 (activé) par défaut depuis 11.6.2 (donc en 11.8 LTS et 12.3)

SET SESSION innodb_snapshot_isolation = OFF;   -- comportement antérieur, pour la session
SET GLOBAL  innodb_snapshot_isolation = OFF;   -- pour les nouvelles sessions
```

Dans le fichier de configuration :

```ini
[mysqld]
innodb_snapshot_isolation = ON
```

## Précision de version et impact sur une migration

⚠️ **Cette isolation par instantané n'est pas une nouveauté de la 12.3.** La variable `innodb_snapshot_isolation` existe depuis **MariaDB 10.6.18** (alors à `OFF`), et son **défaut est passé à `ON` en MariaDB 11.6.2**. Elle est donc déjà active **par défaut en 11.8 LTS** : entre **11.8 et 12.3, ce comportement est inchangé** (`ON` dans les deux cas).

Le **changement de comportement** (`OFF` → `ON`) concerne donc une montée depuis une version **antérieure** où la variable était encore à `OFF` — typiquement **MariaDB 11.4 LTS** ou une version **10.x** (10.6, 10.11) :

- une application qui faisait du *read-modify-write* à `REPEATABLE READ` **sans** `SELECT … FOR UPDATE` explicite peut désormais rencontrer l'erreur « *Record has changed since last read* » là où, sur ces versions, l'opération passait silencieusement ;
- la bonne réponse est d'**ajouter une logique de rejeu** sur cette erreur (comme pour les deadlocks, [§6.5](05-deadlocks-resolution.md)) ;
- en transition, il reste possible de **rétablir l'ancien comportement** en positionnant `innodb_snapshot_isolation = OFF`, le temps d'adapter l'application.

> Pour une migration **11.8 → 12.3**, ce point ne constitue donc **pas** un changement (déjà `ON` de part et d'autre) ; il devient pertinent en venant d'une base plus ancienne. Voir le [Chapitre 19 — Migration et Compatibilité](../19-migration-compatibilite/README.md).

## Bonnes pratiques

- **Adoptez le rejeu** : traitez l'erreur de conflit comme un interblocage — annuler, attendre brièvement, recommencer ([§6.5](05-deadlocks-resolution.md)).
- **Le filet de sécurité ne remplace pas le bon patron** : pour un *read-modify-write*, le `SELECT … FOR UPDATE` explicite ([§6.4](04-verrous.md)) reste la méthode la plus robuste et la plus lisible ; l'isolation par instantané protège **en plus** le code qui l'aurait oublié.
- **Réservez `SERIALIZABLE`** aux invariants multi-lignes (*write skew*) que l'isolation par instantané ne couvre pas.
- **Lors d'une migration**, testez le comportement concurrent de l'application et prévoyez le rejeu avant de basculer (ou désactivez temporairement la variable).

## À retenir

- En **12.3** — comme depuis MariaDB **11.6.2**, donc déjà en **11.8 LTS** — `innodb_snapshot_isolation` est **`ON` par défaut** : `REPEATABLE READ` est une véritable **isolation par instantané**. (Ce n'est **pas** une nouveauté 12.3.)
- Une tentative de **verrouiller/modifier** une ligne **changée et validée depuis l'instantané** lève l'erreur **« *Record has changed since last read* »** (ER_CHECKREAD) au lieu d'une lecture courante silencieuse → **fin des mises à jour perdues** silencieuses.
- À gérer **comme un deadlock** : par un **rejeu** de la transaction ([§6.5](05-deadlocks-resolution.md)).
- ⚠️ Écarte les mises à jour perdues, **pas** l'écriture biaisée (*write skew*) — qui relève toujours de `SERIALIZABLE`.
- Rapproche MariaDB du `REPEATABLE READ` de PostgreSQL et du `SERIALIZABLE` d'Oracle.
- **Migration** : le passage `OFF` → `ON` date de la **11.6.2** — c'est un changement à anticiper en venant de **11.4 LTS / 10.x** (et **non** de 11.8, déjà `ON`) ; ajouter le rejeu ou, en transition, `SET … innodb_snapshot_isolation = OFF` (cf. [Chapitre 19](../19-migration-compatibilite/README.md)).

---

> **Fin du chapitre 6.** Vous maîtrisez désormais les transactions et leurs garanties ACID, les niveaux d'isolation et leur mécanique interne (MVCC, verrous), la gestion des interblocages, les savepoints, les transactions distribuées (XA) et l'isolation par instantané activée par défaut.  
>  
> **Chapitre suivant** : [Chapitre 7 — Moteurs de Stockage](../07-moteurs-de-stockage/README.md), pour explorer en profondeur InnoDB et les autres moteurs sur lesquels reposent ces mécanismes.

⏭️ [Partie 4 : Moteurs de Stockage et Programmation Serveur (Avancé)](/partie-04-moteurs-stockage-programmation.md)
