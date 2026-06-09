🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe I — Changelog de la Formation

> Édition courante : **v2.0** (juin 2026) — alignée sur MariaDB 12.3 LTS.

Cette annexe retrace l'évolution de la formation au fil de ses éditions : changement de version cible, ajouts de contenu, réorganisations. Elle permet de savoir ce qui a changé d'une édition à l'autre et sur quelle version de MariaDB le contenu est aligné. À ne pas confondre avec l'historique des versions de MariaDB lui-même (voir l'[Annexe G](../g-versions-reference/README.md)) : il s'agit ici du journal des modifications de la *formation*. Les entrées sont présentées de la plus récente à la plus ancienne.

La convention du marqueur 🆕 employé dans tout le support est fixée par l'entrée la plus récente : il désigne actuellement les nouveautés de la série 12.x, apparues depuis la 11.8.

**Juin 2026 (v2.0) — Bascule vers MariaDB 12.3 LTS :**
- **Version de référence : 12.3 LTS** (au lieu de 11.8). 11.8 devient la « LTS précédente » de référence pour la migration.
- Inversion du positionnement des versions (note de version, Annexe G, roadmap §1.7 réorientée vers la série 13.x).
- Réattribution des marqueurs 🆕 : ils désignent désormais les **nouveautés 12.x** (depuis 11.8). Les anciennes features 11.8 (PARSEC, TLS zéro-config, utf8mb4/UCA14, TIMESTAMP 2106, privilèges granulaires, espace temporaire, BACKUP STAGE, Online Schema Change, Application Time Period, MariaDB Vector, MCP Server…) deviennent du contenu standard.
- Ajout des features phares 12.x : **binlog réécrit intégré à InnoDB (~4× en écriture, §11.5.4)** et **moteur d'Optimizer Hints (§15.15)**.
- Compatibilité Oracle/MySQL : caching_sha2_password (§10.5.5), TO_DATE/TO_NUMBER/TRUNC (§3.7.1), ( + ) outer join (§3.3.5), tableaux associatifs (§8.1.4), SYS_REFCURSOR et curseurs sur prepared statements (§8.5), type XML `XMLTYPE` (§2.2.6).
- SQL standard : IS JSON (§4.7.4), SET PATH (§19.2.1), UPDATE/DELETE depuis CTE (§4.4.1).
- Triggers multi-événements (§8.3.4) ; clarification de l'isolation par instantané (`innodb_snapshot_isolation` ON par défaut **depuis la 11.6.2**, donc déjà en 11.8 — *pas* un changement 11.8 → 12.3, §6.9) ; contraintes FK uniques par table (§18.12).
- Sécurité : SET SESSION AUTHORIZATION (§10.12), clés SSL avec passphrase (§10.7.4), audit bufferisé (§10.8.3), SHA2 pour file_key_management (§18.7.2).
- Galera : packaging séparé mariadb-server-galera (§14.2.5, §16.x), réplication parallèle inter-clusters (§13.11), retry des write sets (§14.2.6).
- Optimiseur : optimisations sur scans inversés (§5.8.1), nouveaux algorithmes PARTITION BY KEY (§15.9.3).
- Migration dédiée 11.8 → 12.3 (§19.10) et Annexe F reconstruite pour 12.3.
- Variables retirées : big_tables, large_page_size, storage_engine (§11.2.3).

**Novembre 2025 :**
- Ajout MariaDB 11.8 LTS comme version de référence principale
- Intégration complète de MariaDB Vector (18.10, 20.9-20.11)
- Nouveau parcours IA/ML
- Mise à jour politique de versions (3 ans LTS, roadmap 12.x)
- Ajout fonctionnalités JSON avancées (4.8, 4.9)
- Nouveautés sécurité (PARSEC, TLS par défaut, privilèges granulaires)
- Ajout charset utf8mb4 par défaut et extension TIMESTAMP 2106
- Mise à jour MaxScale 25.01 avec Workload Capture/Replay
- Ajout MCP Server et intégrations frameworks IA

---

Le détail des nouveautés introduites par la version cible 12.3 figure en [Annexe F](../f-nouveautes-12-3/README.md), et le positionnement des versions de MariaDB en [Annexe G](../g-versions-reference/README.md).

⏭️ Retour au [Sommaire](/SOMMAIRE.md)
