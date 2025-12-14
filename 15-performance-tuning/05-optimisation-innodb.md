🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.5 Optimisation du moteur InnoDB

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.4 (Méthodologie, Mémoire, Query Cache, I/O)
> - Compréhension approfondie de l'architecture InnoDB
> - Expérience en tuning de bases de données en production
> - Connaissance des workloads OLTP et leur comportement

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre l'architecture interne** d'InnoDB et ses composants critiques
- **Optimiser les redo logs** (taille, nombre, flushing) pour performance et durabilité
- **Configurer le checkpointing** et l'adaptive flushing selon le workload
- **Tuner les purge threads** pour le nettoyage efficace des undo logs
- **Optimiser le change buffer** pour les insertions et mises à jour
- **Gérer le doublewrite buffer** et comprendre son impact
- **Configurer la concurrence** (thread concurrency, spin delays)
- **Monitorer l'état interne** d'InnoDB avec précision
- **Diagnostiquer les problèmes** de performance spécifiques à InnoDB
- **Appliquer les best practices** pour différents types de workloads

---

## Introduction

InnoDB est un **moteur de stockage complexe** avec de nombreux composants internes qui travaillent ensemble pour garantir :

- ✅ **ACID compliance** : Atomicité, Cohérence, Isolation, Durabilité
- ✅ **Performance élevée** : OLTP avec haute concurrence
- ✅ **Intégrité des données** : Crash recovery robuste
- ✅ **Concurrence** : Row-level locking, MVCC

### Architecture globale d'InnoDB

```
┌────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE InnoDB                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │           BUFFER POOL (RAM)                       │     │
│  │  • Data pages (16KB)                              │     │
│  │  • Index pages                                    │     │
│  │  • Adaptive Hash Index (AHI)                      │     │
│  │  • Change Buffer (insertions/updates)             │     │
│  └────────────────────┬──────────────────────────────┘     │
│                       │                                    │
│  ┌────────────────────┴──────────────────────────────┐     │
│  │         LOG BUFFER (RAM)                          │     │
│  │  • Redo log entries                               │     │
│  │  • Flushed to disk at commit                      │     │
│  └────────────────────┬──────────────────────────────┘     │
│                       │                                    │
│                       ▼                                    │
│  ┌───────────────────────────────────────────────────┐     │
│  │         REDO LOGS (DISK) - Critical Path          │     │
│  │  • ib_logfile0, ib_logfile1, ...                  │     │
│  │  • Circular buffer                                │     │
│  │  • Crash recovery                                 │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │         UNDO LOGS (DISK/TABLESPACE)               │     │
│  │  • Old row versions (MVCC)                        │     │
│  │  • Purge thread cleanup                           │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │         DOUBLEWRITE BUFFER                        │     │
│  │  • Protection contre partial page writes          │     │
│  │  • 2MB buffer (128 pages)                         │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
│  ┌───────────────────────────────────────────────────┐     │
│  │         BACKGROUND THREADS                        │     │
│  │  • Master thread (coordinator)                    │     │
│  │  • I/O threads (read/write)                       │     │
│  │  • Purge threads (undo cleanup)                   │     │
│  │  • Page cleaner threads (flush dirty pages)       │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Composants à optimiser

Dans cette section, nous allons optimiser :

1. **Redo Logs** : Taille, nombre, flushing
2. **Checkpointing** : Adaptive flushing, dirty pages
3. **Purge Threads** : Nettoyage undo logs
4. **Change Buffer** : Optimisation INSERT/UPDATE
5. **Doublewrite Buffer** : Sécurité vs performance
6. **Concurrence** : Thread pools, spin waits
7. **Adaptive Hash Index** : Index automatique en RAM
8. **Monitoring** : Métriques internes InnoDB

---

## 1. Redo Logs : Optimisation avancée

### Architecture des redo logs

Les **redo logs** sont **critiques** pour les performances car ils sont sur le **chemin des commits**.

```
┌──────────────────────────────────────────────────┐
│         CYCLE DE VIE REDO LOG                    │
├──────────────────────────────────────────────────┤
│                                                  │
│  1. Transaction modifie données                  │
│     → Écriture dans log buffer (RAM)             │
│                                                  │
│  2. COMMIT                                       │
│     → Flush log buffer → redo log (DISK)         │
│     → Durabilité garantie                        │
│     ⚠️ LATENCE CRITIQUE utilisateur              │
│                                                  │
│  3. Redo logs = circular buffer                  │
│     ib_logfile0 → ib_logfile1 → ... → ib_logfile0│
│                                                  │
│  4. Checkpoint périodique                        │
│     → Dirty pages écrites sur disque             │
│     → Redo log space libéré                      │
│                                                  │
│  5. Crash recovery                               │
│     → Replay redo logs depuis dernier checkpoint │
│     → Restauration cohérence                     │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Paramètres redo log

