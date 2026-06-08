🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.6 Compression de tables

## Pourquoi compresser

Compresser une table consiste à réduire l'espace qu'elle occupe sur le disque, au prix de cycles processeur consacrés à la compression et à la décompression. C'est fondamentalement un **arbitrage entre stockage et CPU** : on accepte un surcoût de calcul pour économiser de l'espace et, souvent, des entrées-sorties.

Le gain dépend fortement de la **nature des données**. Du texte, du JSON ou des valeurs répétitives se compriment très bien et profitent pleinement de la compression ; des données déjà compressées (images, contenus chiffrés) n'y gagnent rien et n'engendrent qu'un gaspillage de CPU. Le contexte matériel compte aussi : sur un stockage lent ou de très gros volumes, l'économie d'E/S est précieuse ; sur des SSD rapides où l'E/S n'est pas le goulot d'étranglement (§15.4.3), le coût processeur peut ne pas être rentabilisé.

Cette section traite avant tout de la compression au sein du moteur **InnoDB**. D'autres formes de compression, propres à certains moteurs spécialisés, sont évoquées en fin de section.

## InnoDB : deux mécanismes

InnoDB propose deux approches distinctes de compression de table :

- la **page compression** (`PAGE_COMPRESSED`), moderne et généralement recommandée ;
- la **compression de format de ligne** (`ROW_FORMAT=COMPRESSED`), plus ancienne, héritée de MySQL.

Toutes deux supposent le mode **file-per-table** (chaque table dans son propre fichier `.ibd`), qui est le réglage par défaut.

## Page compression (`PAGE_COMPRESSED`)

C'est l'approche privilégiée. Chaque page est compressée avant d'être écrite ; l'espace ainsi libéré en fin de page est ensuite « perforé » dans le fichier au moyen de **fichiers creux** (*sparse files*) : le système de fichiers n'alloue physiquement que les blocs réellement utilisés.

```sql
CREATE TABLE journal (
  id      BIGINT PRIMARY KEY,
  contenu TEXT
) PAGE_COMPRESSED=1 PAGE_COMPRESSION_LEVEL=6;
```

Elle offre le **choix de l'algorithme** via la variable `innodb_compression_algorithm` — `zlib` par défaut, mais aussi `lz4` (rapide), `lzo`, `lzma`, `bzip2` ou `snappy` selon les options de compilation :

```sql
SET GLOBAL innodb_compression_algorithm = 'lz4';
```

Le niveau de compression de la table se règle par `PAGE_COMPRESSION_LEVEL` (de 1, le plus rapide, à 9, le plus compact).

> ⚠️ **Prérequis filesystem.** La perforation de trous exige un système de fichiers qui la prenne en charge (XFS, ext4, btrfs sous Linux ; NTFS sous Windows). À défaut, les pages sont bien compressées mais l'espace n'est **pas** récupéré, et le bénéfice disparaît. À vérifier impérativement avant d'opter pour cette méthode.

À noter : la page compression économise l'espace **disque** uniquement ; les pages sont décompressées en mémoire dans le buffer pool.

## `ROW_FORMAT=COMPRESSED`

La compression classique d'InnoDB comprime les pages de l'arbre B à une **taille de bloc cible** fixée par `KEY_BLOCK_SIZE` (1, 2, 4 ou 8 Ko). Plus le bloc est petit, plus la compression est forte, mais plus le surcoût augmente.

