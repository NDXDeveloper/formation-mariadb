🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.10 Moteurs spécialisés

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS

## Des moteurs pour des besoins précis

Au-delà des moteurs généralistes (InnoDB, Aria) et des moteurs orientés analytique ou archivage (ColumnStore, S3), MariaDB embarque une poignée de **moteurs spécialisés**, chacun répondant à un besoin **étroit mais bien identifié** : conserver des données en mémoire vive, compresser au maximum des données figées, répartir une table sur plusieurs serveurs, ou accéder à des données stockées en dehors de MariaDB.

Ces moteurs ne sont presque jamais le **socle** d'une application — ce rôle revient à InnoDB (§7.8) — mais ils résolvent élégamment des problèmes précis lorsqu'on les emploie au bon endroit. Conformément à l'architecture enfichable (§7.1), on peut les utiliser **table par table**, à côté de tables InnoDB, en gardant à l'esprit les limites du mélange de moteurs.

## Panorama des quatre moteurs

Cette section couvre quatre moteurs aux usages très différents :

- **Memory** — des tables stockées **entièrement en mémoire vive** : accès extrêmement rapides, mais données **volatiles** (perdues au redémarrage). Pour des caches ou des données de travail temporaires.
- **Archive** — une **compression maximale** pour des données en **ajout seul** (`INSERT`/`SELECT`, sans `UPDATE` ni `DELETE`). Idéal pour des journaux ou des historiques rarement consultés mais à conserver.
- **Spider** — un moteur de **répartition** : il distribue (shard) les données d'une table sur **plusieurs serveurs** MariaDB/MySQL, présentés comme une seule table. Pour le passage à l'échelle horizontal.
- **CONNECT** — un moteur d'**accès à des données externes** : il rend interrogeables comme des tables MariaDB des **fichiers** (CSV, XML, JSON…) ou d'**autres bases de données**. Pour l'intégration et la fédération de données.

## Intégrés ou sous forme de plugin

Tous ne sont pas disponibles de la même façon (§7.1) :

- **Memory** est **intégré** au serveur : toujours disponible, sans installation ;
- **Archive** est livré comme **plugin** avec le serveur, mais pas toujours chargé par défaut (`INSTALL SONAME 'ha_archive'` selon la distribution — c'est le cas de l'image Docker officielle) ;
- **Spider** et **CONNECT** sont des **plugins**, à charger explicitement (`INSTALL SONAME` / `plugin-load-add`) avant usage.

## Plan de la section

- **7.10.1 — [Memory : Tables en RAM](10.1-memory.md)** — tables en mémoire vive, rapides mais volatiles.
- **7.10.2 — [Archive : Compression maximale](10.2-archive.md)** — archivage très compressé en ajout seul.
- **7.10.3 — [Spider : Sharding distribué](10.3-spider.md)** — répartition d'une table sur plusieurs serveurs.
- **7.10.4 — [CONNECT : Accès données externes](10.4-connect.md)** — accès SQL à des fichiers et bases externes.

## Quand s'y intéresser

La règle est simple : on n'adopte un moteur spécialisé que lorsque **sa force précise correspond à un besoin réel**, jamais par défaut. L'essentiel des données reste dans **InnoDB**. Pour le passage à l'échelle, Spider n'est d'ailleurs qu'une option parmi d'autres (sharding et distribution horizontale, §15.11) ; pour l'intégration de données externes, CONNECT complète des approches comme la capture de changements (§20.8). Les sous-sections qui suivent détaillent chacun de ces moteurs, en commençant par **Memory** (§7.10.1).

## Liens avec d'autres chapitres

- L'**architecture enfichable**, le **chargement des plugins** et le **mélange de moteurs** sont traités en §7.1.
- La **comparaison** de tous les moteurs et la décision figurent en §7.8, le moteur par défaut **InnoDB** en §7.2.
- Le **sharding et la distribution horizontale** (en lien avec Spider) sont abordés en §15.11, et les **architectures pilotées par les événements / l'intégration** (en lien avec CONNECT) au §20.8.

## Ce qu'il faut retenir

- Les **moteurs spécialisés** répondent à des besoins ciblés : **Memory** (RAM, volatile), **Archive** (compression en ajout seul), **Spider** (répartition multi-serveurs), **CONNECT** (données externes).
- **Memory** est **intégré** ; **Archive** (plugin fourni avec le serveur), **Spider** et **CONNECT** sont des **plugins** à charger avant usage.
- On les emploie **table par table**, pour un besoin précis, en gardant **InnoDB comme socle** (§7.8).

⏭️ [Memory : Tables en RAM](/07-moteurs-de-stockage/10.1-memory.md)
