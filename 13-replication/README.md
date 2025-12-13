🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13. Réplication MariaDB

> **Niveau** : Avancé  
> **Durée estimée** : 8-10 heures  
> **Prérequis** : 
> - Maîtrise des concepts de transactions et ACID (Chapitre 6)
> - Compréhension des binary logs (Section 11.5)
> - Connaissance des architectures distribuées
> - Expérience en administration système Linux

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Comprendre les différents modes de réplication (asynchrone, semi-synchrone) et leurs implications
- Configurer une topologie de réplication Master-Slave (Source-Replica) en production
- Mettre en œuvre la réplication basée sur les positions binlog et GTID
- Déployer des architectures avancées (multi-source, cascade)
- Monitorer efficacement la réplication et diagnostiquer les problèmes de lag
- Utiliser les optimisations MariaDB 11.8 pour minimiser le retard de réplication
- Gérer les opérations de failover et switchover en toute sécurité

---

## Introduction

La **réplication** est l'un des mécanismes fondamentaux permettant d'assurer la **haute disponibilité**, la **scalabilité en lecture** et la **redondance des données** dans MariaDB. Elle consiste à copier automatiquement les modifications de données depuis un serveur source (Primary/Master) vers un ou plusieurs serveurs de destination (Replica/Slave).

### Pourquoi la réplication ?

Dans un environnement de production moderne, la réplication répond à plusieurs besoins critiques :

**📈 Scalabilité horizontale**
- Distribution de la charge de lecture sur plusieurs réplicas
- Capacité à servir des milliers de requêtes SELECT simultanées
- Séparation des charges OLTP et analytiques

**🛡️ Haute disponibilité**
- Continuité de service en cas de panne du serveur primary
- Basculement automatique ou manuel vers un replica
- Temps de récupération réduit (RTO/RPO optimisés)

**🔄 Redondance des données**
- Protection contre la perte de données
- Copies multiples pour la sécurité
- Possibilité de restauration à partir d'un replica

**🌍 Géo-distribution**
- Réplication entre datacenters pour la latence
- Conformité réglementaire (données localisées)
- Disaster Recovery cross-région

**⚙️ Maintenance sans interruption**
- Upgrades progressifs (upgrade replica puis basculement)
- Tests de nouvelles versions en parallèle
- Backups depuis un replica sans impact sur le primary

### Architecture de base

```
┌─────────────────┐
│   PRIMARY       │
│  (Master)       │
│                 │
│  Writes + Reads │
└────────┬────────┘
         │ Binary Log
         │ Replication
         ▼
┌─────────────────┐
│   REPLICA       │
│  (Slave)        │
│                 │
│  Reads Only     │
└─────────────────┘
```

Le serveur **Primary** :
- Accepte les écritures (INSERT, UPDATE, DELETE)
- Enregistre toutes les modifications dans le **binary log**
- Répond aux requêtes de lecture

Le serveur **Replica** :
- Se connecte au Primary via un thread I/O
- Récupère les événements du binary log
- Les applique via un thread SQL
- Répond aux requêtes de lecture (read-only par défaut)

---

## Vue d'ensemble du chapitre

Ce chapitre explore la réplication MariaDB dans tous ses aspects, de la configuration de base aux architectures avancées.

### 13.1 Concepts de réplication : Asynchrone vs Semi-synchrone

Nous commençons par comprendre les **modes de réplication** :

**Réplication asynchrone** (par défaut)
- Le Primary n'attend pas la confirmation du Replica
- Performance maximale mais risque de perte de données
- Appropriée pour la scalabilité en lecture

**Réplication semi-synchrone**
- Le Primary attend qu'au moins un Replica ait reçu les événements
- Garantie de durabilité renforcée
- Léger impact sur les performances d'écriture

💡 **Cas d'usage** : La réplication asynchrone convient aux applications tolérantes à une perte de données minimale, tandis que la semi-synchrone est recommandée pour les données critiques.

### 13.2 Réplication Master-Slave (Source-Replica)

Configuration étape par étape d'une topologie classique :

- **Configuration du Primary** : activation du binary log, création d'un utilisateur de réplication
- **Configuration du Replica** : paramétrage du serveur, options de sécurité
- **Commande CHANGE MASTER TO** : établissement de la connexion

