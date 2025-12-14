🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.11 Online Schema Change (ALTER TABLE non-bloquant) 🆕

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Chapitre 11 (Administration), compréhension DDL et réplication

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre les **limitations des ALTER TABLE bloquants**
- Utiliser **ALGORITHM=INPLACE** pour modifications sans copie
- Maîtriser les **modes LOCK** (NONE, SHARED, EXCLUSIVE)
- Implémenter des **migrations zero-downtime**
- Utiliser **pt-online-schema-change** et **gh-ost**
- Gérer les **opérations DDL compatibles online**
- Monitorer les **migrations en cours** (progression, impact)
- Planifier des **stratégies de rollback**
- Optimiser les **performances** des migrations longues

---

## Introduction

L'**Online Schema Change** (OSC) permet de modifier la structure des tables (ALTER TABLE) en production **sans bloquer** les opérations de lecture/écriture, évitant ainsi les downtimes lors des migrations de schéma.

### Problème : ALTER TABLE Bloquant

**Comportement classique** (avant OSC) :

```sql
-- Migration : Ajouter colonne sur table 100M lignes
ALTER TABLE orders ADD COLUMN priority INT DEFAULT 0;

-- Processus :
-- 1. Verrouillage EXCLUSIF de la table
-- 2. Création table temporaire avec nouvelle structure
-- 3. Copie 100M lignes (peut prendre 30+ minutes)
-- 4. Swap table originale ↔ nouvelle
-- 5. Déverrouillage

-- Pendant 30 minutes :
-- ❌ SELECT bloqués (queued)
-- ❌ INSERT/UPDATE/DELETE impossibles
-- ❌ Application en erreur (timeout)
-- ❌ Downtime = durée de la copie
```

**Impact en production** :

```
Connexions actives : 1000
ALTER TABLE démarre → Table LOCKED
  ↓
Queue de 1000 requêtes en attente
  ↓
Timeout applicatif après 30s
  ↓
❌ Indisponibilité service 30 minutes
```

**Symptômes typiques** :
- 💥 Pic d'erreurs applicatives
- 📈 Temps de réponse API > 30s
- 🔥 Queue MySQL saturée (max_connections atteint)
- 😱 Perte de revenus, SLA breach

### Solution : Online Schema Change

**Principe** : Modifier la structure **sans bloquer** les opérations concurrentes.

```
Méthodes OSC :

1. ALGORITHM=INPLACE (MariaDB natif)
   - Modifications sur place (pas de copie complète)
   - Lock minimal (millisecondes)
   
2. Outils externes (pt-osc, gh-ost)
   - Copie progressive en arrière-plan
   - Zero downtime garanti
```

**Bénéfices** :
- ✅ **Zero downtime** : Application continue de fonctionner
- ✅ **Performance maintenue** : Pas de lock prolongé
- ✅ **Rollback possible** : Abandon sans corruption
- ✅ **Monitoring** : Progression en temps réel
- ✅ **Production-safe** : Validation avant swap

---

## ALGORITHM et LOCK : Contrôle Natif MariaDB

### ALGORITHM : Méthode de Migration

MariaDB supporte plusieurs algorithmes pour ALTER TABLE :

#### ALGORITHM=INPLACE

**Description** : Modifie la table **sur place** sans copier toutes les données.

```sql
-- Syntaxe
ALTER TABLE table_name
  ADD COLUMN column_name type,
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

**Processus** :
```
1. Acquiert metadata lock (court)
2. Modifie structure interne InnoDB
3. Ajoute metadata au dictionnaire de données
4. Libère lock
5. Reconstruction index si nécessaire (background)

Durée : Secondes (indépendant du nombre de lignes)
Lock : Millisecondes
```

**Opérations supportées INPLACE** :

| Opération | INPLACE | Remarques |
|-----------|---------|-----------|
| **ADD COLUMN** (fin de table) | ✅ Oui | Rapide, pas de rebuild |
| **ADD COLUMN** (milieu) | ⚠️ Partiel | Peut nécessiter rebuild |
| **DROP COLUMN** | ✅ Oui | Rapide |
| **RENAME COLUMN** | ✅ Oui | Instantané |
| **MODIFY COLUMN** (type compatible) | ⚠️ Partiel | INT→BIGINT ok, VARCHAR→INT non |
| **ADD INDEX** | ✅ Oui | Construction online |
| **DROP INDEX** | ✅ Oui | Instantané |
| **CHANGE COLUMN** (type incompatible) | ❌ Non | Nécessite COPY |
| **ALTER COLUMN DEFAULT** | ✅ Oui | Instantané |
| **RENAME TABLE** | ✅ Oui | Instantané |

**Exemple** :
```sql
-- ✅ INPLACE supporté (colonne en fin, avec défaut)
ALTER TABLE products
  ADD COLUMN stock_location VARCHAR(100) DEFAULT 'Warehouse A',
  ALGORITHM=INPLACE,
  LOCK=NONE;
