🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 — Gestion de l'espace disque

L'espace disque est une ressource finie et critique pour un serveur MariaDB. Contrairement à la mémoire, dont la sur-utilisation se traduit « seulement » par une dégradation des performances, une saturation du disque a des conséquences immédiates et brutales sur la disponibilité du service. Cette section explique **où** MariaDB consomme l'espace, **comment** le mesurer précisément, **comment** récupérer l'espace inutilisé, et **comment** prévenir la saturation. Le contrôle des quotas d'espace temporaire (`max_tmp_space_usage`, `max_total_tmp_space_usage`) fait l'objet de la section suivante ([11.8](08-controle-espace-temporaire.md)) et n'est ici qu'évoqué.

---

## Pourquoi l'espace disque est un enjeu critique

Lorsqu'un système de fichiers hébergeant les données InnoDB arrive à saturation, MariaDB ne peut plus écrire. Les symptômes typiques sont sévères :

- les transactions en écriture échouent ou se bloquent en attente d'espace ;
- InnoDB peut passer le serveur en lecture seule de fait, voire interrompre proprement le service pour éviter une corruption ;
- l'écriture du journal binaire (binlog) échoue, ce qui **rompt la réplication** vers les réplicas ;
- les opérations de maintenance qui nécessitent de l'espace temporaire (reconstruction de table, gros tris, `ALTER TABLE`) deviennent impossibles, précisément au moment où l'on en aurait besoin pour libérer de la place.

La difficulté tient à un effet de seuil : tant qu'il reste de la place, tout fonctionne ; au-delà, plus rien ne fonctionne. La gestion de l'espace disque est donc avant tout une **discipline préventive**, adossée à une supervision continue (voir [11.9](09-monitoring-metriques.md) et [16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md)).

---

## Cartographie : où MariaDB consomme l'espace

Tout commence dans le répertoire de données (`datadir`, typiquement `/var/lib/mysql`). Comprendre l'anatomie de ce répertoire est le préalable à toute gestion de l'espace.

| Fichier / motif | Rôle | Comportement vis-à-vis de l'espace |
|---|---|---|
| `ibdata1` | Tablespace système InnoDB : dictionnaire de données, change buffer, logs undo (si non séparés) | Grandit avec `autoextend` ; ne se réduit pas spontanément en exploitation |
| `*.ibd` | Tablespaces *file-per-table* : données + index d'une table | Un fichier par table ; espace récupérable table par table |
| `*.frm` | Définition des tables (MariaDB conserve les `.frm`, contrairement à MySQL 8 qui utilise un dictionnaire transactionnel) | Très petits |
| `ib_logfile0` | Redo log InnoDB (write-ahead log) | Taille fixe définie par `innodb_log_file_size` |
| `ibtmp1` | Tablespace temporaire InnoDB | Gonfle avec les gros tris / tables temporaires ; voir plus bas |
| `undoNNN` | Tablespaces undo séparés (si configurés) | Croissent avec les transactions longues ; tronquables |
| `binlog-NNNNNN.ibb` | **Journaux binaires nouveau format InnoDB (12.3)** | Préalloués (voir section dédiée plus bas) |
| `binlog.NNNNNN` / `*-bin.NNNNNN` | Journaux binaires format historique | Croissent en continu ; doivent être purgés |
| `*-relay-bin.NNNNNN` | Relay logs (sur un réplica) | Purgés automatiquement une fois appliqués |
| `*.MYD` / `*.MYI` | Données / index MyISAM | Espace récupérable via reconstruction |
| `*.MAD` / `*.MAI` | Données / index Aria | Idem |
| `aria_log.*`, `aria_log_control` | Journaux du moteur Aria | Petits |
| `*.err` (error log) | Journal d'erreurs | Croissance lente, à faire tourner |
| slow / general log | Journaux de requêtes | Le general log peut **exploser** (voir plus bas) |
| `ib_buffer_pool` | Dump du buffer pool au shutdown | Négligeable |

Un point d'architecture important : ces familles de fichiers **ne sont pas obligatoirement sur le même volume**. Plusieurs variables permettent de les répartir :

- `datadir` : emplacement principal des données ;
- `innodb_data_home_dir` : emplacement du tablespace système ;
- `innodb_undo_directory` : emplacement des tablespaces undo ;
- `log_bin` / `log-bin` : emplacement des journaux binaires ;
- `tmpdir` et `innodb_temp_data_file_path` : emplacement des données temporaires.

Cette séparation est un levier de gestion : isoler les binlogs ou le tablespace temporaire sur un volume distinct évite qu'une croissance imprévue de ces fichiers ne sature le volume des données.

---

