🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.9 Réplication Semi-Synchrone

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Sections 13.1 à 13.8 (Réplication complète)
> - Compréhension des garanties de durabilité
> - Notions de théorème CAP et consensus distribué
> - Expérience avec les contraintes de latence réseau

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les différences entre réplication asynchrone et semi-synchrone
- Configurer la réplication semi-synchrone sur Primary et Replicas
- Choisir entre modes AFTER_SYNC et AFTER_COMMIT
- Optimiser les paramètres de timeout et de quorum
- Mesurer et analyser l'impact sur les performances
- Monitorer efficacement la réplication semi-synchrone
- Gérer les scénarios de fallback automatique
- Évaluer le trade-off performance vs durabilité

---

## Introduction

La **réplication semi-synchrone** est un mode de réplication qui garantit qu'**au moins un Replica** a reçu les événements binlog avant que la transaction soit confirmée au client. C'est un compromis entre :

- **Asynchrone** : Performance maximale, risque de perte de données
- **Synchrone** : Durabilité maximale, performance réduite

### Comparaison des modes

```
┌─────────────────────────────────────────────────────────────┐
│              RÉPLICATION ASYNCHRONE                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client          Primary              Replica               │
│    │               │                     │                  │
│    ├─ COMMIT ─────>│                     │                  │
│    │               ├─ Write InnoDB       │                  │
│    │               ├─ Write binlog       │                  │
│    │<─── OK ───────┤                     │                  │
│    │               │                     │                  │
│    │               ├─ Send binlog ──────>│                  │
│    │               │                     ├─ Receive         │
│    │               │                     ├─ Apply           │
│                                                             │
│  Risque: Si Primary crash après "OK" mais avant             │
│          "Send binlog", données perdues ❌                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│            RÉPLICATION SEMI-SYNCHRONE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client          Primary              Replica               │
│    │               │                     │                  │
│    ├─ COMMIT ─────>│                     │                  │
│    │               ├─ Write InnoDB       │                  │
│    │               ├─ Write binlog       │                  │
│    │               ├─ Send binlog ──────>│                  │
│    │               │                     ├─ Receive         │
│    │               │<─── ACK ────────────┤                  │
│    │<─── OK ───────┤                     │                  │
│    │               │                     ├─ Apply           │
│                                                             │
│  Garantie: Replica a reçu binlog AVANT "OK" au client ✓     │
│            → Zero data loss en cas de crash Primary         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Garanties de durabilité

**Réplication asynchrone** :

```
RPO (Recovery Point Objective) = Lag de réplication

Exemple:
- Lag actuel: 2 secondes
- Primary crash
- Perte: ~2 secondes de transactions

RPO > 0 ❌
```

**Réplication semi-synchrone** :

```
RPO = 0 (Zero data loss)

Exemple:
- Transaction confirmée au client
- Garantie: Au moins 1 Replica a le binlog
- Primary crash
- Perte: ZERO ✓

RPO = 0 ✅
```

### Théorème CAP

**CAP Theorem** : Dans un système distribué, vous ne pouvez garantir que 2 des 3 propriétés :
- **C** (Consistency) : Cohérence des données
- **A** (Availability) : Disponibilité du système
- **P** (Partition tolerance) : Tolérance aux pannes réseau

**Réplication asynchrone** :
```
Choix: AP (Availability + Partition tolerance)
- Disponibilité maximale
- Tolérance aux pannes réseau
- Sacrifice: Cohérence (lag possible)
```

**Réplication semi-synchrone** :
```
Choix: CP (Consistency + Partition tolerance)
- Cohérence forte (RPO=0)
- Tolérance aux pannes réseau
- Sacrifice: Disponibilité (peut bloquer si aucun Replica ACK)
```

---

## Configuration de Base

### Activation des plugins

**Sur le Primary** :

```sql
-- Installer le plugin semi-sync master
INSTALL SONAME 'semisync_master';