-- Durée : ~2 secondes (10M lignes)
-- Pendant : SELECT/INSERT/UPDATE/DELETE continuent

-- Vérifier compatibilité avant exécution
ALTER TABLE products
  ADD COLUMN test_column INT,
  ALGORITHM=INPLACE;
-- Si incompatible → Erreur immédiate
-- "ALGORITHM=INPLACE is not supported. Reason: ..."
```

#### ALGORITHM=COPY

**Description** : Crée une **nouvelle table** avec copie complète des données.

```sql
ALTER TABLE table_name
  MODIFY COLUMN column_name new_type,
  ALGORITHM=COPY;
```

**Processus** :
```
1. Crée table temporaire avec nouvelle structure
2. Copie toutes les lignes (peut être long)
3. Swap table originale ↔ temporaire
4. Supprime ancienne table

Durée : Proportionnelle au nombre de lignes
Lock : Selon LOCK mode spécifié
```

**Quand COPY est nécessaire** :
- Modification type incompatible (VARCHAR → INT)
- Ajout colonne au milieu avec NOT NULL sans DEFAULT
- Changement ENGINE (InnoDB → MyISAM)
- Modification PRIMARY KEY

**Exemple** :
```sql
-- ❌ Nécessite COPY (type incompatible)
ALTER TABLE orders
  MODIFY COLUMN total VARCHAR(50),  -- Était DECIMAL(10,2)
  ALGORITHM=COPY,
  LOCK=SHARED;
-- Durée : 15 minutes (50M lignes)
-- Lock : SHARED (lectures ok, écritures bloquées)
```

#### ALGORITHM=DEFAULT

**Description** : MariaDB **choisit automatiquement** INPLACE si possible, sinon COPY.

```sql
ALTER TABLE table_name
  ADD COLUMN column_name type;
-- Équivalent à ALGORITHM=DEFAULT
```

**Recommandation** : Toujours **spécifier explicitement** ALGORITHM en production pour prédictibilité.

### LOCK : Niveau de Verrouillage

Contrôle le niveau de concurrence pendant l'ALTER TABLE.

#### LOCK=NONE

**Description** : **Aucun lock** sur la table. Lectures et écritures continuent.

```sql
ALTER TABLE products
  ADD COLUMN rating DECIMAL(3,2) DEFAULT 0,
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

**Compatible avec** :
- Opérations INPLACE uniquement
- ADD/DROP COLUMN (avec restrictions)
- ADD INDEX (construction online)

**Non compatible avec** :
- ALGORITHM=COPY
- Modifications structure complexes

**Vérification** :
```sql
-- Si LOCK=NONE incompatible → Erreur
ALTER TABLE products
  MODIFY COLUMN price INT,
  LOCK=NONE;
-- ERROR: LOCK=NONE is not supported. Reason: COPY algorithm requires a lock
```

#### LOCK=SHARED

**Description** : Lock partagé. **Lectures ok**, écritures bloquées.

```sql
ALTER TABLE products
  MODIFY COLUMN description TEXT,
  ALGORITHM=COPY,
  LOCK=SHARED;
```

**Comportement** :
- ✅ SELECT continuent
- ❌ INSERT/UPDATE/DELETE bloqués (en queue)

**Usage** : Tables read-heavy avec fenêtre de maintenance pour écritures.

#### LOCK=EXCLUSIVE

**Description** : Lock exclusif. **Tout bloqué** (lectures + écritures).

```sql
ALTER TABLE products
  ENGINE=MyISAM,
  LOCK=EXCLUSIVE;
```

**Comportement** :
- ❌ SELECT bloqués
- ❌ INSERT/UPDATE/DELETE bloqués

**Usage** : Maintenance hors heures de production, ou table peu utilisée.

#### LOCK=DEFAULT

**Description** : MariaDB choisit le **lock minimum** nécessaire.

```sql
ALTER TABLE products
  ADD INDEX idx_category (category);
-- MariaDB choisira LOCK=NONE si ALGORITHM=INPLACE possible
```

**Tableau récapitulatif** :

| LOCK Mode | SELECT | INSERT/UPDATE/DELETE | Durée Lock | Usage |
|-----------|--------|----------------------|------------|-------|
| **NONE** | ✅ Ok | ✅ Ok | Millisecondes | Production, INPLACE |
| **SHARED** | ✅ Ok | ❌ Bloqué | Durée ALTER | Fenêtre maintenance lectures |
| **EXCLUSIVE** | ❌ Bloqué | ❌ Bloqué | Durée ALTER | Hors production |
| **DEFAULT** | Auto | Auto | Auto | Dev/test |

---

## Opérations DDL Online

### ADD COLUMN (En Fin de Table)

**✅ Online (INPLACE + LOCK=NONE)** :