```sql
-- Visualiser la configuration actuelle
SELECT 
    @@innodb_log_file_size / 1024 / 1024 / 1024 as log_file_size_gb,
    @@innodb_log_files_in_group as num_log_files,
    (@@innodb_log_file_size * @@innodb_log_files_in_group) / 1024 / 1024 / 1024 as total_log_size_gb,
    @@innodb_log_buffer_size / 1024 / 1024 as log_buffer_mb,
    @@innodb_flush_log_at_trx_commit as flush_at_commit;
```

#### innodb_log_file_size

**Fonction** : Taille de chaque fichier redo log.

```ini
[mariadb]
# Trop petit (ancien défaut) : 48M
innodb_log_file_size = 48M
# Problème : Checkpoints très fréquents, performance réduite

# Recommandation moderne OLTP : 1-2 GB
innodb_log_file_size = 1G   # Standard
innodb_log_file_size = 2G   # Write-heavy

# Recommandation OLAP/Batch : 4-8 GB
innodb_log_file_size = 4G
```

**Formule de dimensionnement** :

```
Taille idéale = Taux d'écriture (MB/s) × Intervalle checkpoint souhaité (secondes)

Exemple OLTP :
- Taux écriture : 50 MB/s
- Checkpoint tous les 30 secondes souhaité
- Taille = 50 × 30 = 1500 MB ≈ 1.5 GB
→ Configurer 2 GB pour marge
```

**Vérifier si trop petit** :

```sql
-- Fréquence des checkpoints (doit être > 5 minutes idéalement)
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_log_writes') as log_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime') as uptime_sec,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Uptime') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_log_writes'), 0),
        2
    ) as seconds_per_log_write;

-- Si seconds_per_log_write < 300 (5 min) → log trop petit
```

#### innodb_log_files_in_group

**Fonction** : Nombre de fichiers redo log (circular buffer).

```ini
[mariadb]
# Standard : 2 fichiers (défaut)
innodb_log_files_in_group = 2

# Rare : 3+ fichiers si très haute volumétrie write
innodb_log_files_in_group = 3
```

💡 **Conseil** : Plutôt qu'augmenter le nombre, augmentez `innodb_log_file_size`.

#### innodb_flush_log_at_trx_commit

**Fonction** : Contrôle la durabilité vs performance.

```ini
[mariadb]
# Valeur 0 : Performance MAX, durabilité MIN (risque perte 1s données)
innodb_flush_log_at_trx_commit = 0
# Log flushed toutes les 1 seconde, pas à chaque commit
# ⚠️ Perte possible de 1 seconde de transactions en cas de crash

# Valeur 1 : Durabilité MAX, performance standard (ACID complet)
innodb_flush_log_at_trx_commit = 1
# Log flushed à chaque commit (fsync)
# ✅ RECOMMANDÉ pour données critiques

# Valeur 2 : Compromis (protection crash MariaDB, pas crash OS)
innodb_flush_log_at_trx_commit = 2
# Log écrit à chaque commit mais flush OS toutes les 1s
# 🟡 Acceptable si OS/matériel fiable
```

**Impact performance** :

```
Benchmark 10,000 INSERT (SSD) :

innodb_flush_log_at_trx_commit = 0
→ 2.1 secondes (4,760 TPS)

innodb_flush_log_at_trx_commit = 1
→ 8.3 secondes (1,200 TPS)  ← 4x plus lent

innodb_flush_log_at_trx_commit = 2
→ 3.2 secondes (3,125 TPS)

Compromis : Utiliser 1 avec SSD/NVMe rapide
```