-- Vérifier installation
SELECT * FROM information_schema.PLUGINS 
WHERE PLUGIN_NAME LIKE 'rpl_semi_sync%';
-- +--------------------------+---------------+
-- | PLUGIN_NAME              | PLUGIN_STATUS |
-- +--------------------------+---------------+
-- | rpl_semi_sync_master     | ACTIVE        |
-- +--------------------------+---------------+

-- Activer semi-sync
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Vérifier
SELECT @@rpl_semi_sync_master_enabled;
-- +----------------------------------+
-- | @@rpl_semi_sync_master_enabled   |
-- +----------------------------------+
-- |                                1 |
-- +----------------------------------+
```

**Sur les Replicas** :

```sql
-- Installer le plugin semi-sync slave
INSTALL SONAME 'semisync_slave';

-- Vérifier
SELECT * FROM information_schema.PLUGINS 
WHERE PLUGIN_NAME LIKE 'rpl_semi_sync%';
-- +--------------------------+---------------+
-- | PLUGIN_NAME              | PLUGIN_STATUS |
-- +--------------------------+---------------+
-- | rpl_semi_sync_slave      | ACTIVE        |
-- +--------------------------+---------------+

-- Activer semi-sync
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- Redémarrer IO thread pour prise en compte
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;

-- Vérifier
SELECT @@rpl_semi_sync_slave_enabled;
-- +--------------------------------+
-- | @@rpl_semi_sync_slave_enabled  |
-- +--------------------------------+
-- |                              1 |
-- +--------------------------------+
```

### Configuration persistante

**my.cnf sur Primary** :

```ini
[mysqld]
# Plugin semi-sync master
plugin-load-add = semisync_master.so
rpl_semi_sync_master_enabled = ON

# Timeout (ms) avant fallback en async
rpl_semi_sync_master_timeout = 1000

# Comportement si aucun Replica ACK
rpl_semi_sync_master_wait_no_slave = ON

# Point d'attente ACK (AFTER_SYNC recommandé)
rpl_semi_sync_master_wait_point = AFTER_SYNC

# Nombre de Replicas devant ACK (quorum)
rpl_semi_sync_master_wait_for_slave_count = 1
```

**my.cnf sur Replicas** :

```ini
[mysqld]
# Plugin semi-sync slave
plugin-load-add = semisync_slave.so
rpl_semi_sync_slave_enabled = ON
```

**Redémarrage** :

```bash
# Primary
systemctl restart mariadb

# Replicas
systemctl restart mariadb
```

---

## Modes AFTER_SYNC vs AFTER_COMMIT

MariaDB supporte deux modes semi-synchrones :

### Mode AFTER_SYNC (recommandé)

```sql
SET GLOBAL rpl_semi_sync_master_wait_point = AFTER_SYNC;
```

**Séquence d'exécution** :

```
Client sends COMMIT
      ↓
1. Write to InnoDB redo log
2. Sync to disk (if innodb_flush_log_at_trx_commit=1)
3. Write to binlog
4. Sync binlog to disk (if sync_binlog=1)
      ↓
5. Send binlog to Replica(s) ⬅─┐ WAIT ACK
6. Wait for ACK from Replica  ─┘ (Semi-sync wait point)
      ↓
7. COMMIT in storage engine
8. Return OK to client
```

**Caractéristiques** :

✅ **Avantages** :
- **Données cohérentes** : Replica a binlog AVANT commit storage engine
- **Failover safe** : Si Primary crash après ACK, Replica a tout
- **Pas de phantom reads** : Transaction visible seulement après ACK

❌ **Inconvénients** :
- Latence légèrement plus élevée (attente ACK avant commit)

**Scénario de crash** :

```
T0: Client COMMIT
T1: Binlog écrit et envoyé
T2: Replica ACK reçu
T3: COMMIT storage engine
T4: Return OK client