```sql
-- Ajouter colonne à la fin avec DEFAULT
ALTER TABLE orders
  ADD COLUMN delivery_notes TEXT DEFAULT NULL,
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Durée : ~5 secondes (100M lignes)
-- Impact : Aucun (0% downtime)
```

**Processus interne** :
1. Ajoute metadata colonne au dictionnaire InnoDB
2. DEFAULT stocké au niveau table (pas dans chaque ligne)
3. SELECT retourne DEFAULT pour lignes existantes
4. INSERT/UPDATE écrivent vraie valeur

**Limitation** : NOT NULL sans DEFAULT nécessite COPY.

```sql
-- ❌ Nécessite COPY (NOT NULL sans DEFAULT)
ALTER TABLE orders
  ADD COLUMN mandatory_field VARCHAR(100) NOT NULL,
  ALGORITHM=INPLACE;
-- ERROR: ALGORITHM=INPLACE is not supported

-- ✅ Solution : Ajouter avec DEFAULT, puis remplir
ALTER TABLE orders
  ADD COLUMN mandatory_field VARCHAR(100) DEFAULT 'pending',
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Remplir progressivement (batch)
UPDATE orders
SET mandatory_field = calculate_value(order_id)
WHERE order_id BETWEEN 1 AND 10000;
-- Répéter par batches
```

### ADD INDEX (Construction Online)

**✅ Online (INPLACE + LOCK=NONE)** :

```sql
-- Créer index sans bloquer table
ALTER TABLE orders
  ADD INDEX idx_customer_date (customer_id, order_date),
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Ou CREATE INDEX directement
CREATE INDEX idx_customer_date 
ON orders(customer_id, order_date)
ALGORITHM=INPLACE
LOCK=NONE;
```

**Processus** :
1. Scan complet table (en arrière-plan)
2. Trie données pour index
3. Construit B-tree index
4. Active index (atomic)

**Progression** :
```sql
-- Surveiller dans Performance Schema
SELECT 
  EVENT_NAME,
  WORK_COMPLETED,
  WORK_ESTIMATED,
  ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS pct_complete
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE '%Creating sort index%'
   OR EVENT_NAME LIKE '%altering table%';
```

**Durée typique** :
- 10M lignes, 2 colonnes INT : 2-5 minutes
- 100M lignes, 2 colonnes VARCHAR : 20-40 minutes
- Impact CPU : +20-40% pendant construction

### DROP COLUMN

**✅ Online (INPLACE + LOCK=NONE)** :

```sql
-- Supprimer colonne instantanément
ALTER TABLE products
  DROP COLUMN obsolete_field,
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Durée : ~1 seconde (quelle que soit taille table)
```

**Processus** :
- Supprime metadata du dictionnaire
- Espace disque libéré progressivement (purge InnoDB)
- SELECT ne retournent plus la colonne

**Note** : Espace disque récupéré après OPTIMIZE TABLE (optionnel).

### MODIFY COLUMN (Type Compatible)

**✅ Online si types compatibles** :

```sql
-- Agrandir VARCHAR : INPLACE
ALTER TABLE products
  MODIFY COLUMN name VARCHAR(500),  -- Était VARCHAR(255)
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- INT → BIGINT : INPLACE
ALTER TABLE orders
  MODIFY COLUMN order_id BIGINT,  -- Était INT
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

**❌ Copy si types incompatibles** :

```sql
-- VARCHAR → INT : Nécessite COPY
ALTER TABLE products
  MODIFY COLUMN code INT,  -- Était VARCHAR(50)
  ALGORITHM=COPY,
  LOCK=SHARED;
```

**Compatibilité types** :

| Modification | INPLACE | Notes |
|--------------|---------|-------|
| VARCHAR(N) → VARCHAR(M>N) | ✅ Oui | Si <= 255 → <= 255 ou > 255 → > 255 |
| INT → BIGINT | ✅ Oui | Extension entier |
| DECIMAL(M,D) → DECIMAL(M',D') | ⚠️ Partiel | Si M' ≥ M et D' ≥ D |
| VARCHAR → TEXT | ❌ Non | Nécessite COPY |
| DATE → DATETIME | ❌ Non | Nécessite COPY |

### CHANGE DEFAULT

**✅ Online (instantané)** :

```sql
-- Modifier DEFAULT d'une colonne
ALTER TABLE products
  ALTER COLUMN status SET DEFAULT 'active',
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Durée : < 1 seconde
-- Aucun impact sur lignes existantes
```

---

## Outils Externes : pt-osc et gh-ost

### pt-online-schema-change (Percona Toolkit)

**Description** : Outil Perl pour migrations sans downtime, compatible MySQL/MariaDB.

**Principe** :
```
1. Créer table "shadow" (nouvelle structure)
2. Créer triggers pour synchroniser changements
3. Copier données par batches (chunking)
4. Swap atomique table originale ↔ shadow
5. Nettoyer triggers et ancienne table
```

**Installation** :
```bash
# Debian/Ubuntu
sudo apt-get install percona-toolkit

