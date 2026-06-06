🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 Sauvegarde incrémentale avec binary logs

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Toutes les stratégies vues jusqu'ici — complète, incrémentale, différentielle, logique ou physique — fonctionnent par **instantanés** : elles figent l'état de la base à des moments discrets. Entre deux sauvegardes, les modifications les plus récentes ne sont pas encore protégées. Si un incident survient juste avant la prochaine sauvegarde, tout ce qui s'est passé depuis la précédente est perdu.

Le **journal binaire** (*binary log*, ou *binlog*) comble exactement cet intervalle. En enregistrant **en continu** chaque modification apportée à la base, il constitue une couche de **sauvegarde incrémentale permanente** : combiné à une sauvegarde de base, il permet de reconstruire la base **jusqu'à n'importe quel instant** — c'est le fondement de la restauration à un instant précis (*Point-in-Time Recovery*, PITR). C'est le complément « continu » des stratégies par instantanés évoqué dès la [section 12.1](01-strategies-sauvegarde.md).

Cette section se concentre sur l'aspect **sauvegarde** : ce que le binlog enregistre, comment l'activer, l'archiver et le relier à une sauvegarde de base, ainsi que l'outil `mariadb-binlog` qui permet de le rejouer. La **procédure de restauration PITR** complète, pas à pas, est traitée en [section 12.5.2](05.2-pitr.md).

---

## Le journal binaire comme sauvegarde incrémentale

Le journal binaire contient l'enregistrement de **toutes les modifications** apportées aux bases : les changements de données (DML) comme les changements de structure (DDL). Chaque événement y est inscrit **dans l'ordre**, repéré par une **position**. Le serveur l'alimente au fil de l'eau, à mesure que les transactions sont validées.

Il en découle une équation simple, au cœur de toute stratégie de sauvegarde complète :

> **Sauvegarde de base + journaux binaires postérieurs = reconstruction jusqu'à n'importe quel instant.**

Archiver régulièrement les journaux binaires revient donc à **capturer en continu tout ce qui change** depuis la dernière sauvegarde de base. C'est précisément ce qui permet d'atteindre un RPO très faible (quelques secondes si les binlogs sont diffusés en temps réel), là où les seuls instantanés limitent la perte potentielle à l'intervalle qui les sépare.

---

## Prérequis : activer le journal binaire

Le PITR n'est possible **que si le journal binaire est activé** — c'est le prérequis absolu. Sans binlog, il n'existe aucune trace des modifications survenues entre deux sauvegardes, et la restauration ne peut remonter qu'au dernier instantané.

L'activation se fait via l'option **`--log-bin`** (variable `log_bin`), qui définit le préfixe des fichiers de journal et leur fichier d'index. Deux réglages associés méritent attention :

- **Le format** (`binlog_format`) : trois formats existent — *STATEMENT* (SBR), *ROW* (RBR) et *MIXED*. Pour une restauration **fiable**, le format **ROW** est recommandé : il enregistre les modifications ligne par ligne, sans ambiguïté de réexécution, là où le format STATEMENT peut produire des résultats différents pour des instructions non déterministes. Ces formats sont détaillés au chapitre [11.5.2](../11-administration-configuration/05.2-formats-binlog.md).
- **L'identifiant du serveur** (`server_id`), nécessaire au bon fonctionnement du journal et de la réplication.

> 🆕 **Nouveauté MariaDB 12.3.** À partir de la 12.3, les journaux binaires peuvent être **intégrés au moteur InnoDB**, ce qui supprime une synchronisation coûteuse et améliore nettement les performances en écriture (voir chapitre [11.5.4](../11-administration-configuration/05.4-binlog-innodb-performance.md)). Cela ne change rien à leur rôle pour le PITR.

---

## Archiver les journaux binaires

« Sauvegarder » les binlogs consiste à **copier les fichiers de journal vers un stockage sûr**, distinct du serveur. Deux approches coexistent.

La première, la plus simple, consiste à **copier périodiquement les fichiers de journal**. On force d'abord une rotation avec `FLUSH BINARY LOGS` (qui ferme le fichier courant et en ouvre un nouveau), puis on copie les fichiers **clos** vers l'archive. On ne copie jamais le fichier en cours d'écriture.

