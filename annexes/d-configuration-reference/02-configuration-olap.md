🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.2 — OLAP (Data warehouse, analytics)

> 📈 Peu de requêtes lourdes sur de gros volumes : la configuration privilégie les grands parcours, une mémoire généreuse par requête, et — à grande échelle — le moteur ColumnStore.

Le profil **OLAP** (*Online Analytical Processing*) rassemble des requêtes peu nombreuses mais lourdes : parcours étendus, agrégations, jointures volumineuses, rapports, le tout alimenté par des chargements en masse (ETL). Peu d'utilisateurs concurrents, mais chacun mobilise beaucoup de ressources.

> ⚠️ Rappel : configuration de départ à **adapter au matériel** et à valider par la mesure. Voir la [présentation de l'annexe D](README.md).

---

## Le choix du moteur

Deux voies coexistent. Pour de l'analytique sur des volumes modérés, ou sur un serveur à profil mixte, **InnoDB** convient : il bénéficie d'un grand buffer pool et d'index adaptés, et c'est l'objet de la configuration ci-dessous. Pour du **data warehousing à grande échelle**, le moteur **ColumnStore** est conçu pour cela : en stockant les données par colonne, il accélère massivement les agrégations sur de très gros volumes et parallélise les traitements en interne. ColumnStore possède toutefois son **propre système de configuration**, distinct de ce `my.cnf` — voir [Ch. 7.5](../../07-moteurs-de-stockage/README.md) et [Ch. 20.3](../../20-cas-usage-architectures/README.md).

Un point structurant à garder en tête : MariaDB ne **parallélise pas** l'exécution d'un `SELECT` unique sur InnoDB (une requête = un thread). Au-delà d'un certain volume de scan, c'est ColumnStore — et son parallélisme interne — qui fait la différence.

## Configuration de référence — analytique sur InnoDB (`[mysqld]`)

```ini
[mysqld]
# ============================================================
#  Profil OLAP (analytique sur InnoDB) — MariaDB 12.3
#  Peu de requêtes simultanées : on peut se permettre
#  de gros tampons PAR SESSION. À ajuster au matériel.
# ============================================================

# --- Mémoire globale ---
innodb_buffer_pool_size     = <0.6-0.7 * RAM>   # cache des données chaudes
key_buffer_size             = 32M               # petit si aucune table MyISAM

# --- Tampons par requête (généreux car faible concurrence) ---
sort_buffer_size            = 16M               # tris : ORDER BY, GROUP BY
join_buffer_size            = 16M               # jointures sans index (block nested loop)
read_buffer_size            = 4M                # parcours séquentiels
read_rnd_buffer_size        = 8M                # lectures après tri
tmp_table_size              = 512M              # tables temporaires en mémoire (GROUP BY, DISTINCT)
max_heap_table_size         = 512M              # à régler avec tmp_table_size (la plus petite prime)

# --- Garde-fou sur l'espace temporaire disque (gros tris qui débordent) ---
max_tmp_session_space_usage = 8G                # par session
max_tmp_total_space_usage   = 32G               # toutes sessions confondues

# --- Concurrence (faible) ---
max_connections             = 100
thread_handling             = pool-of-threads

# --- Entrées/sorties ---
innodb_io_capacity          = 2000
innodb_io_capacity_max      = 4000
innodb_flush_neighbors      = 0
```

## Mémoire par requête : l'inversion par rapport à l'OLTP

C'est la différence majeure avec le profil OLTP. Comme peu de requêtes s'exécutent en même temps, on peut allouer de **gros tampons par session** — `sort_buffer_size`, `join_buffer_size`, `tmp_table_size` — pour absorber tris, jointures et regroupements volumineux en mémoire. Le principe à respecter : des tampons par connexion généreux ne sont sûrs que tant que peu de sessions les sollicitent simultanément. Il faut donc que le produit (taille des tampons × requêtes concurrentes) tienne dans la RAM, ce qui justifie le `max_connections` volontairement modeste. À noter : la limite effective des tables temporaires en mémoire est la **plus petite** de `tmp_table_size` et `max_heap_table_size` — on relève donc les deux ensemble.

## Espace temporaire : un garde-fou contre les requêtes folles

Une requête analytique mal écrite peut faire déborder un tri sur disque et saturer le système de fichiers. Les variables `max_tmp_session_space_usage` (par session) et `max_tmp_total_space_usage` (global) plafonnent l'usage de l'espace temporaire **sur disque** — fichiers de tri, tables temporaires sur disque, etc. — sans affecter les tables temporaires en mémoire, gouvernées par `tmp_table_size`. C'est un filet de sécurité précieux en OLAP. *Voir Ch. 11.8.*

## Chargements en masse (ETL)

Un entrepôt se remplit par lots, et cette phase a ses propres leviers. On peut **assouplir temporairement la durabilité** pendant le chargement (`innodb_flush_log_at_trx_commit = 2`) pour accélérer l'ETL, à condition de la rétablir ensuite — acceptable pour des données rechargeables, jamais pour le système d'enregistrement de référence. Côté méthode, `LOAD DATA` est plus efficace que des `INSERT` unitaires, et il est souvent préférable de construire les index secondaires **après** le chargement. La construction d'index bénéficie par ailleurs d'une optimisation de copie en masse (`innodb_alter_copy_bulk`, *voir Ch. 15.6*).

## Partitionnement des grandes tables

Sur les très grandes tables de faits, le partitionnement (typiquement par plage de dates) permet l'**élagage des partitions** : l'optimiseur n'examine que les partitions pertinentes pour une requête datée, réduisant fortement le volume parcouru. C'est un complément naturel d'une configuration OLAP. *Voir Ch. 15.9.*

## Ce qu'il ne faut pas configurer

Le **Query Cache** (déprécié) est particulièrement inutile en OLAP, où les résultats, volumineux et variés, sont rarement rejoués à l'identique. Comme pour l'OLTP, on évite aussi les variables ignorées ou retirées : `innodb_buffer_pool_instances`, `innodb_buffer_pool_chunk_size`, `innodb_log_files_in_group`, `innodb_redo_log_capacity`, et les retirées de la série 12.x (`big_tables`, `large_page_size`, `storage_engine`). Enfin, il ne faut pas compter sur un parallélisme de requête sous InnoDB : pour cela, c'est vers ColumnStore qu'il faut se tourner.

## Pour aller plus loin

Le moteur ColumnStore est traité au [Ch. 7.5](../../07-moteurs-de-stockage/README.md) et son usage en entrepôt au [Ch. 20.3](../../20-cas-usage-architectures/README.md). La distinction des charges est développée au [Ch. 20.1 — OLTP vs OLAP](../../20-cas-usage-architectures/README.md), le contrôle de l'espace temporaire au [Ch. 11.8](../../11-administration-configuration/README.md), la construction d'index en masse au [Ch. 15.6](../../15-performance-tuning/README.md), le partitionnement au [Ch. 15.9](../../15-performance-tuning/README.md) et le dimensionnement mémoire au [Ch. 15.2](../../15-performance-tuning/README.md). Pour vérifier une configuration, voir l'[Annexe E — Checklist de Performance](../e-checklist-performance/README.md).

---

⬅️ [D.1 — OLTP (High concurrency, low latency)](01-configuration-oltp.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [D.3 — Mixed workload](03-configuration-mixed-workload.md)

⏭️ [Mixed workload](/annexes/d-configuration-reference/03-configuration-mixed-workload.md)
