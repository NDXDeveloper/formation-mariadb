🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.4 — Gestion des logs

Un serveur MariaDB en production ne se pilote pas à l'aveugle : il consigne en continu son activité dans différents journaux (*logs*). Ces fichiers — ou, selon la configuration, ces tables — constituent la principale source d'information pour diagnostiquer un incident, surveiller la santé du serveur, tracer les accès ou repérer les requêtes problématiques. Savoir quels journaux activer, où les écrire et comment les exploiter fait partie des compétences fondamentales du DBA.

Cette section donne une vue d'ensemble du système de journalisation de MariaDB et présente les principes de configuration communs à tous les journaux. Les trois journaux les plus utilisés au quotidien — le journal d'erreurs, le journal des requêtes lentes et le journal général — sont ensuite détaillés dans les sous-sections qui suivent.

## À quoi servent les journaux ?

La journalisation répond à plusieurs besoins complémentaires. Le diagnostic vient en premier : lorsqu'un service refuse de démarrer ou qu'une réplication s'interrompt, c'est le journal d'erreurs qui en livre la cause. La supervision s'appuie ensuite sur les journaux pour suivre dans la durée le comportement du serveur et alimenter les outils d'observabilité (chapitre 16). L'optimisation repose en grande partie sur le journal des requêtes lentes, qui met en évidence les requêtes coûteuses à retravailler (voir aussi §15.7). L'audit et la conformité, eux, exigent de tracer les connexions et les opérations sensibles, ce qui relève du plugin d'audit (§10.8). Enfin, la réplication et la reprise après incident dépendent du journal binaire, qui enregistre les modifications de données et sert de support au *point-in-time recovery* (§11.5).

## Panorama des journaux de MariaDB

MariaDB distingue plusieurs journaux, chacun dédié à un usage précis. Le tableau ci-dessous les recense ; certains sont traités dans d'autres chapitres lorsqu'ils relèvent d'un sujet spécifique (réplication, audit).

| Journal | Contenu principal | État par défaut | Variable principale | Détaillé dans |
|---------|-------------------|-----------------|---------------------|---------------|
| Journal d'erreurs (*error log*) | Démarrage/arrêt, erreurs, avertissements, messages du serveur | Toujours actif | `log_error` | §11.4.1 |
| Journal des requêtes lentes (*slow query log*) | Requêtes dépassant `long_query_time` ou n'utilisant pas d'index | Désactivé | `slow_query_log` | §11.4.2 |
| Journal général (*general query log*) | Toutes les requêtes et connexions reçues | Désactivé | `general_log` | §11.4.3 |
| Journal binaire (*binary log*) | Modifications de données (réplication, PITR) | Désactivé | `log_bin` | §11.5 |
| Journal de relais (*relay log*) | Événements reçus par un réplica | Automatique (sur réplica) | `relay_log` | Chapitre 13 |
| Journal d'audit | Connexions et requêtes à des fins de traçabilité | Désactivé (plugin) | `server_audit_logging` | §10.8 |

> Le journal binaire a fait l'objet d'une refonte importante dans la série 12.x : intégré au moteur InnoDB, il supprime une étape de synchronisation et apporte un gain de l'ordre de 4× en écriture. Ses spécificités sont traitées à la section 11.5 (et §11.5.4 pour ce changement).

## Trois destinations possibles : fichier, table ou journal système

Pour le journal général et le journal des requêtes lentes, MariaDB permet de choisir la destination via la variable `log_output`, qui accepte `FILE`, `TABLE`, `NONE` ou une combinaison (`'FILE,TABLE'`). La valeur `FILE` (par défaut) écrit dans les fichiers désignés par `general_log_file` et `slow_query_log_file`. La valeur `TABLE` écrit respectivement dans les tables `mysql.general_log` et `mysql.slow_log`, directement interrogeables en SQL. La valeur `NONE`, enfin, désactive l'écriture de ces deux journaux même si leur variable d'activation reste positionnée à `ON`.

Le journal d'erreurs et le journal binaire ne sont pas concernés par `log_output` : le premier s'écrit toujours dans un fichier (ou dans le journal système, voir ci-dessous), le second dans ses propres fichiers binaires.

Sur les systèmes gérés par systemd, si `log_error` n'est pas défini, les messages d'erreur sont par défaut redirigés vers le journal système (`journald`), consultable avec `journalctl -u mariadb`. Définir explicitement `log_error` reste recommandé en production : on dispose ainsi d'un fichier dédié, plus simple à archiver et à analyser.