```sql
-- Sur le Primary
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'StrongPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- Sur le Replica
CHANGE MASTER TO
  MASTER_HOST='primary.example.com',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='StrongPassword123!',
  MASTER_LOG_FILE='mariadb-bin.000042',
  MASTER_LOG_POS=1234;

START SLAVE;
```

### 13.3 Réplication basée sur les positions binlog

La méthode traditionnelle utilise les **coordonnées binlog** :
- Nom du fichier binlog (`mariadb-bin.000042`)
- Position dans le fichier (`1234`)

⚠️ **Limitation** : Complexité lors des failovers (nécessite de calculer la position exacte sur le nouveau Primary)

### 13.4 GTID (Global Transaction Identifier)

Le **GTID** est une révolution dans la réplication MariaDB :

```
0-1-1000
│ │  └─── Sequence number
│ └────── Server ID
└──────── Domain ID
```

**Avantages décisifs** :
- Identification unique de chaque transaction
- Failover automatisé simplifié
- Réplication multi-source facilitée
- Résolution automatique des conflits

```sql
-- Activation GTID
SET GLOBAL gtid_strict_mode=ON;
SET GLOBAL gtid_domain_id=0;

-- Réplication avec GTID
CHANGE MASTER TO
  MASTER_HOST='primary.example.com',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='StrongPassword123!',
  MASTER_USE_GTID=slave_pos;
```

🆕 **MariaDB 11.8** : Améliorations de la gestion GTID pour les topologies complexes et meilleure compatibilité avec MySQL GTID.

### 13.5 Réplication multi-source

Permet à un Replica de répliquer depuis **plusieurs Primary** simultanément :

```
┌──────────┐     ┌──────────┐
│ Primary1 │     │ Primary2 │
│ (Sales)  │     │ (HR)     │
└─────┬────┘     └────┬─────┘
      │               │
      └───────┬───────┘
              ▼
        ┌───────────┐
        │ Replica   │
        │(Reporting)│
        └───────────┘
```

**Cas d'usage** :
- Consolidation de données pour le reporting
- Agrégation de bases séparées
- Migration progressive

### 13.6 Réplication en cascade

Permet de **chaîner les serveurs** pour réduire la charge sur le Primary :

```
Primary → Replica1 → Replica2 → Replica3
          (Relay)
```

**Configuration** :
```sql
-- Sur Replica1 (intermédiaire)
SET GLOBAL log_slave_updates=ON;
```

⚠️ **Attention** : Augmente la latence de réplication proportionnellement au nombre de niveaux.

### 13.7 Monitoring et troubleshooting

**Commandes essentielles** :

```sql
-- État détaillé de la réplication
SHOW REPLICA STATUS\G

-- ou (ancien nom)
SHOW SLAVE STATUS\G
```

**Métriques critiques** :
- `Slave_IO_Running` : Thread I/O actif ?
- `Slave_SQL_Running` : Thread SQL actif ?
- `Seconds_Behind_Master` : Retard en secondes
- `Last_Error` : Dernière erreur rencontrée

**Diagnostic du lag** :

```sql
-- Vérifier le lag actuel
SELECT 
  TIMESTAMPDIFF(SECOND, 
    ts, 
    NOW()
  ) AS replication_lag_seconds
FROM mysql.heartbeat
WHERE server_id = @@server_id;
```

