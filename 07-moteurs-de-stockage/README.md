🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 7 — Moteurs de Stockage

> **Partie 4 : Moteurs de Stockage et Programmation Serveur (Avancé)**  
> Version de référence : **MariaDB 12.3 LTS**

## Introduction

Si une seule caractéristique devait résumer ce qui distingue MariaDB d'une grande partie des autres systèmes de gestion de bases de données, ce serait sans doute son architecture à **moteurs de stockage enfichables** (*pluggable storage engines*). Le moteur de stockage est la couche responsable de la façon dont les données sont réellement écrites sur le disque, lues, indexées, verrouillées et récupérées après incident. Dans la plupart des SGBD, ce mécanisme est unique et figé. Dans MariaDB, il devient un choix d'architecture : on sélectionne, **table par table**, le moteur le mieux adapté à la charge de travail visée.

Cette flexibilité a des conséquences très concrètes. Une même base peut héberger des tables transactionnelles fortement concurrentes, des tables analytiques balayées par de lourdes agrégations, des tables d'archives compressées, des données froides déportées sur du stockage objet, voire des vecteurs destinés à de la recherche sémantique pour l'IA — chacune servie par le moteur le plus pertinent. Comprendre les forces et les limites de chaque moteur est donc une compétence centrale pour tout développeur ou DBA souhaitant tirer le meilleur de MariaDB.

Ce chapitre dresse le panorama des principaux moteurs disponibles. Il part d'InnoDB — le moteur par défaut, transactionnel et robuste — puis parcourt les moteurs historiques (MyISAM, Aria), analytiques (ColumnStore), orientés cloud (S3) et IA (Vector/HNSW), avant de proposer une grille de décision et de présenter les moteurs spécialisés. Il se clôt sur la conversion d'une table d'un moteur vers un autre. Nous vous recommandons de suivre les sections dans l'ordre, InnoDB servant de référence pour les comparaisons ultérieures.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- expliquer le rôle d'un moteur de stockage et le principe de l'architecture enfichable de MariaDB ;
- décrire les caractéristiques d'InnoDB (ACID, clés étrangères, verrouillage au niveau ligne, gestion mémoire, journalisation) et le configurer ;
- situer MyISAM et son successeur Aria, et comprendre dans quels contextes ils interviennent encore ;
- identifier les moteurs orientés analytique (ColumnStore) et archivage cloud (S3), ainsi que la recherche vectorielle (Vector/HNSW), fonctionnalité intégrée plutôt que moteur distinct ;
- comparer les moteurs et choisir celui qui convient à un cas d'usage donné ;
- convertir une table d'un moteur à un autre en connaissant les précautions associées ;
- reconnaître les moteurs spécialisés (Memory, Archive, Spider, CONNECT) et leurs usages typiques.

## Prérequis

Ce chapitre fait partie d'un cycle avancé. Il suppose acquis :

- les fondamentaux de MariaDB et la création de tables (chapitres 1 et 2) ;
- les notions d'index et leur fonctionnement (chapitre 5), car le comportement des index diffère selon le moteur ;
- les concepts de transaction, d'ACID et de verrouillage (chapitre 6), particulièrement utiles pour comprendre InnoDB.

## Plan du chapitre

- **7.1 — [Vue d'ensemble : Architecture Pluggable Storage Engine](01-vue-ensemble-architecture.md)**
  Le principe de l'architecture enfichable et la façon dont MariaDB délègue le stockage à des moteurs interchangeables.
- **7.2 — [InnoDB : Le moteur par défaut](02-innodb.md)** — transactionnel, robuste, adapté à l'OLTP.
  - 7.2.1 — [Caractéristiques : ACID, FK, Row-level locking](02.1-innodb-caracteristiques.md)
  - 7.2.2 — [Buffer Pool et gestion mémoire](02.2-innodb-buffer-pool.md)
  - 7.2.3 — [Redo Log et Undo Log](02.3-innodb-redo-undo-log.md)
  - 7.2.4 — [Configuration avancée](02.4-innodb-configuration.md)
- **7.3 — [MyISAM : Moteur legacy](03-myisam.md)** — historique, sans transactions, verrouillage au niveau table.
- **7.4 — [Aria : Le successeur de MyISAM](04-aria.md)** — moteur *crash-safe*, notamment utilisé pour les tables système et les tables temporaires internes.
  - 7.4.1 — [Segmented key cache (`aria_pagecache_segments`)](04.1-aria-segmented-key-cache.md) 🆕
- **7.5 — [ColumnStore : Analytique et OLAP](05-columnstore.md)** — stockage en colonnes pour l'entrepôt de données et les agrégations massives.
- **7.6 — [Moteur S3 : Archivage de données froides](06-moteur-s3.md)** — tables en lecture seule déportées sur stockage objet (AWS S3, MinIO).
- **7.7 — [Vector/HNSW : recherche vectorielle pour l'IA](07-moteur-vector-hnsw.md)** — le type `VECTOR` et l'index HNSW pour la recherche sémantique et le RAG (une fonctionnalité intégrée à InnoDB, **non un moteur distinct**).
- **7.8 — [Comparaison et choix du moteur approprié](08-comparaison-choix-moteur.md)** — grille de décision selon la charge de travail.
- **7.9 — [Conversion entre moteurs (`ALTER TABLE … ENGINE`)](09-conversion-entre-moteurs.md)** — migrer une table d'un moteur à un autre et précautions associées.
- **7.10 — [Moteurs spécialisés](10-moteurs-specialises.md)**
  - 7.10.1 — [Memory : Tables en RAM](10.1-memory.md)
  - 7.10.2 — [Archive : Compression maximale](10.2-archive.md)
  - 7.10.3 — [Spider : Sharding distribué](10.3-spider.md)
  - 7.10.4 — [CONNECT : Accès à des données externes](10.4-connect.md)

## À noter pour MariaDB 12.3 LTS 🆕

Dans ce chapitre, la nouveauté de la série 12.x à signaler concerne **Aria** : le *segmented key cache* (`aria_pagecache_segments`), détaillé en §7.4.1, qui segmente le page cache d'Aria afin de réduire la contention sous forte concurrence. Les autres moteurs présentés (InnoDB, ColumnStore, S3…) — ainsi que la recherche vectorielle Vector/HNSW (fonctionnalité intégrée, non un moteur) — constituent désormais du contenu standard, plusieurs d'entre eux ayant été introduits ou stabilisés au fil des versions précédentes (jusqu'à la 11.8 LTS).

## Liens avec d'autres chapitres

Le choix et le réglage d'un moteur de stockage débordent du cadre de ce chapitre :

- le **tuning d'InnoDB** (buffer pool, I/O, partitionnement) est approfondi au chapitre 15 (*Performance et Tuning*) ;
- la **recherche vectorielle** et ses cas d'usage IA/RAG sont développés en §18.10 et au chapitre 20 ;
- le **data warehousing avec ColumnStore** est traité en §20.3 ;
- les notions d'**ACID et d'isolation** qui sous-tendent InnoDB relèvent du chapitre 6.

⏭️ [Vue d'ensemble : Architecture Pluggable Storage Engine](/07-moteurs-de-stockage/01-vue-ensemble-architecture.md)