### Monitoring des redo logs

```sql
-- Dashboard redo log complet
SELECT 
    -- Configuration
    @@innodb_log_file_size / 1024 / 1024 / 1024 as log_file_gb,
    @@innodb_log_files_in_group as num_files,
    
    -- Utilisation
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_os_log_written') / 1024 / 1024 / 1024 as total_written_gb,
    
    -- Taux d'écriture
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_os_log_written') /
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Uptime') / 1024 / 1024,
        2
    ) as mb_written_per_sec,
    
    -- Log writes et waits
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_log_writes') as log_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_log_waits') as log_waits,  -- Doit être 0
    
    -- Pending writes
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_os_log_pending_writes') as pending_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_os_log_pending_fsyncs') as pending_fsyncs;
```

**Alertes** :

```sql
-- Alerte si log_waits > 0
SELECT 
    CASE 
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Innodb_log_waits') > 0
        THEN CONCAT('ALERT: Log buffer trop petit - ', 
                   (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
                    WHERE VARIABLE_NAME = 'Innodb_log_waits'),
                   ' waits')
        ELSE 'OK: Pas de log waits'
    END as log_buffer_status;
```

---

## 2. Checkpointing et Adaptive Flushing

### Concept du checkpointing

Le **checkpoint** est le processus qui écrit les dirty pages du buffer pool sur disque, libérant ainsi de l'espace dans les redo logs.

```
┌────────────────────────────────────────────────┐
│         PROCESSUS DE CHECKPOINT                │
├────────────────────────────────────────────────┤
│                                                │
│  Buffer Pool          Redo Log                 │
│  ───────────          ────────                 │
│                                                │
│  [Data Pages]         [Log Entries]            │
│   ├─ Clean            ├─ Applied               │
│   └─ Dirty ──┐        └─ Not Applied           │
│              │                                 │
│              ▼                                 │
│         CHECKPOINT                             │
│         ──────────                             │
│         1. Flush dirty pages → Disk            │
│         2. Mark log entries as applied         │
│         3. Advance checkpoint LSN              │
│         4. Free redo log space                 │
│                                                │
│  Fréquence : Dépend de dirty pages ratio       │
│              et taille redo log                │
│                                                │
└────────────────────────────────────────────────┘
```

### Paramètres de checkpointing

#### innodb_max_dirty_pages_pct

**Fonction** : Pourcentage maximum de dirty pages dans le buffer pool.

```ini
[mariadb]
# Ancien défaut (conservateur)
innodb_max_dirty_pages_pct = 75

# Recommandation moderne SSD/NVMe
innodb_max_dirty_pages_pct = 90
# Permet plus de dirty pages car flush rapide sur SSD

# HDD (legacy)
innodb_max_dirty_pages_pct = 75
# Plus conservateur car flush lent
```

**Impact** :

```
Si dirty pages > innodb_max_dirty_pages_pct :
→ InnoDB force un flush agressif
→ Peut causer des micro-pauses
→ Mais libère de l'espace redo log
```

#### innodb_max_dirty_pages_pct_lwm

**Fonction** : Low Water Mark - seuil pour commencer le flushing préemptif.

```ini
[mariadb]
# Recommandation : 50-70% du max
innodb_max_dirty_pages_pct = 90
innodb_max_dirty_pages_pct_lwm = 50

# Comportement :
# - Dirty < 50% : Flushing normal (background)
# - Dirty 50-90% : Flushing accéléré progressivement
# - Dirty > 90% : Flushing agressif (force)
```

#### innodb_adaptive_flushing

**Fonction** : Ajustement dynamique du taux de flush selon la charge.

```ini
[mariadb]
# TOUJOURS activer (défaut dans versions modernes)
innodb_adaptive_flushing = ON

# Permet à InnoDB d'ajuster le flushing automatiquement
# Évite les pics de flushing (checkpoint stalls)
```

#### innodb_adaptive_flushing_lwm

**Fonction** : Seuil redo log pour activer adaptive flushing.

```ini
[mariadb]
# Défaut : 10% de l'espace redo log
innodb_adaptive_flushing_lwm = 10

# Valeur plus agressive pour workload write-heavy
innodb_adaptive_flushing_lwm = 5

# Signification :
# Si redo log usage > 10% (ou 5%) de la capacité totale
# → Activer adaptive flushing plus tôt
```

