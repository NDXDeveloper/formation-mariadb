🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4 GTID (Global Transaction Identifier)

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : 
> - Section 13.3 (Réplication basée sur les positions)
> - Compréhension des systèmes distribués
> - Connaissance des concepts de consensus et d'ordering
> - Expérience avec les topologies de réplication

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le concept de GTID et son utilité dans les systèmes distribués
- Différencier l'implémentation GTID de MariaDB de celle de MySQL
- Expliquer la structure et la sémantique des GTID MariaDB
- Analyser les avantages décisifs du GTID sur la réplication par positions
- Identifier les cas d'usage appropriés pour GTID
- Comprendre l'impact sur les performances et l'opérabilité
- Préparer une migration vers GTID

---

## Introduction

Le **GTID (Global Transaction Identifier)** représente une évolution majeure dans la réplication de bases de données. Il remplace le système de coordonnées (fichier, position) par des **identifiants uniques et globaux** pour chaque transaction répliquée.

### Le problème résolu par GTID

**Avec positions binlog** (méthode traditionnelle) :

```
Primary:   mariadb-bin.000042:4567
           ↓
           "Où suis-je dans ce fichier spécifique ?"
           
Problème lors du failover:
- Positions incompatibles entre serveurs
- Calcul manuel de positions équivalentes
- Risque d'erreurs et de perte de données
```

**Avec GTID** :

```
Transaction: 0-1-1000
             ↓
             "Cette transaction unique, peu importe où elle est"
             
Avantage lors du failover:
- Identification universelle
- Réplication automatique depuis la bonne position
- Zero calcul manuel
```

💡 **Analogie** : 
- **Positions** = Adresse postale locale (rue, numéro, ville)
- **GTID** = Code-barres universel (ISBN pour livres)

### Historique

**Timeline** :

```
2013 - MySQL 5.6
       │ Introduction du GTID MySQL
       │ Format: UUID:Transaction_ID
       ▼
       
2014 - MariaDB 10.0
       │ Implémentation propre de GTID MariaDB
       │ Format: Domain-Server-Sequence
       │ Incompatible avec MySQL GTID
       ▼
       
2017 - MariaDB 10.3
       │ Améliorations GTID strict mode
       │ Meilleure gestion des topologies complexes
       ▼
       
2024 - MariaDB 11.4 LTS
       │ Optimisations performances GTID
       │ Stabilité accrue
       ▼
       
2025 - MariaDB 11.8 LTS
       │ GTID recommandé par défaut
       │ Meilleure compatibilité MySQL GTID (lecture seule)
       │ Outils de migration améliorés
```

---

## Structure du GTID MariaDB

### Format de base

Un GTID MariaDB suit le format **Domain-Server-Sequence** :

```
0-1-1000
│ │  └─── Sequence Number (transaction)
│ └────── Server ID (serveur origine)
└──────── Domain ID (domaine de réplication)
```

**Exemple concret** :

```sql
SELECT @@gtid_current_pos;
-- +---------------------+
-- | @@gtid_current_pos  |
-- +---------------------+
-- | 0-1-1000            |
-- +---------------------+

-- Signification :
-- Domain 0  : Domaine par défaut
-- Server 1  : Transaction créée sur le serveur avec server-id=1
-- Sequence 1000 : Numéro de séquence de cette transaction
```

### Composants détaillés

#### 1. Domain ID

Le **Domain ID** permet de gérer des **flux de réplication indépendants**.

```
┌─────────────────────────────────────────────────────────┐
│ Topologie Multi-Source                                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Primary A            Primary B                        │
│   (Domain 0)           (Domain 1)                       │
│   GTID: 0-1-500        GTID: 1-2-300                    │
│       │                    │                            │
│       └────────┬───────────┘                            │
│                │                                        │
│                ▼                                        │
│           Replica C                                     │
│           GTID position:                                │
│           Domain 0: 0-1-500                             │
│           Domain 1: 1-2-300                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Use cases** :
- ✅ Réplication multi-source (chaque source = 1 domain)
- ✅ Topologies circular/bi-directionnelles
- ✅ Consolidation de bases distinctes

**Configuration** :

```sql
-- Sur Primary A
SET GLOBAL gtid_domain_id = 0;

