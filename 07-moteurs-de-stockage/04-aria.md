🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.4 Aria : Le successeur de MyISAM

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS

## Aria, le MyISAM *crash-safe*

Aria est le moteur de stockage que MariaDB propose comme **successeur de MyISAM**. Conçu dès l'origine pour être un **équivalent de MyISAM résistant aux pannes** (*crash-safe*), il en reprend l'essentiel des fonctionnalités tout en corrigeant son principal défaut : la fragilité en cas d'arrêt brutal. Comme MyISAM, Aria **n'est pas transactionnel au sens d'InnoDB** (pas de `COMMIT`/`ROLLBACK` multi-instructions) et **ne gère pas les clés étrangères** ; pour ces besoins, c'est InnoDB qu'il faut retenir (§7.2).

Aria est développé depuis 2007 par les ingénieurs à l'origine du serveur MySQL et des moteurs MyISAM, MERGE et MEMORY. Il s'est d'abord appelé **« Maria »** (en référence à la fille cadette de Michael « Monty » Widenius), puis a été renommé **Aria** en 2010 pour éviter la confusion avec MariaDB ; les deux noms restent acceptés dans les versions actuelles. C'est un moteur **propre à MariaDB** : il n'est livré ni avec MySQL ni avec Percona Server.

## Une présence discrète mais omniprésente

Un point souvent ignoré : **tout utilisateur de MariaDB se sert d'Aria**, même sans jamais créer de table Aria. Le serveur s'appuie en effet sur Aria pour ses **tables temporaires internes** (celles que MariaDB matérialise sur disque pour certaines requêtes) et pour la plupart de ses **tables système**. Bien dimensionner et comprendre Aria a donc un intérêt général, et pas seulement pour les rares cas où l'on choisit explicitement `ENGINE = Aria`.

## Architecture et stockage

Une table Aria est stockée dans **trois fichiers**, sur le modèle de MyISAM :

- `.frm` — la **définition** de la table ;
- `.MAD` — les **données** (*MAria Data*) ;
- `.MAI` — les **index** (*MAria Index*).

Aria « prend en charge tous les aspects de MyISAM, sauf exceptions », ce qui facilite la migration depuis MyISAM — il en hérite par exemple le **comptage instantané** (`SELECT COUNT(*)` sans `WHERE`), le nombre de lignes étant conservé dans les métadonnées de la table. Après un arrêt propre, les fichiers Aria peuvent être copiés d'un serveur à l'autre. L'outil hors-ligne `aria_chk` opère sur les fichiers `.MAI` pour vérifier et réparer les tables, et `aria_pack` produit des tables compressées **en lecture seule** (généralement 40 à 70 % plus compactes).

## Le page cache : données *et* index en mémoire

C'est l'une des différences les plus importantes avec MyISAM. Là où MyISAM dispose d'un *key cache* qui ne met en cache que les **blocs d'index**, Aria utilise un **page cache** (dimensionné par `aria_pagecache_buffer_size`, 128 Mo par défaut) qui, avec le format PAGE, met en cache **les pages de données *et* d'index**. Sur ce plan, le page cache d'Aria joue un rôle comparable au **buffer pool d'InnoDB**.

Cette mise en cache plus complète a un léger coût en écriture (Aria écrit et journalise davantage que MyISAM), compensé par la sécurité et de meilleures performances de lecture sur les systèmes où le cache de données du système d'exploitation est insuffisant. La taille de bloc/page se règle via `aria_block_size` (8 Ko par défaut) ; la modifier impose toutefois de recréer toutes les tables Aria. On surveille l'efficacité du cache en comparant `aria_pagecache_reads` (lectures physiques) à `aria_pagecache_read_requests` (lectures logiques), exactement comme pour le buffer pool d'InnoDB.

> 🆕 Depuis la série 12.x, ce page cache peut être **découpé en segments** (`aria_pagecache_segments`) afin d'améliorer les performances en accès parallèle. Ce mécanisme — le *segmented key cache* — fait l'objet de la sous-section **§7.4.1**.

## Les formats de ligne : PAGE, FIXED, DYNAMIC

Aria propose trois formats de ligne, choisis via l'option `ROW_FORMAT` :

- **PAGE** (par défaut) : les données sont organisées en pages. C'est le **seul format *crash-safe***, et celui qui apporte un gain notable sur les systèmes au cache de données médiocre.
- **FIXED** et **DYNAMIC** : identiques aux formats de MyISAM, présents surtout pour des raisons de **compatibilité**.

À noter que le format **COMPRESSED** de MyISAM n'existe pas dans Aria ; pour de la compression en lecture seule, on utilise l'outil `aria_pack`.

## La résistance aux pannes

C'est la raison d'être d'Aria et son avantage décisif sur MyISAM. Grâce à sa journalisation, Aria est **crash-safe** : en cas d'arrêt brutal, les modifications sont **ramenées à l'état du début de l'instruction** en cours (ou de la dernière instruction `LOCK TABLES`). Aria sait rejouer depuis son journal presque toutes les opérations — y compris des opérations de définition comme `CREATE`, `DROP`, `RENAME` et `TRUNCATE` —, à tel point qu'on peut sauvegarder une instance Aria en copiant simplement son journal.

