🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.3 — Mixed workload

> ⚖️ Un même serveur sert à la fois l'OLTP et l'OLAP : la configuration est un compromis assumé — et, idéalement, une invitation à séparer les charges.

Le profil **mixte** correspond à un serveur qui doit traiter en même temps des transactions courtes et fréquentes (OLTP) et des requêtes analytiques occasionnelles mais lourdes (OLAP). C'est le cas le plus délicat à régler, car les deux profils tirent la configuration dans des directions opposées.

> ⚠️ Rappel : configuration de départ à **adapter au matériel** et à valider par la mesure. Voir la [présentation de l'annexe D](README.md).

---

## D'abord : faut-il vraiment tout mélanger ?

Avant de chercher le compromis parfait, il faut poser la vraie question. Optimiser une charge mixte est difficile précisément parce que l'OLTP veut **beaucoup de petits tampons par connexion** tandis que l'OLAP veut **quelques très gros tampons** : ces deux exigences sont incompatibles sur un même réglage global.

La solution la plus propre, lorsqu'elle est possible, consiste à **séparer les charges** : conserver l'OLTP sur le serveur primaire et déporter l'analytique sur un réplica asynchrone — une option devenue peu coûteuse grâce au binlog InnoDB de la 12.3 — puis router les lectures via la séparation lecture/écriture de MaxScale. *Voir Ch. 13 (réplication), Ch. 14.4.2 (read/write split) et Ch. 20.1.* La configuration ci-dessous s'adresse au cas où un seul serveur doit assumer les deux.

## Configuration de référence (`[mysqld]`)

```ini
[mysqld]
# ============================================================
#  Profil mixte OLTP + OLAP — MariaDB 12.3
#  Compromis : buffer pool généreux, tampons par session MODÉRÉS
#  (à cause de la concurrence OLTP) ; les gros tampons sont
#  réservés aux sessions analytiques via SET SESSION.
# ============================================================

# --- Mémoire globale ---
innodb_buffer_pool_size        = <0.65-0.75 * RAM>
innodb_log_file_size           = 2G              # fichier unique en MariaDB 12.x
innodb_log_buffer_size         = 64M

# --- Durabilité (priorité à la sécurité, pour le volet OLTP) ---
innodb_flush_log_at_trx_commit = 1
innodb_flush_method            = O_DIRECT        # déprécié depuis la 11.0 (O_DIRECT déjà par défaut) ; cf. D.1

# --- Tampons par session : MODÉRÉS par défaut ---
sort_buffer_size               = 2M
join_buffer_size               = 2M
read_rnd_buffer_size           = 2M
tmp_table_size                 = 64M
max_heap_table_size            = 64M

# --- Garde-fou espace temporaire (protège l'OLTP d'une requête analytique folle) ---
max_tmp_session_space_usage    = 4G
max_tmp_total_space_usage      = 16G

# --- Concurrence (élevée, côté OLTP) ---
thread_handling                = pool-of-threads
thread_pool_size               = <nb_cœurs>
max_connections                = 500

# --- Entrées/sorties ---
innodb_io_capacity             = 2000
innodb_io_capacity_max         = 4000
innodb_flush_neighbors         = 0

# --- Réplication / binlog (options détaillées en D.1) ---
server_id                      = 1
binlog_format                  = ROW
```

## La clé : modéré en global, généreux par session

C'est le principe central d'une configuration mixte. Comme l'OLTP apporte de nombreuses connexions concurrentes, les tampons par session (`sort_buffer_size`, `join_buffer_size`, `tmp_table_size`) doivent rester **modestes au niveau global** : une grande valeur globale se multiplierait par le nombre de connexions et épuiserait la mémoire. Mais les requêtes analytiques, elles, réclament de gros tampons.

La réponse consiste à garder des valeurs globales modérées, puis à les **augmenter au niveau de la session** uniquement pour les connexions qui exécutent les traitements lourds :

```sql
-- Dans la session dédiée aux rapports, avant la requête lourde :
SET SESSION sort_buffer_size    = 64 * 1024 * 1024;
SET SESSION join_buffer_size    = 32 * 1024 * 1024;
SET SESSION tmp_table_size      = 512 * 1024 * 1024;
SET SESSION max_heap_table_size = 512 * 1024 * 1024;
```

La requête analytique dispose ainsi de ses gros tampons sans les imposer aux milliers de transactions OLTP courtes qui partagent le serveur.

## Durabilité et binlog

Le volet OLTP impose la sécurité des données : on conserve donc `innodb_flush_log_at_trx_commit = 1`. Le binlog InnoDB de la 12.3 (option A de la [section D.1](01-configuration-oltp.md)) convient bien ici — un seul `fsync` par commit pour l'OLTP, et un réplica asynchrone peu coûteux pour déporter l'analytique. On gardera à l'esprit ses prérequis (GTID) et ses incompatibilités (notamment Galera, qui impose alors le binlog fichier classique).

## Le garde-fou temporaire prend tout son sens

Dans un contexte mixte, les limites d'espace temporaire sont particulièrement précieuses : elles empêchent qu'une seule requête analytique mal maîtrisée ne sature le disque avec ses fichiers de tri et ne dégrade le service OLTP hébergé sur la même machine. C'est une protection mutuelle entre les deux charges. *Voir Ch. 11.8.*

## Ce qu'il ne faut pas configurer

Les mêmes réglages que pour les autres profils sont à proscrire : le **Query Cache** (déprécié), ainsi que les variables ignorées ou retirées (`innodb_buffer_pool_instances`, `innodb_buffer_pool_chunk_size`, `innodb_log_files_in_group`, `innodb_redo_log_capacity`, et les retirées de la série 12.x — `big_tables`, `large_page_size`, `storage_engine`). *Voir Ch. 11.2.3.*

## Pour aller plus loin

La séparation des charges (le meilleur remède à un profil mixte) repose sur la réplication ([Ch. 13](../../13-replication/README.md)) et la séparation lecture/écriture de MaxScale ([Ch. 14.4.2](../../14-haute-disponibilite/README.md)). Les deux profils de référence se trouvent en [D.1 — OLTP](01-configuration-oltp.md) et [D.2 — OLAP](02-configuration-olap.md). La portée des variables (globale vs session) est détaillée au [Ch. 11.2](../../11-administration-configuration/README.md), et le contrôle de l'espace temporaire au [Ch. 11.8](../../11-administration-configuration/README.md). Pour vérifier une configuration, voir l'[Annexe E — Checklist de Performance](../e-checklist-performance/README.md).

---

⬅️ [D.2 — OLAP (Data warehouse, analytics)](02-configuration-olap.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [D.4 — Développement local](04-configuration-developpement-local.md)

⏭️ [Développement local](/annexes/d-configuration-reference/04-configuration-developpement-local.md)