-- Sur Primary B
SET GLOBAL gtid_domain_id = 1;
```

💡 **Recommandation** : 
- Petit déploiement (1 Primary) : `gtid_domain_id = 0` (défaut)
- Multi-source : 1 domain ID par source de réplication

#### 2. Server ID

Le **Server ID** identifie le serveur qui a **créé** la transaction.

```
Server ID = Identité unique du serveur MariaDB
          = Doit être unique dans TOUTE la topologie
```

**Configuration** :

```ini
[mysqld]
server-id = 1    # Primary
server-id = 2    # Replica1
server-id = 3    # Replica2
# etc.
```

**Importance critique** :

```
Scénario problématique:
┌──────────┐
│ Primary  │ server-id = 1
└─────┬────┘
      │
      ▼
┌──────────┐
│ Replica  │ server-id = 1  ← MÊME ID ! ❌
└──────────┘

Résultat:
- Boucles de réplication
- Transactions ignorées
- Corruption de données possible
```

⚠️ **RÈGLE D'OR** : `server-id` DOIT être unique dans toute la topologie.

**Vérification** :

```sql
-- Sur chaque serveur
SELECT @@server_id;

-- Script de vérification topologie
SELECT DISTINCT server_id FROM mysql.gtid_slave_pos;
```

#### 3. Sequence Number

Le **Sequence Number** est un **compteur monotone croissant** de transactions.

```
Serveur avec server-id = 1:

Transaction 1 → GTID: 0-1-1
Transaction 2 → GTID: 0-1-2
Transaction 3 → GTID: 0-1-3
...
Transaction 1000 → GTID: 0-1-1000
```

**Propriétés** :
- ✅ Strictement croissant par serveur
- ✅ Gaps possibles (transactions rollback)
- ✅ Permet l'ordering des transactions

**Exemple avec gaps** :

```sql
BEGIN;
INSERT INTO users (name) VALUES ('Alice');
-- GTID assigné: 0-1-1001
ROLLBACK;  -- Transaction annulée

BEGIN;
INSERT INTO users (name) VALUES ('Bob');
COMMIT;
-- GTID assigné: 0-1-1002 (1001 est "sauté")
```

### Format GTID Position

Une **position GTID** peut contenir **plusieurs GTID** (un par domain) :

```sql
-- Position simple (1 domain)
SELECT @@gtid_current_pos;
-- 0-1-1000

-- Position multi-domain
SELECT @@gtid_current_pos;
-- 0-1-1000,1-2-500,2-3-750

-- Explication:
-- Domain 0: Transactions jusqu'à 0-1-1000
-- Domain 1: Transactions jusqu'à 1-2-500
-- Domain 2: Transactions jusqu'à 2-3-750
```

**Intervalles** :

MariaDB supporte les **intervalles de GTID** pour compacité :

```sql
-- Sans intervalle (verbose)
0-1-1,0-1-2,0-1-3,0-1-4,0-1-5,...,0-1-1000

-- Avec intervalle (compact)
0-1-1000

-- Intervalles non-contigus
0-1-100,0-1-200-300,0-1-500
-- = Transactions 1-100, 200-300, et 500
```

---

## GTID MariaDB vs MySQL GTID

MariaDB et MySQL ont des **implémentations incompatibles** de GTID.

### Comparaison structurelle

| Aspect | MariaDB GTID | MySQL GTID |
|--------|--------------|------------|
| **Format** | `Domain-Server-Sequence` | `UUID:Transaction` |
| **Exemple** | `0-1-1000` | `3E11FA47-71CA-11E1-9E33-C80AA9429562:1000` |
| **Domain ID** | ✅ Supporté | ❌ N/A |
| **Multi-source** | ✅ Natif (via domains) | ⚠️ Complexe |
| **Compacité** | ✅ Court, lisible | ❌ UUID long (36 chars) |
| **Bi-directionnel** | ✅ Supporté | ❌ Problématique |
| **Intervalle** | ✅ Oui (`0-1-100`) | ✅ Oui (`UUID:1-100`) |

### Différences d'implémentation

**MariaDB GTID** :

```sql
-- Variable position
@@gtid_current_pos       -- Position actuelle
@@gtid_binlog_pos        -- Position dans binlog
@@gtid_slave_pos         -- Position répliquée