# Red Hat/CentOS
sudo yum install percona-toolkit

# Vérifier installation
pt-online-schema-change --version
```

**Exemple basique** :
```bash
pt-online-schema-change \
  --alter "ADD COLUMN priority INT DEFAULT 0" \
  --host=localhost \
  --user=root \
  --password=secret \
  --database=ecommerce \
  --table=orders \
  --execute

# Options importantes :
# --alter : DDL à exécuter
# --execute : Exécuter (sans = dry-run)
# --chunk-size : Taille des batches (défaut 1000)
# --max-lag : Pause si réplication en retard
# --critical-load : Abandon si charge élevée
```

**Exemple avancé** :
```bash
pt-online-schema-change \
  --alter "
    ADD COLUMN shipping_address TEXT,
    ADD COLUMN tracking_number VARCHAR(100),
    ADD INDEX idx_tracking (tracking_number)
  " \
  --host=prod-db-01 \
  --port=3306 \
  --user=migration_user \
  --password=$DB_PASSWORD \
  --database=ecommerce \
  --table=orders \
  \
  --chunk-size=2000 \
  --chunk-time=0.5 \
  --max-lag=2 \
  --critical-load="Threads_running=100" \
  --max-load="Threads_running=50" \
  \
  --progress=time,30 \
  --print \
  --execute

# Paramètres :
# --chunk-size=2000 : 2000 lignes par batch
# --chunk-time=0.5 : Viser 0.5s par chunk
# --max-lag=2 : Pause si slave > 2s retard
# --critical-load : Abandon si > 100 threads
# --max-load : Pause si > 50 threads
# --progress=time,30 : Afficher progression chaque 30s
```

**Monitoring progression** :
```bash
# Terminal 1 : Exécution
pt-online-schema-change ... --execute

# Terminal 2 : Surveiller
watch -n 5 "mysql -e '
  SELECT 
    TABLE_NAME,
    TABLE_ROWS,
    DATA_LENGTH / 1024 / 1024 AS data_mb
  FROM information_schema.TABLES
  WHERE TABLE_SCHEMA = \"ecommerce\"
    AND TABLE_NAME LIKE \"_orders%\"
  ORDER BY TABLE_NAME
'"

# Sortie :
# TABLE_NAME       | TABLE_ROWS | data_mb
# _orders_new      | 25000000   | 1250.5  (shadow, progression)
# orders           | 50000000   | 2500.0  (originale)
```

**Avantages pt-osc** :
- ✅ Contrôle précis (chunk-size, load limits)
- ✅ Pause/resume automatique (max-load)
- ✅ Abandon si réplication lag
- ✅ Compatible MySQL/MariaDB/Percona
- ✅ Mature, largement utilisé

**Inconvénients** :
- ⚠️ Triggers overhead (~10-15%)
- ⚠️ Double espace disque temporairement
- ⚠️ Complexe pour très grandes tables (> 500M lignes)

### gh-ost (GitHub Online Schema Migration)

**Description** : Outil Go développé par GitHub, sans triggers.

**Principe** :
```
1. Créer table "ghost" (nouvelle structure)
2. Lire binlog pour capturer changements
3. Copier données + appliquer changements binlog
4. Cutover atomique (rename)
```

**Installation** :
```bash
# Télécharger binaire
wget https://github.com/github/gh-ost/releases/download/v1.1.6/gh-ost-binary-linux-amd64-20231207141714.tar.gz
tar -xzf gh-ost-*.tar.gz
sudo mv gh-ost /usr/local/bin/
chmod +x /usr/local/bin/gh-ost

# Vérifier
gh-ost --version
```

**Exemple basique** :
```bash
gh-ost \
  --user="root" \
  --password="secret" \
  --host="localhost" \
  --database="ecommerce" \
  --table="orders" \
  --alter="ADD COLUMN priority INT DEFAULT 0" \
  --execute

# Options importantes :
# --execute : Exécuter (sans = test mode)
# --allow-on-master : Autoriser sur master (requis)
# --chunk-size : Taille batches (défaut 1000)
```

**Exemple avancé (production)** :
```bash
gh-ost \
  --user="migration_user" \
  --password="$DB_PASSWORD" \
  --host="prod-db-master.example.com" \
  --port=3306 \
  --database="ecommerce" \
  --table="orders" \
  --alter="
    ADD COLUMN shipping_provider VARCHAR(50),
    ADD INDEX idx_provider (shipping_provider)
  " \
  \
  --allow-on-master \
  --assume-rbr \
  --chunk-size=5000 \
  --max-lag-millis=2000 \
  --critical-load="Threads_running=100" \
  --max-load="Threads_running=80" \
  --heartbeat-interval-millis=500 \
  \
  --initially-drop-ghost-table \
  --initially-drop-old-table \
  --ok-to-drop-table \
  \
  --serve-socket-file="/tmp/gh-ost.orders.sock" \
  --execute

