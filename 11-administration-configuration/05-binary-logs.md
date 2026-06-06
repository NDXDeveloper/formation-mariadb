🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 — Binary logs et logs de transactions

Le journal binaire (*binary log*, ou *binlog*) est l'un des mécanismes les plus importants de MariaDB : il constitue le socle de la réplication et de la récupération à un instant donné (*point-in-time recovery*, PITR). À la différence des journaux vus jusqu'ici — d'erreurs, lent, général —, qui servent au diagnostic et à la supervision, le journal binaire est un journal *fonctionnel* : il enregistre les modifications de données elles-mêmes, dans le but de les rejouer ailleurs ou plus tard.

Cette section introduit le journal binaire, le situe par rapport aux journaux internes de transactions d'InnoDB, et présente la grande évolution apportée par MariaDB 12.3 : son intégration directe au moteur InnoDB. Les sous-sections détailleront ensuite sa configuration, ses formats, sa purge, puis cette nouvelle implémentation.

## Un journal logique des changements

Le journal binaire enregistre tous les événements qui *modifient les données* : les instructions de manipulation (`INSERT`, `UPDATE`, `DELETE`) qui changent effectivement des lignes, ainsi que les instructions de définition (`CREATE`, `ALTER`, `DROP`). Les requêtes de pure lecture comme `SELECT` n'y figurent pas, puisqu'elles ne modifient rien.

Deux propriétés sont essentielles. D'abord, l'enregistrement a lieu **au moment de la validation** (`COMMIT`) et dans l'ordre des commits — contrairement au journal général, qui consigne chaque requête dès sa réception (§11.4.3). Une transaction annulée n'apparaît donc jamais dans le journal binaire. Ensuite, c'est un journal *logique* : il décrit *ce qui a changé* (au niveau des instructions ou des lignes), et non l'état physique des pages sur le disque.

## Deux usages fondamentaux : réplication et PITR

L'intérêt du journal binaire tient à ses deux usages principaux. Pour la **réplication**, le serveur primaire écrit ses modifications dans son journal binaire ; les réplicas les lisent et les rejouent pour rester synchronisés. C'est le fondement de toute la réplication standard de MariaDB (chapitre 13). Pour la **récupération à un instant donné** (PITR), on restaure une sauvegarde puis on rejoue le journal binaire jusqu'à un moment précis — par exemple juste avant une erreur humaine — afin de revenir à un état cohérent (§12.4 et §12.5.2).

Le journal binaire alimente également les architectures orientées événements et la capture de changements (*Change Data Capture*) avec des outils comme Debezium et Kafka (§20.8), et fournit accessoirement une trace des modifications de données.

## Journal binaire et journaux de transactions InnoDB

Le titre de cette section distingue deux familles de journaux qu'il ne faut pas confondre. Le journal binaire opère au **niveau du serveur**, tous moteurs confondus, dans une optique de réplication et de sauvegarde. InnoDB possède en parallèle ses propres journaux, **internes au moteur**, qui poursuivent d'autres objectifs.

| Journal | Niveau | Nature | Rôle principal | Détaillé dans |
|---------|--------|--------|----------------|---------------|
| Journal binaire | Serveur (tous moteurs) | Logique (événements) | Réplication et PITR | Ce chapitre (§11.5) |
| Redo log (InnoDB) | Moteur InnoDB | Physique (pages) | Durabilité et récupération après crash | §7.2.3 |
| Undo log (InnoDB) | Moteur InnoDB | Versions de lignes | MVCC et `ROLLBACK` | §7.2.3, §6.6 |

Le *redo log* garantit la durabilité : après un crash, InnoDB rejoue les changements validés mais pas encore écrits dans les fichiers de données. C'est un journal circulaire, de taille fixe, dont la fréquence de synchronisation est réglée par `innodb_flush_log_at_trx_commit` (valeur 1 par défaut). L'*undo log*, lui, conserve les versions précédentes des lignes pour servir le MVCC et permettre l'annulation des transactions (§6.6).

