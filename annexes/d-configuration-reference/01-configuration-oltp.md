🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.1 — OLTP (High concurrency, low latency)

> ⚡ Forte concurrence, faible latence : la configuration privilégie la résidence mémoire, des validations rapides et durables, et la concurrence.

Le profil **OLTP** (*Online Transaction Processing*) se caractérise par de nombreuses transactions courtes et concurrentes, des accès ciblés (par clé primaire ou index), et une exigence de faible latence doublée, le plus souvent, d'une garantie de durabilité. Le moteur de référence est InnoDB.

> ⚠️ Rappel : cette configuration est un **point de départ à adapter au matériel** (RAM, cœurs, type de disque) et à valider par la mesure. Voir la [présentation de l'annexe D](README.md) pour les précautions générales.

---

## Configuration de référence (`[mysqld]`)

```ini
[mysqld]
# ============================================================
#  Profil OLTP — MariaDB 12.3
#  Les valeurs <…> et les exemples sont à AJUSTER au matériel.
# ============================================================

# --- Mémoire ---
# Levier dominant : ~70-80 % de la RAM sur un serveur dédié.
innodb_buffer_pool_size        = <0.75 * RAM>   # ex. 24G sur une machine de 32 Go
innodb_log_buffer_size         = 64M            # marge pour les transactions volumineuses

# --- Journal redo et durabilité ---
innodb_log_file_size           = 2G             # fichier unique (ib_logfile0) en MariaDB 12.x
innodb_flush_log_at_trx_commit = 1              # 1 = durabilité ACID ; 2 = plus rapide, risque accru
innodb_flush_method            = O_DIRECT       # O_DIRECT = défaut depuis la 10.6 ; variable dépréciée en 11.0 (voir le texte)

# --- Entrées/sorties (à accorder au stockage) ---
innodb_io_capacity             = 2000           # SSD ; ~200 pour un disque mécanique
innodb_io_capacity_max         = 4000
innodb_flush_neighbors         = 0              # 0 sur SSD/NVMe
innodb_read_io_threads         = 8
innodb_write_io_threads        = 8

# --- Concurrence ---
thread_handling                = pool-of-threads  # thread pool intégré
thread_pool_size               = <nb_cœurs>       # ~ nombre de cœurs CPU
max_connections                = 500              # selon le besoin applicatif
table_open_cache               = 4000

# --- Réplication / PITR (voir la section dédiée plus bas) ---
server_id                      = 1
binlog_format                  = ROW
```

## Mémoire

Le `innodb_buffer_pool_size` est le réglage le plus déterminant : plus le jeu de données « chaud » tient en mémoire, moins le serveur sollicite le disque. Sur une machine dédiée, on vise 70 à 80 % de la RAM, en laissant de la marge pour le système, les connexions et les autres tampons.

Deux variables d'un autre temps sont à **ne plus utiliser** : `innodb_buffer_pool_instances` (**supprimée** — le buffer pool est à instance unique depuis la 10.5.1 ; elle n'existe plus comme variable système, donc tout `SET GLOBAL`/`@@` la concernant échoue, même si un `my.cnf` la tolère encore en l'ignorant) et `innodb_buffer_pool_chunk_size` (présente mais ignorée). La taille du buffer pool peut par ailleurs être redimensionnée à chaud, jusqu'à `innodb_buffer_pool_size_max` si cette borne a été fixée au démarrage.

## Journal redo et durabilité

En MariaDB 12.x, le journal redo est un fichier unique dimensionné par `innodb_log_file_size` (les anciennes `innodb_log_files_in_group` et `innodb_redo_log_capacity` n'ont pas cours ici). Un redo plus grand espace les points de contrôle et lisse les écritures soutenues, au prix d'une reprise après crash plus longue. La valeur est dynamique depuis la 10.9.

Le couple durabilité/performance se règle via `innodb_flush_log_at_trx_commit` : la valeur `1` garantit qu'une transaction validée survit à une panne (conforme ACID) ; `2` accélère les écritures en repoussant le `fsync`, au prix d'un risque de perte des toutes dernières transactions en cas de coupure brutale. `innodb_flush_method = O_DIRECT` évite que les pages soient mises en cache à la fois par InnoDB et par le système. **Précision pour la 12.3** : O_DIRECT est désormais le comportement **par défaut** (depuis la 10.6), et la variable `innodb_flush_method` est elle-même **dépréciée** depuis la 11.0 — un serveur 12.3 démarre avec un avertissement si elle est définie. Le réglage fin moderne passe par des variables booléennes modifiables à chaud : `innodb_data_file_buffering` et `innodb_log_file_buffering` (laissées à `OFF`, leur valeur par défaut, pour un accès direct sans cache du système d'exploitation). La conserver à `O_DIRECT` reste toutefois fonctionnel et sans risque.

## Entrées/sorties

`innodb_io_capacity` indique à InnoDB le débit d'écriture de fond qu'il peut soutenir : modeste sur disque mécanique, élevé sur SSD ou NVMe. Sur stockage flash, `innodb_flush_neighbors = 0` évite de regrouper inutilement l'écriture de pages voisines, et l'on peut augmenter les fils d'E/S (`innodb_read_io_threads`, `innodb_write_io_threads`).

## Concurrence

Sous forte concurrence, le **thread pool** intégré (`thread_handling = pool-of-threads`) sert mieux de nombreuses connexions que le modèle « un thread par connexion », en limitant le coût des changements de contexte. On dimensionne `thread_pool_size` autour du nombre de cœurs. `max_connections` se règle selon le besoin réel de l'application (mieux vaut un pool applicatif bien réglé qu'un plafond démesuré).

## Journalisation binaire : deux options

La 12.3 introduit un choix structurant pour l'OLTP. Le binlog peut désormais être **stocké à l'intérieur d'InnoDB**, ce qui supprime le commit en deux phases entre binlog et moteur : la durabilité ne dépend plus que d'`innodb_flush_log_at_trx_commit`, et un seul `fsync` par commit suffit (au lieu de deux). C'est, pour l'OLTP, le gain de performance le plus marquant de la version.

**Option A — binlog InnoDB (recommandée pour un serveur autonome ou avec réplica asynchrone) :**

```ini
log_bin                              # sans valeur (les noms de fichiers sont fixes : binlog-NNNNNN.ibb)
binlog_storage_engine = innodb       # binlog stocké dans InnoDB (MariaDB 12.3)
```

Avec cette option, `sync_binlog` devient inutile (les fichiers binlog sont déjà résistants aux crashs). Elle suppose toutefois la réplication par **GTID** et présente des limites à connaître : elle n'est **pas compatible avec Galera**, la **réplication semi-synchrone y est indisponible** (toute tentative de l'activer échoue avec l'erreur `4248` « *Semi-synchronous replication is not yet supported with --binlog-storage-engine* »), les positions par nom de fichier/offset ne sont plus utilisables, et les outils tiers lisant directement les fichiers binlog classiques ne comprennent pas le nouveau format.

**Option B — binlog fichier classique (pour Galera, ou par souci de compatibilité) :**

```ini
log_bin               = mariadb-bin
binlog_format         = ROW
sync_binlog           = 1            # avec innodb_flush_log_at_trx_commit = 1 : durabilité maximale
```

*Le binlog InnoDB est détaillé au Ch. 11.5.4 ; la réplication et Galera, aux Ch. 13 et 14.*

## Note : isolation par instantané

`innodb_snapshot_isolation` est activé par défaut **depuis la 11.6.2** (donc déjà en 11.8, et bien sûr en 12.3 — ce n'est pas un changement propre au passage 11.8 → 12.3). Sous `REPEATABLE READ`, le serveur signale les conflits d'écriture par une erreur (`1020`) plutôt que de les ignorer silencieusement — un comportement à connaître pour les applications OLTP qui mettent à jour des lignes en concurrence. *Voir Ch. 6.9.*

## Ce qu'il ne faut pas configurer en 12.3

Plusieurs réglages hérités d'anciens `my.cnf` sont désormais inutiles, voire absents : le **Query Cache** (`query_cache_*`), déprécié ; `innodb_buffer_pool_instances` (supprimée) et `innodb_buffer_pool_chunk_size` (ignorée) ; `innodb_log_files_in_group` et `innodb_redo_log_capacity`, sans objet en MariaDB ; et les variables retirées de la série 12.x — `big_tables`, `large_page_size`, `storage_engine` (*voir Ch. 11.2.3*).

## Pour aller plus loin

Le dimensionnement du buffer pool est traité au [Ch. 15.2](../../15-performance-tuning/README.md), la configuration des E/S au [Ch. 15.4](../../15-performance-tuning/README.md), le binlog InnoDB au [Ch. 11.5.4](../../11-administration-configuration/README.md), le thread pool au [Ch. 11.10](../../11-administration-configuration/README.md), et l'isolation par instantané au [Ch. 6.9](../../06-transactions-et-concurrence/README.md). Pour vérifier une configuration, voir l'[Annexe E — Checklist de Performance](../e-checklist-performance/README.md).

---

⬅️ [Annexe D — Configuration de Référence (présentation)](README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [D.2 — OLAP (Data warehouse, analytics)](02-configuration-olap.md)

⏭️ [OLAP (Data warehouse, analytics)](/annexes/d-configuration-reference/02-configuration-olap.md)
