🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1 Concepts de Réplication : Asynchrone vs Semi-synchrone

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** : 
> - Compréhension des transactions ACID (Chapitre 6)
> - Connaissance des binary logs (Section 11.5)
> - Bases en systèmes distribués (consensus, latence réseau)
> - Théorème CAP (Consistency, Availability, Partition tolerance)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Expliquer le fonctionnement détaillé de la réplication asynchrone et semi-synchrone
- Comprendre les implications du théorème CAP sur chaque mode
- Analyser les trade-offs performance vs durabilité
- Choisir le mode approprié selon les contraintes métier
- Configurer et optimiser chaque mode de réplication
- Mesurer et interpréter les métriques de performance

---

## Introduction

La **réplication** dans MariaDB repose sur la propagation des modifications de données depuis un serveur source (Primary/Master) vers un ou plusieurs serveurs de destination (Replica/Slave). Le **mode de réplication** détermine le niveau de synchronisation entre ces serveurs et a un impact direct sur :

- La **durabilité** des données (garantie anti-perte)
- La **performance** des écritures
- La **cohérence** lecture/écriture
- Le **comportement en cas de panne**

MariaDB propose principalement deux modes :
1. **Réplication asynchrone** (par défaut)
2. **Réplication semi-synchrone** (plugin additionnel)

💡 **Analogie** : Pensez à l'envoi d'un colis. La réplication asynchrone est comme déposer le colis dans une boîte postale sans attendre de confirmation. La semi-synchrone, c'est attendre que le facteur confirme qu'il a bien le colis avant de partir.

---

## Réplication Asynchrone

### Principe de fonctionnement

La réplication asynchrone est le mode **par défaut** de MariaDB. Elle fonctionne selon ce mécanisme :

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVEUR PRIMARY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Client COMMIT                                           │
│  2. Transaction écrite dans InnoDB                          │
│  3. Transaction écrite dans binlog                          │
│  4. COMMIT confirmé au client ✓                             │
│                                                             │
│  Le Primary n'attend PAS le Replica                         │
│                                                             │
└────────────────────┬────────────────────────────────────────┘
                     │ Binary Log Events
                     │ (Transfert réseau asynchrone)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    SERVEUR REPLICA                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  5. IO Thread : Récupère les événements binlog              │
│  6. Écriture dans relay log                                 │
│  7. SQL Thread : Applique les événements                    │
│                                                             │
│  Aucune confirmation envoyée au Primary                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Séquence détaillée** :

1. **Client envoie COMMIT** au Primary
2. **Primary écrit dans InnoDB** (redo log, buffer pool)
3. **Primary écrit dans binary log**
4. **Primary confirme COMMIT au client** ✅ **(sans attendre le Replica)**
5. **IO Thread du Replica** se connecte au Primary et lit le binlog
6. **Relay log** : Les événements sont stockés localement sur le Replica
7. **SQL Thread du Replica** applique les événements du relay log

### Caractéristiques techniques

**Threads impliqués** :

```sql
-- Sur le Primary
SHOW PROCESSLIST;
-- On voit : Binlog Dump thread(s) pour chaque Replica connecté

-- Sur le Replica
SHOW PROCESSLIST;
-- On voit :
-- 1. Slave_IO thread (connexion au Primary)
-- 2. Slave_SQL thread (application des événements)
```

**Flux de données** :

```
Primary Thread        Network         Replica Threads
──────────────        ───────         ───────────────
                                      
Binlog Dump  ─────→  TCP/IP  ─────→  IO Thread
                                         │
                                         ▼
                                     Relay Log
                                         │
                                         ▼
                                     SQL Thread
                                         │
                                         ▼
                                      InnoDB
```

### Avantages

✅ **Performance maximale**
- Latence d'écriture minimale
- Le Primary ne bloque jamais en attendant le Replica
- Throughput optimal pour les charges OLTP

✅ **Simplicité**
- Configuration par défaut, aucun plugin
- Gestion transparente des pannes Replica
- Le Primary continue même si tous les Replicas sont down

✅ **Scalabilité**
- Le Primary peut avoir de nombreux Replicas sans impact
- Ajout/suppression de Replicas à chaud

### Inconvénients et risques

❌ **Perte de données possible**