## Mesurer l'occupation réelle

La gestion commence par la mesure, à deux niveaux complémentaires.

### Au niveau du système de fichiers

Le système d'exploitation donne la vérité physique. C'est lui qu'il faut surveiller en priorité, car c'est lui qui sature.

```bash
# Espace disponible sur le volume des données
df -h /var/lib/mysql

# Les 20 plus gros consommateurs dans le datadir
du -sh /var/lib/mysql/* | sort -rh | head -20
```

### Au niveau de MariaDB

`INFORMATION_SCHEMA` permet de ventiler l'occupation par base et par table. La requête suivante classe les bases par taille :

```sql
SELECT
    table_schema                                              AS base,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 1)   AS taille_mb,
    ROUND(SUM(data_free) / 1024 / 1024, 1)                    AS espace_libre_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY taille_mb DESC;
```

Pour identifier les tables les plus volumineuses et repérer la fragmentation :

```sql
SELECT
    table_schema                                            AS base,
    table_name                                              AS nom_table,
    engine                                                  AS moteur,
    ROUND((data_length + index_length) / 1024 / 1024, 1)    AS taille_mb,
    ROUND(data_free / 1024 / 1024, 1)                       AS espace_libre_mb,
    table_rows                                              AS lignes_estimees
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

La vue `INNODB_SYS_TABLESPACES` distingue la taille **apparente** du fichier de la taille **physiquement allouée** (utile sur les systèmes de fichiers qui gèrent les fichiers creux) :

```sql
SELECT
    name                                        AS tablespace,
    ROUND(file_size / 1024 / 1024, 1)           AS taille_apparente_mb,
    ROUND(allocated_size / 1024 / 1024, 1)      AS taille_physique_mb
FROM information_schema.innodb_sys_tablespaces
ORDER BY file_size DESC
LIMIT 20;
```

> ⚠️ La colonne `DATA_FREE` est une **estimation** fournie par InnoDB. Elle reflète l'espace réutilisable à l'intérieur du fichier de la table, pas l'espace qui sera rendu au système. Elle sert d'indicateur de fragmentation, pas de mesure exacte.

---

## Le piège de la fragmentation : pourquoi `DELETE` ne libère pas l'espace

C'est l'incompréhension la plus fréquente en administration MariaDB : **supprimer des lignes ne réduit pas la taille du fichier sur disque.**

Lorsqu'on exécute `DELETE`, InnoDB ne restitue pas immédiatement l'espace au système de fichiers. Les pages libérées sont conservées dans une liste interne et seront réutilisées par de futurs `INSERT`. Le fichier `.ibd` ne rétrécit pas ; il contient simplement de l'espace « creux » réutilisable en interne. Le tablespace système `ibdata1`, lui, ne rend **jamais** spontanément l'espace issu de suppressions.

Il faut donc distinguer :

- l'**espace logique libre** (réutilisable par MariaDB, visible via `DATA_FREE`) ;
- l'**espace physique** réellement rendu au système d'exploitation, qui ne diminue qu'à l'issue d'opérations explicites décrites ci-dessous.

---

## Récupérer l'espace disque

### `innodb_file_per_table` : le prérequis

Activé par défaut (`ON`), ce paramètre fait que chaque table InnoDB possède son propre fichier `.ibd`. C'est la condition qui rend l'espace récupérable **table par table** : seul un fichier dédié peut être tronqué, supprimé ou reconstruit pour rendre de l'espace au système.

Si ce paramètre est désactivé, toutes les données partagent `ibdata1`, et la seule façon de récupérer de l'espace devient un export logique complet suivi d'un rechargement — une opération lourde et indisponibilisante. En pratique, on laisse `innodb_file_per_table = ON`.

### Reconstruire une table

Pour qu'une table « compacte » son fichier et rende l'espace fragmenté, il faut la reconstruire :

```sql
-- Reconstruit la table, défragmente et rend l'espace au système (file-per-table)
OPTIMIZE TABLE commandes;