# Paramètres clés :
# --assume-rbr : Binlog en ROW format (requis)
# --chunk-size=5000 : Batches de 5000 lignes
# --max-lag-millis : Pause si replica > 2s lag
# --serve-socket-file : Socket pour contrôle interactif
```

**Contrôle interactif** :
```bash
# Pendant migration, envoyer commandes via socket

# Voir statut
echo "status" | nc -U /tmp/gh-ost.orders.sock

# Pausez migration
echo "throttle" | nc -U /tmp/gh-ost.orders.sock

# Reprendre
echo "no-throttle" | nc -U /tmp/gh-ost.orders.sock

# Forcer cutover immédiat
echo "unpostpone" | nc -U /tmp/gh-ost.orders.sock

# Abandonner (rollback safe)
echo "panic" | nc -U /tmp/gh-ost.orders.sock
```

**Avantages gh-ost** :
- ✅ **Pas de triggers** (pas d'overhead)
- ✅ Pause/resume à tout moment
- ✅ Rollback safe (avant cutover)
- ✅ Contrôle interactif (socket)
- ✅ Moins d'impact performance

**Inconvénients** :
- ⚠️ Nécessite binlog ROW format
- ⚠️ Binlog doit être activé
- ⚠️ Moins mature que pt-osc

### Comparaison pt-osc vs gh-ost

| Aspect | pt-online-schema-change | gh-ost |
|--------|-------------------------|--------|
| **Méthode** | Triggers | Binlog streaming |
| **Overhead** | +10-15% (triggers) | +5-8% (binlog read) |
| **Rollback** | Difficile après swap | Facile (avant cutover) |
| **Contrôle** | Via signaux Unix | Socket interactif |
| **Binlog requis** | ❌ Non | ✅ Oui (ROW format) |
| **Maturité** | ✅ Très mature | ⚠️ Assez récent |
| **Pause/Resume** | ✅ Oui | ✅ Oui |
| **Très grandes tables** | ⚠️ Peut être lent | ✅ Meilleur |

**Recommandation** :
- **pt-osc** : Environnements sans binlog, migrations < 100M lignes
- **gh-ost** : Productions critiques, très grandes tables (> 100M), besoin contrôle fin

---

## Stratégies de Migration Zero-Downtime

### Stratégie 1 : ALGORITHM=INPLACE (Natif)

**Quand utiliser** :
- Opération compatible INPLACE
- Table < 1 milliard de lignes
- Pas besoin contrôle fin

**Processus** :
```sql
-- 1. Vérifier compatibilité (dry-run)
ALTER TABLE orders
  ADD COLUMN priority INT DEFAULT 0,
  ALGORITHM=INPLACE,
  LOCK=NONE;
-- Si erreur → Pas compatible, utiliser outil externe

-- 2. Si compatible, exécuter
-- (production-safe)

-- 3. Monitorer progression
SELECT 
  EVENT_NAME,
  WORK_COMPLETED,
  WORK_ESTIMATED
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE '%alter%';
```

**Avantages** :
- ✅ Simple (SQL natif)
- ✅ Rapide (secondes)
- ✅ Pas d'outil externe

**Inconvénients** :
- ⚠️ Limité aux opérations INPLACE
- ⚠️ Pas de pause/resume
- ⚠️ Pas de rollback facile

### Stratégie 2 : Colonne Temporaire + Migration Progressive

**Quand utiliser** :
- Modification type incompatible INPLACE
- Besoin contrôle total
- Migration critique

**Processus** :
```sql
-- Exemple : Changer type colonne status VARCHAR → ENUM

-- Étape 1 : Ajouter nouvelle colonne (INPLACE)
ALTER TABLE orders
  ADD COLUMN status_new ENUM('pending','processing','shipped','delivered') DEFAULT 'pending',
  ALGORITHM=INPLACE,
  LOCK=NONE;
-- Durée : ~5 secondes

-- Étape 2 : Copier données progressivement (batch)
-- Script exécuté en arrière-plan
SET @batch_size = 10000;
SET @last_id = 0;

WHILE @last_id < (SELECT MAX(order_id) FROM orders) DO
  UPDATE orders
  SET status_new = CASE status
    WHEN 'P' THEN 'pending'
    WHEN 'R' THEN 'processing'
    WHEN 'S' THEN 'shipped'
    WHEN 'D' THEN 'delivered'
    ELSE 'pending'
  END
  WHERE order_id > @last_id
    AND order_id <= @last_id + @batch_size
    AND status_new IS NULL;
  
  SET @last_id = @last_id + @batch_size;
  
  -- Pause entre batches (éviter surcharge)
  DO SLEEP(0.1);