## Activer et configurer un journal

La configuration des journaux se fait de deux manières. De façon permanente, dans un fichier de configuration (typiquement sous la section `[mariadbd]` de `my.cnf` ; voir §11.1) :

```ini
[mariadbd]
# Journal d'erreurs dédié
log_error = /var/log/mysql/error.log

# Journal des requêtes lentes
slow_query_log      = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time     = 1

# Destination des journaux general et slow
log_output = FILE
```

La plupart des paramètres de journalisation sont également dynamiques : ils peuvent être modifiés à chaud, sans redémarrage, à l'aide de `SET GLOBAL`. C'est particulièrement utile pour activer ponctuellement un journal le temps d'un diagnostic, puis le désactiver :

```sql
-- Activer le journal général temporairement
SET GLOBAL general_log = ON;

-- Le diriger vers une table plutôt qu'un fichier
SET GLOBAL log_output = 'TABLE';

-- ... investigation ...

-- Le désactiver une fois l'analyse terminée
SET GLOBAL general_log = OFF;
```

Pour inspecter l'état courant de la journalisation, les variables se consultent avec `SHOW VARIABLES` :

```sql
SHOW VARIABLES LIKE 'log_%';
SHOW VARIABLES LIKE 'general_log%';
SHOW VARIABLES LIKE 'slow_query%';
```

À noter : quelques variables, comme l'activation du journal binaire (`log_bin`), restent statiques et nécessitent un redémarrage du serveur.

## Espace disque, rotation et purge

Les journaux peuvent croître rapidement — le journal général en particulier, qui enregistre chaque requête. Sans surveillance, ils risquent de saturer le système de fichiers et, dans le pire des cas, de bloquer le serveur. Trois précautions s'imposent.

La rotation consiste à archiver périodiquement le fichier courant et à en ouvrir un nouveau. Sous Linux, elle est généralement confiée à `logrotate`, dont MariaDB fournit une configuration par défaut. La commande `FLUSH LOGS` (ou `mariadb-admin flush-logs`) ferme et rouvre les fichiers de journaux, étape indispensable après qu'un script de rotation a renommé l'ancien fichier :

```sql
FLUSH LOGS;
```

La purge concerne surtout le journal binaire, dont l'accumulation est la cause la plus fréquente de saturation disque ; sa gestion (`PURGE BINARY LOGS`, `binlog_expire_logs_seconds`) est détaillée en §11.5.3.

Enfin, la surveillance de l'espace disque dédié aux journaux doit être intégrée au monitoring (chapitre 16) afin d'être alerté avant que l'incident ne survienne.

## Sécurité et confidentialité des journaux

Les journaux contiennent des informations sensibles. Le journal général, notamment, enregistre le texte intégral des requêtes : il peut donc faire apparaître des données métier confidentielles, voire des secrets (par exemple lors de certaines commandes de gestion d'utilisateurs). Pour cette raison, il convient de restreindre les permissions des fichiers de journaux au seul compte système de MariaDB, de protéger les répertoires d'archive, et de n'activer le journal général que de manière ciblée et temporaire. Lorsque les journaux sont écrits en table (`log_output = TABLE`), l'accès aux tables `mysql.general_log` et `mysql.slow_log` doit être encadré par le système de privilèges (chapitre 10).

## Centralisation et observabilité

Dans une infrastructure de plusieurs serveurs, consulter les journaux machine par machine devient vite ingérable. La pratique courante consiste à centraliser les journaux dans une pile de collecte dédiée et à exposer les métriques associées à des outils comme Prometheus et Grafana. Ces aspects, qui relèvent de l'exploitation moderne, sont abordés au chapitre 16 (§16.9 et §16.10).

## Pour la suite

Les sous-sections suivantes détaillent les trois journaux que l'administrateur manipule le plus souvent : le journal d'erreurs (§11.4.1), premier réflexe en cas d'incident ; le journal des requêtes lentes (§11.4.2), point d'entrée de l'optimisation ; et le journal général (§11.4.3), outil de débogage exhaustif mais coûteux à laisser actif. Pour les journaux relevant d'un domaine spécifique, on se reportera à la section 11.5 (journal binaire et logs de transactions), au chapitre 13 (journal de relais et réplication) ainsi qu'à la section 10.8 (audit et logging).

⏭️ [Error log](/11-administration-configuration/04.1-error-log.md)