-- Mode
@@gtid_strict_mode       -- ON/OFF
@@gtid_domain_id         -- Domain actif
```

**MySQL GTID** :

```sql
-- Variable position (différent !)
@@gtid_executed          -- GTID exécutés
@@gtid_purged            -- GTID purgés

-- Mode
@@gtid_mode              -- ON/OFF/OFF_PERMISSIVE/ON_PERMISSIVE
@@enforce_gtid_consistency  -- ON/OFF
```

### Compatibilité MariaDB ↔ MySQL

**MariaDB 10.5+ peut lire MySQL GTID** :

```
MySQL Primary (MySQL GTID)
       ↓
MariaDB Replica (lit en mode compatible)
       → Convertit en MariaDB GTID en interne
```

**Configuration** :

```sql
-- Sur MariaDB Replica
CHANGE MASTER TO
  MASTER_HOST = 'mysql-primary.example.com',
  MASTER_USE_GTID = slave_pos;

-- MariaDB détecte automatiquement MySQL GTID
-- et le traduit en format MariaDB
```

⚠️ **Limitation** : MariaDB → MySQL avec GTID **non supporté** (formats incompatibles).

---

## Avantages Décisifs du GTID

### 1. Failover Automatisé

**Problème avec positions** :

```
Primary CRASH ! 💥
Replica1: mariadb-bin.000042:8934
Replica2: mariadb-bin.000042:8920

Promotion de Replica1 → nouveau Primary
Mais Replica2 cherche mariadb-bin.000042:8920
Le nouveau Primary a replica1-bin.000001:4

❌ Positions incompatibles → Calcul manuel requis
```

**Solution avec GTID** :

```
Primary CRASH ! 💥
Replica1: 0-1-1000
Replica2: 0-1-998

Promotion de Replica1 → nouveau Primary
Replica2 cherche: "Transactions après 0-1-998"

CHANGE MASTER TO MASTER_USE_GTID=slave_pos;
✅ Réplication reprend automatiquement depuis 0-1-999
```

**Exemple concret** :

```sql
-- Sur Replica2 (après promotion de Replica1)
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST = 'replica1.example.com',  -- Nouveau Primary
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'password',
  MASTER_USE_GTID = slave_pos;  -- Magic ! Position calculée auto

START SLAVE;

-- MariaDB calcule automatiquement:
-- Position actuelle de Replica2: 0-1-998
-- Demande au nouveau Primary: "Donne-moi à partir de 0-1-999"
```

### 2. Topologies Complexes Simplifiées

**Multi-Source** :

```sql
-- Avec positions: Gestion de 2 jeux (fichier, position)
CHANGE MASTER 'source1' TO
  MASTER_LOG_FILE = 'source1-bin.000042',
  MASTER_LOG_POS = 4567;

CHANGE MASTER 'source2' TO
  MASTER_LOG_FILE = 'source2-bin.000123',
  MASTER_LOG_POS = 9876;

-- Avec GTID: Simple et unifié
CHANGE MASTER 'source1' TO
  MASTER_USE_GTID = slave_pos;

CHANGE MASTER 'source2' TO
  MASTER_USE_GTID = slave_pos;
```

**Cascade** :

```
Primary → Relay → Replica1
                → Replica2

Avec GTID:
- Relay génère son propre binlog avec GTID
- Replica1 et Replica2 suivent les GTID du Primary
- Transparence totale de la cascade
```

### 3. Réplication Bi-directionnelle

**Scenario** : Active-Active (avec précautions)

```
Primary A ←──→ Primary B
(writes)        (writes)

Avec GTID:
Domain 0: Transactions de A
Domain 1: Transactions de B

Chaque serveur ignore ses propres transactions
Pas de boucles infinies
```

**Configuration** :

```sql
-- Sur Primary A
SET GLOBAL gtid_domain_id = 0;
SET GLOBAL server_id = 1;

-- Sur Primary B
SET GLOBAL gtid_domain_id = 1;
SET GLOBAL server_id = 2;