END WHILE;

-- Étape 3 : Vérifier migration complète
SELECT 
  COUNT(*) AS total,
  COUNT(status_new) AS migrated,
  COUNT(*) - COUNT(status_new) AS remaining
FROM orders;
-- Attendre remaining = 0

-- Étape 4 : Applications migrent vers status_new

-- Étape 5 : Swap colonnes (après validation)
ALTER TABLE orders
  DROP COLUMN status,
  CHANGE COLUMN status_new status ENUM('pending','processing','shipped','delivered'),
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

**Avantages** :
- ✅ Contrôle total
- ✅ Rollback facile (garder ancienne colonne)
- ✅ Validation avant swap
- ✅ Zero downtime

**Inconvénients** :
- ⚠️ Complexe (plusieurs étapes)
- ⚠️ Double stockage temporairement

### Stratégie 3 : Table Shadow + Swap

**Quand utiliser** :
- Restructuration complète
- Très grandes tables
- Besoin rollback garanti

**Processus** :
```sql
-- Exemple : Partitionner table existante

-- Étape 1 : Créer table shadow avec nouvelle structure
CREATE TABLE orders_partitioned LIKE orders;

ALTER TABLE orders_partitioned
PARTITION BY RANGE (YEAR(order_date)) (
  PARTITION p_2023 VALUES LESS THAN (2024),
  PARTITION p_2024 VALUES LESS THAN (2025),
  PARTITION p_2025 VALUES LESS THAN (2026),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Étape 2 : Utiliser gh-ost ou pt-osc pour copier données
gh-ost \
  --user="root" \
  --password="secret" \
  --host="localhost" \
  --database="ecommerce" \
  --table="orders" \
  --alter="ENGINE=InnoDB" \
  --ghost-table="orders_partitioned" \
  --execute

# Pendant copie : orders continue de recevoir données
# gh-ost synchronise via binlog

-- Étape 3 : Cutover (swap atomique)
# gh-ost fait automatiquement :
RENAME TABLE 
  orders TO orders_old,
  orders_partitioned TO orders;
# Durée : < 1 seconde, atomique

-- Étape 4 : Validation
-- Si problème détecté, rollback rapide :
RENAME TABLE
  orders TO orders_broken,
  orders_old TO orders;

-- Étape 5 : Nettoyage (après validation complète)
DROP TABLE orders_old;
```

---

## Monitoring et Diagnostique

### Suivre Progression ALTER TABLE

```sql
-- 1. Performance Schema : Stages actuels
SELECT 
  EVENT_NAME,
  WORK_COMPLETED,
  WORK_ESTIMATED,
  ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS pct_complete,
  TIMER_WAIT / 1000000000 AS elapsed_sec
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE '%alter%'
   OR EVENT_NAME LIKE '%Creating%'
   OR EVENT_NAME LIKE '%copy%';

-- Exemple output :
-- EVENT_NAME                  | WORK_COMPLETED | WORK_ESTIMATED | pct_complete | elapsed_sec
-- stage/sql/copy to tmp table | 25000000       | 100000000      | 25.00        | 1250

-- 2. Processlist : ALTER en cours
SHOW PROCESSLIST;
-- Chercher "Altering table", "Creating sort index", etc.

-- Ou via information_schema
SELECT 
  ID,
  USER,
  HOST,
  DB,
  COMMAND,
  TIME AS duration_sec,
  STATE,
  INFO
FROM information_schema.PROCESSLIST
WHERE STATE LIKE '%alter%'
   OR INFO LIKE '%ALTER TABLE%';
```

### Estimer Durée Restante

```sql
-- Formule : (temps_écoulé / pct_complete) * (100 - pct_complete)

SELECT 
  EVENT_NAME,
  WORK_COMPLETED,
  WORK_ESTIMATED,
  ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS pct_complete,
  TIMER_WAIT / 1000000000 AS elapsed_sec,
  ROUND(
    (TIMER_WAIT / 1000000000) / (WORK_COMPLETED / WORK_ESTIMATED) * 
    (1 - WORK_COMPLETED / WORK_ESTIMATED)
  , 0) AS estimated_remaining_sec
FROM performance_schema.events_stages_current
WHERE WORK_ESTIMATED > 0;

-- Exemple :
-- 25% complété en 300s
-- → Durée totale estimée : 300 / 0.25 = 1200s (20 min)
-- → Restant : 1200 - 300 = 900s (15 min)
```

### Impact Performance en Temps Réel

```sql
-- CPU et I/O
SELECT 
  EVENT_NAME,
  COUNT_STAR,
  SUM_TIMER_WAIT / 1000000000 AS total_time_sec,
  AVG_TIMER_WAIT / 1000000000 AS avg_time_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE '%io%'
   OR EVENT_NAME LIKE '%cpu%'
ORDER BY total_time_sec DESC
LIMIT 10;

-- Locks en attente
SELECT 
  r.trx_id AS waiting_trx_id,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx_id,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
INNER JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
INNER JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id;
```