-- Équivalents par reconstruction explicite
ALTER TABLE commandes FORCE;
ALTER TABLE commandes ENGINE = InnoDB, ALGORITHM = INPLACE, LOCK = NONE;
```

Sur InnoDB, `OPTIMIZE TABLE` est traité comme une reconstruction de table (les détails de syntaxe et de comportement sont vus en [11.6.1](06.1-optimize-table.md)). La variante `ALGORITHM = INPLACE, LOCK = NONE` permet une opération en ligne, peu bloquante (voir aussi l'*Online Schema Change*, [18.11](../18-fonctionnalites-avancees/11-online-schema-change.md)).

> ⚠️ **Point de vigilance majeur** : une reconstruction crée une copie de la table avant de remplacer l'ancienne. Elle exige donc temporairement un espace libre **au moins égal à la taille de la table**. Reconstruire la plus grosse table d'un serveur déjà proche de la saturation est précisément ce qui ne peut pas se faire. C'est une raison de plus de conserver une marge.

### Le tablespace système (`ibdata1`)

`ibdata1` stocke le dictionnaire de données, le change buffer et, par défaut, les logs undo. En exploitation normale, il ne rétrécit pas. Deux nuances utiles :

- **Au redémarrage**, MariaDB récupère automatiquement l'espace inutilisé en fin de tablespace système (comportement standard depuis la 11.2, donc actif en 12.3). Cela atténue le principal motif historique de gonflement (accumulation d'undo).
- En cas de croissance pathologique persistante, le retour à un `ibdata1` compact reste **destructif** : sauvegarde logique complète, arrêt du serveur, suppression des fichiers de tablespace système et de redo, réinitialisation puis rechargement. C'est une opération de dernier recours, à mener avec une extrême prudence et toujours après sauvegarde validée. Mieux vaut prévenir la cause (transactions longues, undo non séparé) que de la subir.

### Les logs undo

Les logs undo conservent les « images avant » des lignes modifiées, nécessaires au rollback et au MVCC ([6.6](../06-transactions-et-concurrence/06-mvcc.md)). Ils gonflent lorsque des transactions restent ouvertes longtemps ou que la purge prend du retard. Pour borner et récupérer cet espace, on sépare les undo dans leurs propres tablespaces et on active la troncature :

```ini
[mariadb]
innodb_undo_tablespaces              = 3
innodb_undo_log_truncate             = ON
innodb_max_undo_log_size             = 1G
innodb_purge_rseg_truncate_frequency = 32
```

La troncature exige au moins **deux** tablespaces undo actifs (l'un reste disponible pendant que l'autre est tronqué). Un bon indicateur de surveillance est la *history list length*, lisible via `SHOW ENGINE INNODB STATUS` : une valeur qui croît durablement signale une purge à la peine, souvent provoquée par une transaction restée ouverte.

### Le tablespace temporaire (`ibtmp1`)

`ibtmp1` accueille les tables temporaires InnoDB et les résultats intermédiaires des gros tris. Une seule requête mal écrite peut le faire gonfler de plusieurs gigaoctets. Particularités :

- ce tablespace est **supprimé et recréé à chaque arrêt/démarrage propre** : un simple redémarrage le ramène à sa taille minimale ;
- depuis la 11.3, on peut le réduire **sans redémarrer** :

```sql
SET GLOBAL innodb_truncate_temporary_tablespace_now = ON;
```

- on peut le borner via `innodb_temp_data_file_path`, en plafonnant l'auto-extension :

```ini
[mariadb]
innodb_temp_data_file_path = ibtmp1:256M:autoextend:max:20G
```

Pour imposer des **quotas** par requête ou globaux et éviter qu'une requête ne sature à elle seule l'espace temporaire, voir la section dédiée [11.8](08-controle-espace-temporaire.md) (`max_tmp_space_usage`, `max_total_tmp_space_usage`).

---

## Les journaux binaires et l'espace disque

Les binlogs sont une cause d'incident d'espace disque récurrente : ils s'accumulent silencieusement tant qu'une politique de purge n'est pas définie. La rétention se pilote principalement par le temps :

```sql
-- Conserver 7 jours de journaux binaires
SET GLOBAL binlog_expire_logs_seconds = 604800;