#### innodb_flushing_avg_loops

**Fonction** : Fenêtre de lissage pour adaptive flushing.

```ini
[mariadb]
# Défaut : 30 loops
innodb_flushing_avg_loops = 30

# Plus petit = réaction plus rapide aux changements
innodb_flushing_avg_loops = 20  # Workload variable

# Plus grand = lissage plus important
innodb_flushing_avg_loops = 50  # Workload stable
```

### Monitoring du checkpointing

```sql
-- État du checkpointing
SELECT 
    -- Dirty pages
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') as dirty_pages,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') as total_pages,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100,
        2
    ) as dirty_pct,
    
    -- Seuils configurés
    @@innodb_max_dirty_pages_pct as max_dirty_pct,
    @@innodb_max_dirty_pages_pct_lwm as lwm_pct,
    
    -- Flushing activity
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_flushed') as pages_flushed,
    
    -- Checkpoint age (espace redo log utilisé)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_checkpoint_age') / 1024 / 1024 as checkpoint_age_mb,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_checkpoint_max_age') / 1024 / 1024 as max_checkpoint_age_mb;
```

**Interprétation** :

```
Dirty pages % :
  < 25%  : Excellent - Flushing suit bien
  25-50% : Bon - Sous le LWM
  50-75% : Attention - Flushing accéléré
  > 75%  : Problème - Risque de checkpoint stall

Checkpoint age vs max :
  < 50%  : Excellent
  50-75% : Bon
  75-90% : Attention - Approche limite
  > 90%  : Critique - Risque de blocage
```

---

## 3. Purge Threads : Nettoyage des Undo Logs

### Rôle des undo logs et purge

Les **undo logs** stockent les anciennes versions des lignes pour le **MVCC** (Multi-Version Concurrency Control).

```
┌────────────────────────────────────────────────┐
│         CYCLE DE VIE UNDO LOG                  │
├────────────────────────────────────────────────┤
│                                                │
│  1. UPDATE table SET col = 'new'               │
│     → Ancienne valeur stockée dans undo log    │
│     → Pour transactions concurrentes           │
│                                                │
│  2. COMMIT                                     │
│     → Nouvelle valeur visible                  │
│     → Undo log marqué "purgeable"              │
│                                                │
│  3. Purge threads (background)                 │
│     → Nettoient les undo logs obsolètes        │
│     → Libèrent l'espace                        │
│                                                │
│  Problème si purge lent :                      │
│  → Undo logs accumulent (History List Length)  │
│  → Espace disque consommé                      │
│  → Performance dégradée                        │
│                                                │
└────────────────────────────────────────────────┘
```

### Paramètres purge

#### innodb_purge_threads

**Fonction** : Nombre de threads dédiés au nettoyage des undo logs.

```ini
[mariadb]
# Défaut ancien : 1 thread
innodb_purge_threads = 1
# ⚠️ Insuffisant pour workload write-heavy

# Recommandation moderne
innodb_purge_threads = 4
# Standard pour la plupart des workloads

# Workload très write-heavy (millions UPDATE/sec)
innodb_purge_threads = 8
# Maximum utile généralement
```

**Formule** :

```
Règle générale :
- 1-2 threads : Workload read-heavy
- 4 threads : Workload mixte (recommandé)
- 6-8 threads : Workload très write-heavy

⚠️ Plus de threads ≠ toujours mieux
Si History List Length reste bas, inutile d'augmenter
```

#### innodb_purge_batch_size

**Fonction** : Nombre d'undo logs traités par batch.

```ini
[mariadb]
# Défaut : 300
innodb_purge_batch_size = 300

# Workload write-heavy
innodb_purge_batch_size = 1000
# Plus gros batches = moins d'overhead

# ⚠️ Valeurs excessives peuvent causer des pauses
innodb_purge_batch_size = 5000  # Max raisonnable
```

#### innodb_max_purge_lag

**Fonction** : Limite la croissance de l'History List Length en ralentissant les DML.