-- A réplique depuis B (domain 1)
-- B réplique depuis A (domain 0)
-- ✅ Pas de conflit
```

⚠️ **Attention** : Requires application-level conflict resolution !

### 4. État Global Cohérent

**Avec GTID** : Vue globale unifiée de l'état de réplication

```sql
-- Sur n'importe quel serveur
SELECT @@gtid_current_pos;
-- 0-1-1000,1-2-500

-- Signifie universellement:
-- "J'ai appliqué toutes les transactions jusqu'à:
--   - Domain 0: 0-1-1000
--   - Domain 1: 1-2-500"
```

**Monitoring simplifié** :

```sql
-- Comparer 2 serveurs
-- Serveur A
SELECT @@gtid_current_pos AS server_a;
-- 0-1-1000

-- Serveur B
SELECT @@gtid_current_pos AS server_b;
-- 0-1-998

-- Lag = 1000 - 998 = 2 transactions
```

### 5. Recovery Simplifié

**Point-in-Time Recovery** :

```sql
-- Restaurer jusqu'à un GTID spécifique
mariabackup --backup --target-dir=/backup/full

-- Lire la position GTID
cat /backup/full/xtrabackup_binlog_info
-- 0-1-1000

-- Restaurer
mariabackup --prepare --target-dir=/backup/full
mariabackup --copy-back --target-dir=/backup/full

-- Reprendre réplication depuis ce GTID exact
CHANGE MASTER TO MASTER_USE_GTID = slave_pos;
```

### 6. Consistance Garantie

**GTID Strict Mode** :

```sql
SET GLOBAL gtid_strict_mode = ON;

-- Refuse maintenant:
-- 1. Transactions sans GTID (si GTID activé)
-- 2. Changements de gtid_domain_id pendant réplication
-- 3. Écritures sur Replica (sauf réplication)

-- Garantit cohérence totale
```

---

## Fonctionnement Interne

### Cycle de vie d'une transaction GTID

```
┌─────────────────────────────────────────────────────────────┐
│ PRIMARY (server-id=1, gtid_domain_id=0)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Client: BEGIN                                           │
│     INSERT INTO users (name) VALUES ('Alice');              │
│     COMMIT;                                                 │
│     ↓                                                       │
│  2. MariaDB génère GTID: 0-1-1000                           │
│     ↓                                                       │
│  3. Écriture dans binlog:                                   │
│     - GTID_EVENT (0-1-1000)                                 │
│     - QUERY_EVENT (BEGIN)                                   │
│     - TABLE_MAP_EVENT                                       │
│     - WRITE_ROWS_EVENT                                      │
│     - XID_EVENT (COMMIT)                                    │
│     ↓                                                       │
│  4. Mise à jour @@gtid_current_pos → 0-1-1000               │
│     Mise à jour @@gtid_binlog_pos → 0-1-1000                │
│     ↓                                                       │
│  5. Confirmation au client ✓                                │
│                                                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Réplication
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ REPLICA (server-id=2)                                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  6. IO Thread: Reçoit GTID_EVENT (0-1-1000)                 │
│     Écrit dans relay log                                    │
│     ↓                                                       │
│  7. SQL Thread: Lit GTID_EVENT                              │
│     Vérifie: ai-je déjà 0-1-1000 ?                          │
│     ↓                                                       │
│     NON → Applique la transaction                           │
│     OUI → SKIP (déjà appliquée)                             │
│     ↓                                                       │
│  8. Mise à jour @@gtid_slave_pos → 0-1-1000                 │
│     Mise à jour mysql.gtid_slave_pos (persisté)             │
│     ↓                                                       │
│  9. Transaction visible sur Replica ✓                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Tables système GTID

**mysql.gtid_slave_pos** :