-- Purge manuelle, jusqu'à un fichier donné ou jusqu'à une date
PURGE BINARY LOGS TO 'binlog.000123';
PURGE BINARY LOGS BEFORE '2026-06-01 00:00:00';
```

Les détails de configuration, de format et de rotation sont traités en [11.5](05-binary-logs.md) (et [11.5.3](05.3-purge-rotation.md) pour la purge). Règle de sécurité absolue : **ne jamais supprimer manuellement** des fichiers de binlog au niveau du système de fichiers — cela peut rendre incohérente la réplication des réplicas qui n'auraient pas encore tout consommé. On purge toujours par les commandes prévues.

### Nouveau binlog InnoDB (12.3) et planification de l'espace

La 12.3 introduit un nouveau format de journal binaire intégré à InnoDB (voir [11.5.4](05.4-binlog-innodb-performance.md)). Activé par `binlog_storage_engine = innodb`, il stocke les événements dans des tablespaces gérés par InnoDB sous forme de fichiers `binlog-NNNNNN.ibb` au lieu de fichiers plats.

```ini
[mariadb]
log-bin
binlog_storage_engine = innodb
```

Une conséquence directe pour la **planification de capacité** : ces fichiers `.ibb` sont **préalloués (de l'ordre de 1 Go par fichier)**. Sur un serveur peu chargé, voire au repos, l'occupation disque « paraît » donc élevée immédiatement après l'activation. **Ce n'est pas une fuite** : c'est de l'espace réservé d'avance. Il faut simplement en tenir compte dans le dimensionnement du volume hébergeant les binlogs, et ne pas confondre cette préallocation avec une croissance anormale.

---

## Les journaux textuels

Au-delà des fichiers de données, les journaux peuvent occuper un espace non négligeable :

- **Error log** : croissance lente ; à faire tourner périodiquement (rotation logrotate + `FLUSH ERROR LOGS`).
- **General log** : enregistre **chaque** requête. Il peut atteindre des dizaines de gigaoctets en quelques heures sur un serveur actif. **À ne pas laisser activé en production**, sauf diagnostic ponctuel et borné dans le temps.
- **Slow query log** : utile et généralement modéré, mais à surveiller si le seuil `long_query_time` est très bas.
- **Relay logs** (réplica) : purgés automatiquement une fois appliqués ; une accumulation signale un réplica en retard.

La rotation s'organise classiquement avec `logrotate` côté système, complété par `FLUSH LOGS` pour que MariaDB rouvre proprement ses fichiers. La configuration fine des journaux est détaillée en [11.4](04-gestion-logs.md).

---

## Réduire l'empreinte : compression et archivage

Plutôt que de courir après l'espace, on peut structurellement réduire l'empreinte des données :

- **Compression de tables** (`ROW_FORMAT=COMPRESSED`, page compression) pour les données peu sollicitées en écriture — voir [18.6](../18-fonctionnalites-avancees/06-compression-tables.md).
- **ColumnStore** pour les charges analytiques, avec un taux de compression élevé sur de gros volumes — voir [7.5](../07-moteurs-de-stockage/05-columnstore.md).
- **Moteur S3** pour archiver les données froides sur un stockage objet et les sortir du volume principal — voir [7.6](../07-moteurs-de-stockage/06-moteur-s3.md).
- **Partitionnement** : découper une grosse table par période permet de supprimer des partitions anciennes d'un seul `ALTER TABLE ... DROP PARTITION`, ce qui rend l'espace bien plus efficacement qu'un `DELETE` massif — voir [15.9](../15-performance-tuning/09-partitionnement-tables.md) et [15.10](../15-performance-tuning/10-gestion-avancee-partitions.md).

---

## Bonnes pratiques de gestion de l'espace disque

En synthèse, une gestion saine de l'espace disque repose sur quelques principes :

- **Superviser le système de fichiers en continu**, avec des alertes à plusieurs seuils (par exemple 70 % puis 85 %), bien avant la saturation, afin de disposer du temps nécessaire pour agir.
- **Conserver une marge de manœuvre** au moins égale à la taille de la plus grosse table, sans quoi les reconstructions deviennent impossibles précisément quand elles sont nécessaires.
- **Séparer les volumes** (données, binlogs, tablespace temporaire, undo) pour qu'une croissance imprévue de l'un ne sature pas les autres.
- **Borner explicitement** la rétention des binlogs, la taille des undo (`innodb_max_undo_log_size` + troncature) et l'espace temporaire (voir [11.8](08-controle-espace-temporaire.md)).
- **Ne jamais laisser le general log actif** en production en dehors d'un diagnostic ponctuel et limité dans le temps.
- **Anticiper la préallocation des fichiers `.ibb`** si le nouveau binlog InnoDB de la 12.3 est activé, pour ne pas confondre réservation et fuite.
- **Réviser périodiquement** les plus gros consommateurs et la fragmentation (`DATA_FREE`), et planifier les reconstructions ciblées hors heures de pointe.
- **Disposer d'une procédure d'urgence** documentée pour les situations de quasi-saturation : purger d'abord les anciens binlogs (`PURGE BINARY LOGS`), réduire le tablespace temporaire (redémarrage ou `innodb_truncate_temporary_tablespace_now`), puis seulement reconstruire les tables fragmentées une fois un peu d'espace regagné.

La gestion de l'espace disque n'est pas une tâche ponctuelle mais une **composante permanente de l'exploitation** : mesure régulière, bornage des sources de croissance, et supervision proactive sont les trois piliers qui évitent l'incident de production le plus banal — et le plus évitable — d'un serveur de base de données.

⏭️ [Contrôle espace temporaire (max_tmp_space_usage, max_total_tmp_space_usage)](/11-administration-configuration/08-controle-espace-temporaire.md)
