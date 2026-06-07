🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3 — Réplication basée sur les positions (binlog coordinates)

> **Chapitre 13 — Réplication** · Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La réplication **par coordonnées binlog** est la méthode de positionnement **historique** : le réplica repère sa place dans le flux de la source au moyen d'un couple **(nom de fichier binlog, position dans ce fichier)**. C'est la mécanique d'origine de MariaDB, toujours fonctionnelle, mais **supplantée par le GTID** (13.4) pour tout ce qui touche à la robustesse des bascules.

> ⚠️ **À savoir dès maintenant (12.3) :** la réplication par coordonnées suppose le **binlog traditionnel** basé sur fichiers. Elle est **incompatible avec le binlog intégré à InnoDB** (nouveauté 12.3, cf. 13.2.1), lequel **impose le mode GTID**. Si vous adoptez le binlog InnoDB, ce chapitre 13.3 ne s'applique pas : passez directement au GTID (13.4).

---

## 1. Qu'est-ce qu'une coordonnée binlog ?

Une coordonnée est une **paire** :

```
(nom_de_fichier_binlog , position)
```

par exemple `(mariadb-bin.000003, 1245)`. La **position** est un **offset en octets** dans le fichier, qui pointe vers le début de l'événement suivant à lire. Chaque événement écrit dans le binary log occupe une plage d'octets ; la position progresse au fil des écritures.

Lorsque le binary log **tourne** (rotation après `max_binlog_size`, `FLUSH BINARY LOGS`, redémarrage…), un **nouveau fichier** est créé (`...000004`) et la position **repart** au début. Une coordonnée n'a donc de sens **que sur le serveur qui l'a produite** : c'est là toute la fragilité de cette méthode (voir §6).

---

## 2. Relever la position de la source

Sur la source, la position courante se lit avec **`SHOW BINLOG STATUS`** (alias historique **`SHOW MASTER STATUS`**) :

```sql
SHOW BINLOG STATUS\G
```

```
*************************** 1. row ***************************
             File: mariadb-bin.000003
         Position: 1245
     Binlog_Do_DB:
 Binlog_Ignore_DB:
  Gtid_Binlog_Pos: 0-1-42
```

- **`File`** et **`Position`** : les coordonnées à fournir au réplica.
- **`Binlog_Do_DB` / `Binlog_Ignore_DB`** : filtres de journalisation éventuels (cf. 13.2.1).
- **`Gtid_Binlog_Pos`** : depuis **MariaDB 12.3**, cette colonne figure directement dans le résultat — auparavant, il fallait une requête supplémentaire `SELECT @@global.gtid_binlog_pos`.

Cette commande requiert le privilège **`BINLOG MONITOR`**. Pour lister les fichiers binlog présents : `SHOW BINARY LOGS;`.

> 💡 En pratique, on **ne relève pas** la position à la main : elle est capturée **en même temps que l'instantané initial** (par `mariadb-dump --master-data` ou par Mariabackup, cf. 13.2.2 et chapitre 12), garantissant la cohérence entre les données copiées et le point de départ.

---

## 3. Configurer le réplica avec `MASTER_LOG_FILE` / `MASTER_LOG_POS`

Côté réplica, on indique le point de départ via `CHANGE MASTER TO` (cf. 13.2.3) :

```sql
CHANGE MASTER TO
    MASTER_HOST     = '192.168.1.10',
    MASTER_USER     = 'repl',
    MASTER_PASSWORD = 'mot_de_passe_robuste',
    MASTER_LOG_FILE = 'mariadb-bin.000003',
    MASTER_LOG_POS  = 1245,
    MASTER_USE_GTID = no;     -- mode coordonnées (pas de GTID)

START REPLICA;
```

Le thread IO commence alors à télécharger les événements de la source **à partir de cette coordonnée**, et le thread SQL les applique dans l'ordre.

---

## 4. Comment le réplica suit sa progression

Une fois la réplication lancée, le réplica maintient **plusieurs positions**, visibles dans `SHOW REPLICA STATUS` :

| Champ | Signification |
|-------|---------------|
| `Master_Log_File` / `Read_Master_Log_Pos` | Jusqu'où le **thread IO** a **lu** dans le binlog de la source. |
| `Relay_Master_Log_File` / `Exec_Master_Log_Pos` | La coordonnée du binlog **source** correspondant à ce que le **thread SQL** a déjà **exécuté**. |
| `Relay_Log_File` / `Relay_Log_Pos` | La position dans le **relay log local**. |

L'écart entre `Read_Master_Log_Pos` (lu) et `Exec_Master_Log_Pos` (exécuté) reflète le travail restant à appliquer.

> ⚠️ **Non *crash-safe*** : en mode coordonnées, ces positions sont stockées dans les fichiers `master.info` et `relay-log.info`, mis à jour **indépendamment** des données. Après un crash, ils peuvent se désynchroniser des changements réellement appliqués (doublons, corruption silencieuse). C'est l'une des raisons majeures de préférer le GTID (rappel de 13.2.2).