```sql
-- Structure
DESC mysql.gtid_slave_pos;
-- +-----------+---------------------+------+-----+---------+-------+
-- | Field     | Type                | Null | Key | Default | Extra |
-- +-----------+---------------------+------+-----+---------+-------+
-- | domain_id | int(10) unsigned    | NO   | PRI | NULL    |       |
-- | sub_id    | bigint(20) unsigned | NO   | PRI | NULL    |       |
-- | server_id | int(10) unsigned    | NO   |     | NULL    |       |
-- | seq_no    | bigint(20) unsigned | NO   |     | NULL    |       |
-- +-----------+---------------------+------+-----+---------+-------+

-- Contenu
SELECT * FROM mysql.gtid_slave_pos;
-- +-----------+--------+-----------+--------+
-- | domain_id | sub_id | server_id | seq_no |
-- +-----------+--------+-----------+--------+
-- |         0 |      1 |         1 |   1000 |
-- +-----------+--------+-----------+--------+
```

**Utilité** :
- Persiste la position GTID du Replica
- Survit aux redémarrages
- Permet de reprendre réplication au bon endroit

### Variables GTID essentielles

```sql
-- Position actuelle globale
SELECT @@gtid_current_pos;
-- 0-1-1000

-- Position dans le binlog local
SELECT @@gtid_binlog_pos;
-- 0-1-1000

-- Position répliquée (sur Replica uniquement)
SELECT @@gtid_slave_pos;
-- 0-1-1000

-- GTID de la transaction actuelle (dans transaction)
SELECT @@gtid_binlog_state;

-- Configuration
SELECT 
  @@gtid_domain_id AS domain,
  @@gtid_strict_mode AS strict_mode,
  @@server_id AS server_id;
-- +--------+-------------+-----------+
-- | domain | strict_mode | server_id |
-- +--------+-------------+-----------+
-- |      0 |           1 |         1 |
-- +--------+-------------+-----------+
```

### Skip automatique de transactions

**Mécanisme de déduplication** :

```
Scenario:
┌──────────┐
│ Primary  │ Génère: 0-1-1000
└─────┬────┘
      │
      ├──────────────┬──────────────┐
      │              │              │
      ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────┐
│Replica A │   │Replica B │   │Replica C │
│0-1-1000  │   │0-1-1000  │   │0-1-999   │
└──────────┘   └──────────┘   └──────────┘

Replica C est en retard, reçoit 0-1-1000 du Primary

SQL Thread de Replica C:
1. Lit GTID_EVENT: 0-1-1000
2. Vérifie mysql.gtid_slave_pos
   → Domain 0, seq_no 999 (pas 1000)
3. Transaction 0-1-1000 NON présente
4. → APPLIQUE la transaction
5. Mise à jour gtid_slave_pos → 0-1-1000

Si Replica C recevait à nouveau 0-1-1000 (par erreur):
1. Lit GTID_EVENT: 0-1-1000
2. Vérifie mysql.gtid_slave_pos
   → Domain 0, seq_no 1000 (déjà présent !)
3. → SKIP la transaction (pas d'erreur)
```

💡 **Avantage** : Robustesse face aux doublons, simplification du failover.

---

## Cas d'Usage et Quand Utiliser GTID

### ✅ Scénarios recommandés pour GTID

**1. Haute disponibilité avec failover automatique**

```
Production avec SLA strict:
- Failover automatisé (Orchestrator, MHA)
- Zero calcul de position manuel
- RTO (Recovery Time Objective) < 1 minute

→ GTID obligatoire
```

**2. Topologies complexes**

```
- Multi-source replication
- Cascade multi-niveaux
- Topologies en étoile ou mesh
- Circular replication (active-active)

→ GTID hautement recommandé
```

**3. Environnements cloud-native**

```
- Kubernetes avec MariaDB Operator
- Scaling horizontal automatique
- Ajout/retrait dynamique de Replicas
- Infrastructure as Code (Terraform, Ansible)

→ GTID simplifie l'automatisation
```

**4. Migration de données**

```
- Consolidation de bases
- Split de bases
- Réorganisation de topologie
- Changement de Primary

→ GTID facilite les opérations
```

**5. Audit et compliance**

```
- Traçabilité précise de chaque transaction
- Vérification d'intégrité multi-serveur
- Point-in-time recovery précis
- Forensics après incident

→ GTID améliore la traçabilité
```

### ❌ Scénarios où GTID peut être évité

**1. Setup simple et stable**

