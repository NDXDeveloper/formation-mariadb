🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Concept de transaction et propriétés ACID

Cette première section pose les **fondations conceptuelles** de tout le chapitre. Avant d'aborder la gestion concrète des transactions (§6.2), les niveaux d'isolation (§6.3) ou les verrous (§6.4), il est essentiel de comprendre ce qu'est une transaction et quelles garanties elle apporte. Ces garanties sont résumées par l'acronyme **ACID**.

## Qu'est-ce qu'une transaction ?

Une **transaction** est une **unité logique de traitement** : une séquence d'une ou plusieurs opérations SQL que le SGBD considère comme un tout **indivisible**. Le principe fondateur est celui du **« tout ou rien »** : soit l'ensemble des opérations de la transaction est validé, soit aucune ne l'est.

L'exemple canonique est le **virement bancaire**. Transférer 100 € du compte A vers le compte B suppose deux opérations :

```sql
START TRANSACTION;

UPDATE comptes SET solde = solde - 100 WHERE id = 1;  -- débit du compte A
UPDATE comptes SET solde = solde + 100 WHERE id = 2;  -- crédit du compte B

COMMIT;
```

Ces deux mises à jour ne doivent **jamais** être dissociées. Si le débit réussissait mais que le crédit échouait (panne, erreur, coupure réseau), 100 € disparaîtraient purement et simplement. La transaction garantit qu'on ne se retrouve jamais dans cet état intermédiaire incohérent : soit le virement est intégralement effectué, soit la base reste exactement dans l'état où elle se trouvait avant.

Une transaction suit un **cycle de vie** simple :

1. elle **débute** (explicitement avec `START TRANSACTION` / `BEGIN`, ou implicitement) ;
2. elle exécute une ou plusieurs opérations (`INSERT`, `UPDATE`, `DELETE`, `SELECT`…) ;
3. elle se **termine** de deux manières possibles :
   - **`COMMIT`** : toutes les modifications sont validées et rendues permanentes ;
   - **`ROLLBACK`** : toutes les modifications sont annulées, la base revient à son état initial.

> La mécanique détaillée de ces instructions — démarrage, mode *autocommit*, validations implicites — est traitée en [§6.2](02-gestion-transactions.md). Ici, nous nous concentrons sur le **pourquoi** : les propriétés que ce mécanisme garantit.

## Les propriétés ACID

Les garanties offertes par une transaction sont formalisées par quatre propriétés, désignées par l'acronyme **ACID** : **A**tomicité, **C**ohérence (*Consistency*), **I**solation et **D**urabilité. Cette formalisation a été proposée par Theo Härder et Andreas Reuter en 1983, à partir des travaux de Jim Gray sur le concept de transaction.

### A — Atomicité (*Atomicity*)

Une transaction est **atomique** : elle est exécutée **entièrement ou pas du tout**. Si une seule de ses opérations échoue, ou si la transaction est explicitement annulée, toutes les modifications déjà réalisées sont défaites comme si elles n'avaient jamais eu lieu.

Dans le cas du virement, l'atomicité est ce qui empêche le débit d'être appliqué sans le crédit. C'est la propriété qui sous-tend le « tout ou rien ».