**Scénario de perte** :
```
Timeline:
─────────────────────────────────────────────────────────►

T0: Client COMMIT transaction T1 sur Primary ✓
T1: Primary confirme au client
T2: Primary écrit dans binlog
T3: Primary CRASH 💥 (avant que le Replica ne récupère T1)
T4: Replica promu en nouveau Primary
    → Transaction T1 est PERDUE ❌
```

**Window de vulnérabilité** :
- Durée entre le COMMIT côté client et la réception par le Replica
- Typiquement 100ms-1s selon la latence réseau et la charge
- Peut atteindre plusieurs secondes sous forte charge

❌ **Cohérence lecture/écriture non garantie**

```sql
-- Application web
-- Serveur 1 (Write to Primary)
INSERT INTO orders (customer_id, total) VALUES (123, 99.99);
COMMIT; -- Confirmé ✓

-- Serveur 2 (Read from Replica) - Quelques ms plus tard
SELECT COUNT(*) FROM orders WHERE customer_id = 123;
-- Peut ne PAS voir la nouvelle commande ! (lag de réplication)
```

❌ **Lag de réplication variable**

- Le Replica peut être en retard de quelques secondes à plusieurs heures
- Facteurs : charge, bande passante réseau, DDL sur grosses tables
- Impact sur les lectures sur Replica (données "stale")

### Configuration

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf

[mysqld]
# Primary configuration
server-id = 1                    # Unique par serveur
log-bin = /var/log/mysql/mariadb-bin
binlog_format = ROW              # Recommandé
max_binlog_size = 100M
expire_logs_days = 7

# Replica configuration
server-id = 2                    # Différent du Primary !
relay-log = /var/log/mysql/relay-bin
log-slave-updates = ON           # Si cascade ou multi-master
read-only = ON                   # Protection écriture accidentelle
```

**Établissement de la réplication** :

```sql
-- Sur le Primary : Créer utilisateur de réplication
CREATE USER 'repl_user'@'%' 
  IDENTIFIED BY 'SecureP@ssw0rd!';
  
GRANT REPLICATION SLAVE ON *.* 
  TO 'repl_user'@'%';

FLUSH PRIVILEGES;

-- Obtenir la position courante
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000042 |     4567 |              |                  |
-- +--------------------+----------+--------------+------------------+

-- Sur le Replica : Configurer la réplication
CHANGE MASTER TO
  MASTER_HOST = '192.168.1.100',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'SecureP@ssw0rd!',
  MASTER_LOG_FILE = 'mariadb-bin.000042',
  MASTER_LOG_POS = 4567;

START SLAVE;

-- Vérifier l'état
SHOW SLAVE STATUS\G
```

### Cas d'usage appropriés

**✓ Scénarios recommandés** :

1. **Scalabilité en lecture**
   - Application à forte charge de lecture
   - Reporting/Analytique sans impact sur production
   - Tolérance à des données légèrement "stale"

2. **Développement/Testing**
   - Environnements non-critiques
   - Coût de perte de données acceptable

3. **Géo-réplication longue distance**
   - Latence réseau élevée (>50ms)
   - Impact performance semi-sync serait prohibitif

**✗ Scénarios déconseillés** :

1. Données financières critiques
2. Conformité réglementaire stricte (RGPD, HIPAA)
3. Applications nécessitant cohérence lecture après écriture

---

## Réplication Semi-synchrone

### Principe de fonctionnement

La réplication semi-synchrone ajoute une **étape de confirmation** :

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVEUR PRIMARY                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Client COMMIT                                           │
│  2. Transaction écrite dans InnoDB                          │
│  3. Transaction écrite dans binlog                          │
│  4. Envoi événements → Replica(s)                           │
│  5. ⏳ ATTENTE ACK d'au moins 1 Replica                     │
│  6. COMMIT confirmé au client ✓                             │
│                                                             │
└────────────────────┬────────────────────────────────────────┘
                     │ Binary Log Events
                     │ + ACK obligatoire
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    SERVEUR REPLICA                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  5. IO Thread : Récupère les événements binlog              │
│  6. Écriture dans relay log                                 │
│  7. Envoi ACK au Primary ✓                                  │
│  8. SQL Thread : Applique les événements                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Différence clé** : Le Primary **attend** qu'au moins un Replica ait confirmé la **réception** des événements binlog avant de confirmer le COMMIT au client.

⚠️ **Important** : Le Replica confirme la **réception** (écriture relay log), PAS l'**application** (exécution SQL). Il peut donc y avoir un léger lag même en semi-sync.

### Garanties de durabilité

**Scénario de crash avec semi-sync** :

```
Timeline:
─────────────────────────────────────────────────────────►

