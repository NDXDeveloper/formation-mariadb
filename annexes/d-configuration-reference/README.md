🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe D — Configuration de Référence par Cas d'Usage

> ⚙️ Des configurations `my.cnf` prêtes à adapter, selon le profil de charge, pour MariaDB 12.3 LTS.

Cette annexe rassemble des configurations de référence ajustées à différents profils de charge. Elle fournit un point de départ structuré pour chaque cas d'usage, accompagné du raisonnement qui sous-tend les choix — afin que l'on puisse les adapter en connaissance de cause.

## ⚠️ À lire avant tout : ce sont des points de départ

Ces configurations sont des **bases à adapter, pas des réglages universels à appliquer tels quels**. Les bonnes valeurs dépendent étroitement du matériel (quantité de RAM, nombre de cœurs, type de disque) et de la charge réelle ; le dimensionnement du buffer pool, en particulier, est proportionnel à la mémoire disponible. Aucune de ces configurations ne doit être appliquée en production sans tests et sans observation des métriques. Enfin, certains réglages de durabilité échangent de la sécurité des données contre du débit : il faut en comprendre les implications avant de les assouplir.

## Ce qui distingue les profils

D'un profil à l'autre, ce sont surtout quelques leviers qui varient. La **mémoire** d'abord : la part allouée au buffer pool InnoDB (`innodb_buffer_pool_size`), qui domine les charges transactionnelles. La **durabilité face au débit** ensuite : les réglages de synchronisation (`innodb_flush_log_at_trx_commit`, `sync_binlog`) arbitrent entre garantie de non-perte et vitesse d'écriture. Les **entrées/sorties** (`innodb_io_capacity`, `innodb_flush_method`), à accorder au support de stockage. La **concurrence** (thread pool, `max_connections`). Et enfin le **moteur** privilégié : InnoDB pour l'OLTP, ColumnStore pour l'OLAP.

## Cadre : MariaDB 12.3

Les configurations visent la 12.3 et n'emploient que des variables en vigueur. Elles évitent notamment celles retirées au cours de la série 12.x (`big_tables`, `large_page_size`, `storage_engine` — *voir Ch. 11.2.3*) ainsi que le Query Cache, déprécié (*voir Ch. 15.3*).

## Comment utiliser ces configurations

Les réglages se placent dans la section `[mysqld]` du fichier `my.cnf` (ou d'un fichier d'inclusion). La plupart nécessitent un redémarrage du serveur ; certaines variables sont modifiables à chaud via `SET GLOBAL`. La structure des fichiers de configuration et leur ordre de lecture sont détaillés au [Ch. 11.1 — Fichiers de configuration](../../11-administration-configuration/README.md).

## Contenu de l'annexe

### D.1 — [OLTP (High concurrency, low latency)](01-configuration-oltp.md)

Charges transactionnelles : de nombreuses transactions courtes et concurrentes, où prime la latence. L'accent porte sur un buffer pool généreux, des validations rapides et une bonne gestion de la concurrence.

### D.2 — [OLAP (Data warehouse, analytics)](02-configuration-olap.md)

Charges analytiques : quelques requêtes lourdes parcourant de grands volumes. L'accent porte sur les gros parcours, la mémoire dédiée aux tris et jointures, et le recours à ColumnStore.

### D.3 — [Mixed workload](03-configuration-mixed-workload.md)

Les deux à la fois : un compromis assumé entre la réactivité attendue de l'OLTP et le débit attendu de l'OLAP, lorsque le même serveur doit servir les deux.

### D.4 — [Développement local](04-configuration-developpement-local.md)

Un poste de travail : ressources modestes, où l'on privilégie la commodité et l'empreinte réduite plutôt que la performance ou la durabilité maximale.

## Pour aller plus loin

Ces configurations s'appuient sur des notions développées ailleurs dans la formation.

| Pour approfondir… | Voir |
|-------------------|------|
| Structure et lecture des fichiers de configuration | [Ch. 11.1 — Fichiers de configuration](../../11-administration-configuration/README.md) |
| Variables système et leur portée | [Ch. 11.2 — Variables système et de session](../../11-administration-configuration/README.md) |
| Dimensionnement mémoire et tuning | [Ch. 15 — Performance et Tuning](../../15-performance-tuning/README.md) |
| Distinction OLTP / OLAP | [Ch. 20.1 — OLTP vs OLAP](../../20-cas-usage-architectures/README.md) |
| Moteur ColumnStore (OLAP) | [Ch. 7.5 — ColumnStore](../../07-moteurs-de-stockage/README.md) |
| Audit de configuration | [Annexe E — Checklist de Performance](../e-checklist-performance/README.md) |

## Conventions

Les valeurs dépendant du matériel apparaissent comme des paramètres à substituer, par exemple `<RAM_TOTALE>` ou un chemin de fichier ; les noms de variables et les commandes sont notés en `police à chasse fixe`. Sauf mention contraire, les valeurs proposées sont des bases à affiner par le test et la mesure.

---

⬅️ [Annexe C — Requêtes SQL de Référence](../c-requetes-sql-reference/README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [D.1 — OLTP (High concurrency, low latency)](01-configuration-oltp.md)

⏭️ [OLTP (High concurrency, low latency)](/annexes/d-configuration-reference/01-configuration-oltp.md)