La seconde, plus robuste pour un faible RPO, consiste à **diffuser les journaux en continu** depuis le serveur, à la manière d'une réplique, grâce à `mariadb-binlog` :

```bash
mariadb-binlog \
  --read-from-remote-server \
  --raw \
  --stop-never \
  --host=db.exemple.com --user=repli --password=*** \
  mariadb-bin.000001
```

Avec `--read-from-remote-server`, l'outil se connecte au serveur ; `--raw` écrit les fichiers de journal bruts (et non du SQL) ; `--stop-never` maintient la connexion ouverte pour récupérer les nouveaux événements **en temps réel**. On obtient ainsi une copie des binlogs presque instantanément à jour.

### Gérer l'expiration : ne pas purger avant d'avoir archivé

Le danger principal est que le serveur **supprime un journal avant qu'il n'ait été archivé**. Il faut donc maîtriser l'expiration des binlogs :

- **`expire_logs_days`** : durée, en jours, au-delà de laquelle un fichier de journal est automatiquement supprimé. Fixée à **0 par défaut** (aucune suppression). Les fichiers ne sont vérifiés **qu'à la rotation** : un journal qui se remplit lentement peut donc survivre au-delà du délai. On peut forcer la rotation (et donc l'expiration) avec `FLUSH BINARY LOGS`.
- **`binlog_expire_logs_seconds`** : disponible depuis MariaDB 10.6, elle offre un contrôle **plus précis** (en secondes) et **prend le dessus** si les deux variables sont non nulles.
- Pour une purge manuelle, `PURGE BINARY LOGS` supprime les journaux antérieurs à une date ou à un fichier donné ; `RESET MASTER` les supprime tous.

> 💡 **Règle d'or.** Réglez toujours l'expiration sur une durée **supérieure à l'intervalle entre deux sauvegardes de base** (et supérieure à tout retard possible d'une réplique). Mieux vaut un journal de trop qu'un trou irrécupérable dans la chaîne de restauration. La purge et la rotation sont détaillées au chapitre [11.5.3](../11-administration-configuration/05.3-purge-rotation.md).

---

## Le lien avec la sauvegarde de base : les coordonnées

Pour rejouer les binlogs au bon endroit, il faut savoir **à partir de quel point** commencer. C'est le rôle des **coordonnées** que chaque sauvegarde de base enregistre au moment où elle est prise :

- une sauvegarde **Mariabackup** consigne ces coordonnées dans le fichier **`mariadb_backup_binlog_info`**, sous la forme `fichier position [GTID]` — par exemple `mariadb-bin.000002 208286 0-1-423` ;
- une sauvegarde **logique** les écrit via l'option `--master-data=2` (en commentaire dans le dump).

Une coordonnée se compose d'un **nom de fichier** et d'une **position**. La position est un entier qui croît à chaque transaction, mais de façon **non séquentielle** (les sauts sont irréguliers) : une position n'a donc de sens **qu'associée à son nom de fichier**. Le **GTID** (*Global Transaction Identifier*) offre une alternative plus robuste pour identifier le point de départ, indépendante des positions de fichier.

Le PITR consiste alors à restaurer la sauvegarde de base, puis à rejouer les binlogs **à partir de cette coordonnée**.

---

## Du binlog au SQL : l'outil `mariadb-binlog`

Les fichiers de journal binaire ne sont pas lisibles directement. L'outil **`mariadb-binlog`** (ancien nom : `mysqlbinlog`) les lit et en restitue le contenu sous forme d'**instructions SQL**, que l'on **redirige vers le client `mariadb`** pour les réexécuter :

```bash
mariadb-binlog \
  --start-position=208286 \
  --stop-datetime="2026-06-06 11:59:59" \
  /var/lib/mysql/mariadb-bin.000002 \
  | mariadb -u root -p
```

Les options qui délimitent la plage à rejouer sont au cœur du PITR :

- **`--start-position`** / **`--stop-position`** : bornent le rejeu par **position** ; c'est la méthode la plus **précise**.
- **`--start-datetime`** / **`--stop-datetime`** : bornent par **date et heure**. `--stop-datetime` arrête la génération à la **première transaction dont l'horodatage est égal ou postérieur** à la valeur indiquée.

On peut rejouer **plusieurs fichiers** à la suite, dans l'ordre (`mariadb-binlog bin.000002 bin.000003 | mariadb …`), ou écrire le SQL dans un fichier avant de l'exécuter.