### Dashboard Grafana

**Métriques à surveiller** :
```yaml
# Prometheus + mysqld_exporter

# 1. DDL Operations
mysql_info_schema_innodb_metrics_ddl_pending_alter_table
mysql_info_schema_innodb_metrics_ddl_online_create_index

# 2. Table Size Growth
mysql_info_schema_table_size_data_length
mysql_info_schema_table_size_index_length

# 3. Connections
mysql_global_status_threads_running
mysql_global_status_threads_connected

# 4. Queries per Second
rate(mysql_global_status_questions[1m])

# 5. Slow Queries
mysql_global_status_slow_queries
```

---

## Best Practices Production

### 1. Toujours Tester en Staging

```bash
# ❌ NE JAMAIS faire directement en prod
ALTER TABLE orders ADD COLUMN priority INT;

# ✅ Processus recommandé

# 1. Clone prod → staging (données réelles)
mysqldump --single-transaction prod_db orders | mysql staging_db

# 2. Tester migration sur staging
ALTER TABLE orders
  ADD COLUMN priority INT DEFAULT 0,
  ALGORITHM=INPLACE,
  LOCK=NONE;

# 3. Mesurer durée et impact
# - Temps d'exécution
# - Locks générés
# - Impact CPU/I/O
# - Taille disque

# 4. Si OK, planifier en prod avec fenêtre de maintenance
```

### 2. Utiliser Dry-Run

```bash
# pt-osc : Dry-run (simulation)
pt-online-schema-change \
  --alter "ADD COLUMN priority INT" \
  --database=ecommerce \
  --table=orders \
  --dry-run  # ← Pas --execute
  
# Output :
# Operation, tries, wait:
#   copy_rows, 10, 0.25
# Chunk size: 1000
# Estimated rows: 50000000
# Estimated time: 12 hours

# gh-ost : Dry-run
gh-ost \
  --alter "ADD COLUMN priority INT" \
  --database=ecommerce \
  --table=orders \
  --test-on-replica  # ← Test mode
```

### 3. Planification et Communication

```markdown
# Migration Plan : orders.priority column

## Contexte
- Ajouter colonne priority INT DEFAULT 0
- Table orders : 50M lignes, 15 GB
- Trafic peak : 1000 req/s

## Méthode
- ✅ ALGORITHM=INPLACE compatible (testé en staging)
- ✅ LOCK=NONE (zero downtime)
- Durée estimée : 10 secondes
- Fenêtre : 2025-01-20 02:00 UTC (heures creuses)

## Rollback Plan
- Si problème : Aucun rollback nécessaire (colonne optionnelle)
- Si corruption : Restore depuis backup (RTO: 30 min)

## Communication
- 📧 Email équipe dev : 2025-01-15
- 🔔 Slack #production : 1h avant migration
- 📊 Status page : "Maintenance planifiée"

## Validation Post-Migration
- [ ] SELECT COUNT(*) FROM orders → OK
- [ ] Vérifier applications fonctionnent
- [ ] Monitorer erreurs (10 min)
- [ ] Comparer métriques (before/after)
```

### 4. Backup Avant Migration

```bash
# ✅ Toujours backup avant DDL critique

# Option 1 : mysqldump (table spécifique)
mysqldump \
  --single-transaction \
  --routines \
  --triggers \
  ecommerce orders > backup_orders_$(date +%Y%m%d_%H%M%S).sql

# Option 2 : Percona XtraBackup (full backup)
xtrabackup --backup \
  --target-dir=/backup/pre-migration-$(date +%Y%m%d)

# Option 3 : Snapshot disque (cloud)
aws ec2 create-snapshot \
  --volume-id vol-xxxxx \
  --description "Pre-migration backup orders table"
```

### 5. Monitoring Automatisé

```bash
# Script surveillance migration

#!/bin/bash
# monitor-migration.sh

DB="ecommerce"
TABLE="orders"
INTERVAL=30  # secondes

while true; do
  echo "=== $(date) ==="
  
  # Progression
  mysql -e "
    SELECT 
      EVENT_NAME,
      ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 2) AS pct_complete,
      TIMER_WAIT / 1000000000 AS elapsed_sec
    FROM performance_schema.events_stages_current
    WHERE WORK_ESTIMATED > 0
  " 2>/dev/null
  
  # Threads running
  mysql -e "SHOW STATUS LIKE 'Threads_running'" | awk '{print $2}'
  
  # Table size
  mysql -e "
    SELECT 
      ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
      ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB' AND TABLE_NAME LIKE '%$TABLE%'
  "
  
  sleep $INTERVAL
done
```