```ini
[mariadb]
# Défaut : 0 (désactivé)
innodb_max_purge_lag = 0

# Activer si History List Length explose
innodb_max_purge_lag = 1000000
# Si HLL > 1M, ralentir les INSERT/UPDATE/DELETE
# Donne du temps au purge pour rattraper

# ⚠️ Utiliser en dernier recours uniquement
# Préférable : Augmenter purge_threads d'abord
```

### Monitoring du purge

```sql
-- État du purge et History List Length
SELECT 
    -- History List Length (critique!)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_history_list_length') as history_list_length,
    
    -- Purge activity
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_purge_trx_id') as purge_trx_id,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_purge_undo_no') as purge_undo_no,
    
    -- Configuration
    @@innodb_purge_threads as purge_threads,
    @@innodb_purge_batch_size as batch_size,
    @@innodb_max_purge_lag as max_lag;
```

**Interprétation History List Length** :

```
HLL :
  < 1,000     : Excellent - Purge suit parfaitement
  1k - 10k    : Bon - Normal pour workload actif
  10k - 100k  : Attention - Surveiller
  100k - 1M   : Problème - Purge en retard
  > 1M        : Critique - Action immédiate requise

Action si HLL élevé :
1. Vérifier innodb_purge_threads (augmenter à 4-8)
2. Vérifier transactions longues (bloquent le purge)
3. Considérer innodb_max_purge_lag temporairement
4. Optimiser le workload (moins d'UPDATE)
```

**Procédure d'alerte** :

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE check_purge_lag()
BEGIN
    DECLARE v_hll BIGINT;
    
    SELECT VARIABLE_VALUE INTO v_hll
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Innodb_history_list_length';
    
    IF v_hll > 1000000 THEN
        SELECT CONCAT('CRITICAL: History List Length = ', v_hll, 
                     ' (> 1M) - Augmenter purge_threads!') as alert;
    ELSEIF v_hll > 100000 THEN
        SELECT CONCAT('WARNING: History List Length = ', v_hll,
                     ' (> 100k) - Surveiller') as alert;
    ELSE
        SELECT CONCAT('OK: History List Length = ', v_hll) as status;
    END IF;
END //
DELIMITER ;

-- Exécuter périodiquement
CALL check_purge_lag();
```

---

## 4. Change Buffer : Optimisation des Insertions

### Rôle du Change Buffer

Le **change buffer** cache les modifications des index secondaires non-uniques pour éviter les I/O aléatoires immédiates.

```
┌────────────────────────────────────────────────┐
│         CHANGE BUFFER WORKFLOW                 │
├────────────────────────────────────────────────┤
│                                                │
│  INSERT INTO orders (customer_id, amount)      │
│  VALUES (12345, 99.99);                        │
│                                                │
│  1. Primary key insert                         │
│     → Écriture immédiate (clustered index)     │
│                                                │
│  2. Secondary index (idx_customer_id)          │
│     → Si page INDEX en buffer pool : Update    │
│     → Si page INDEX PAS en BP :                │
│        ├─ SANS change buffer : Read page (I/O) │
│        └─ AVEC change buffer : Buffer change   │
│           (évite I/O aléatoire)                │
│                                                │
│  3. Merge différé                              │
│     → Quand page index lue pour autre raison   │
│     → Ou lors du shutdown                      │
│     → Ou purge periodique                      │
│                                                │
│  Gain : Réduit I/O aléatoires pour INSERT      │
│                                                │
└────────────────────────────────────────────────┘
```

### Paramètres Change Buffer

#### innodb_change_buffering

**Fonction** : Contrôle quelles opérations sont bufferisées.

```ini
[mariadb]
# Valeurs possibles :
# - none : Désactivé
# - inserts : Seulement INSERT
# - deletes : Seulement DELETE
# - changes : INSERT + DELETE
# - purges : Purge operations
# - all : Tout (défaut recommandé)

# Recommandation standard
innodb_change_buffering = all

# Si workload majoritairement READ
innodb_change_buffering = none
# Pas de bénéfice à buffering si peu de writes
```

#### innodb_change_buffer_max_size

**Fonction** : Taille maximale du change buffer (% du buffer pool).

```ini
[mariadb]
# Défaut : 25% du buffer pool
innodb_change_buffer_max_size = 25