Crash à T3.5 (après ACK, avant OK client):
- Replica a le binlog ✓
- Primary: Transaction peut être en cours de commit
- Failover: Replica promu, transaction présente ✓
- Résultat: ZERO data loss ✓
```

### Mode AFTER_COMMIT (legacy)

```sql
SET GLOBAL rpl_semi_sync_master_wait_point = AFTER_COMMIT;
```

**Séquence d'exécution** :

```
Client sends COMMIT
      ↓
1. Write to InnoDB redo log
2. Sync to disk
3. Write to binlog
4. Sync binlog to disk
5. COMMIT in storage engine ⬅──┐ Commit AVANT ACK
      ↓                        │
6. Send binlog to Replica(s) ⬅─┤ WAIT ACK
7. Wait for ACK from Replica  ─┘ (Semi-sync wait point)
      ↓
8. Return OK to client
```

**Caractéristiques** :

✅ **Avantages** :
- Compatible MySQL 5.5-5.6
- Commit storage engine plus rapide

❌ **Inconvénients** :
- **Phantom reads possibles** : Transaction visible sur Primary avant ACK Replica
- Moins safe en cas de crash entre commit et ACK

**Scénario problématique** :

```
T0: Client COMMIT
T1: Binlog écrit
T2: COMMIT storage engine ⬅ Transaction VISIBLE sur Primary
T3: Binlog envoyé à Replica
T4: En attente ACK...

Autre client lit sur Primary:
- Voit la transaction (commitée)

Crash Primary avant ACK:
- Replica N'A PAS le binlog ❌
- Failover: Transaction perdue
- Client qui avait lu la donnée: Incohérence !
```

**Recommandation** : **Toujours utiliser AFTER_SYNC** sauf contrainte de compatibilité MySQL legacy.

---

## Paramètres de Configuration

### rpl_semi_sync_master_timeout

**Définition** : Temps maximum (ms) d'attente ACK avant fallback en async

```sql
-- Par défaut: 10000 (10 secondes)
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 seconde
```

**Impact** :

```
Timeout COURT (500ms):
✅ Fallback rapide en async si problème réseau
❌ Risque de fallback trop fréquent (faux positifs)

Timeout LONG (10000ms):
✅ Tolère pics de latence réseau
❌ Blocage prolongé si Replica vraiment down
```

**Tuning** :

```sql
-- Mesurer latence réseau P95
-- Sur Primary:
SELECT 
  AVG(rpl_semi_sync_master_tx_avg_wait_time) AS avg_wait_ms,
  MAX(rpl_semi_sync_master_tx_avg_wait_time) AS max_wait_ms
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_tx_avg_wait_time';

-- Règle empirique:
-- timeout = P95_latency_network * 2
-- Exemple: P95 = 50ms → timeout = 100-200ms

SET GLOBAL rpl_semi_sync_master_timeout = 200;
```

### rpl_semi_sync_master_wait_no_slave

**Définition** : Comportement si AUCUN Replica ACK disponible

```sql
-- ON (défaut): Fallback en async
SET GLOBAL rpl_semi_sync_master_wait_no_slave = ON;

-- OFF: Attendre indéfiniment (BLOQUER)
SET GLOBAL rpl_semi_sync_master_wait_no_slave = OFF;
```

**Comparaison** :

| Mode | Comportement | Use case |
|------|--------------|----------|
| **ON** | Fallback async si aucun Replica | Production (disponibilité prioritaire) |
| **OFF** | Bloquer écritures si aucun Replica | Ultra-critique (durabilité absolue) |

**Exemple avec OFF** :

```sql
SET GLOBAL rpl_semi_sync_master_wait_no_slave = OFF;

-- Scénario: Tous les Replicas tombent
-- Client tente INSERT:
INSERT INTO users (name) VALUES ('Alice');
-- ⏳ BLOQUÉ indéfiniment
-- Attente ACK qui ne viendra jamais