Historiquement, ces deux mondes — le journal binaire d'un côté, le *redo log* d'InnoDB de l'autre — devaient être tenus cohérents à chaque commit par un protocole de **validation en deux phases** (*two-phase commit*, 2PC), au prix d'opérations de synchronisation (`fsync`) supplémentaires. C'est précisément ce coût que la nouveauté de la 12.3 vient supprimer.

## Activation et organisation des fichiers

Le journal binaire est **désactivé par défaut** sur une instance autonome ; on l'active avec l'option `log_bin`, dont la configuration fait l'objet de la sous-section suivante (§11.5.1). Dans le modèle traditionnel, MariaDB écrit une suite de fichiers numérotés accompagnés d'un fichier d'index ; ces fichiers sont périodiquement renouvelés (rotation), puis supprimés lorsqu'ils ne sont plus nécessaires (§11.5.3).

Chaque transaction peut par ailleurs être identifiée de façon globale par un **GTID** (*Global Transaction Identifier*), qui simplifie considérablement les bascules (*failover*) et la reconfiguration des réplicas (§13.4).

## Trois formats d'enregistrement

Le journal binaire peut consigner les modifications sous trois formes, selon la variable `binlog_format` : `STATEMENT` (l'instruction SQL est journalisée telle quelle), `ROW` (ce sont les modifications de lignes qui sont enregistrées) et `MIXED` (MariaDB choisit l'une ou l'autre selon l'instruction). Chacun a ses avantages et ses limites, détaillés en §11.5.2.

## La grande nouveauté de la 12.3 : le binlog intégré à InnoDB

L'évolution majeure de MariaDB 12.3 — l'une des deux fonctionnalités phares de la version — réorganise en profondeur le fonctionnement du journal binaire. Plutôt que d'écrire dans des fichiers plats séparés, MariaDB peut désormais stocker les événements du binlog **directement dans des tablespaces gérés par InnoDB**, fichiers portant l'extension `.ibb` :

```sql
SHOW BINARY LOGS;
+-------------------+------------+
| Log_name          | File_size  |
+-------------------+------------+
| binlog-000000.ibb |   60735488 |
| binlog-000001.ibb | 1073741824 |
+-------------------+------------+
```

En faisant partager au binlog le mécanisme de *redo log* d'InnoDB, cette implémentation **élimine la validation en deux phases** entre le journal binaire et le moteur : il n'y a plus d'étape de synchronisation propre au binlog, et l'option `sync_binlog` devient inutile puisque les fichiers de binlog sont désormais intrinsèquement résistants aux crashs. À la clé, des gains d'écriture importants : l'éditeur annonce « ~4× » (sans méthodologie de benchmark publiée), tandis que des mesures indépendantes relèvent des gains d'écriture allant d'environ **1,5× à plus de 4×** selon le profil de charge. Le journal binaire et les fichiers de données InnoDB restant en permanence synchronisés, la récupération après crash s'en trouve simplifiée, ce qui autorise à assouplir `innodb_flush_log_at_trx_commit` (valeurs 0 ou 2) sans compromettre la sûreté.

Cette implémentation s'active explicitement (option `--binlog-storage-engine=innodb`) et vise les charges à fort débit transactionnel exigeant une durabilité stricte, de préférence sur des topologies de réplication déjà basées sur les GTID. Récente, elle est encore appelée à évoluer (l'instrumentation dans `performance_schema`, par exemple, n'est pas encore complète). Le détail figure en §11.5.4.

## Plan de la section et pour la suite

Les sous-sections suivantes approfondissent successivement la configuration du journal binaire (§11.5.1), ses trois formats `STATEMENT`/`ROW`/`MIXED` (§11.5.2), sa purge et sa rotation (§11.5.3), puis la nouvelle implémentation intégrée à InnoDB (§11.5.4). Au-delà de ce chapitre, le journal binaire est au cœur de la réplication (chapitre 13, GTID en §13.4), de la sauvegarde incrémentale et du PITR (§12.4, §12.5.2) ainsi que des architectures de capture de changements (§20.8) ; les journaux internes d'InnoDB, eux, sont traités avec le moteur de stockage (§7.2.3) et la concurrence (§6.6).

⏭️ [Configuration binlog](/11-administration-configuration/05.1-configuration-binlog.md)