# Workload très INSERT-heavy
innodb_change_buffer_max_size = 50
# Plus d'espace pour buffer les changes

# Workload READ-heavy
innodb_change_buffer_max_size = 10
# Libérer de l'espace pour les data pages
```

### Monitoring du Change Buffer

```sql
-- État du change buffer
SELECT 
    -- Size
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_size') as ibuf_size_pages,
    
    -- Free space
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_free_list') as ibuf_free_pages,
    
    -- Segment size
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_segment_size') as segment_size,
    
    -- Merges
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_merges') as merges,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_merged_inserts') as merged_inserts,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_ibuf_merged_deletes') as merged_deletes,
    
    -- Configuration
    @@innodb_change_buffering as change_buffering,
    @@innodb_change_buffer_max_size as max_size_pct;
```

---

## 5. Doublewrite Buffer

### Rôle et fonctionnement

Le **doublewrite buffer** protège contre les **partial page writes** (corruption de page).

```
┌────────────────────────────────────────────────┐
│         DOUBLEWRITE BUFFER PROTECTION          │
├────────────────────────────────────────────────┤
│                                                │
│  Problème : Partial page write                 │
│  ─────────────────────────────                 │
│  • InnoDB page = 16 KB                         │
│  • Secteur disque = 512 bytes ou 4 KB          │
│  • Crash pendant écriture 16 KB                │
│    → Seulement 8 KB écrits                     │
│    → Page corrompue !                          │
│                                                │
│  Solution : Doublewrite buffer                 │
│  ────────────────────────────                  │
│  1. Écrire page → doublewrite buffer (seq)     │
│  2. Flush doublewrite buffer                   │
│  3. Écrire page → emplacement final (random)   │
│                                                │
│  Recovery :                                    │
│  Si page finale corrompue                      │
│  → Restaurer depuis doublewrite buffer         │
│                                                │
│  Coût : ~5-10% performance write               │
│  Bénéfice : Protection corruption              │
│                                                │
└────────────────────────────────────────────────┘
```

### Paramètres Doublewrite

#### innodb_doublewrite

**Fonction** : Activer/désactiver le doublewrite buffer.

```ini
[mariadb]
# Défaut : ON (recommandé)
innodb_doublewrite = ON
# Protection contre partial page writes

# Désactiver SEULEMENT si :
innodb_doublewrite = OFF
# ✅ Filesystem garantit atomic writes (ex: ZFS, Btrfs)
# ✅ Réplication: esclave non-critique peut désactiver
# ⚠️ Gain performance : seulement ~5-10%
# ⚠️ Risque : Corruption en cas de crash
```

**Cas d'usage désactivation** :

```
✅ Acceptable de désactiver :
- Filesystem atomic write (ZFS, Btrfs)
- Replica read-only non-critique
- Tests/staging (pas production primaire)

❌ NE JAMAIS désactiver :
- Production critique
- Primary server
- Filesystem standard (ext4, XFS)
- Sans backup récent
```

### Monitoring Doublewrite

```sql
-- Activité doublewrite
SELECT 
    @@innodb_doublewrite as doublewrite_enabled,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_dblwr_writes') as dblwr_writes,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_dblwr_pages_written') as dblwr_pages,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_dblwr_pages_written') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_dblwr_writes'), 0),
        2
    ) as pages_per_write;
```

---

## 6. Thread Concurrency et Spin Delays

### innodb_thread_concurrency

**Fonction** : Limite le nombre de threads OS simultanés dans InnoDB.

```ini
[mariadb]
# Défaut : 0 (illimité, recommandé sur systèmes modernes)
innodb_thread_concurrency = 0

# Si contention excessive sur serveur >32 cores
innodb_thread_concurrency = 32
# Formule: 2 × num_cores (mais tester!)

# ⚠️ La plupart du temps, laisser à 0
```

### innodb_spin_wait_delay et innodb_sync_spin_loops

**Fonction** : Contrôle le spin waiting avant de dormir sur un mutex.

```ini
[mariadb]
# Défaut : Généralement OK
innodb_spin_wait_delay = 6
innodb_sync_spin_loops = 30

# Sur CPU ultra-rapides (>4 GHz)
innodb_spin_wait_delay = 4
# Moins de spin avant de dormir