-- Solution: Promouvoir un Replica ou passer en async
```

**Recommandation** :
- **Production générale** : `ON` (fallback)
- **Financier/critique** : `OFF` + monitoring strict

### rpl_semi_sync_master_wait_for_slave_count

**Définition** : Nombre minimum de Replicas devant ACK (quorum)

```sql
-- Par défaut: 1
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;

-- Quorum de 2 Replicas
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;
```

**Impact durabilité** :

```
Count = 1:
- Garantie: Au moins 1 Replica a le binlog
- Risque: Si ce Replica unique crash aussi, perte possible

Count = 2:
- Garantie: Au moins 2 Replicas ont le binlog
- Risque réduit: Très peu probable que 2 Replicas crashent
- Performance: Latence = MAX(ACK_replica1, ACK_replica2)
```

**Architecture pour count=2** :

```
                Primary
                   │
      ┌────────────┼────────────┐
      │            │            │
      ▼            ▼            ▼
   Replica1     Replica2     Replica3
      │            │            │
      ACK ─────────┤            │
                   ACK ─────────┘

Primary attend ACK de 2 Replicas (Replica1 et Replica2)
Replica3 peut être en async ou semi-sync (bonus)
```

**Tuning** :

```
Topologie avec 2 Replicas:
→ count = 1 (au moins 1 sur 2)

Topologie avec 3+ Replicas:
→ count = 2 (quorum majoritaire)

Topologie avec 1 Replica:
→ count = 1 (pas le choix)
```

### rpl_semi_sync_master_trace_level

**Définition** : Niveau de verbosité logs semi-sync

```sql
-- Niveaux:
-- 1  : Errors only
-- 16 : General info
-- 32 : Detailed info
-- 64 : Trace level (debug)

SET GLOBAL rpl_semi_sync_master_trace_level = 32;

-- Logs dans error log:
-- [Note] Semi-sync replication enabled on master
-- [Note] Semi-sync: 1 replicas acknowledged
```

---

## Monitoring Semi-Sync

### Variables de statut Primary

```sql
-- Sur Primary
SHOW STATUS LIKE 'Rpl_semi_sync%';

-- Variables critiques:
-- +-----------------------------------------+-------+
-- | Variable_name                           | Value |
-- +-----------------------------------------+-------+
-- | Rpl_semi_sync_master_status             | ON    | ← Active ou fallback async
-- | Rpl_semi_sync_master_clients            | 2     | ← Nombre Replicas semi-sync
-- | Rpl_semi_sync_master_yes_tx             | 15420 | ← Transactions ACK reçu
-- | Rpl_semi_sync_master_no_tx              | 12    | ← Transactions NO ACK (timeout)
-- | Rpl_semi_sync_master_tx_wait_time       | 234   | ← Temps total attente (ms)
-- | Rpl_semi_sync_master_tx_avg_wait_time   | 0.015 | ← Temps moyen attente (ms)
-- | Rpl_semi_sync_master_net_wait_time      | 180   | ← Temps réseau (ms)
-- | Rpl_semi_sync_master_timefunc_failures  | 0     | ← Erreurs timeout
-- | Rpl_semi_sync_master_wait_sessions      | 0     | ← Transactions en attente ACK
-- +-----------------------------------------+-------+
```

**Interprétation** :

```sql
-- Taux de succès semi-sync
SELECT 
  @yes := VARIABLE_VALUE AS yes_tx
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx';

SELECT 
  @no := VARIABLE_VALUE AS no_tx
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx';

SELECT 
  @yes / (@yes + @no) * 100 AS success_rate_pct;
-- +-------------------+
-- | success_rate_pct  |
-- +-------------------+
-- |            99.92  | ← Excellent (>99%)
-- +-------------------+