Quelques opérations ne sont **pas encore rejouables** et appellent donc une vigilance particulière : les insertions en masse dans une table vide (`LOAD DATA INFILE`, `INSERT … SELECT`, `INSERT` multi-lignes) et `ALTER TABLE` ; par ailleurs, les fichiers `.frm` ne sont pas recréés par la relecture du journal. La fréquence des points de cohérence se règle via `aria_checkpoint_interval` (en secondes).

## Caractéristiques et atouts face à MyISAM

Au-delà de la résistance aux pannes, Aria apporte plusieurs améliorations :

- prise en charge des index **FULLTEXT**, des types **spatiaux (OpenGIS)** et des **colonnes virtuelles** ;
- **insertions concurrentes** par plusieurs sessions dans une même table ;
- une **longueur de clé maximale de 2300 octets** avec la page de 8 Ko par défaut, contre 1000 pour MyISAM (la limite, relevée à 2000 octets en 10.5 via MDEV-20279, a depuis été portée à 2300) ;
- les outils dédiés `aria_chk` (vérification/réparation) et `aria_pack` (compression en lecture seule).

En contrepartie, Aria ne prend pas en charge `INSERT DELAYED` et, historiquement, ne permettait pas de définir plusieurs caches d'index nommés comme MyISAM — limite que la segmentation du page cache (§7.4.1) vient précisément adresser sur l'axe du parallélisme.

## Configuration de base

```sql
-- Une table Aria en format crash-safe (PAGE est le format par défaut)
CREATE TABLE journal_acces (
  id    BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  ip    INET6        NOT NULL,
  url   VARCHAR(255) NOT NULL,
  vu_le DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE = Aria ROW_FORMAT = PAGE;
```

```ini
[mariadb]
aria_pagecache_buffer_size = 256M   # cache de pages Aria (données + index) ; défaut 128M
aria_block_size            = 8K     # taille de page ; la modifier impose de recréer les tables Aria
aria_checkpoint_interval   = 30     # intervalle des checkpoints, en secondes
```

Pour migrer une table MyISAM héritée vers Aria :

```sql
ALTER TABLE ma_table ENGINE = Aria;
```

Le détail de la conversion entre moteurs figure en §7.9.

## Quand utiliser Aria

Aria est le bon choix pour des **tables non transactionnelles mais que l'on souhaite résistantes aux pannes** : tables de référence ou de consultation majoritairement en lecture, données de travail temporaires, ou remplacement de tables MyISAM héritées. Pour tout besoin de **transactions, de clés étrangères ou de forte concurrence en écriture**, on se tourne vers **InnoDB** (§7.2) ; pour l'**analytique massive**, vers **ColumnStore** (§7.5). La grille de décision complète figure en §7.8.

## Ce que couvre la sous-section suivante

- **7.4.1 — [Segmented key cache (`aria_pagecache_segments`)](04.1-aria-segmented-key-cache.md)** 🆕 — comment segmenter le page cache d'Aria pour améliorer les performances en accès parallèle, une nouveauté de la série 12.x.

## Liens avec d'autres chapitres

- **MyISAM**, le prédécesseur d'Aria, est présenté en §7.3.
- **InnoDB**, l'alternative transactionnelle, en §7.2.
- La **segmentation du page cache** (nouveauté 12.x) est détaillée en §7.4.1.
- Les commandes **`CHECK TABLE` / `REPAIR TABLE`** sont traitées en §11.6, et le contexte du *key cache* MyISAM en §15.2.3.
- La **comparaison des moteurs** (§7.8) et la **conversion entre moteurs** (§7.9) complètent le tableau.

## Ce qu'il faut retenir

- Aria est le **successeur *crash-safe* de MyISAM**, propre à MariaDB : il en reprend les fonctionnalités tout en survivant aux arrêts brutaux, mais reste **non transactionnel** et **sans clés étrangères**.
- Tout utilisateur de MariaDB se sert d'Aria sans le savoir : **tables temporaires internes** et **tables système**.
- Son **page cache** (`aria_pagecache_buffer_size`) met en cache **données et index** (format PAGE), à la manière du buffer pool d'InnoDB — contrairement au key cache de MyISAM, limité aux index.
- Le format **PAGE** (par défaut) est le seul **crash-safe** ; FIXED et DYNAMIC ne servent que la compatibilité MyISAM.
- Aria apporte FULLTEXT, spatial, colonnes virtuelles, clés jusqu'à 2300 octets (page de 8 Ko) et les outils `aria_chk`/`aria_pack`.
- Pour du transactionnel, du référentiel ou de la forte concurrence en écriture, préférez **InnoDB** ; la segmentation du page cache (12.x) est traitée en **§7.4.1**.

⏭️ [Segmented key cache (aria_pagecache_segments)](/07-moteurs-de-stockage/04.1-aria-segmented-key-cache.md)