### 6. Rollback Plan

```sql
-- Scénario 1 : ALTER TABLE échoue
-- → Automatique, rien à faire

-- Scénario 2 : ALTER TABLE réussit mais problème applicatif
-- → Rollback selon type modification

-- 2a. ADD COLUMN → Supprimer colonne
ALTER TABLE orders
  DROP COLUMN priority,
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- 2b. ADD INDEX → Supprimer index
DROP INDEX idx_priority ON orders;

-- 2c. MODIFY COLUMN → Restore from backup
mysql ecommerce < backup_orders_20250120.sql

-- 2d. Table swap → Swap inverse
RENAME TABLE
  orders TO orders_new,
  orders_old TO orders;
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **ALTER TABLE bloquant** : Copie complète, downtime = durée copie
- ✅ **ALGORITHM=INPLACE** : Modifications sur place, lock minimal
- ✅ **ALGORITHM=COPY** : Nouvelle table, copie complète
- ✅ **LOCK modes** : NONE (online), SHARED (lectures ok), EXCLUSIVE (tout bloqué)

### Opérations Online (INPLACE + LOCK=NONE)
- ✅ **ADD COLUMN** (fin de table, avec DEFAULT)
- ✅ **DROP COLUMN** (instantané)
- ✅ **ADD INDEX** (construction background)
- ✅ **DROP INDEX** (instantané)
- ✅ **RENAME COLUMN** (instantané)
- ✅ **ALTER DEFAULT** (instantané)
- ⚠️ **MODIFY COLUMN** (si types compatibles)

### Outils Externes
- ✅ **pt-online-schema-change** : Triggers, mature, contrôle précis
- ✅ **gh-ost** : Binlog, pas de triggers, contrôle interactif
- 💡 **Choix** : pt-osc pour < 100M lignes, gh-ost pour > 100M ou contrôle fin

### Stratégies Zero-Downtime
1. **INPLACE natif** : Simple, rapide, limité aux opérations compatibles
2. **Colonne temporaire** : Migration progressive, rollback facile
3. **Table shadow + swap** : Restructuration complète, validation garantie

### Monitoring
- ✅ **Performance Schema** : Progression (WORK_COMPLETED / WORK_ESTIMATED)
- ✅ **PROCESSLIST** : État actuel (Altering table, Creating index)
- ✅ **Estimation durée** : (temps_écoulé / pct) * (100 - pct)
- ✅ **Impact** : CPU, I/O, locks, threads_running

### Best Practices
- ✅ **Toujours tester en staging** avec données réelles
- ✅ **Dry-run** : pt-osc --dry-run, gh-ost --test-on-replica
- ✅ **Backup avant migration** (mysqldump, xtrabackup, snapshot)
- ✅ **Plan de communication** (équipe, timing, fenêtre maintenance)
- ✅ **Rollback plan** documenté et testé
- ✅ **Monitoring automatisé** pendant migration
- ⚠️ **Éviter heures de pointe** (même si online)

### Limitations
- ⚠️ **INPLACE** : Limité à certaines opérations
- ⚠️ **Outils externes** : Nécessitent double espace disque
- ⚠️ **Très grandes tables** (> 1 milliard) : Migration peut prendre jours
- ⚠️ **Modifications complexes** : Parfois nécessitent COPY

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB
- 📖 [Online DDL Overview](https://mariadb.com/kb/en/innodb-online-ddl-overview/)
- 📖 [ALTER TABLE](https://mariadb.com/kb/en/alter-table/) - Syntaxe complète
- 📖 [ALGORITHM and LOCK](https://mariadb.com/kb/en/alter-table/#algorithm)
- 📖 [InnoDB Online DDL Operations](https://mariadb.com/kb/en/innodb-online-ddl-operations-with-the-inplace-alter-algorithm/)

### Outils Externes
- 🛠️ [pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html) - Documentation Percona
- 🛠️ [gh-ost](https://github.com/github/gh-ost) - GitHub repository
- 📝 [gh-ost Documentation](https://github.com/github/gh-ost/blob/master/doc/README.md)

### Guides et Tutoriels
- 📚 [Online Schema Change Best Practices](https://www.percona.com/blog/online-schema-change-best-practices/)
- 📚 [Zero Downtime Migrations](https://mariadb.com/resources/blog/zero-downtime-schema-changes/)
- 📝 [gh-ost vs pt-osc Comparison](https://github.blog/2016-08-01-gh-ost-github-s-online-migration-tool-for-mysql/)

### Monitoring
- 📊 [Performance Schema DDL](https://mariadb.com/kb/en/performance-schema-events-stages-current-table/)
- 🔍 [Monitoring ALTER TABLE Progress](https://www.percona.com/blog/how-to-monitor-alter-table-progress/)

---


⏭️ [Migration et Compatibilité](/19-migration-compatibilite/README.md)