-- Si < 95% : Investiguer (timeout trop court, réseau, Replicas lents)
```

**Latence moyenne** :

```sql
SELECT 
  VARIABLE_VALUE AS avg_wait_ms
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_tx_avg_wait_time';
-- +-------------+
-- | avg_wait_ms |
-- +-------------+
-- |       0.015 | ← 0.015ms = Excellent (LAN)
-- +-------------+

-- Attendu:
-- LAN: 0.01-0.5ms
-- WAN même région: 1-10ms
-- WAN inter-continental: 50-200ms
```

### Variables de statut Replica

```sql
-- Sur Replica
SHOW STATUS LIKE 'Rpl_semi_sync_slave%';

-- +--------------------------+--------+
-- | Variable_name            | Value  |
-- +--------------------------+--------+
-- | Rpl_semi_sync_slave_status | ON   | ← Active
-- +--------------------------+--------+
```

### Requête de monitoring consolidée

```sql
-- Sur Primary
SELECT 
  'Semi-Sync Status' AS metric,
  CASE 
    WHEN status.rpl_status = 'ON' AND clients.replica_count > 0 
      THEN 'ACTIVE'
    WHEN status.rpl_status = 'ON' AND clients.replica_count = 0 
      THEN 'ENABLED (no replicas)'
    ELSE 'ASYNC (fallback)'
  END AS value
