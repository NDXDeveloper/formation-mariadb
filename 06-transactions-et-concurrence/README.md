🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 6 — Transactions et Concurrence

> **Partie 3 — Index, Transactions et Performance** · Niveau : Intermédiaire  
> Version de référence : **MariaDB 12.3 LTS**  
> Parcours concernés : **Développeur**, **Formation complète** (notions également fondamentales pour les DBA)

Dès qu'une base de données est sollicitée par plusieurs utilisateurs ou plusieurs processus en même temps, deux exigences entrent en tension : garantir que chaque opération laisse les données dans un état cohérent, et permettre à ces opérations de s'exécuter simultanément sans se bloquer inutilement. C'est précisément le rôle des **transactions** et des mécanismes de **gestion de la concurrence**.

Une transaction regroupe un ensemble d'opérations SQL en une unité logique « tout ou rien » : soit toutes ses modifications sont validées ensemble, soit aucune ne l'est. Autour de cette idée simple, MariaDB — principalement via le moteur **InnoDB** — déploie tout un appareillage (niveaux d'isolation, verrous, MVCC, points de sauvegarde) qui détermine ce que chaque transaction « voit » des autres et comment les conflits sont arbitrés.

Ce chapitre constitue le socle conceptuel et pratique sur lequel reposent la fiabilité des applications transactionnelles (OLTP), la cohérence des traitements concurrents et, plus loin dans la formation, le tuning de la performance (Partie 7) ainsi que la réplication (Partie 6).

## 🎯 Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- expliquer ce qu'est une transaction et énoncer les quatre propriétés **ACID** ;
- délimiter explicitement une transaction (`START TRANSACTION` / `BEGIN`, `COMMIT`, `ROLLBACK`) et maîtriser le mode *autocommit* ;
- choisir le **niveau d'isolation** adapté à un besoin et anticiper les anomalies associées (lecture sale, lecture non répétable, lecture fantôme) ;
- comprendre le rôle des **verrous** et des lectures verrouillantes (`SELECT … FOR UPDATE` / `LOCK IN SHARE MODE`) ;
- diagnostiquer et prévenir les interblocages (**deadlocks**) ;
- expliquer le fonctionnement du **MVCC** d'InnoDB et son intérêt pour la concurrence en lecture ;
- utiliser les **savepoints** pour réaliser des annulations partielles ;
- situer les **transactions distribuées (XA)** et leur cas d'usage ;
- comprendre l'**isolation par instantané** (*snapshot*) que MariaDB applique par défaut à `REPEATABLE READ` (`innodb_snapshot_isolation`, activée par défaut depuis la 11.6.2 — donc en 11.8 LTS comme en 12.3) et savoir gérer l'erreur de conflit qu'elle introduit.

## Prérequis

- Notions de base du SQL et manipulation de données : voir [Chapitre 2 — Bases du SQL](../02-bases-du-sql/README.md) ;
- Familiarité avec le moteur **InnoDB** : ses grandes lignes suffisent ici, le détail étant traité au [Chapitre 7 — Moteurs de Stockage](../07-moteurs-de-stockage/02-innodb.md) ;
- Idéalement, une première lecture du [Chapitre 5 — Index et Performance](../05-index-et-performance/README.md), car verrous et plans d'exécution sont étroitement liés.

## Contenu du chapitre

- **6.1 — [Concept de transaction et propriétés ACID](01-concept-transaction-acid.md)** : ce qu'est une transaction et les garanties d'**A**tomicité, **C**ohérence, **I**solation et **D**urabilité.
- **6.2 — [Gestion des transactions](02-gestion-transactions.md)** : délimiter et contrôler une transaction (`START TRANSACTION`, `BEGIN`, `COMMIT`, `ROLLBACK`) ; comportement du mode *autocommit*.
- **6.3 — [Niveaux d'isolation](03-niveaux-isolation.md)** : les quatre niveaux standard SQL — `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ` (défaut InnoDB) et `SERIALIZABLE` — et les anomalies que chacun écarte.
- **6.4 — [Verrous](04-verrous.md)** : verrouillage de tables et lectures verrouillantes (`SELECT … FOR UPDATE` / `LOCK IN SHARE MODE`).
- **6.5 — [Deadlocks : détection et résolution](05-deadlocks-resolution.md)** : comment InnoDB détecte les interblocages, comment les résoudre et les prévenir.
- **6.6 — [MVCC (Multi-Version Concurrency Control)](06-mvcc.md)** : le contrôle de concurrence multi-versions qui permet des lectures cohérentes sans blocage.
- **6.7 — [Savepoints : points de sauvegarde](07-savepoints.md)** : poser des points de reprise pour annuler partiellement une transaction.
- **6.8 — [Transactions distribuées (XA)](08-transactions-distribuees-xa.md)** : coordonner une transaction sur plusieurs ressources via le protocole de validation en deux phases (*two-phase commit*).
- **6.9 — [Snapshot Isolation](09-snapshot-isolation.md)** : `innodb_snapshot_isolation` (activé par défaut depuis la 11.6.2, donc en 11.8 LTS et 12.3) et son impact sur le comportement de `REPEATABLE READ`.

## À noter : l'isolation par instantané activée par défaut

Un **comportement par défaut important** d'InnoDB mérite d'être signalé d'emblée : la variable `innodb_snapshot_isolation` est **activée par défaut** en MariaDB 12.3. ⚠️ Ce n'est **pas** une nouveauté de la 12.3 — ce défaut est en place **depuis MariaDB 11.6.2** (donc déjà en 11.8 LTS) —, mais il change profondément le comportement de `REPEATABLE READ` par rapport aux versions plus anciennes et à de nombreux autres SGBD.

Concrètement, le niveau `REPEATABLE READ` d'InnoDB applique une véritable isolation par instantané (*snapshot*) : si une ligne lue par la transaction a été modifiée entre-temps par une autre transaction validée, une tentative de la verrouiller ou de la modifier déclenche une **erreur** (`ER_CHECKREAD`) plutôt qu'une mise à jour silencieuse. Ce comportement renforce la protection contre les **mises à jour perdues**, mais peut faire apparaître des erreurs à gérer dans des applications conçues pour des versions antérieures (≤ 11.4 LTS / 10.x), où la variable était à `OFF`.

Le détail de ce mécanisme — ainsi que ses implications lors d'une migration — est traité en [§6.9](09-snapshot-isolation.md).

---

> **Pour démarrer**, commencez par la [§6.1](01-concept-transaction-acid.md), qui pose les fondations conceptuelles (transactions et propriétés ACID) sur lesquelles s'appuie tout le reste du chapitre.

⏭️ [Concept de transaction et propriétés ACID](/06-transactions-et-concurrence/01-concept-transaction-acid.md)
