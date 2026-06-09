🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.6 Moteur S3 : Archivage de données froides sur AWS S3/MinIO

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS

## Archiver sur du stockage objet, sans perdre l'accès SQL

Le moteur **S3** répond à un besoin précis : **archiver des données froides** sur du **stockage objet** (Amazon S3, MinIO, ou tout service compatible avec l'API S3) tout en les gardant **interrogeables en SQL**. C'est un moteur **en lecture seule** : on y dépose des tables devenues peu actives mais que l'on ne peut pas supprimer, afin de **libérer le stockage local** (souvent coûteux) au profit d'un stockage objet bon marché et durable, sans renoncer à pouvoir les consulter.

Le terme « S3 » désigne ici l'**API de stockage objet** définie par Amazon AWS. Le moteur fonctionne donc avec le S3 d'Amazon comme avec n'importe quel service tiers qui implémente cette API — au premier rang desquels **MinIO**, une solution S3-compatible que l'on peut héberger soi-même.

Techniquement, le moteur S3 est **fondé sur le code d'Aria** (§7.4) : il en hérite et détourne le chemin de lecture pour aller chercher les données **sur S3** plutôt que sur le disque local.

## Lecture seule : un moteur d'archivage

C'est la caractéristique fondamentale : une table S3 est **immuable côté données**. Aucun `INSERT`, `UPDATE` ou `DELETE` n'est possible. En revanche :

- les `SELECT` fonctionnent normalement ;
- les **changements de structure** (`ALTER TABLE`) restent autorisés ;
- `DROP TABLE` supprime également les données **sur S3**.

Comme Aria et MyISAM, le moteur S3 conserve le nombre de lignes en métadonnées, ce qui rend `SELECT COUNT(*)` rapide. À l'inverse, la **latence de lecture** depuis S3 est plus élevée que depuis un disque local : le moteur convient à des requêtes **occasionnelles** sur des données d'archive, pas à des données « chaudes ».

## Le flux d'archivage : `ALTER TABLE … ENGINE=S3`

Archiver une table consiste à convertir une table locale (par exemple InnoDB ou Aria) vers le moteur S3 :

```sql
-- Archiver une table devenue inactive vers S3 (lecture seule ensuite)
ALTER TABLE commandes_2019 ENGINE = S3;

-- La requêter normalement, malgré son stockage distant
SELECT COUNT(*) FROM commandes_2019;

-- La ramener en local si l'on doit de nouveau la modifier
ALTER TABLE commandes_2019 ENGINE = InnoDB;
```

Après la conversion vers S3, **seul le fichier `.frm`** (la définition de la table) **reste en local** ; les données et l'index partent **sur S3** (où ils sont d'ailleurs stockés séparément). Pour des opérations plus manuelles, l'outil **`aria_s3_copy`** permet de copier des tables Aria **vers** ou **depuis** S3 (`--op=to` / `--op=from`), avec compression possible :

```bash
# Copier une table Aria vers S3 (compressée)
aria_s3_copy --op=to --database=entrepot --compress commandes_2019
```

## Activation et configuration

Le moteur S3 est un **plugin** (`ha_s3`) qui **n'est pas chargé par défaut** ; il a longtemps été de maturité **alpha** (expérimentale), aussi convient-il de **vérifier son niveau de maturité pour votre version** avant tout usage en production. On l'active et on le configure via la variable `s3` et la famille de variables `s3-*`.

Configuration pour **Amazon S3** :

```ini
[mariadb]
plugin-load-add = ha_s3
s3             = ON
s3-bucket      = mariadb-archive
s3-access-key  = AKIA…            # identifiants IAM RESTREINTS de préférence
s3-secret-key  = ……              # stockés en clair : limiter strictement les droits
s3-region      = eu-north-1
s3-host-name   = s3.amazonaws.com
# Primaire et réplica partageant les mêmes tables S3 :
s3-slave-ignore-updates = 1

[aria_s3_copy]
s3-bucket     = mariadb-archive
s3-access-key = AKIA…
s3-secret-key = ……
s3-region     = eu-north-1
s3-host-name  = s3.amazonaws.com
```