T0: Client COMMIT transaction T1 sur Primary
T1: Primary écrit dans binlog
T2: Primary envoie à Replica
T3: Replica reçoit et écrit relay log
T4: Replica envoie ACK ✓
T5: Primary confirme au client ✓
T6: Primary CRASH 💥
T7: Replica promu en nouveau Primary
    → Transaction T1 est PRÉSENTE ✓ (dans relay log)
    → Aucune perte de données
```

**Garantie** : Si le client reçoit un COMMIT confirmé, au moins un Replica possède la transaction dans son relay log. Promotion de ce Replica = zero data loss.

### Architecture AFTER_SYNC vs AFTER_COMMIT

MariaDB 10.3+ propose deux modes semi-synchrones :

**1. AFTER_SYNC (par défaut, recommandé)** :

```
Séquence:
1. Écriture binlog
2. Synchronisation disque binlog (fsync)
3. Envoi aux Replicas
4. Attente ACK
5. Commit InnoDB
6. Confirmation client
```

**Avantage** : Aucune transaction visible sur le Primary qui ne serait pas sur au moins un Replica. Garantie de cohérence maximale.

**2. AFTER_COMMIT (legacy)** :

```
Séquence:
1. Écriture binlog
2. Commit InnoDB
3. Envoi aux Replicas
4. Attente ACK
5. Confirmation client
```

**Inconvénient** : Fenêtre où une transaction est visible sur Primary mais pas encore ACKée par Replica. En cas de crash, légère incohérence possible.

**Configuration** :

```sql
-- Mode recommandé (AFTER_SYNC)
SET GLOBAL rpl_semi_sync_master_wait_point = 'AFTER_SYNC';
```

### Installation et configuration

**1. Installation du plugin** :

```sql
-- Sur le Primary
INSTALL SONAME 'semisync_master';

-- Vérifier l'installation
SHOW PLUGINS;
-- +-----------------------+--------+--------------------+---------+---------+
-- | Name                  | Status | Type               | Library | License |
-- +-----------------------+--------+--------------------+---------+---------+
-- | rpl_semi_sync_master  | ACTIVE | REPLICATION        | ...     | GPL     |
-- +-----------------------+--------+--------------------+---------+---------+
```

```sql
-- Sur chaque Replica
INSTALL SONAME 'semisync_slave';

SHOW PLUGINS;
-- +-----------------------+--------+--------------------+---------+---------+
-- | Name                  | Status | Type               | Library | License |
-- +-----------------------+--------+--------------------+---------+---------+
-- | rpl_semi_sync_slave   | ACTIVE | REPLICATION        | ...     | GPL     |
-- +-----------------------+--------+--------------------+---------+---------+
```

**2. Activation** :

```sql
-- Primary
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 seconde
SET GLOBAL rpl_semi_sync_master_wait_no_slave = ON;  -- Important !

-- Replica
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- Redémarrer IO thread pour activer
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

**3. Configuration permanente** :

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf

# PRIMARY
[mysqld]
plugin-load-add = semisync_master.so
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_timeout = 1000
rpl_semi_sync_master_wait_point = AFTER_SYNC
rpl_semi_sync_master_wait_no_slave = ON

# REPLICA
[mysqld]
plugin-load-add = semisync_slave.so
rpl_semi_sync_slave_enabled = ON
```

### Paramètres critiques

**rpl_semi_sync_master_timeout**

Temps maximal d'attente d'un ACK avant de basculer en mode asynchrone.

```sql
-- Default: 10000ms (10 secondes)
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 seconde

-- Trop court : Basculements fréquents async ↔ semi-sync
-- Trop long : Latence excessive en cas de problème Replica
```

💡 **Recommandation** : 
- LAN : 500-1000ms
- WAN : 2000-5000ms
- Cross-datacenter : 5000-10000ms

**rpl_semi_sync_master_wait_no_slave**

Comportement si aucun Replica semi-sync n'est disponible.

```sql
-- ON (recommandé) : Continue en mode asynchrone
SET GLOBAL rpl_semi_sync_master_wait_no_slave = ON;