> 💡 **Bonne pratique.** Utilisez d'abord les bornes `--start-datetime` / `--stop-datetime` pour **localiser** la zone qui vous intéresse et repérer la position exacte, puis effectuez le rejeu réel avec `--start-position` / `--stop-position` : borner la plage par position est plus **fiable** que par horodatage (moindre risque de manquer ou d'inclure un événement de trop).

---

## L'intérêt : récupérer à un instant précis

Le bénéfice est décisif : on ne se limite plus au dernier instantané, on peut remonter à **n'importe quel moment**. Le scénario emblématique est celui de la **fausse manœuvre** — un `DROP TABLE` ou un `DELETE` exécuté par erreur en production. Avec un instantané seul, on perdrait tout ce qui a été produit depuis la dernière sauvegarde. Avec les binlogs, la récupération devient **chirurgicale** :

1. on restaure la sauvegarde de base ;
2. on rejoue les binlogs **jusqu'à juste avant** l'instruction fautive (via `--stop-position`) ;
3. on récupère ainsi **tout sauf l'erreur**.

Il est même possible de **sauter une seule transaction** indésirable : on rejoue jusqu'à la position précédant l'erreur, puis on reprend à la position qui la suit. C'est une capacité propre à l'approche par journal binaire, qu'aucune stratégie d'instantané ne peut offrir.

La **procédure complète** de restauration PITR — de la restauration de la base au rejeu borné des journaux — est détaillée en [section 12.5.2](05.2-pitr.md).

> 📌 **Note prospective.** La série **13.x** introduit une approche PITR alternative fondée sur l'**archivage du write-ahead log d'InnoDB** (`innodb_log_archive`), qui rejoue les journaux InnoDB plutôt que les journaux binaires. Sur la 12.3, le PITR repose sur les binlogs décrits ici.

---

## Bonnes pratiques

- **Activez le journal binaire** (`--log-bin`) en format **ROW** : sans lui, aucun PITR n'est possible.
- **Archivez les binlogs fréquemment**, idéalement par diffusion continue (`--read-from-remote-server --raw --stop-never`) pour un RPO de quelques secondes.
- **Maîtrisez l'expiration** : réglez `expire_logs_days` / `binlog_expire_logs_seconds` au-delà de l'intervalle entre sauvegardes de base, pour ne jamais purger un journal non archivé.
- **Conservez les coordonnées** de chaque sauvegarde de base (`mariadb_backup_binlog_info` ou `--master-data`) : elles indiquent où démarrer le rejeu.
- **Stockez les binlogs séparément** du serveur (règle 3-2-1, voir [12.1](01-strategies-sauvegarde.md)).
- **Localisez par date, rejouez par position** pour un PITR fiable.
- **Testez le rejeu** régulièrement, comme toute sauvegarde (voir [12.7](07-tests-restauration-pra.md)).

---

## À retenir

- Le **journal binaire** enregistre en continu toutes les modifications (DML et DDL) ; il constitue la couche de **sauvegarde incrémentale permanente** qui comble l'intervalle entre deux instantanés.
- **Sauvegarde de base + binlogs postérieurs = restauration jusqu'à n'importe quel instant** (PITR).
- Le PITR exige que le journal binaire soit **activé** ; le format **ROW** est recommandé pour un rejeu fiable.
- « Sauvegarder » les binlogs = les **archiver** régulièrement (copie après `FLUSH BINARY LOGS`, ou diffusion continue) **avant** qu'ils n'expirent (`expire_logs_days`, `binlog_expire_logs_seconds`).
- Les **coordonnées** de la sauvegarde de base (`mariadb_backup_binlog_info`, `--master-data`) indiquent le point de départ du rejeu ; une position n'a de sens qu'avec son nom de fichier.
- **`mariadb-binlog`** convertit les journaux en SQL rejoué via le client `mariadb` ; on **localise par `--start/--stop-datetime`** puis on **rejoue par `--start/--stop-position`**.

---

⏮️ [12.3.3 — Support BACKUP STAGE](03.3-backup-stage.md) · ⏭️ [12.5 — Restauration](05-restauration.md)

⏭️ [Restauration](/12-sauvegarde-restauration/05-restauration.md)