```
1 Primary + 1-2 Replicas
- Pas de failover prévu
- Topologie figée
- Faible volumétrie

→ Positions binlog suffisantes (plus simple)
```

**2. Compatibilité legacy requise**

```
- Outils tiers ne supportant pas GTID
- Migration depuis version très ancienne (< 10.0)
- Contraintes organisationnelles

→ Rester sur positions binlog
```

**3. Overhead non acceptable**

```
- Très haute fréquence de transactions (>50K TPS)
- Mémoire limitée (GTID = léger overhead)
- Besoin de performance absolue

→ Benchmarker avant migration
```

---

## Performance et Overhead

### Impact sur les performances

**Overhead de stockage** :

```
Binary log avec GTID:
┌──────────────────────────┐
│ GTID_EVENT (26-40 bytes) │ ← Overhead
├──────────────────────────┤
│ QUERY_EVENT (BEGIN)      │
│ TABLE_MAP_EVENT          │
│ WRITE_ROWS_EVENT         │
│ XID_EVENT (COMMIT)       │
└──────────────────────────┘

Sans GTID:
┌──────────────────────────┐
│ QUERY_EVENT (BEGIN)      │
│ TABLE_MAP_EVENT          │
│ WRITE_ROWS_EVENT         │
│ XID_EVENT (COMMIT)       │
└──────────────────────────┘

Overhead: ~30 bytes par transaction
        = Négligeable pour transactions normales
        = Significatif pour micro-transactions
```

**Benchmarks** :

```
Sysbench OLTP Write-Only (10 tables, 1M rows)
Configuration: 16 threads, 300 secondes

Sans GTID:
- TPS: 15,420
- Latence P95: 8.2ms
- Binlog size: 1073741824 bytes

Avec GTID:
- TPS: 15,180 (-1.6%)
- Latence P95: 8.4ms (+0.2ms)
- Binlog size: 1085839360 bytes (+1.1%)

Conclusion: Impact minimal sur workloads normaux
```

**Mémoire** :

```sql
-- Vérifier l'utilisation mémoire GTID
SHOW GLOBAL STATUS LIKE 'Gtid%';
-- Généralement < 1MB pour des milliers de GTID
```

### Optimisations

**1. Compaction automatique**

MariaDB compacte automatiquement les GTID en intervalles :

```sql
-- Avant compaction
0-1-1,0-1-2,0-1-3,...,0-1-1000

-- Après compaction automatique
0-1-1000

-- Économie mémoire et stockage
```

**2. Purge des anciens GTID**

```sql
-- Purge binlogs = purge GTID associés
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;

-- Les GTID des binlogs purgés sont oubliés
```

**3. GTID strict mode pour performance**

```sql
-- Évite vérifications supplémentaires
SET GLOBAL gtid_strict_mode = ON;

-- Garantit:
-- - Pas de transactions sans GTID
-- - Pas de changements domain_id inopinés
-- - Performance légèrement améliorée
```

---

## Limitations et Considérations

### 1. Incompatibilité MySQL GTID

```
MariaDB GTID ≠ MySQL GTID

MariaDB → MySQL avec GTID : ❌ Non supporté
MySQL → MariaDB avec GTID : ✅ Supporté (lecture seule)
```

**Workaround** : Migration MySQL → MariaDB

```sql
-- MySQL Primary avec MySQL GTID
-- MariaDB Replica peut lire et convertir en interne
CHANGE MASTER TO
  MASTER_HOST = 'mysql-primary',
  MASTER_USE_GTID = slave_pos;

-- MariaDB convertit MySQL UUID:seq en Domain-Server-Seq
```

### 2. Transactions locales

```sql
-- Sur un Replica avec GTID activé
-- Écriture locale (non répliquée)
SET sql_log_bin = 0;
INSERT INTO local_table VALUES (1);
SET sql_log_bin = 1;

-- Problème: Pas de GTID généré pour cette transaction
-- → Potentiel gap dans la séquence
```

**Solution** :

```sql
-- Toujours utiliser GTID même pour transactions locales
-- Ou activer gtid_strict_mode pour interdire ce cas
SET GLOBAL gtid_strict_mode = ON;
```

### 3. Migration progressive