-- OFF : Bloque les écritures ! ⚠️ Dangereux
SET GLOBAL rpl_semi_sync_master_wait_no_slave = OFF;
```

⚠️ **Attention** : Avec `OFF`, si tous les Replicas échouent, le Primary refuse les COMMIT ! À utiliser seulement si durabilité > disponibilité.

**rpl_semi_sync_master_wait_point**

```sql
-- AFTER_SYNC (recommandé, par défaut depuis 10.3)
SET GLOBAL rpl_semi_sync_master_wait_point = 'AFTER_SYNC';

-- AFTER_COMMIT (legacy, pour compatibilité)
SET GLOBAL rpl_semi_sync_master_wait_point = 'AFTER_COMMIT';
```

### Monitoring de la semi-sync

**Variables de statut essentielles** :

```sql
-- État global
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

**Métriques clés** :

| Variable | Signification |
|----------|---------------|
| `Rpl_semi_sync_master_status` | ON = actif, OFF = fallback async |
| `Rpl_semi_sync_master_clients` | Nombre de Replicas semi-sync connectés |
| `Rpl_semi_sync_master_yes_tx` | Transactions confirmées en semi-sync |
| `Rpl_semi_sync_master_no_tx` | Transactions en mode async (timeout) |
| `Rpl_semi_sync_master_wait_sessions` | Sessions actuellement en attente ACK |
| `Rpl_semi_sync_master_avg_wait_time` | Temps moyen d'attente ACK (µs) |

**Exemple de monitoring** :

```sql
-- Taux de succès semi-sync
SELECT 
  (Rpl_semi_sync_master_yes_tx / 
   (Rpl_semi_sync_master_yes_tx + Rpl_semi_sync_master_no_tx)) * 100 
  AS semi_sync_success_rate
FROM (
  SELECT 
    VARIABLE_VALUE AS Rpl_semi_sync_master_yes_tx
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx'
) yes_tx,
(
  SELECT 
    VARIABLE_VALUE AS Rpl_semi_sync_master_no_tx
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx'
) no_tx;
```

**Alertes recommandées** :

```sql
-- Semi-sync désactivé (fallback async)
Rpl_semi_sync_master_status = OFF
→ Alert: Perte de garantie de durabilité

-- Trop de timeouts (>5%)
(Rpl_semi_sync_master_no_tx / total_tx) > 0.05
→ Alert: Problème réseau ou Replica surchargé

-- Latence excessive (>100ms)
Rpl_semi_sync_master_avg_wait_time > 100000  -- µs
→ Alert: Optimiser réseau ou Replica
```

### Impact sur les performances

**Latence d'écriture** :

```
Asynchrone:      5-10ms   ████████
                          ↓
Semi-sync (LAN): 7-15ms   ██████████
                          ↓ +2-5ms
Semi-sync (WAN): 50-150ms ████████████████████████████
                          ↓ +45-140ms
```

**Throughput** :

```sql
-- Benchmark sysbench (OLTP write-only)
-- Serveur : 16 vCPU, 64GB RAM, SSD NVMe
-- Réseau : LAN 10Gbps

Mode           TPS     Latence P95    Latence P99
──────────────────────────────────────────────────
Asynchrone     15,420  8.2ms          12.5ms
Semi-sync      13,850  10.1ms         15.8ms  (-10%)
```

**Facteurs d'impact** :

1. **Latence réseau** : Principal facteur
   - LAN (< 1ms) : Impact minimal (5-10%)
   - WAN (10-50ms) : Impact significatif (20-40%)
   - Intercontinental (100-200ms) : Impact majeur (50-80%)

2. **Nombre de Replicas semi-sync** :
   - Le Primary attend le **premier** ACK seulement
   - 1 Replica ou 10 Replicas : même latence !

3. **Configuration `sync_binlog`** :
   ```sql
   -- sync_binlog=1 (durabilité max, recommandé avec semi-sync)
   SET GLOBAL sync_binlog = 1;  -- fsync à chaque commit
   
   -- sync_binlog=0 (performance max, risque si crash OS)
   SET GLOBAL sync_binlog = 0;  -- fsync async par l'OS
   ```

### Avantages

✅ **Durabilité garantie**
- Zero data loss en cas de crash Primary
- Au moins un Replica possède toutes les transactions confirmées

✅ **Failover simplifié**
- Le Replica semi-sync peut être promu immédiatement
- Pas de calcul de position binlog complexe

