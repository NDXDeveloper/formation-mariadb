🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Deadlocks : détection et résolution

Un **interblocage** (*deadlock*) survient lorsque **deux transactions (ou plus) s'attendent mutuellement** : chacune détient un verrou que l'autre réclame, formant un **cycle** d'attente dont aucune ne peut sortir seule. C'est une conséquence directe du verrouillage concurrent ([§6.4](04-verrous.md)) ; impossible à éliminer totalement, l'interblocage doit donc être **détecté**, **résolu** et, autant que possible, **prévenu**.

## Deadlock ≠ dépassement de délai

Deux situations distinctes peuvent bloquer une transaction en attente d'un verrou :

| | Attente simple (*lock wait*) | Interblocage (*deadlock*) |
|---|---|---|
| **Situation** | une transaction attend un verrou détenu par une autre, **sans cycle** | **cycle** d'attente mutuel : A attend B qui attend A |
| **Issue** | la transaction attend, puis échoue si le délai expire | InnoDB **détecte le cycle immédiatement** et annule une victime |
| **Délai** | `innodb_lock_wait_timeout` (50 s par défaut) | détection **instantanée**, sans attendre le délai |
| **Erreur** | **1205** — *Lock wait timeout exceeded* | **1213** — *Deadlock found when trying to get lock* |

Dans les deux cas, le message invite à **rejouer la transaction** (*try restarting transaction*).

## Démonstration d'un interblocage

Deux lignes, `id = 1` et `id = 2`. Les sessions A et B les verrouillent dans l'**ordre inverse** l'une de l'autre.

| Étape | Session A | Session B |
|:---:|---|---|
| ① | `START TRANSACTION;` `UPDATE comptes SET … WHERE id = 1;` *(verrou X sur 1)* | |
| ② | | `START TRANSACTION;` `UPDATE comptes SET … WHERE id = 2;` *(verrou X sur 2)* |
| ③ | `UPDATE comptes SET … WHERE id = 2;` → **attend** *(B détient 2)* | |
| ④ | | `UPDATE comptes SET … WHERE id = 1;` → **cycle !** |
| ⑤ | (se débloque) | InnoDB **détecte le deadlock**, choisit B comme **victime** et l'**annule** (erreur 1213) |

À l'étape ④, le cycle se referme (A attend B, B attend A). InnoDB l'interrompt aussitôt : il **annule entièrement** l'une des transactions — ici B — ce qui libère ses verrous et permet à A de poursuivre. B reçoit l'erreur 1213 et devra **recommencer**.

## La détection par InnoDB

InnoDB maintient en permanence un **graphe d'attente** (*wait-for graph*) des transactions. Dès qu'un **cycle** y apparaît, il déclenche la résolution :

- il choisit une **transaction victime** — généralement celle qui a effectué le **moins de modifications** (la moins coûteuse à annuler) ;
- il **annule** (`ROLLBACK`) intégralement cette victime et libère ses verrous ;
- la ou les autres transactions du cycle peuvent alors progresser.

> La détection automatique est pilotée par `innodb_deadlock_detect` (**`ON` par défaut**). Dans des environnements à **très forte concurrence**, son coût peut devenir non négligeable ; on peut alors la désactiver, auquel cas les interblocages ne sont plus détectés activement mais résolus par **expiration de `innodb_lock_wait_timeout`**. C'est un réglage avancé, à n'envisager qu'en connaissance de cause.

## Diagnostiquer les interblocages

- **`SHOW ENGINE INNODB STATUS`** — sa section **`LATEST DETECTED DEADLOCK`** détaille le **dernier** interblocage : transactions impliquées, requêtes, verrous en jeu et victime choisie.
- **`innodb_print_all_deadlocks`** — lorsqu'elle est activée (`ON`), **chaque** interblocage est journalisé dans le **log d'erreurs**. Indispensable en production pour repérer les interblocages **récurrents** (et non seulement le dernier).

> Voir les requêtes de monitoring en [Annexe C.2](../annexes/c-requetes-sql-reference/02-requetes-monitoring.md).

### Lire un rapport de deadlock

La sortie de `LATEST DETECTED DEADLOCK` paraît dense, mais se lit selon une grille fixe. Voici celle produite par le scénario ci-dessus (sortie réelle **abrégée** — lignes techniques et vidages hexadécimaux des enregistrements omis) :

```
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 24, ACTIVE 2 sec starting index read
UPDATE comptes SET solde = solde - 1 WHERE id = 1
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS … index PRIMARY of table `bank`.`comptes` trx id 24 lock_mode X locks rec but not gap waiting
*** CONFLICTING WITH:
RECORD LOCKS … index PRIMARY of table `bank`.`comptes` trx id 23 lock_mode X locks rec but not gap
*** (2) TRANSACTION:
TRANSACTION 23, ACTIVE 3 sec starting index read
UPDATE comptes SET solde = solde - 1 WHERE id = 2
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS … index PRIMARY of table `bank`.`comptes` trx id 23 lock_mode X locks rec but not gap waiting
*** WE ROLL BACK TRANSACTION (2)
```

La lecture est systématique :