```
Migration positions → GTID:
- Nécessite mode hybride temporaire
- Test approfondi requis
- Rollback possible mais complexe
```

**Phases** :

```
Phase 1: Activer GTID sans le rendre obligatoire
Phase 2: Vérifier génération GTID sur Primary
Phase 3: Basculer Replicas un par un vers GTID
Phase 4: Activer gtid_strict_mode
Phase 5: Documenter et former équipes
```

### 4. Outils tiers

Certains outils anciens peuvent ne pas supporter GTID :

```
✅ Supportent GTID:
- Percona Toolkit (récent)
- Orchestrator
- ProxySQL
- MaxScale
- pt-table-checksum (avec --replicate-do-table)

⚠️ Support partiel ou absent:
- Anciens scripts maison
- Outils legacy pré-2015
```

---

## Sous-sections du Chapitre 13.4

Ce chapitre se décompose en 2 sous-sections détaillées :

### 📖 13.4.1 Configuration GTID

- Activation GTID sur Primary et Replicas
- Paramètres de configuration (`gtid_domain_id`, `gtid_strict_mode`, etc.)
- Migration depuis positions binlog (step-by-step)
- Mode hybride (positions + GTID)
- Configuration multi-source avec domains
- Troubleshooting activation

### 📖 13.4.2 Avantages pour Failover

- Scénarios de failover détaillés
- Promotion automatique de Replica
- Calcul automatique de positions
- Outils d'orchestration (Orchestrator, MHA, MaxScale)
- Switchover planifié vs Failover d'urgence
- Procédures de test de failover
- Recovery après panne Primary

---

## ✅ Points clés à retenir

1. **Format MariaDB GTID** : `Domain-Server-Sequence` (ex: `0-1-1000`)

2. **3 composants essentiels** : Domain ID (flux indépendants), Server ID (origine), Sequence (compteur)

3. **Incompatibilité MySQL** : MariaDB GTID ≠ MySQL GTID (formats différents)

4. **Failover automatisé** : Plus besoin de calculer positions manuellement

5. **Topologies complexes** : Multi-source, cascade, bi-directionnel simplifiés

6. **Skip automatique** : Transactions déjà appliquées ignorées (pas d'erreur duplicate)

7. **Overhead minimal** : ~1-2% performance, +1% stockage binlog

8. **GTID strict mode** : Recommandé pour garantir cohérence totale

9. **Migration progressive** : Possible en mode hybride sans downtime

10. **Recommandation 2025** : GTID par défaut pour tout nouveau déploiement

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Global Transaction ID](https://mariadb.com/kb/en/gtid/)
- [📖 GTID System Variables](https://mariadb.com/kb/en/gtid-system-variables/)
- [📖 Using MariaDB GTID](https://mariadb.com/kb/en/using-mariadb-gtid/)
- [📖 Differences between MySQL and MariaDB GTID](https://mariadb.com/kb/en/gtid/#differences-between-mariadb-and-mysql-gtid)

### Articles techniques

- [🔗 MariaDB GTID Explained](https://mariadb.com/resources/blog/mariadb-gtid-explained/)
- [🔗 Migrating to GTID-based Replication](https://mariadb.com/kb/en/migrating-to-gtid-based-replication/)
- [🔗 GTID Best Practices](https://mariadb.com/resources/blog/gtid-best-practices/)

### Outils

- **Orchestrator** : Failover automatique avec support GTID natif
- **pt-table-checksum** : Vérification cohérence avec GTID
- **MaxScale** : Routing intelligent avec awareness GTID

---

## ➡️ Sections suivantes

### 13.4.1 Configuration GTID

Guide complet d'activation et configuration GTID : paramètres système, migration depuis positions binlog, mode hybride, multi-source avec domains, et troubleshooting.

### 13.4.2 Avantages pour Failover

Étude approfondie des scénarios de failover avec GTID : promotion automatique, orchestration, outils de HA, procédures de test, et recovery après incident.

GTID transforme radicalement la gestion opérationnelle de la réplication MariaDB. Prêt à plonger dans la configuration ?

---


⏭️ [Configuration GTID](/13-replication/04.1-configuration-gtid.md)