---

## 5. Inspecter le binary log

Pour examiner le contenu d'un binlog et ses positions :

```sql
-- Depuis le serveur
SHOW BINLOG EVENTS IN 'mariadb-bin.000003' FROM 1245 LIMIT 10;
```

```bash
# En ligne de commande (indépendamment du serveur)
mariadb-binlog mariadb-bin.000003
```

L'outil **`mariadb-binlog`** (anciennement `mysqlbinlog`) décode les événements et affiche, pour le binlog traditionnel, leur offset (`end_log_pos`).

> 🔎 **Spécificités du binlog InnoDB (12.3) :** avec le binlog intégré à InnoDB, les événements affichent `end_log_pos 0` (le suivi de position est géré en interne par InnoDB), un même événement peut **s'étendre sur plusieurs fichiers** et des parties d'événements peuvent y être **entrelacées**. `mariadb-binlog` les recompose de façon transparente, à condition de lui passer tous les fichiers concernés, dans l'ordre. Ce modèle confirme pourquoi le positionnement par offset n'a plus cours avec le binlog InnoDB.

---

## 6. Avantages et limites

### Avantages

- **Simplicité conceptuelle** : un fichier et un offset, sans configuration supplémentaire.
- **Universalité** : compris par tous les outils de l'écosystème qui lisent directement les fichiers binlog (CDC, audit…).

### Limites

- **Coordonnées locales au serveur** : une position sur la source n'a **aucun sens** sur un autre serveur. Chaque serveur numérote ses propres binlogs indépendamment.
- **Fragilité lors d'un *failover* / *switchover*** : promouvoir un nouveau serveur source oblige à **retrouver manuellement** la coordonnée équivalente sur ce nouveau serveur pour chaque réplica restant — opération délicate et source d'erreurs (voir 13.8).
- **Non *crash-safe*** (voir §4).
- **Incompatible avec le binlog InnoDB** (12.3), qui impose le GTID.

---

## 7. Coordonnées vs GTID (et l'impact de la 12.3)

Le **GTID** (Global Transaction Identifier, 13.4) répond précisément à ces limites : chaque transaction reçoit un **identifiant global unique**, reconnu par tous les serveurs de la topologie. Le réplica n'a plus besoin de coordonnées locales : il demande « toutes les transactions que je n'ai pas encore vues », ce qui rend **bascules et reconfigurations** nettement plus simples.

**Migrer des coordonnées vers le GTID** est direct : la fonction `BINLOG_GTID_POS('fichier', position)` renvoie la position GTID correspondant à une coordonnée donnée, après quoi on bascule le réplica avec `MASTER_USE_GTID = slave_pos` (cf. 13.4).

**Dans le contexte 12.3**, le choix est encore plus tranché : adopter le **binlog intégré à InnoDB** — l'une des nouveautés phares de la version — **exige** le mode GTID. La réplication par coordonnées devient alors indisponible. Pour tout nouveau déploiement, le **GTID est donc le choix par défaut recommandé**, la réplication par coordonnées restant surtout utile pour comprendre les fondations, maintenir des topologies anciennes ou interfacer des outils lisant les binlogs.

---

## Idées clés à retenir

- Une **coordonnée binlog** est un couple (fichier, offset) **local au serveur** qui l'a produit.
- La position de la source se relève avec **`SHOW BINLOG STATUS`** (alias `SHOW MASTER STATUS`) ; depuis 12.3, le résultat inclut `Gtid_Binlog_Pos`.
- Le réplica démarre via `MASTER_LOG_FILE` + `MASTER_LOG_POS` (`MASTER_USE_GTID = no`) et suit sa progression (`Read_*` / `Exec_Master_Log_Pos`).
- La méthode est **non *crash-safe***, **fragile aux bascules** et **incompatible avec le binlog InnoDB** (12.3).
- Le **GTID** (13.4) corrige ces limites ; `BINLOG_GTID_POS()` permet la conversion. Le GTID est le **choix recommandé** — et **obligatoire** avec le binlog InnoDB.

---

## Pour aller plus loin

- **13.4** — [GTID (Global Transaction Identifier)](04-gtid.md) : la méthode de positionnement moderne et robuste.
- **13.2.3** — [CHANGE MASTER TO / CHANGE REPLICATION SOURCE](02.3-change-master-to.md) : `MASTER_LOG_FILE` / `MASTER_LOG_POS` et `MASTER_USE_GTID`.
- **13.7** — [Monitoring et troubleshooting](07-monitoring-troubleshooting.md) : interpréter les champs de position de `SHOW REPLICA STATUS`.
- **13.8** — [Failover et switchover](08-failover-switchover.md) : pourquoi les coordonnées compliquent les bascules.
- **Chapitre 11.5** — [Binary logs et logs de transactions](../11-administration-configuration/05-binary-logs.md) : rotation, formats et binlog InnoDB.

⏭️ [GTID (Global Transaction Identifier)](/13-replication/04-gtid.md)
