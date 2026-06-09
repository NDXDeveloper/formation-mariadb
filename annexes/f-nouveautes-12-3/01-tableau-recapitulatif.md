🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.1 — Tableau récapitulatif des features majeures (12.0 → 12.3) 🆕

Ce tableau récapitule l'ensemble des nouveautés de la série *rolling* 12.x — apparues depuis la 11.8 et consolidées dans la **12.3 LTS**. Elles sont regroupées en sept axes fonctionnels, complétés par une vue transversale des changements de comportement à surveiller lors d'une migration. Pour la mise en contexte de la version, voir l'introduction de l'[Annexe F](README.md).

> **Note d'attribution** — La formation traite la série 12.x comme un tout stabilisé par la 12.3. L'attribution de chaque fonctionnalité à une version mineure précise (12.0, 12.1 ou 12.2) n'est pas détaillée ici ; la colonne *Section* renvoie au chapitre qui approfondit chaque point.

**Légende**

- **⭐** — *feature* phare, approfondie en [F.2](02-features-phares.md).
- **Section** — renvoi vers le chapitre détaillant la fonctionnalité.
- Toutes les entrées listées sont des nouveautés de la série 12.x (marqueur 🆕 dans la formation).

## Moteur, performance et optimiseur

| Fonctionnalité | En bref | Section |
|---|---|---|
| ⭐ Binlog intégré à InnoDB | Journal binaire réécrit et intégré à InnoDB ; suppression de la synchronisation redondante, gain d'environ 4× en écriture. | §11.5.4 |
| ⭐ Optimizer Hints | Moteur d'indices d'optimisation pour piloter finement le plan : blocs de requête (`QB_NAME`), hints de table/index, d'ordre de jointure, de sous-requête, et `MAX_EXECUTION_TIME`. | §15.15 |
| Optimisations sur scans inversés | *Rowid Filtering*, *Index Condition Pushdown* et *Loose Index Scan* sur clés `DESC`. | §5.8.1 |
| Algorithmes `PARTITION BY KEY` | Nouveaux algorithmes de hachage pour le partitionnement par clé. | §15.9.3 |
| Cache de clés segmenté (Aria) | Segmentation du cache de pages Aria (`aria_pagecache_segments`) pour réduire la contention. | §7.4.1 |
| Optimisations Vector | Calcul de distance par extrapolation et exécution au niveau du moteur de stockage. | §15.16, §18.10.7 |

## Compatibilité Oracle

| Fonctionnalité | En bref | Section |
|---|---|---|
| Fonctions de conversion (Oracle) | `TO_DATE`, `TRUNC`, `TO_NUMBER` ; complètent `TO_CHAR` (avec format `FM`), déjà présente de plus longue date (10.6). | §3.7.1 |
| Jointures externes `( + )` | Syntaxe `( + )` pour les jointures externes en mode Oracle. | §3.3.5 |
| Tableaux associatifs | `DECLARE TYPE … TABLE OF … INDEX BY`. | §8.1.4 |
| Type XML | Type `XML` basique (`XMLTYPE`). | §2.2.6 |
| `SET PATH` | Chemin de résolution des routines. | §19.2.1 |

## Compatibilité MySQL

| Fonctionnalité | En bref | Section |
|---|---|---|
| `caching_sha2_password` | Plugin d'authentification compatible avec MySQL 8. | §10.5.5 |
| Fonctions GIS MySQL 8 | Alignement sur de nouvelles fonctions géospatiales. | §19.1.1 |

## Standard SQL

| Fonctionnalité | En bref | Section |
|---|---|---|
| Prédicat `IS JSON` | Prédicat normalisé ; suppression de la limite de profondeur JSON. | §4.7.4 |
| `UPDATE` / `DELETE` depuis une CTE | Modification de données en s'appuyant sur une CTE. | §4.4.1 |

## Programmation côté serveur et curseurs

| Fonctionnalité | En bref | Section |
|---|---|---|
| Triggers multi-événements | Un même trigger pour `INSERT`, `UPDATE` et `DELETE`. | §8.3.4 |
| Curseurs sur *prepared statements* | Curseurs portant sur des instructions préparées. | §8.5.1 |
| `SYS_REFCURSOR` | `REF CURSOR` faible et variable `max_open_cursors` (compatibilité Oracle). | §8.5.2 |

## Sécurité

| Fonctionnalité | En bref | Section |
|---|---|---|
| `SET SESSION AUTHORIZATION` | Exécuter des actions au nom d'un autre utilisateur. | §10.12 |
| Clés SSL avec *passphrase* | Clés privées TLS protégées par *passphrase* (`ssl_passphrase`). | §10.7.4 |
| Audit bufferisé | Journal d'audit bufferisé (`server_audit_file_buffer_size`), cibles `HOST:PORT` et `tls_version`. | §10.8.3 |
| SHA2 pour `file_key_management` | Hachage SHA2 des clés et purge du cache Hashicorp. | §18.7.2 |

## Réplication, Galera et haute disponibilité

| Fonctionnalité | En bref | Section |
|---|---|---|
| Packaging Galera séparé | Paquet `mariadb-server-galera` distinct ; dépendance retirée du serveur. | §14.2.5 |
| *Retry* des *write sets* | Rejeu des *write sets* via `wsrep_applier_retry_count`. | §14.2.6 |
| Réplication parallèle inter-clusters | `slave_parallel_threads` appliqué entre clusters Galera. | §13.11 |
| Tables temporaires en réplication | Comportement prévisible via `create_tmp_table_binlog_formats`. | §13.12 |

## Changements de comportement (à surveiller en migration)

Vue transversale des évolutions susceptibles d'affecter une application existante lors du passage depuis la 11.8. Certaines entrées renvoient à des fonctionnalités déjà listées plus haut ; leur analyse détaillée figure en [F.4](04-impact-migration-compatibilite.md).

| Changement | Conséquence / point d'attention | Section |
|---|---|---|
| `innodb_snapshot_isolation = ON` | **Déjà actif par défaut en 11.8** (depuis la 11.6.2) — *pas* un changement propre au passage 11.8 → 12.3 ; ne concerne que les migrations depuis une version antérieure ou MySQL. Modifie le comportement de `REPEATABLE READ`. | §6.9 |
| Noms de contraintes FK | Uniques par table seulement (et non plus par base) : risque de collision lors d'imports. | §18.12 |
| Variables système retirées | `big_tables`, `large_page_size`, `storage_engine` supprimées : à retirer des fichiers de configuration. | §11.2.3 |
| Binlog intégré à InnoDB | Nouvelle architecture du journal binaire ; synchronisation redondante supprimée. | §11.5.4 |
| Packaging Galera | Installer le paquet `mariadb-server-galera` séparé (impacte images Docker et déploiements). | §14.2.5 |
| `DROP USER` avec sessions actives | Avertissement par défaut ; échec en mode Oracle. | §10.2.1 |

---

Les deux *features* phares (⭐) sont approfondies en [F.2](02-features-phares.md). La compatibilité Oracle et MySQL est synthétisée en [F.3](03-compatibilite-oracle-mysql.md), et les changements de comportement sont analysés en [F.4](04-impact-migration-compatibilite.md). Pour la procédure de montée de version, voir [§19.10 — Migration 11.8 → 12.3](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md).

⏭️ [Les features phares : binlog réécrit (~4× écriture) et Optimizer Hints](/annexes/f-nouveautes-12-3/02-features-phares.md)