Dans MariaDB, InnoDB assure l'atomicité grâce à l'**undo log** (journal d'annulation), qui conserve de quoi inverser chaque modification en cas de `ROLLBACK` ou de panne.

### C — Cohérence (*Consistency*)

Une transaction fait passer la base d'un **état valide à un autre état valide**. À aucun moment une transaction validée ne doit laisser la base dans un état qui violerait ses règles d'intégrité : contraintes de clé primaire et étrangère, contraintes `UNIQUE`, `NOT NULL`, `CHECK`, types de données, déclencheurs (*triggers*), etc.

La cohérence est une **responsabilité partagée** :

- le SGBD garantit le respect des règles **déclarées** dans le schéma (contraintes, types…) ; toute opération qui les violerait fait échouer la transaction ;
- l'application reste responsable de la cohérence **métier** qui n'est pas exprimable par des contraintes (par exemple une règle de gestion complexe), qu'elle doit traduire en opérations cohérentes au sein de la transaction.

### I — Isolation

L'**isolation** garantit que des transactions exécutées **simultanément** ne se perturbent pas mutuellement : chacune se comporte, dans une certaine mesure, **comme si elle était seule** à accéder à la base. Sans isolation, une transaction pourrait par exemple lire des données partiellement modifiées par une autre, encore non validées.

L'isolation est la propriété la plus **nuancée** des quatre, car elle est **réglable**. MariaDB propose plusieurs **niveaux d'isolation** (du plus permissif au plus strict) qui arbitrent entre la rigueur des garanties et la performance en concurrence. Un niveau plus faible autorise certaines **anomalies** (lecture sale, lecture non répétable, lecture fantôme) en échange d'un meilleur débit ; un niveau plus fort les écarte au prix d'un surcoût.

> Les niveaux d'isolation et leurs anomalies font l'objet de la [§6.3](03-niveaux-isolation.md) ; le niveau par défaut d'InnoDB est `REPEATABLE READ`. Le mécanisme qui rend l'isolation efficace sans bloquer les lectures, le **MVCC**, est détaillé en [§6.6](06-mvcc.md). Enfin, MariaDB applique par défaut (depuis la 11.6, donc en 12.3) une **isolation par instantané** qui renforce le comportement de `REPEATABLE READ` (voir [§6.9](09-snapshot-isolation.md)).

### D — Durabilité (*Durability*)

Une fois qu'une transaction est **validée** (`COMMIT`), ses effets sont **permanents** : ils survivent à toute défaillance ultérieure — arrêt brutal du serveur, coupure de courant, crash du processus. Après redémarrage, les données validées doivent être intégralement retrouvées.

InnoDB assure la durabilité par une technique de **journalisation en écriture anticipée** (*write-ahead logging*) : les modifications sont d'abord inscrites dans le **redo log** (journal de reprise), de façon séquentielle et rapide, avant d'être appliquées aux fichiers de données. En cas de panne, le redo log permet de **rejouer** les transactions validées qui n'auraient pas encore été écrites sur le disque.

Le compromis entre durabilité stricte et performance est piloté par la variable `innodb_flush_log_at_trx_commit` (valeur par défaut `1` = durabilité ACID complète, chaque `COMMIT` étant écrit puis synchronisé sur le disque). Le réglage fin de ce paramètre relève du tuning (voir Partie 7).

## Comment MariaDB garantit (ou non) les propriétés ACID

Le respect des propriétés ACID **dépend du moteur de stockage** utilisé. C'est un point crucial, et une source fréquente de mauvaises surprises.

- **InnoDB** — moteur **par défaut** de MariaDB, **pleinement transactionnel et ACID**. Il combine trois mécanismes : le **redo log** (durabilité), l'**undo log** (atomicité et lectures cohérentes du MVCC) et les **verrous** (isolation).
- **Moteurs non transactionnels** — **MyISAM** et **MEMORY**, notamment, **ignorent les transactions** : leurs écritures sont immédiates et un `ROLLBACK` n'a aucun effet sur elles. Aria est *crash-safe* pour ses propres tables, mais n'offre pas de transactions ACID au sens d'InnoDB.

> ⚠️ **Conséquence pratique** : une transaction qui modifie une table MyISAM ne pourra **pas** être annulée. Pire, une transaction « mixte » touchant à la fois une table InnoDB et une table MyISAM ne sera annulée **que partiellement** (la partie InnoDB), laissant la base dans un état incohérent. Pour bénéficier des garanties ACID, **toutes** les tables concernées doivent utiliser un moteur transactionnel comme InnoDB.

Enfin, par défaut, MariaDB fonctionne en mode **autocommit** : en l'absence de `START TRANSACTION`, **chaque instruction** est traitée comme une transaction autonome, validée immédiatement. Ce comportement et ses implications sont détaillés en [§6.2](02-gestion-transactions.md).

## Synthèse

| Propriété | Garantit que… | Mécanisme InnoDB |
|-----------|---------------|------------------|
| **A**tomicité | la transaction est appliquée entièrement ou pas du tout | *undo log* |
| **C**ohérence | la base passe d'un état valide à un autre (règles respectées) | contraintes, types, triggers (+ logique applicative) |
| **I**solation | les transactions concurrentes ne se perturbent pas (selon le niveau choisi) | verrous + MVCC |
| **D**urabilité | les données validées survivent aux pannes | *redo log* (*write-ahead logging*) |

---

> **Section suivante** : [6.2 — Gestion des transactions](02-gestion-transactions.md), pour mettre en pratique le contrôle des transactions (`START TRANSACTION`, `COMMIT`, `ROLLBACK`) et comprendre le mode *autocommit*.

⏭️ [Gestion des transactions (START TRANSACTION, BEGIN, COMMIT, ROLLBACK)](/06-transactions-et-concurrence/02-gestion-transactions.md)