✅ **Conformité réglementaire**
- Répond aux exigences de durabilité strictes
- Auditabilité : traçabilité des transactions

### Inconvénients

❌ **Impact performance**
- Latence d'écriture augmentée
- Dépendance à la latence réseau

❌ **Disponibilité réduite**
- Si tous les Replicas semi-sync tombent : fallback async ou blocage (selon config)

❌ **Complexité opérationnelle**
- Configuration supplémentaire
- Monitoring additionnel requis

### Cas d'usage appropriés

**✓ Scénarios recommandés** :

1. **Données financières critiques**
   - Transactions bancaires
   - Paiements e-commerce
   - Comptabilité

2. **Conformité réglementaire**
   - RGPD (données personnelles)
   - HIPAA (santé)
   - PCI-DSS (cartes bancaires)

3. **SLA strict**
   - RPO (Recovery Point Objective) = 0
   - Zero data loss requirement

**✗ Scénarios déconseillés** :

1. Géo-réplication longue distance (latence > 100ms)
2. Charges d'écriture extrêmement élevées (>10K TPS)
3. Budget latence très serré (< 10ms P99)

---

## Comparaison Approfondie

### Tableau récapitulatif

| Critère | Asynchrone | Semi-synchrone |
|---------|------------|----------------|
| **Durabilité** | ⚠️ Possible perte de données | ✅ Zero data loss |
| **Performance écriture** | ✅ Maximale | ⚠️ +5-50ms selon réseau |
| **Throughput** | ✅ Maximum | ⚠️ -5-30% |
| **Disponibilité** | ✅ Indépendant des Replicas | ⚠️ Dépend d'au moins 1 Replica |
| **Cohérence** | ⚠️ Éventuelle | ✅ Forte |
| **Complexité** | ✅ Simple | ⚠️ Moyenne |
| **Failover** | ⚠️ Calcul position requis | ✅ Immédiat |
| **Cas d'usage** | Scalabilité lecture | Durabilité critique |

### Trade-offs selon le théorème CAP

