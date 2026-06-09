🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe F — Nouveautés MariaDB 12.3 LTS en un Coup d'Œil 🆕

> Synthèse transversale des nouveautés de la série 12.x, consolidées dans la 12.3 LTS.

Cette annexe rassemble, sous une forme condensée, les principales nouveautés apparues au fil de la série *rolling* 12.0 → 12.2 et stabilisées dans **MariaDB 12.3 LTS**. Dans le corps de la formation, ces fonctionnalités sont signalées par le marqueur 🆕 et détaillées dans leur contexte (moteur de stockage, sécurité, réplication, optimiseur…). Elles sont ici regroupées pour offrir une lecture transversale : évaluer rapidement l'apport de la version, préparer une montée de version depuis la 11.8, ou simplement disposer d'une vue d'ensemble avant d'entrer dans le détail des chapitres.

L'annexe ne remplace pas les sections dédiées : chaque nouveauté y est présentée brièvement, accompagnée d'un renvoi vers le chapitre qui l'approfondit.

## Positionnement de la version

MariaDB 12.3 est la dernière version **LTS** (*Long Term Support*) à la date de cette formation. Sa version GA est la **12.3.2**, publiée fin mai 2026 (la 12.3.1 ayant servi de *release candidate*), et elle est supportée jusqu'en **juin 2029**. Elle consolide en un socle stable les fonctionnalités introduites progressivement par les versions *rolling* 12.0, 12.1 et 12.2.

Elle succède à la **11.8 LTS** (juin 2025, supportée jusqu'en 2028), qui reste très largement déployée et qui sert, dans cette formation, de point de comparaison pour la migration. Le cycle suivant s'ouvre avec la série **13.x**, dont la première version (13.0) est en préversion puis *release candidate* à la date de cette formation — sa GA n'étant pas encore parue. Le positionnement complet des versions et leurs dates de support figurent en [Annexe G](../g-versions-reference/README.md).

## Grandes familles de nouveautés

Pour faciliter la lecture, les nouveautés de la 12.x peuvent se regrouper en quelques grands axes. Le détail exhaustif, version par version, est présenté dans la section [F.1](01-tableau-recapitulatif.md) ; les points ci-dessous n'en donnent qu'un aperçu représentatif.

- **Performance et architecture du moteur** — au premier rang, le *binary log* réécrit et intégré à InnoDB (suppression de la synchronisation redondante, gain d'environ 4× en écriture, §11.5.4) et le nouveau moteur d'**Optimizer Hints** (§15.15). S'y ajoutent les optimisations sur scans inversés — *Rowid Filtering*, *Index Condition Pushdown*, *Loose Index Scan* sur clés `DESC` (§5.8.1) — et les optimisations de la recherche vectorielle (§15.16, §18.10.7).
- **Compatibilité Oracle** — fonctions de conversion `TO_DATE`, `TRUNC`, `TO_NUMBER` (§3.7.1, qui complètent `TO_CHAR`, présente depuis la 10.6), syntaxe `( + )` pour les jointures externes (§3.3.5), tableaux associatifs `TABLE OF … INDEX BY` (§8.1.4), `SYS_REFCURSOR` et curseurs sur *prepared statements* (§8.5), type `XML` (§2.2.6) et `SET PATH` (§19.2.1).
- **Compatibilité MySQL** — prise en charge de `caching_sha2_password` pour l'authentification (§10.5.5) et alignement sur de nouvelles fonctions GIS de MySQL 8 (§19.1.1).
- **Standard SQL** — prédicat `IS JSON` et levée de la limite de profondeur (§4.7.4), `UPDATE` / `DELETE` lisant une CTE (§4.4.1).
- **Sécurité** — `SET SESSION AUTHORIZATION` (§10.12), clés SSL protégées par *passphrase* (§10.7.4), journal d'audit bufferisé (§10.8.3) et SHA2 pour `file_key_management` (§18.7.2).
- **Galera et haute disponibilité** — *packaging* séparé du paquet `mariadb-server-galera` (§14.2.5), réplication parallèle entre clusters (§13.11) et nouveau mécanisme de *retry* des *write sets* (§14.2.6).
- **Changements de comportement** — à surveiller particulièrement lors d'une migration : noms de contraintes de clés étrangères désormais uniques par table et non plus par base (§18.12) ; variables système retirées — `big_tables`, `large_page_size`, `storage_engine` (§11.2.3). À noter : `innodb_snapshot_isolation` (activé par défaut depuis la 11.6.2, donc déjà en 11.8) modifie le niveau `REPEATABLE READ`, mais ne constitue *pas* un changement propre au passage 11.8 → 12.3 (§6.9).

## Plan de l'annexe

- **F.1 — [Tableau récapitulatif des features majeures (12.0 → 12.3)](01-tableau-recapitulatif.md)** : panorama complet des nouveautés, classées par thème et reliées à leur chapitre d'origine.
- **F.2 — [Les features phares](02-features-phares.md)** : zoom sur les deux apports les plus structurants — le binlog réécrit (~4× en écriture) et le moteur d'Optimizer Hints.
- **F.3 — [Compatibilité renforcée Oracle & MySQL](03-compatibilite-oracle-mysql.md)** : synthèse des fonctions et syntaxes ajoutées pour faciliter les migrations depuis ces deux SGBD.
- **F.4 — [Impact sur migration et compatibilité](04-impact-migration-compatibilite.md)** : récapitulatif des changements de comportement et de leurs conséquences pratiques.
- **F.5 — [Recommandations d'adoption](05-recommandations-adoption.md)** : critères pour décider d'adopter la 12.3, et selon quel calendrier.

## Comment lire cette annexe

Le marqueur 🆕 employé dans toute la formation désigne les nouveautés de la série 12.x, c'est-à-dire celles apparues depuis la 11.8. Les fonctionnalités introduites en 11.8 (PARSEC, TLS zéro-configuration, `utf8mb4` avec collations UCA 14.0.0, extension `TIMESTAMP` jusqu'en 2106, privilèges granulaires, MariaDB Vector, MCP Server…) sont désormais considérées comme acquises et relèvent du contenu standard.

Pour la marche à suivre concrète lors d'une montée de version, voir la section dédiée [§19.10 — Migration 11.8 → 12.3](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md).

⏭️ [Tableau récapitulatif des features majeures (12.0 → 12.3)](/annexes/f-nouveautes-12-3/01-tableau-recapitulatif.md)