FROM 
  (SELECT VARIABLE_VALUE AS rpl_status 
   FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_status') status,
  (SELECT VARIABLE_VALUE AS replica_count 
   FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients') clients

UNION ALL

SELECT 
  'Connected Replicas',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients'

UNION ALL

SELECT 
  'Success Rate %',
  ROUND(
    yes.yes_tx / (yes.yes_tx + no.no_tx) * 100, 
    2
  )
FROM 
  (SELECT VARIABLE_VALUE AS yes_tx 
   FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx') yes,
  (SELECT VARIABLE_VALUE AS no_tx 
   FROM information_schema.GLOBAL_STATUS 
   WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx') no

UNION ALL

SELECT 
  'Avg Wait Time (ms)',
  VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_tx_avg_wait_time';

-- +----------------------+------------------+
-- | metric               | value            |
-- +----------------------+------------------+
-- | Semi-Sync Status     | ACTIVE           |
-- | Connected Replicas   | 2                |
-- | Success Rate %       | 99.92            |
-- | Avg Wait Time (ms)   | 0.015            |
-- +----------------------+------------------+
```

### Alerting

**Prometheus rules** :

```yaml
groups:
  - name: mariadb_semisync
    rules:
      - alert: SemiSyncDisabled
        expr: mysql_global_status_rpl_semi_sync_master_status == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Semi-sync replication disabled on {{ $labels.instance }}"
          
      - alert: SemiSyncLowSuccessRate
        expr: |
          (mysql_global_status_rpl_semi_sync_master_yes_tx / 
           (mysql_global_status_rpl_semi_sync_master_yes_tx + 
            mysql_global_status_rpl_semi_sync_master_no_tx)) < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Semi-sync success rate < 95% on {{ $labels.instance }}"
          
      - alert: SemiSyncHighLatency
        expr: mysql_global_status_rpl_semi_sync_master_tx_avg_wait_time > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Semi-sync average wait time > 100ms on {{ $labels.instance }}"
```

**Script Bash de monitoring** :

```bash
#!/bin/bash
# semisync_monitor.sh

STATUS=$(mysql -N -e "
  SELECT VARIABLE_VALUE 
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_status'
")

CLIENTS=$(mysql -N -e "
  SELECT VARIABLE_VALUE 
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients'
")

YES_TX=$(mysql -N -e "
  SELECT VARIABLE_VALUE 
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx'
")

NO_TX=$(mysql -N -e "
  SELECT VARIABLE_VALUE 
  FROM information_schema.GLOBAL_STATUS 
  WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx'
")

if [ "$STATUS" != "ON" ]; then
  echo "CRITICAL: Semi-sync disabled (fallback to async)"
  exit 2
fi

if [ "$CLIENTS" -eq 0 ]; then
  echo "WARNING: Semi-sync enabled but no replicas connected"
  exit 1
fi

SUCCESS_RATE=$(awk "BEGIN {print ($YES_TX / ($YES_TX + $NO_TX)) * 100}")

if (( $(echo "$SUCCESS_RATE < 95" | bc -l) )); then
  echo "WARNING: Semi-sync success rate ${SUCCESS_RATE}% < 95%"
  exit 1
fi

echo "OK: Semi-sync active, $CLIENTS replicas, ${SUCCESS_RATE}% success"
exit 0
```

---

## Impact Performance

### Overhead théorique

**Latence ajoutée** :

```
Latence_semi_sync = Latence_async + Temps_attente_ACK

Où Temps_attente_ACK = 
  Latence_réseau_aller +  # Primary → Replica
  Temps_traitement_Replica +  # Minime (write relay log)
  Latence_réseau_retour   # Replica → Primary

Total typique:
- LAN (< 1ms latency): +0.5-2ms
- WAN même région (5ms): +10-15ms
- WAN inter-continental (100ms): +200-250ms
```

### Benchmarks

**Configuration test** :
- Hardware: 16 vCPU, 64GB RAM, NVMe SSD
- Workload: Sysbench OLTP Mixed
- Threads: 16
- Durée: 300 secondes

**Résultats** :

| Mode | TPS | Latence P95 | Latence P99 | vs Async |
|------|-----|-------------|-------------|----------|
| **Async** | 15,420 | 8.2ms | 12.5ms | Baseline |
| **Semi-sync (LAN)** | 13,850 | 10.1ms | 15.8ms | -10% TPS |
| **Semi-sync (WAN 50ms)** | 4,200 | 125ms | 180ms | -73% TPS |

**Observations** :

```
LAN (latence < 1ms):
- Impact modéré: -5 à -15% TPS
- Latence +2-4ms
- Acceptable pour la plupart des workloads

WAN (latence > 50ms):
- Impact sévère: -50 à -80% TPS
- Latence +100-200ms
- Inacceptable pour applications interactives
```

### Facteurs d'impact

**1. Latence réseau** (facteur principal)

```sql
-- Mesurer latence réseau
-- Sur Primary:
SELECT 
  NOW(6) AS start_time,
  SLEEP(0) AS dummy;

-- Sur Replica (simultanément):
SELECT 
  NOW(6) AS replica_time;

-- Comparer timestamps (NTP synchronisé)
-- Différence ≈ latence réseau one-way
```

**2. Charge réseau**

```bash
# Vérifier saturation
iftop -i eth0

# Si bandwidth > 80% capacité:
# → Considérer upgrade réseau ou compression binlog
```

**3. Configuration sync_binlog**

```sql
-- Sur Primary
SELECT @@sync_binlog;

-- sync_binlog = 1: Fsync chaque transaction
-- → Latence disque ajoutée
-- → Sécurité maximale

-- sync_binlog = 0: Pas de fsync (OS cache)
-- → Latence réduite
-- → Risque en crash OS
```

**4. Nombre de Replicas attendus**

```sql
-- wait_for_slave_count = 1
-- Latence = ACK du Replica le plus rapide

-- wait_for_slave_count = 2
-- Latence = ACK du 2ème Replica (plus lent des 2)
-- Impact: Latence × 1.5-2
```

### Optimisations

**1. Parallélisation connections Replicas**

```sql
-- Primary envoie binlog en parallèle
-- Chaque Replica ACK indépendamment
-- Pas de sérialisation
```

**2. Compression binlog (WAN)**

```sql
-- Sur Replica
CHANGE MASTER TO
  MASTER_COMPRESSION_ALGORITHMS = 'zstd';

-- Réduit bande passante
-- Légère latence CPU (compression/décompression)
-- Net positif sur WAN
```

**3. Tuning buffer sizes**

```ini
[mysqld]
# Sur Primary
max_binlog_cache_size = 1G
binlog_cache_size = 4M

# Sur Replica
slave_net_timeout = 120
```

**4. Hardware réseau**

```
- NIC 10Gbps minimum
- Latence réseau < 1ms (LAN)
- MTU 9000 (Jumbo Frames) si supporté
```

---

## Scénarios Avancés

### Fallback automatique

**Scénario** : Tous les Replicas deviennent indisponibles

```
Timeline:

T0: Semi-sync actif, 2 Replicas
    rpl_semi_sync_master_status = ON

T1: Replica1 crash
    rpl_semi_sync_master_clients = 1
    Still OK (1 Replica restant)

T2: Replica2 crash
    rpl_semi_sync_master_clients = 0
    
T3: Nouveau COMMIT sur Primary
    Attente ACK...
    Timeout après rpl_semi_sync_master_timeout (1000ms)
    
T4: Fallback automatique en ASYNC
    rpl_semi_sync_master_status = OFF
    
T5: Transaction continue en mode async
    Performance restaurée
    
T6: Replica2 revient
    rpl_semi_sync_master_clients = 1
    rpl_semi_sync_master_status = ON ← Reactivation auto
```

**Configuration** :

```sql
-- Activer fallback
SET GLOBAL rpl_semi_sync_master_wait_no_slave = ON;

-- Désactiver fallback (bloquer)
SET GLOBAL rpl_semi_sync_master_wait_no_slave = OFF;
```

### Topologies hybrides

**Cas d'usage** : Semi-sync sur certains Replicas, async sur d'autres

```
Architecture:

            Primary
               │
               ├─────────────┬─────────────┐
               │             │             │
               ▼             ▼             ▼
          Replica1      Replica2      Replica3
          (Semi-sync)   (Semi-sync)   (Async)
          Critical HA   Critical HA   Reporting
```

**Configuration** :

```sql
-- Sur Replica1 et Replica2
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD; START SLAVE IO_THREAD;

-- Sur Replica3
SET GLOBAL rpl_semi_sync_slave_enabled = OFF;
-- Reste en async

-- Sur Primary
SELECT 
  VARIABLE_VALUE AS semisync_replicas
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients';
-- +--------------------+
-- | semisync_replicas  |
-- +--------------------+
-- |                  2 | ← Replica1 et Replica2
-- +--------------------+
```

### Semi-sync avec cascade

**Architecture** :

```
Primary (Semi-sync) → Relay (Semi-sync) → Replicas (Async)
```

**Configuration** :

```sql
-- Sur Primary
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Sur Relay
SET GLOBAL rpl_semi_sync_slave_enabled = ON;   -- Vers Primary
SET GLOBAL rpl_semi_sync_master_enabled = OFF; -- Vers Replicas (async)

-- Résultat:
-- Primary → Relay: Semi-sync (RPO=0)
-- Relay → Replicas: Async (performance)
```

---

## Troubleshooting

### Problème 1 : Semi-sync ne s'active pas

```sql
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';
-- +---------------------------------+-------+
-- | Variable_name                   | Value |
-- +---------------------------------+-------+
-- | Rpl_semi_sync_master_status     | OFF   |
-- +---------------------------------+-------+
```

**Diagnostic** :

```sql
-- Vérifier plugin installé
SELECT * FROM information_schema.PLUGINS 
WHERE PLUGIN_NAME = 'rpl_semi_sync_master';

-- Vérifier enabled
SELECT @@rpl_semi_sync_master_enabled;

-- Vérifier Replicas
SELECT 
  VARIABLE_VALUE AS replica_count
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_clients';
```

**Causes et solutions** :

```sql
-- Cause 1: Plugin non installé
INSTALL SONAME 'semisync_master';

-- Cause 2: Pas activé
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Cause 3: Aucun Replica semi-sync
-- Sur Replica:
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD; START SLAVE IO_THREAD;
```

### Problème 2 : Taux de succès faible

```sql
SELECT 
  yes.yes_tx / (yes.yes_tx + no.no_tx) * 100 AS success_rate
FROM 
  (SELECT VARIABLE_VALUE AS yes_tx FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_yes_tx') yes,
  (SELECT VARIABLE_VALUE AS no_tx FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_no_tx') no;
-- +--------------+
-- | success_rate |
-- +--------------+
-- |        75.3  | ← Faible !
-- +--------------+
```

**Causes** :

1. **Timeout trop court**

```sql
SELECT @@rpl_semi_sync_master_timeout;
-- 100ms (trop court pour WAN)

-- Solution: Augmenter
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 seconde
```

2. **Latence réseau élevée**

```bash
# Mesurer depuis Primary vers Replica
ping replica.example.com
# 150ms (élevé)

# Solution:
# - Augmenter timeout
# - Activer compression
# - Évaluer si semi-sync approprié pour ce lien
```

3. **Replica lent (CPU/I/O)**

```sql
-- Sur Replica: Vérifier charge
SHOW PROCESSLIST;

-- CPU élevé? I/O saturé?
-- Solution: Upgrade Replica ou désactiver semi-sync sur ce Replica
```

### Problème 3 : Latence élevée

```sql
SELECT 
  VARIABLE_VALUE AS avg_wait_ms
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Rpl_semi_sync_master_tx_avg_wait_time';
-- +-------------+
-- | avg_wait_ms |
-- +-------------+
-- |       250   | ← Très élevé !
-- +-------------+
```

**Solutions** :

```sql
-- 1. Vérifier latence réseau
-- Si WAN: Évaluer si acceptable ou passer en async

-- 2. Réduire wait_for_slave_count
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;

-- 3. Utiliser Replicas plus proches (même datacenter)

-- 4. Si inacceptable: Désactiver semi-sync
SET GLOBAL rpl_semi_sync_master_enabled = OFF;
```

---

## ✅ Points clés à retenir

1. **RPO=0** : Semi-sync garantit zero data loss (Replica a binlog avant ACK client)

2. **AFTER_SYNC** : Mode recommandé (vs AFTER_COMMIT legacy)

3. **Timeout critique** : Tuner selon latence réseau (P95 × 2)

4. **Quorum** : wait_for_slave_count = nombre de Replicas souhaités pour ACK

5. **Fallback automatique** : wait_no_slave=ON permet passage en async si aucun Replica

6. **Impact performance** : -10% TPS en LAN, -50 à -80% en WAN

7. **Monitoring** : Rpl_semi_sync_master_status et taux succès yes_tx/(yes_tx+no_tx)

8. **Théorème CAP** : Semi-sync = CP (Consistency + Partition tolerance)

9. **LAN recommandé** : Semi-sync optimal avec latence < 5ms

10. **Tests requis** : Benchmarker avant production, valider RTO/RPO

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Semisynchronous Replication](https://mariadb.com/kb/en/semisynchronous-replication/)
- [📖 Replication Variables](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/)

### Articles techniques

- [🔗 Understanding Semi-sync Replication](https://mariadb.com/resources/blog/understanding-semi-sync-replication/)
- [🔗 AFTER_SYNC vs AFTER_COMMIT](https://www.percona.com/blog/semisync-after-sync-vs-after-commit/)

---

## ➡️ Section suivante

**13.10 Optimistic ALTER TABLE (MariaDB 11.8)** : Nouveauté majeure de MariaDB 11.8 LTS pour réduction du lag de réplication lors d'opérations DDL (ALTER TABLE), fonctionnement détaillé, configuration, et impact sur les topologies de réplication.

---


⏭️ [Optimistic ALTER TABLE pour réduction du lag](/13-replication/10-optimistic-alter-table.md)