# ⚠️ Rarement besoin de modifier
# Seulement si profiling montre contention spin
```

---

## 7. Page Cleaners : Parallélisation du Flushing

### innodb_page_cleaners

**Fonction** : Nombre de threads pour flush les dirty pages.

```ini
[mariadb]
# Défaut ancien : 1 thread
innodb_page_cleaners = 1

# Recommandation moderne (égal aux buffer pool instances)
innodb_buffer_pool_instances = 16
innodb_page_cleaners = 16
# Un cleaner par instance

# Maximum utile
innodb_page_cleaners = 4
# Suffisant si buffer pool instances < 4
```

**Règle** :

```
innodb_page_cleaners = MIN(innodb_buffer_pool_instances, 4)

Exemple :
- 48 GB buffer pool, 48 instances
- page_cleaners = MIN(48, 4) = 4

Raison : Au-delà de 4, peu de gain
```

---

## 8. Monitoring Global InnoDB

### Dashboard de santé InnoDB

```sql
-- Vue complète de l'état InnoDB
CREATE OR REPLACE VIEW v_innodb_health AS
SELECT 
    -- Buffer Pool
    ROUND(
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        ), 4
    ) as bp_hit_rate_pct,
    
    -- Dirty pages
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100,
        2
    ) as dirty_pages_pct,
    
    -- History List Length
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_history_list_length') as history_list_length,
    
    -- Log waits (doit être 0)
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_log_waits') as log_waits,
    
    -- Row locks
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_row_lock_current_waits') as current_lock_waits,
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_row_lock_time_avg') / 1000,
        2
    ) as avg_lock_wait_ms,
    
    -- Threads
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_running') as threads_running,
    
    -- Timestamp
    NOW() as checked_at;

-- Utiliser
SELECT * FROM v_innodb_health;
```

### Métriques critiques à surveiller

```sql
-- Alertes automatiques
DELIMITER //
CREATE OR REPLACE PROCEDURE innodb_health_alerts()
BEGIN
    DECLARE v_bp_hit_rate DECIMAL(10,4);
    DECLARE v_dirty_pct DECIMAL(10,2);
    DECLARE v_hll BIGINT;
    DECLARE v_log_waits BIGINT;
    
    -- Buffer pool hit rate
    SELECT 
        100 - (
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0) * 100
        )
    INTO v_bp_hit_rate;
    
    -- Dirty pages
    SELECT 
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') /
        NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total'), 0) * 100
    INTO v_dirty_pct;
    
    -- History List Length
    SELECT VARIABLE_VALUE INTO v_hll
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Innodb_history_list_length';
    
    -- Log waits
    SELECT VARIABLE_VALUE INTO v_log_waits
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Innodb_log_waits';
    
    -- Générer alertes
    IF v_bp_hit_rate < 95 THEN
        SELECT CONCAT('ALERT: Buffer pool hit rate = ', v_bp_hit_rate, '% (< 95%)') as alert;
    END IF;
    
    IF v_dirty_pct > 75 THEN
        SELECT CONCAT('ALERT: Dirty pages = ', v_dirty_pct, '% (> 75%)') as alert;
    END IF;
    
    IF v_hll > 100000 THEN
        SELECT CONCAT('ALERT: History List Length = ', v_hll, ' (> 100k)') as alert;
    END IF;
    
    IF v_log_waits > 0 THEN
        SELECT CONCAT('ALERT: Log waits = ', v_log_waits, ' (> 0)') as alert;
    END IF;
END //
DELIMITER ;

-- Exécuter
CALL innodb_health_alerts();
```

---

## Configuration Recommandée par Type de Workload

### OLTP High Performance (SSD/NVMe)

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION INNODB OLTP MODERNE (64+ GB RAM, NVMe)
# ─────────────────────────────────────────────────────

# Redo Logs
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 1  # ACID complet

# Checkpointing
innodb_max_dirty_pages_pct = 90
innodb_max_dirty_pages_pct_lwm = 50
innodb_adaptive_flushing = ON
innodb_adaptive_flushing_lwm = 10
innodb_page_cleaners = 4

# Purge
innodb_purge_threads = 4
innodb_purge_batch_size = 300

# Change Buffer
innodb_change_buffering = all
innodb_change_buffer_max_size = 25

# Doublewrite
innodb_doublewrite = ON

# Concurrency
innodb_thread_concurrency = 0  # Illimité
innodb_read_io_threads = 16
innodb_write_io_threads = 16
```