**Erreurs courantes** :
- **1062 (Duplicate entry)** : Insertion d'une clé déjà existante
- **1032 (Can't find record)** : Ligne à modifier/supprimer introuvable
- **2003 (Can't connect)** : Problème réseau avec le Primary

**Résolution** :
```sql
-- Ignorer une erreur ponctuelle (avec précaution !)
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;

-- Ou définir des erreurs à ignorer
SET GLOBAL slave_skip_errors = 1062,1032;
```

### 13.8 Failover et switchover

**Failover** (panne du Primary) :
1. Identifier le Replica le plus à jour
2. Promouvoir ce Replica en Primary
3. Reconfigurer les autres Replicas

**Switchover** (maintenance planifiée) :
1. Arrêter les écritures sur le Primary
2. Attendre que tous les Replicas soient synchronisés
3. Promouvoir le Replica cible
4. Basculer le trafic applicatif

💡 **Outils** : Orchestrator, MHA (Master High Availability), MaxScale Auto-Failover

### 13.9 Réplication semi-synchrone

Garantit qu'au moins un Replica a reçu la transaction avant que le Primary ne confirme le COMMIT :

```sql
-- Sur le Primary
INSTALL SONAME 'semisync_master';
SET GLOBAL rpl_semi_sync_master_enabled=ON;
SET GLOBAL rpl_semi_sync_master_timeout=1000; -- 1 seconde

-- Sur le Replica
INSTALL SONAME 'semisync_slave';
SET GLOBAL rpl_semi_sync_slave_enabled=ON;
```

**Trade-off** :
- ✅ Durabilité accrue (pas de perte de données en cas de crash)
- ⚠️ Latence d'écriture légèrement augmentée

### 🆕 13.10 Optimistic ALTER TABLE pour réduction du lag

**Nouveauté MariaDB 11.8** : Une innovation majeure pour minimiser l'impact des DDL sur la réplication.

**Problème traditionnel** :
```
Primary executes ALTER TABLE → Blocks writes for 30 minutes
                              ↓
Replica receives binlog event → Blocks replication for 30 minutes
                              ↓
Replication lag: 30+ minutes ❌
```

**Solution Optimistic ALTER** :
```sql
-- Sur le Primary
SET SESSION alter_algorithm='INSTANT', lock='NONE';
ALTER TABLE large_table ADD COLUMN new_col INT;
```

Le Replica peut appliquer l'ALTER de manière **non-bloquante** :
- Utilise un algorithme optimiste
- Permet aux autres transactions de continuer
- Réduit drastiquement le lag

**Configuration** :
```sql
-- Sur le Replica
SET GLOBAL slave_parallel_threads=4;
SET GLOBAL slave_parallel_mode='optimistic';
SET GLOBAL slave_run_triggers_for_rbr='YES';
```

**Résultats** :
- Lag réduit de **80-95%** pour les grosses tables
- Continuité de service améliorée
- Maintenance moins disruptive

⚠️ **Prérequis** : Compatible avec `ALTER ALGORITHM=INSTANT` ou `COPY` selon le cas.

---

## Architecture de réplication avancée

### Topologie complète

```
                  ┌──────────────┐
                  │   PRIMARY    │
                  │  (Master)    │
                  └───────┬──────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐
    │ Replica 1 │  │ Replica 2 │  │ Replica 3 │
    │ (Reads)   │  │ (Reporting)│ │ (Backup)  │
    └───────────┘  └─────┬─────┘  └───────────┘
                         │
                         ▼
                  ┌───────────┐
                  │ Replica 4 │
                  │ (Cascade) │
                  └───────────┘
```

### Bonnes pratiques de production

**1. Sécurité**
```ini
[mysqld]
# Replica en read-only
read_only=1
super_read_only=1

# Utiliser SSL pour la réplication
ssl-ca=/etc/mysql/ssl/ca-cert.pem
ssl-cert=/etc/mysql/ssl/server-cert.pem
ssl-key=/etc/mysql/ssl/server-key.pem
```

**2. Performance**
```ini
# Parallélisation de la réplication
slave_parallel_threads=4
slave_parallel_mode=optimistic

# Optimisation checksum binlog
binlog_checksum=CRC32
master_verify_checksum=ON
slave_sql_verify_checksum=ON
```

**3. Fiabilité**
```ini
# Durabilité des relay logs
sync_relay_log=1
relay_log_recovery=ON

# Position de réplication persistée
relay_log_info_repository=TABLE
master_info_repository=TABLE
```

**4. Monitoring**
```sql
-- Script de monitoring quotidien
SELECT 
  @@hostname AS replica_host,
  CASE 
    WHEN Slave_IO_Running='Yes' AND Slave_SQL_Running='Yes' THEN 'OK'
    ELSE 'ERROR'
  END AS replication_status,
  Seconds_Behind_Master AS lag_seconds,
  Master_Host AS primary_host,
  Master_Log_File AS current_binlog,
  Read_Master_Log_Pos AS binlog_position
FROM information_schema.REPLICA_STATUS;
```

---

## Cas d'usage réels

### Exemple 1 : E-commerce avec réplication géographique

```
Europe DC              ←→              US DC
┌──────────┐                      ┌──────────┐
│ Primary  │                      │ Replica  │
│ (Writes) │  ─ Réplication  ─→   │ (Reads)  │
└──────────┘      asynchrone      └──────────┘
```

**Configuration** :
- Primary en Europe pour les écritures
- Replica aux US pour les lectures locales
- Latence de réplication : 100-200ms acceptable
- Réduction de la latence utilisateur : 80%

### Exemple 2 : Reporting sans impact sur la production

```
Production DB          Reporting DB
┌──────────┐          ┌──────────┐
│ Primary  │          │ Replica  │
│ OLTP     │  ─────→  │ OLAP     │
└──────────┘          └──────────┘
                      ColumnStore
```

**Avantages** :
- Requêtes analytiques lourdes sur le Replica
- Zero impact sur le Primary
- Possibilité d'utiliser ColumnStore sur le Replica

---

## ✅ Points clés à retenir

1. **Modes de réplication** : Asynchrone (par défaut, performant) vs Semi-synchrone (durable, légère latence)

2. **GTID** : Remplace avantageusement les positions binlog pour une gestion simplifiée et un failover automatisé

3. **Monitoring essentiel** : Surveiller `Slave_IO_Running`, `Slave_SQL_Running` et `Seconds_Behind_Master` en permanence

4. **Réplication multi-source** : Permet de consolider plusieurs bases pour le reporting et l'analytique

5. **Semi-synchrone** : Obligatoire pour les données critiques nécessitant une garantie de durabilité

6. **Optimistic ALTER TABLE (11.8)** : Réduit drastiquement le lag lors des opérations DDL sur de grosses tables

7. **Parallélisation** : Utiliser `slave_parallel_threads` pour accélérer l'application des événements

8. **Sécurité** : Toujours configurer `read_only=1` sur les Replicas et utiliser SSL

9. **Failover** : GTID simplifie grandement les opérations de basculement et de reconfiguration

10. **Testing** : Tester régulièrement les procédures de failover et de switchover

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Replication Overview](https://mariadb.com/kb/en/replication-overview/)
- [📖 Setting Up Replication](https://mariadb.com/kb/en/setting-up-replication/)
- [📖 Global Transaction ID (GTID)](https://mariadb.com/kb/en/gtid/)
- [📖 Semi-synchronous Replication](https://mariadb.com/kb/en/semisynchronous-replication/)
- [📖 Multi-Source Replication](https://mariadb.com/kb/en/multi-source-replication/)
- [📖 Parallel Replication](https://mariadb.com/kb/en/parallel-replication/)

### Articles et guides

- [🔗 MariaDB Replication Best Practices (2025)](https://mariadb.com/resources/blog/mariadb-replication-best-practices/)
- [🔗 GTID Migration Guide](https://mariadb.com/kb/en/migrating-to-gtid-based-replication/)
- [🔗 Troubleshooting Replication](https://mariadb.com/kb/en/troubleshooting-replication/)

### Outils

- **Orchestrator** : Gestion automatisée de topologies de réplication
- **MHA (Master High Availability)** : Failover automatique
- **MaxScale** : Proxy avec gestion de réplication intégrée
- **pt-heartbeat** (Percona Toolkit) : Mesure précise du lag

---

## 🎓 Prochaines étapes

Après avoir maîtrisé la réplication, vous êtes prêt à aborder :

### ➡️ Chapitre 14 : Haute Disponibilité

Découvrez comment construire des architectures **hautement disponibles** avec :
- **Galera Cluster** : Réplication synchrone multi-master
- **MaxScale** : Load balancing et query routing
- **Failover automatique** : Solutions de basculement intelligent
- Architectures de **disaster recovery**

La réplication est la fondation ; la haute disponibilité est l'édifice que vous allez construire dessus !

---


⏭️ [Concepts de réplication : Asynchrone vs Semi-synchrone](/13-replication/01-concepts-replication.md)
