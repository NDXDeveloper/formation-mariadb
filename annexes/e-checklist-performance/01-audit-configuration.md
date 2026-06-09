🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.1 — Audit de configuration

> 🔧 Vérifier que la configuration du serveur soutient la performance, plutôt qu'elle ne la freine.

Cette checklist passe en revue les réglages serveur qui pèsent le plus sur la performance. Pour chaque point, on indique comment le constater — via une requête de l'[Annexe C](../c-requetes-sql-reference/README.md) — et à quoi le comparer — les profils de l'[Annexe D](../d-configuration-reference/README.md). On garde en tête la démarche d'ensemble : mesurer, ne changer qu'une chose, re-mesurer (voir la [présentation de l'annexe E](README.md)).

---

## Mémoire

- [ ] **Buffer pool dimensionné au jeu de données** — `innodb_buffer_pool_size` à environ 60-80 % de la RAM sur un serveur dédié, sans dépasser la mémoire disponible (sous peine de *swap*).
- [ ] **Taux de succès du buffer pool satisfaisant** — viser > ~99 % en OLTP ; le calculer avec la requête de l'Annexe C.2. Un taux durablement bas signale un buffer pool sous-dimensionné.
- [ ] **Tampons par session cohérents avec la concurrence** — `sort_buffer_size`, `join_buffer_size`, `tmp_table_size` modérés en global si les connexions sont nombreuses (cf. D.3), généreux seulement à faible concurrence (cf. D.2).
- [ ] **`tmp_table_size` et `max_heap_table_size` réglés ensemble** — la limite effective des tables temporaires en mémoire est la plus petite des deux.
- [ ] **Garde-fou sur l'espace temporaire actif** — `max_tmp_session_space_usage` et `max_tmp_total_space_usage` définis pour éviter qu'une requête sature le disque (cf. Ch. 11.8).

## Durabilité

- [ ] **`innodb_flush_log_at_trx_commit` adapté à l'exigence** — `1` en production (ACID) ; `0` ou `2` uniquement là où la perte des dernières transactions est acceptable (dev, ETL temporaire).
- [ ] **Stratégie de binlog cohérente** — avec le binlog InnoDB de la 12.3, `sync_binlog` est sans objet (la durabilité dépend du seul `innodb_flush_log_at_trx_commit`) ; avec le binlog fichier classique, `sync_binlog = 1` pour une durabilité maximale (cf. D.1).
- [ ] **Redo log correctement dimensionné** — `innodb_log_file_size` assez grand pour lisser les points de contrôle sous charge soutenue (fichier unique en MariaDB 12.x ; ni `innodb_log_files_in_group` ni `innodb_redo_log_capacity`, absents).

## Entrées/sorties

- [ ] **Accès disque direct** — sous Linux, `O_DIRECT` évite le double cache (OS + InnoDB). En 12.3, c'est le **défaut** (depuis la 10.6) et `innodb_flush_method` est **déprécié** (11.0) au profit des booléens `innodb_data_file_buffering` / `innodb_log_file_buffering` (cf. D.1).
- [ ] **`innodb_io_capacity` accordé au stockage** — élevé sur SSD/NVMe, modeste sur disque mécanique ; `innodb_io_capacity_max` au-dessus.
- [ ] **`innodb_flush_neighbors = 0` sur stockage flash**, où regrouper l'écriture de pages voisines n'apporte rien.

## Variables obsolètes ou retirées

- [ ] **Aucun reliquat de Query Cache** (`query_cache_*`), déprécié.
- [ ] **Aucune variable supprimée ou ignorée** — `innodb_buffer_pool_instances` (**supprimée** comme variable système, `SET GLOBAL`/`@@` échouent ; tolérée mais ignorée dans un `my.cnf` — le buffer pool est à instance unique depuis la 10.5.1), `innodb_buffer_pool_chunk_size` (présente mais ignorée).
- [ ] **Aucune variable retirée en 12.x** — `big_tables`, `large_page_size`, `storage_engine` (cf. Ch. 11.2.3).
- [ ] **Journal d'erreurs vérifié au démarrage** — un `my.cnf` migré depuis une ancienne version traîne souvent ces réglages ; les avertissements « *was removed… exists only for compatibility* » au démarrage les révèlent (un nom de variable réellement inconnu, lui, bloque le démarrage avec une erreur).

## Connexions et threads

- [ ] **`max_connections` calibré sur le besoin réel** — à comparer à `Max_used_connections` ; un plafond démesuré gaspille de la mémoire sans bénéfice.
- [ ] **Thread pool sous forte concurrence** — `thread_handling = pool-of-threads` lorsque les connexions sont nombreuses (cf. D.1).
- [ ] **Indicateurs de saturation surveillés** — `Threads_running` et `Aborted_connects` (requêtes en Annexe C.2).

## Niveau système (OS)

- [ ] **Swap maîtrisé** — `vm.swappiness` bas ; le buffer pool ne doit jamais être déplacé en swap.
- [ ] **Transparent Huge Pages examiné** — souvent désactivé pour une latence plus prévisible ; les *huge pages* explicites peuvent en revanche aider les très gros buffer pools.
- [ ] **Descripteurs de fichiers suffisants** — `open_files_limit` aligné sur le nombre de tables et de connexions.
- [ ] **Stockage adapté** — SSD/NVMe pour les charges limitées par les E/S.

## Cohérence avec le profil de charge

- [ ] **Configuration alignée sur le profil** — OLTP, OLAP, mixte ou développement, selon le cas d'usage réel (cf. Annexe D).
- [ ] **`sql_mode` homogène entre environnements** — en particulier identique entre développement et production, pour éviter les surprises (cf. D.4).

## Pour aller plus loin

La méthodologie est rappelée dans la [présentation de l'annexe E](README.md) et développée au [Ch. 15.1](../../15-performance-tuning/README.md). Le dimensionnement mémoire est traité au [Ch. 15.2](../../15-performance-tuning/README.md), la configuration des E/S au [Ch. 15.4](../../15-performance-tuning/README.md), et les variables système au [Ch. 11.2](../../11-administration-configuration/README.md) (dont les retirées en 11.2.3). Les requêtes de constat figurent à l'[Annexe C.2](../c-requetes-sql-reference/02-requetes-monitoring.md) et les cibles de configuration à l'[Annexe D](../d-configuration-reference/README.md).

---

⬅️ [Annexe E — Checklist de Performance (présentation)](README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [E.2 — Audit d'indexation](02-audit-indexation.md)

⏭️ [Audit d'indexation](/annexes/e-checklist-performance/02-audit-indexation.md)