### OLAP / Data Warehouse

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION INNODB OLAP (Analytics, Batch)
# ─────────────────────────────────────────────────────

# Redo Logs (plus grands pour gros batch)
innodb_log_file_size = 4G
innodb_log_files_in_group = 2
innodb_log_buffer_size = 128M
innodb_flush_log_at_trx_commit = 2  # Acceptable pour analytics

# Checkpointing (tolérant)
innodb_max_dirty_pages_pct = 90
innodb_max_dirty_pages_pct_lwm = 60
innodb_adaptive_flushing = ON
innodb_page_cleaners = 8

# Purge (actif pour gros UPDATE)
innodb_purge_threads = 8
innodb_purge_batch_size = 1000

# Change Buffer (large pour bulk inserts)
innodb_change_buffering = all
innodb_change_buffer_max_size = 50

# I/O élevé
innodb_read_io_threads = 16
innodb_write_io_threads = 16
```

### Workload Mixte (Standard)

```ini
[mariadb]
# ─────────────────────────────────────────────────────
# CONFIGURATION INNODB MIXTE (Équilibrée)
# ─────────────────────────────────────────────────────

# Redo Logs
innodb_log_file_size = 1G
innodb_log_files_in_group = 2
innodb_log_buffer_size = 32M
innodb_flush_log_at_trx_commit = 1

# Checkpointing
innodb_max_dirty_pages_pct = 75
innodb_max_dirty_pages_pct_lwm = 50
innodb_adaptive_flushing = ON
innodb_page_cleaners = 4

# Purge
innodb_purge_threads = 4
innodb_purge_batch_size = 300

# Change Buffer
innodb_change_buffering = all
innodb_change_buffer_max_size = 25

# Doublewrite
innodb_doublewrite = ON

# I/O standard
innodb_read_io_threads = 8
innodb_write_io_threads = 8
```

---

## ✅ Points clés à retenir

- 📝 **Redo logs = critique** : Taille impact checkpoints, 1-2 GB standard
- 🔄 **Checkpointing adaptatif** : innodb_adaptive_flushing = ON toujours
- 🧹 **Purge threads** : 4 threads standard, surveiller History List Length
- 💾 **Change buffer** : Réduit I/O sur INSERT, all = recommandé
- 🛡️ **Doublewrite** : Protection essentielle, ne désactiver que si ZFS/Btrfs
- 📊 **Monitoring continu** : BP hit rate, dirty pages, HLL, log waits
- ⚡ **Page cleaners** : Paralléliser flush, 4 threads suffisants
- 🎯 **innodb_flush_log_at_trx_commit** : 1 = ACID, 2 = compromis, 0 = risqué
- 📈 **HLL < 100k** : Objectif pour purge efficace
- 🔧 **Tuning workload-specific** : OLTP vs OLAP = configs différentes

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [📖 InnoDB Redo Log](https://mariadb.com/kb/en/innodb-redo-log/)
- [📖 InnoDB Purge](https://mariadb.com/kb/en/innodb-purge/)
- [📖 InnoDB Change Buffer](https://mariadb.com/kb/en/innodb-change-buffering/)

### Lectures avancées

- [MySQL InnoDB Internals](https://dev.mysql.com/doc/internals/en/innodb.html)
- [Percona Blog - InnoDB](https://www.percona.com/blog/category/mysql/innodb/)

---

## ➡️ Section suivante

**[15.6 innodb_alter_copy_bulk](/15-performance-tuning/06-innodb-alter-copy-bulk.md)** : Explorons en détail cette nouveauté MariaDB 11.8 qui accélère drastiquement la construction d'index sur SSD/NVMe.

---

*L'optimisation d'InnoDB nécessite une compréhension approfondie de son architecture interne. Les paramètres par défaut sont conservateurs - un tuning adapté peut multiplier les performances par 2-5x.*

⏭️ [innodb_alter_copy_bulk : Construction d'index efficace](/15-performance-tuning/06-innodb-alter-copy-bulk.md)