- **`*** (1) TRANSACTION`** et **`*** (2) TRANSACTION`** décrivent les deux transactions du cycle, avec leur **dernière requête** (ici les deux `UPDATE` croisés).
- **`WAITING FOR THIS LOCK TO BE GRANTED`** donne, pour chacune, le **verrou réclamé** : un verrou **exclusif** (`lock_mode X`) sur un enregistrement de l'index `PRIMARY` de `comptes` ; **`CONFLICTING WITH`** indique la transaction qui le **détient déjà**. Le cycle est explicite : (1) attend un verrou tenu par (2), et (2) attend un verrou tenu par (1).
- **`*** WE ROLL BACK TRANSACTION (2)`** : InnoDB a désigné la transaction **(2)** comme **victime** et l'a annulée — elle reçoit l'erreur 1213, tandis que (1) poursuit.

> La mention `lock_mode X locks rec but not gap` confirme qu'il s'agit de **verrous d'enregistrement** (sans *gap lock* ici) ; on y retrouve les variantes de verrous décrites en [§6.4](04-verrous.md).

## Résolution : rejouer la transaction

Un interblocage **n'est pas un bug** : c'est un événement **normal** en système concurrent. La victime ayant été **entièrement annulée**, la bonne réponse est de **rejouer la transaction depuis le début**, idéalement avec un court délai croissant (*backoff*) :

```text
répéter au plus N fois :
    démarrer la transaction
    exécuter les opérations
    si erreur 1213 (deadlock) ou 1205 (timeout) :
        annuler ; attendre un court instant (backoff) ; recommencer
    sinon :
        valider ; sortir
```

Côté serveur, dans une routine stockée, un *handler* peut intercepter l'interblocage (voir [§8.6 — Gestion des erreurs et exceptions](../08-programmation-cote-serveur/06-gestion-erreurs-exceptions.md)) :

```sql
DECLARE EXIT HANDLER FOR SQLSTATE '40001'   -- deadlock
BEGIN
    ROLLBACK;
    -- journaliser / signaler pour que l'appelant relance la transaction
END;
```

> **À noter** : avec l'isolation par instantané activée par défaut ([§6.9](09-snapshot-isolation.md)), une **erreur de conflit** supplémentaire (`ER_CHECKREAD`, 1020) peut survenir lorsqu'une transaction tente de modifier une ligne changée depuis son instantané. La logique de **rejeu** prévue pour les interblocages doit donc aussi **couvrir ce cas**.

## Prévenir les interblocages

On ne supprime pas les interblocages, mais on en **réduit fortement** la fréquence :

- **Accéder aux ressources dans un ordre constant** entre toutes les transactions (par exemple, toujours par `id` croissant). C'est la parade la plus efficace : sans ordres d'acquisition opposés, pas de cycle. *(C'est précisément cet ordre inverse qui a provoqué le deadlock de la démonstration.)*
- **Garder les transactions courtes et ciblées** : moins de verrous tenus, moins longtemps → moins de chances de cycle.
- **Indexer les colonnes filtrées** : des verrous précis (peu de lignes) entrent moins souvent en conflit qu'un verrouillage étendu ([§6.4](04-verrous.md)).
- **Acquérir tôt les verrous nécessaires**, dans un ordre maîtrisé (par exemple via des `SELECT … FOR UPDATE` ordonnés), plutôt que de les réclamer dispersés au fil de la transaction.
- **Choisir un niveau d'isolation adapté** : `SERIALIZABLE` multiplie les verrous, donc les interblocages ([§6.3.4](03.4-serializable.md)) ; à l'inverse, `READ COMMITTED` (sans *gap locks*) en génère moins ([§6.3.2](03.2-read-committed.md)).
- **Désengorger les points chauds** (*hotspots*) : une même ligne très sollicitée (compteur global, etc.) concentre les conflits ; envisager des techniques de répartition.
- Pour les **files de tâches**, préférer `SKIP LOCKED` / `NOWAIT` afin d'éviter les attentes qui dégénèrent en cycles ([§6.4](04-verrous.md)).

## À retenir

- Un **deadlock** = **cycle** d'attente entre transactions ; à distinguer du simple **dépassement de délai** (erreur 1205 vs 1213).
- InnoDB **détecte** les interblocages via un graphe d'attente et **annule une victime** (la moins coûteuse), erreur **1213** — comportement piloté par `innodb_deadlock_detect` (`ON` par défaut).
- **Diagnostic** : `SHOW ENGINE INNODB STATUS` (dernier deadlock) et `innodb_print_all_deadlocks` (tous, dans le log).
- **Résolution** : c'est normal — **rejouer** la transaction annulée (avec *backoff*).
- **Prévention** : ordre d'accès **constant**, transactions courtes, bons index, isolation adaptée, désengorgement des points chauds.
- La logique de rejeu doit aussi couvrir l'erreur de conflit de l'**isolation par instantané** (`ER_CHECKREAD`, activée par défaut ; [§6.9](09-snapshot-isolation.md)).

---

> **Section suivante** : [6.6 — MVCC (Multi-Version Concurrency Control)](06-mvcc.md), le mécanisme qui permet à InnoDB d'offrir des lectures cohérentes sans verrou — fondement de tout ce que nous venons de voir.

⏭️ [MVCC (Multi-Version Concurrency Control)](/06-transactions-et-concurrence/06-mvcc.md)