Pour un serveur **MinIO** auto-hébergé, on pointe vers son hôte et son port, en HTTP :

```ini
[mariadb]
plugin-load-add = ha_s3
s3            = ON
s3-bucket     = storage-engine
s3-access-key = minio
s3-secret-key = minioadmin
s3-host-name  = 127.0.0.1
s3-port       = 9000
s3-use-http   = ON
```

Quelques points de configuration utiles :

- la section `[aria_s3_copy]` reprend les mêmes paramètres pour que l'outil en ligne de commande accède au même bucket ;
- `s3-block-size` vaut **4 Mo** par défaut, ce qui convient à la plupart des charges ; il n'est généralement pas recommandé de l'augmenter ;
- le moteur dispose, comme Aria, d'un **page cache** dédié (variable `s3_pagecache_buffer_size`) ;
- selon le fournisseur, des options comme `s3-protocol-version` peuvent être nécessaires ;
- **sécurité** : les identifiants sont stockés **en clair** dans la configuration ; utilisez des identifiants IAM aux droits limités (idéalement en lecture seule sur le bucket d'archive).

## Partage des tables entre serveurs

Parce qu'elles sont en lecture seule, les **mêmes tables S3 peuvent être lues par plusieurs serveurs** MariaDB simultanément. Dans un contexte de réplication, on indique au réplica de ne pas tenter de réappliquer les modifications sur ces tables partagées via `s3-slave-ignore-updates=1`. On peut ainsi exposer un **socle d'archives commun** à toute une flotte de serveurs sans dupliquer le stockage local.

## Quand utiliser le moteur S3 — et quand non

Le moteur S3 est pertinent pour :

- l'**archivage de données historiques** (anciennes commandes, journaux, données réglementaires) qui doivent rester **consultables** mais changent rarement ;
- la **réduction des coûts** de stockage en déportant ces données vers du stockage objet ;
- le **partage** d'un même jeu d'archives entre plusieurs serveurs.

En revanche, il ne convient pas :

- aux données **chaudes ou transactionnelles** (mises à jour fréquentes, forte concurrence) → **InnoDB** (§7.2) ;
- à l'**analytique intensive** sur de très grands volumes, où l'on recherche le maximum de performance → **ColumnStore** (§7.5) ;
- à toute donnée nécessitant des **écritures**, puisque le moteur est en lecture seule.

La grille de décision complète figure en §7.8.

## Liens avec d'autres chapitres

- Le moteur **Aria**, dont S3 hérite, est présenté en §7.4.
- L'alternative analytique **ColumnStore** est traitée en §7.5, et le moteur transactionnel **InnoDB** en §7.2.
- La **sauvegarde cloud-native** (S3 et stockage objet) est abordée en §12.8.1.
- La **comparaison des moteurs** (§7.8) et la **conversion entre moteurs** (§7.9) complètent le tableau.

## Ce qu'il faut retenir

- Le moteur **S3** archive des **tables froides** sur du **stockage objet S3-compatible** (AWS S3, MinIO…), en **lecture seule**, tout en les laissant interrogeables en SQL.
- Il est **fondé sur Aria** et s'active comme **plugin** (`ha_s3`, non chargé par défaut, maturité historiquement alpha — à vérifier).
- On archive avec **`ALTER TABLE … ENGINE=S3`** (seul le `.frm` reste local) ; l'outil **`aria_s3_copy`** gère les transferts manuels.
- La configuration repose sur les variables **`s3-*`** : `s3-bucket`, identifiants, `s3-region`, `s3-host-name` (avec `s3-port`/`s3-use-http` pour MinIO) ; les identifiants étant en clair, **restreignez les droits IAM**.
- Les **mêmes tables S3 sont partageables** entre serveurs (`s3-slave-ignore-updates=1`).
- À réserver à l'**archivage froid** : pour le transactionnel, **InnoDB** ; pour l'analytique, **ColumnStore**.

⏭️ [Vector/HNSW : recherche vectorielle pour l'IA](/07-moteurs-de-stockage/07-moteur-vector-hnsw.md)