```sql
CREATE TABLE archive_doc (
  id      BIGINT PRIMARY KEY,
  contenu TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

Sa particularité est que les **pages peuvent demeurer compressées dans le buffer pool** : elle économise donc à la fois le disque **et** la mémoire. En contrepartie, elle n'utilise que `zlib` et impose un surcoût en écriture plus marqué — une page qui ne se recomprime plus dans son bloc cible doit être réorganisée.

## Choisir entre les deux

| Critère | `PAGE_COMPRESSED` | `ROW_FORMAT=COMPRESSED` |
|---|---|---|
| Mécanisme | Compression de page + perforation (fichiers creux) | Pages B-tree compressées à une taille de bloc fixe |
| Économie | Disque uniquement | Disque **et** mémoire (buffer pool) |
| Algorithmes | zlib, lz4, lzo, lzma, bzip2, snappy (selon build) | zlib uniquement |
| Réglage | `PAGE_COMPRESSION_LEVEL` (1–9) | `KEY_BLOCK_SIZE` (1/2/4/8 Ko) |
| Prérequis | file-per-table + FS perforable | file-per-table |
| Positionnement | Approche moderne, recommandée par défaut | Classique ; pertinente si la mémoire est aussi contrainte |

En pratique, on retient la **page compression** dans la majorité des cas, et `ROW_FORMAT=COMPRESSED` lorsque l'on cherche à soulager *aussi* le buffer pool sur une instance à la mémoire serrée.

## Appliquer et vérifier

La compression se déclare à la création (comme ci-dessus) ou s'applique à une table existante — ce qui en **reconstruit** le contenu :

```sql
ALTER TABLE journal PAGE_COMPRESSED=1;
```

`SHOW CREATE TABLE` confirme les options en vigueur :

```sql
SHOW CREATE TABLE journal\G
```

Pour mesurer l'efficacité de la page compression, on s'appuie sur les variables de statut `Innodb` liées à la compression : `Innodb_page_compression_saved` (octets économisés), ainsi que `Innodb_num_pages_page_compressed` et `Innodb_num_pages_page_decompressed` (nombre de pages compressées et décompressées). Un motif large les capture toutes :

```sql
SHOW STATUS LIKE 'Innodb%compress%';
```

La comparaison de la taille des fichiers avant et après reste également un indicateur direct.

## Quand compresser, et quand s'en abstenir

La compression est avantageuse pour des données **qui se compriment bien** — texte, JSON, logs, valeurs répétitives — surtout lorsqu'elles sont volumineuses, plutôt lues qu'écrites, et stockées sur un support où l'E/S pèse. Elle se combine par ailleurs avec le **chiffrement au repos** (§18.7), la compression intervenant avant le chiffrement.

À l'inverse, mieux vaut s'abstenir pour des données **déjà compressées ou chiffrées** (la recompression n'apporte rien), pour des tables **très sollicitées en écriture** (le surcoût de recompression devient pénalisant), ou lorsque le stockage rapide rend l'économie d'E/S marginale au regard du coût CPU. Comme souvent, un test sur des données et une charge représentatives tranche mieux que toute règle générale.

## Au-delà d'InnoDB

D'autres mécanismes de compression existent en dehors d'InnoDB, traités dans leurs sections respectives :

- le moteur **Archive** (§7.10.2), conçu pour une compression maximale de données froides en ajout seul ;
- **ColumnStore** (§7.5), dont le stockage en colonnes intègre sa propre compression, particulièrement efficace pour l'analytique et le data warehousing (§20.3) ;
- les tables **MyISAM compressées** via l'utilitaire `myisampack`, en lecture seule — une option ancienne, à ne considérer que sur des données historiques figées.

## Points clés à retenir

- Compresser, c'est échanger du **CPU** contre de l'**espace** (et souvent des E/S) ; le gain dépend de la compressibilité des données et du matériel.
- InnoDB offre la **page compression** (`PAGE_COMPRESSED`, économie disque, choix d'algorithme, fichiers creux) et `ROW_FORMAT=COMPRESSED` (économie disque **et** mémoire, zlib, `KEY_BLOCK_SIZE`).
- La page compression **exige un système de fichiers perforable** ; sans cela, aucun espace n'est récupéré.
- On l'applique à la création ou par `ALTER TABLE` (qui reconstruit la table), et on mesure le gain via les statuts `Innodb…compress…` (`Innodb_page_compression_saved`, `Innodb_num_pages_page_compressed/_decompressed`) et la taille des fichiers.
- À réserver aux données **compressibles** et plutôt lues ; à éviter sur des données déjà compressées/chiffrées ou très sollicitées en écriture. Se combine avec le chiffrement (§18.7).
- Hors InnoDB : moteurs **Archive** (§7.10.2) et **ColumnStore** (§7.5).

⏭️ [Encryption at rest](/18-fonctionnalites-avancees/07-encryption-at-rest.md)
