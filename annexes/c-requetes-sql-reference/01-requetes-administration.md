🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.1 — Requêtes d'administration

> 🚨 Les requêtes que l'on dégaine au quotidien et pendant un incident : sessions, verrous, empreinte disque.

Cette section regroupe des requêtes opérationnelles : voir qui est connecté et ce qui s'exécute, repérer les blocages, et mesurer la taille des tables et des index. Toutes sont en lecture seule et puisent dans les schémas système ; la plupart supposent le privilège `PROCESS`.

> ⚠️ **Spécificité MariaDB.** La surveillance des verrous s'appuie ici sur les tables `information_schema.INNODB_TRX`, `INNODB_LOCKS` et `INNODB_LOCK_WAITS`. C'est un point de vigilance pour qui arrive de MySQL 8, où ces informations ont migré vers `performance_schema.data_locks` / `data_lock_waits` — tables **absentes de MariaDB**.

---

## Sessions et requêtes en cours

La table `information_schema.PROCESSLIST` offre une version filtrable et triable de `SHOW PROCESSLIST` (*voir Annexe B.2 pour la commande équivalente et `KILL`*). MariaDB y ajoute des colonnes utiles, dont `time_ms`, `progress` et `memory_used`.

```sql
-- Sessions actives (hors connexions au repos), des plus longues aux plus courtes
SELECT id, user, host, db, command, time, state, info
FROM   information_schema.PROCESSLIST
WHERE  command <> 'Sleep'
ORDER  BY time DESC;
```

```sql
-- Nombre de connexions par utilisateur
SELECT user, COUNT(*) AS nb_connexions
FROM   information_schema.PROCESSLIST
GROUP  BY user
ORDER  BY nb_connexions DESC;
```

```sql
-- Connexions au repos depuis longtemps (> 5 min) : candidates à un nettoyage
SELECT id, user, host, db, time
FROM   information_schema.PROCESSLIST
WHERE  command = 'Sleep' AND time > 300
ORDER  BY time DESC;
```

## Verrous et blocages

Le premier réflexe consiste à lister les transactions en attente de verrou (`trx_state = 'LOCK WAIT'`) :

```sql
SELECT trx_id, trx_state, trx_started, trx_wait_started,
       trx_mysql_thread_id AS thread_id, trx_query
FROM   information_schema.INNODB_TRX
WHERE  trx_state = 'LOCK WAIT';
```

Pour identifier **qui bloque qui**, on relie `INNODB_LOCK_WAITS` aux transactions des deux côtés (requérante et bloquante), et à `INNODB_LOCKS` pour connaître l'objet verrouillé :

```sql
SELECT
  bt.trx_mysql_thread_id AS thread_bloquant,
  rt.trx_mysql_thread_id AS thread_bloque,
  bl.lock_table          AS objet_verrouille,
  bl.lock_mode           AS mode_verrou,
  rt.trx_query           AS requete_en_attente,
  bt.trx_query           AS requete_bloquante
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX   rt ON rt.trx_id  = w.requesting_trx_id
JOIN information_schema.INNODB_TRX   bt ON bt.trx_id  = w.blocking_trx_id
JOIN information_schema.INNODB_LOCKS bl ON bl.lock_id = w.blocking_lock_id;
```

La colonne `thread_bloquant` correspond à l'identifiant de session (`PROCESSLIST.id`) : c'est lui que l'on passe à `KILL` pour libérer la situation, une fois le diagnostic posé.

Il est également utile de repérer les transactions ouvertes depuis longtemps, souvent à l'origine des blocages et du gonflement de l'undo log :

```sql
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duree_s,
       trx_mysql_thread_id AS thread_id,
       trx_rows_locked, trx_rows_modified, trx_query
FROM   information_schema.INNODB_TRX
ORDER  BY trx_started ASC;
```

Pour le détail brut d'un blocage (sections `TRANSACTIONS` et `LATEST DETECTED DEADLOCK`), la commande `SHOW ENGINE INNODB STATUS` reste complémentaire (*voir Annexe B.2*). Les mécanismes de verrouillage et de résolution des deadlocks sont traités au [Ch. 6 — Transactions et Concurrence](../../06-transactions-et-concurrence/README.md).

## Taille des tables et des index

La table `information_schema.TABLES` donne l'empreinte disque, en distinguant données (`data_length`) et index (`index_length`).

```sql
-- Les 20 plus grosses tables (données + index)
SELECT
  table_schema,
  table_name,
  ROUND((data_length + index_length) / 1024 / 1024, 1) AS taille_mb,
  ROUND(data_length  / 1024 / 1024, 1)                 AS donnees_mb,
  ROUND(index_length / 1024 / 1024, 1)                 AS index_mb,
  table_rows                                            AS lignes_estimees
FROM   information_schema.TABLES
WHERE  table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER  BY (data_length + index_length) DESC
LIMIT  20;
```

```sql
-- Taille totale par base
SELECT
  table_schema,
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS taille_mb
FROM   information_schema.TABLES
GROUP  BY table_schema
ORDER  BY SUM(data_length + index_length) DESC;
```

```sql
-- Espace potentiellement récupérable (indicateur de fragmentation)
SELECT table_schema, table_name,
       ROUND(data_free / 1024 / 1024, 1) AS espace_libre_mb
FROM   information_schema.TABLES
WHERE  data_free > 0
ORDER  BY data_free DESC
LIMIT  20;
```

Deux précautions de lecture : pour les tables InnoDB, `table_rows` est une **estimation** issue des statistiques (à ne pas confondre avec un `COUNT(*)` exact), et un `data_free` élevé signale un espace récupérable, par exemple via `OPTIMIZE TABLE` (*voir Ch. 11.6 — Maintenance des tables*).

## Pour aller plus loin

Les verrous, niveaux d'isolation et deadlocks sont développés au [Ch. 6](../../06-transactions-et-concurrence/README.md) ; les schémas système interrogés ici, au [Ch. 9.7](../../09-vues-et-donnees-virtuelles/README.md) ; la maintenance et la récupération d'espace, au [Ch. 11.6](../../11-administration-configuration/README.md). Pour un instantané immédiat sans requête, voir les commandes de l'[Annexe B.2](../b-commandes-cli/02-informations-systeme.md).

---

⬅️ [Annexe C — Requêtes SQL de Référence (présentation)](README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [C.2 — Requêtes de monitoring](02-requetes-monitoring.md)

⏭️ [Requêtes de monitoring (buffer pool, slow queries, replication)](/annexes/c-requetes-sql-reference/02-requetes-monitoring.md)