Le [théorème CAP](https://en.wikipedia.org/wiki/CAP_theorem) stipule qu'un système distribué ne peut garantir simultanément que 2 des 3 propriétés :
- **C**onsistency (cohérence)
- **A**vailability (disponibilité)
- **P**artition tolerance (tolérance aux partitions)

**Réplication asynchrone : AP (Availability + Partition tolerance)**

```
Primary     Replica
   │           │
   │  Network  │
   │  Partition│
   │     ╳     │
   ▼           ▼
 Writes     Reads
continue   continue
(Stale data possible)
```

- **Availability** : Le Primary continue même si Replica inaccessible
- **Partition tolerance** : Tolère la partition réseau
- **Consistency** : ⚠️ Sacrifiée (données "stale" sur Replica)

**Réplication semi-synchrone : CP (Consistency + Partition tolerance)**

```
Primary     Replica
   │           │
   │  Network  │
   │  Partition│
   │     ╳     │
   ▼           
 Writes
  BLOCK
(Si wait_no_slave=OFF)
```

- **Consistency** : Garantie par l'ACK obligatoire
- **Partition tolerance** : Tolère la partition (fallback async possible)
- **Availability** : ⚠️ Sacrifiée si `wait_no_slave=OFF`

### Choix du mode selon les contraintes

**Matrice de décision** :

```
                        Latence réseau
                    Low (<10ms)    High (>50ms)
                  ┌─────────────┬──────────────┐
  Durabilité      │             │              │
  Critical        │ Semi-sync   │ Semi-sync    │
  (RPO=0)         │ (optimal)   │ (coûteux)    │
                  ├─────────────┼──────────────┤
  Durabilité      │ Async       │ Async        │
  Tolerant        │ (optimal)   │ (optimal)    │
  (RPO>0)         │             │              │
                  └─────────────┴──────────────┘
```

**Algorithme de décision** :

```
SI (RPO = 0 ET latence_réseau < 20ms) 
  → Semi-synchrone AFTER_SYNC

SINON SI (RPO = 0 ET latence_réseau > 20ms)
  → Semi-synchrone AFTER_SYNC + timeout élevé
  → OU Galera Cluster (chapitre 14)

SINON SI (RPO > 0 ET throughput > 10K TPS)
  → Asynchrone

SINON SI (RPO > 0 ET latence_réseau > 100ms)
  → Asynchrone

SINON
  → Semi-synchrone (meilleure garantie par défaut)
FIN SI
```

---

## Configurations Avancées

### Réplication semi-sync avec multiple ACKs

MariaDB 10.6+ permet d'exiger des ACK de **plusieurs** Replicas :

```sql
-- Attendre ACK de 2 Replicas minimum (quorum)
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;
```

**Use case** : Garantie de durabilité encore plus forte, tolérance à la panne d'un Replica.

⚠️ **Trade-off** : Latence augmentée (attente du 2ème Replica le plus lent).

### Hybrid mode : Semi-sync sur un sous-ensemble

```sql
-- Activer semi-sync uniquement sur les Replicas critiques
-- Replica 1 : Semi-sync (datacenter principal)
-- Replica 2 : Async (datacenter distant)
-- Replica 3 : Async (reporting)

-- Configuration Replica 1
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- Configuration Replicas 2-3
SET GLOBAL rpl_semi_sync_slave_enabled = OFF;
```

### Monitoring complet

**Script de monitoring production-ready** :

```sql
-- Créer une vue de monitoring
CREATE OR REPLACE VIEW replication_health AS
SELECT 
  'semi_sync_status' AS metric,
  VARIABLE_VALUE AS value
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_status'

UNION ALL

SELECT 
  'semi_sync_clients',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients'

UNION ALL

SELECT 
  'semi_sync_avg_wait_ms',
  ROUND(VARIABLE_VALUE / 1000, 2)  -- µs → ms
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_avg_wait_time'

UNION ALL

SELECT 
  'semi_sync_success_rate_pct',
  ROUND(
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx') /
    NULLIF((
      SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx'
    ) + (
      SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx'
    ), 0) * 100,
    2
  );

-- Utilisation
SELECT * FROM replication_health;
-- +-----------------------------+----------+
-- | metric                      | value    |
-- +-----------------------------+----------+
-- | semi_sync_status            | ON       |
-- | semi_sync_clients           | 2        |
-- | semi_sync_avg_wait_ms       | 8.45     |
-- | semi_sync_success_rate_pct  | 98.76    |
-- +-----------------------------+----------+
```

**Intégration Prometheus** :

```yaml
# mysqld_exporter config
scrape_configs:
  - job_name: 'mariadb'
    static_configs:
      - targets: ['localhost:9104']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        
# Queries exportées
queries:
  - name: mariadb_semi_sync_status
    help: "Semi-sync replication status (1=ON, 0=OFF)"
    query: |
      SELECT 
        CASE WHEN VARIABLE_VALUE = 'ON' THEN 1 ELSE 0 END AS value
      FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_status';
      
  - name: mariadb_semi_sync_avg_wait_seconds
    help: "Average semi-sync wait time in seconds"
    query: |
      SELECT VARIABLE_VALUE / 1000000 AS value
      FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_avg_wait_time';
```

---

## Bonnes Pratiques de Production

### 1. Toujours tester le fallback async

```bash
#!/bin/bash
# Script de test : Que se passe-t-il si tous les Replicas tombent ?

echo "1. État initial semi-sync"
mariadb -e "SHOW STATUS LIKE 'Rpl_semi_sync_master_status';"

echo "2. Arrêt de tous les Replicas"
ssh replica1 "systemctl stop mariadb"
ssh replica2 "systemctl stop mariadb"

echo "3. Test écriture (doit continuer en mode async)"
mariadb test -e "INSERT INTO test_table VALUES (NOW());"

echo "4. Vérifier basculement automatique"
mariadb -e "SHOW STATUS LIKE 'Rpl_semi_sync_master_status';"
# Doit afficher OFF

echo "5. Redémarrage Replicas"
ssh replica1 "systemctl start mariadb"
mariadb -e "SHOW STATUS LIKE 'Rpl_semi_sync_master_status';"
# Doit afficher ON
```

### 2. Dimensionner le timeout correctement

```sql
-- Mesurer la latence réseau réelle
-- Depuis le Primary
DO BENCHMARK(1000000, 1);  -- Warm-up

-- Test ping-pong avec Replica
-- Script à exécuter en boucle
CREATE TABLE ping_test (ts TIMESTAMP(6));
INSERT INTO ping_test VALUES (NOW(6));

-- Sur Replica, mesurer le lag
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000 AS lag_ms
FROM ping_test 
ORDER BY ts DESC 
LIMIT 1;

-- Définir timeout = P95 latence × 2
SET GLOBAL rpl_semi_sync_master_timeout = 
  (SELECT CEIL(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY lag_ms) * 2)
   FROM lag_measurements);
```

### 3. Monitoring continu

```sql
-- Créer une alerte si semi-sync se désactive
CREATE EVENT check_semi_sync_status
ON SCHEDULE EVERY 1 MINUTE
DO
  BEGIN
    DECLARE semi_sync_on BOOLEAN;
    
    SELECT VARIABLE_VALUE = 'ON' INTO semi_sync_on
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_status';
    
    IF NOT semi_sync_on THEN
      -- Logger dans une table
      INSERT INTO alerts (severity, message, created_at)
      VALUES ('CRITICAL', 'Semi-sync replication DISABLED', NOW());
      
      -- Ou envoyer alerte externe (webhook, etc.)
    END IF;
  END;
```

### 4. Documentation de la topologie

```sql
-- Documenter la configuration dans une table
CREATE TABLE replication_topology (
  host VARCHAR(255) PRIMARY KEY,
  role ENUM('primary', 'replica'),
  replication_mode ENUM('async', 'semi-sync'),
  datacenter VARCHAR(100),
  purpose VARCHAR(255),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO replication_topology VALUES
('db-primary-01.prod', 'primary', 'semi-sync', 'DC1', 'Production writes'),
('db-replica-01.prod', 'replica', 'semi-sync', 'DC1', 'Production reads + HA'),
('db-replica-02.prod', 'replica', 'async', 'DC2', 'DR site'),
('db-replica-03.prod', 'replica', 'async', 'DC1', 'Reporting/Analytics');
```

---

## ✅ Points clés à retenir

1. **Réplication asynchrone** : Mode par défaut, performance maximale, mais perte de données possible en cas de crash

2. **Réplication semi-synchrone** : Garantit zero data loss, ajoute une latence réseau mais essentielle pour données critiques

3. **AFTER_SYNC vs AFTER_COMMIT** : Toujours utiliser AFTER_SYNC pour la cohérence maximale

4. **Timeout crucial** : Dimensionner `rpl_semi_sync_master_timeout` selon latence réseau réelle (P95 × 2)

5. **Théorème CAP** : Async privilégie disponibilité (AP), semi-sync privilégie cohérence (CP)

6. **Monitoring essentiel** : Surveiller `Rpl_semi_sync_master_status`, taux de succès, et latence moyenne

7. **Fallback automatique** : Semi-sync bascule en async si timeout, assure disponibilité

8. **Impact performance** : -5 à -30% throughput selon latence réseau, tester en conditions réelles

9. **Multiple ACKs** : `rpl_semi_sync_master_wait_for_slave_count > 1` pour tolérance à la panne d'un Replica

10. **Use case driven** : Choisir selon contraintes métier (RPO, RTO, throughput, latence)

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Replication Overview](https://mariadb.com/kb/en/replication-overview/)
- [📖 Semisynchronous Replication](https://mariadb.com/kb/en/semisynchronous-replication/)
- [📖 Replication and Binary Log System Variables](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/)

### Articles techniques

- [🔗 Understanding Semi-Synchronous Replication](https://mariadb.com/resources/blog/understanding-mariadb-semisynchronous-replication/)
- [🔗 CAP Theorem and Databases](https://www.ibm.com/topics/cap-theorem)
- [🔗 AFTER_SYNC vs AFTER_COMMIT](https://jfg-mysql.blogspot.com/2016/09/semisync-after-sync-vs-after-commit.html)

### Outils

- **pt-heartbeat** (Percona Toolkit) : Mesure précise du lag
- **Orchestrator** : Gestion automatisée de topologies
- **PrometheusDB/Grafana** : Monitoring et alerting

---

## ➡️ Section suivante

**13.2 Réplication Master-Slave (Source-Replica)** : Configuration complète étape par étape d'une topologie de réplication, création de l'utilisateur de réplication, établissement de la connexion avec `CHANGE MASTER TO`, et sécurisation de l'environnement.

Vous allez mettre en pratique les concepts de cette section !

---


⏭️ [Réplication Master-Slave (Source-Replica)](/13-replication/02-replication-master-slave.md)
