🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C — Requêtes SQL de Référence

> 🧰 Une boîte à outils de requêtes prêtes à copier pour administrer, surveiller et analyser MariaDB 12.3 LTS.

Cette annexe rassemble un jeu de requêtes SQL prêtes à l'emploi pour les tâches courantes d'administration, de supervision et d'analyse. Elle se consulte comme une boîte à outils : on y choisit la requête qui répond au besoin, on adapte les noms d'objets, puis on l'exécute. Toutes sont en lecture seule : elles observent l'état du serveur sans le modifier.

## Que vont interroger ces requêtes ?

L'essentiel de ces requêtes ne porte pas sur vos tables métier, mais sur les **schémas système** qui exposent le fonctionnement interne du serveur. On y trouve principalement `INFORMATION_SCHEMA` (métadonnées normalisées, toujours disponible), `PERFORMANCE_SCHEMA` (instrumentation détaillée de l'exécution, qui doit parfois être activée via `performance_schema=ON`), le *sys schema* (vues plus lisibles bâties au-dessus des deux précédents) et les tables système de la base `mysql`. Ces schémas sont présentés au [Ch. 9.7 — Vues système](../../09-vues-et-donnees-virtuelles/README.md).

## Complément de l'annexe B

L'annexe B fournit des commandes client donnant un instantané immédiat (`SHOW PROCESSLIST`, `SHOW ENGINE INNODB STATUS`…). L'annexe C prend le relais lorsqu'il faut aller plus loin : interroger les mêmes données sous-jacentes avec des requêtes que l'on peut filtrer, trier, joindre et planifier. En pratique, on commence souvent par un `SHOW` (annexe B), puis on bascule sur une requête de cette annexe dès qu'on a besoin de précision.

## Prérequis et précautions

L'exécution suppose des droits adaptés : le privilège `PROCESS` pour voir l'ensemble des sessions, le droit `SELECT` sur les schémas système, et `PERFORMANCE_SCHEMA` activé pour certaines requêtes de supervision. Par ailleurs, si ces requêtes ne modifient rien, quelques-unes — notamment les parcours volumineux d'`INFORMATION_SCHEMA` sur des serveurs comptant beaucoup d'objets — peuvent être coûteuses : à employer avec discernement en production.

## Contenu de l'annexe

### C.1 — [Requêtes d'administration](01-requetes-administration.md)

Le quotidien opérationnel : identifier qui est connecté et qui bloque qui, repérer les verrous en cours, et mesurer l'empreinte disque des tables et des index (*table sizes*). Ce sont les requêtes que l'on dégaine pendant un incident.

### C.2 — [Requêtes de monitoring](02-requetes-monitoring.md)

La santé continue du serveur : efficacité du buffer pool, requêtes lentes ou trop gourmandes, et état de la réplication (retard, position). Le socle d'un suivi régulier.

### C.3 — [Requêtes d'analyse](03-requetes-analyse.md)

L'analyse de fond pour guider l'optimisation : statistiques sur les tables et les colonnes, et usage des index (index réellement sollicités, index inutilisés). De quoi étayer les décisions d'indexation et de tuning.

## Pour aller plus loin

Ces requêtes sont des points d'entrée vers les chapitres qui développent chaque domaine.

| Pour approfondir… | Voir |
|-------------------|------|
| Schémas et vues système (`INFORMATION_SCHEMA`, `PERFORMANCE_SCHEMA`) | [Ch. 9.7 — Vues système](../../09-vues-et-donnees-virtuelles/README.md) |
| Verrous, deadlocks et transactions | [Ch. 6 — Transactions et Concurrence](../../06-transactions-et-concurrence/README.md) |
| Administration, métriques et monitoring | [Ch. 11 — Administration et Configuration](../../11-administration-configuration/README.md) |
| Analyse des requêtes lentes et tuning | [Ch. 15 — Performance et Tuning](../../15-performance-tuning/README.md) |
| Surveillance et troubleshooting de la réplication | [Ch. 13.7 — Réplication](../../13-replication/README.md) |
| Stratégies d'indexation | [Ch. 5 — Index et Performance](../../05-index-et-performance/README.md) |
| Commandes client équivalentes (instantané rapide) | [Annexe B — Commandes mariadb CLI](../b-commandes-cli/README.md) |

## Conventions

Les paramètres à remplacer apparaissent entre chevrons, par exemple `<base>` ou `<table>` ; les noms d'objets et de colonnes sont notés en `police à chasse fixe`. Sauf mention contraire, les requêtes sont en lecture seule et conçues pour MariaDB 12.3 ; il convient d'adapter les noms à votre environnement.

---

⬅️ [Annexe B — Commandes mariadb CLI Essentielles](../b-commandes-cli/README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [C.1 — Requêtes d'administration](01-requetes-administration.md)

⏭️ [Requêtes d'administration (locks, processlist, table sizes)](/annexes/c-requetes-sql-reference/01-requetes-administration.md)
