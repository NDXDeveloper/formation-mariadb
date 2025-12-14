🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.3 Query Cache : Pourquoi il est déprécié

> **Niveau** : Expert  
> **Durée estimée** : 1-2 heures  
> **Prérequis** : 
> - Compréhension de l'architecture MariaDB
> - Connaissances en caching et invalidation
> - Expérience avec les patterns de workload OLTP

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre le fonctionnement** du query cache et pourquoi il a été introduit
- **Identifier les problèmes** fondamentaux du query cache en production moderne
- **Mesurer l'impact négatif** du query cache sur vos performances
- **Désactiver correctement** le query cache sur systèmes existants
- **Implémenter des alternatives** modernes et efficaces
- **Migrer** d'une architecture basée query cache vers des solutions appropriées
- **Éviter les pièges** des anciennes recommandations encore présentes en ligne

---

## Introduction

Le **Query Cache** a été l'une des fonctionnalités les plus mal comprises et problématiques de MySQL/MariaDB. Malgré son apparence séduisante, il a causé plus de problèmes de performance qu'il n'en a résolu dans la grande majorité des cas.

### Historique rapide

```
┌────────────────────────────────────────────────────────┐
│           CHRONOLOGIE DU QUERY CACHE                   │
├────────────────────────────────────────────────────────┤
│                                                        │
│  2000   MySQL 4.0 : Introduction du Query Cache        │
│         "Cache magique pour accélérer les requêtes"    │
│                                                        │
│  2000s  Adoption massive                               │
│         Recommandé dans tous les guides                │
│                                                        │
│  2010s  Problèmes de scalabilité découverts            │
│         Contention sur serveurs multi-cœurs            │
│                                                        │
│  2017   MySQL 5.7.20 : Déprécié officiellement         │
│         Oracle reconnaît les problèmes fondamentaux    │
│                                                        │
│  2018   MySQL 8.0 : SUPPRIMÉ complètement              │
│         Plus de query cache dans MySQL 8+              │
│                                                        │
│  2024   MariaDB 11.x : Toujours présent mais...        │
│         Documentation officielle déconseille son usage │
│                                                        │
└────────────────────────────────────────────────────────┘
```

⚠️ **État actuel** :
- **MySQL 8.0+** : Supprimé, n'existe plus
- **MariaDB 11.8** : Présent mais **déconseillé** par la documentation officielle
- **Recommandation universelle** : `query_cache_type = OFF`

---

## Comment le Query Cache était censé fonctionner

### Le concept séduisant

L'idée originale était simple et attractive :

```
┌────────────────────────────────────────────────┐
│  THÉORIE : Le "cache magique"                  │
├────────────────────────────────────────────────┤
│                                                │
│  1. Requête SQL arrive                         │
│     SELECT * FROM users WHERE id = 123         │
│                                                │
│  2. Vérifier si exactement la même requête     │
│     a déjà été exécutée                        │
│                                                │
│  3. Si OUI : Retourner résultat du cache       │
│     → Pas d'exécution, ultra rapide !          │
│                                                │
│  4. Si NON : Exécuter, stocker résultat        │
│     → Cache pour la prochaine fois             │
│                                                │
│  5. Si données modifiées (INSERT/UPDATE/DELETE)│
│     → Invalider le cache de la table           │
│                                                │
└────────────────────────────────────────────────┘
```

### Exemple de "succès" théorique

```sql
-- Configuration query cache (ANCIEN, ne pas utiliser !)
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_size = 128M;

-- Requête 1 : Exécutée normalement (50ms)
SELECT * FROM products WHERE category = 'electronics';

-- Requête 2 : IDENTIQUE → Servie depuis cache (0.1ms)
SELECT * FROM products WHERE category = 'electronics';

-- "Wow, 500x plus rapide !"
```

**Le piège** : Cette amélioration ne fonctionne que dans des conditions très spécifiques et irréalistes.

---

## Les problèmes fondamentaux

### 1. Invalidation en cascade catastrophique

Le query cache invalide **toute la table** à la moindre modification.

```sql
-- Scénario catastrophique (très fréquent en production)

-- Cache contient 1000 résultats différents de la table orders
-- Taille du cache utilisée : 50 MB pour cette table

-- Un seul UPDATE arrive :
UPDATE orders SET status = 'shipped' WHERE id = 12345;

-- BOUM ! Toutes les 1000 requêtes en cache sont invalidées
-- 50 MB de cache perdus pour une modification d'une seule ligne
```

**Impact en production** :

```
┌──────────────────────────────────────────────────────┐
│  TABLE : orders (table très active)                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  READ  : 1000 requêtes/sec                           │
│  WRITE : 50 updates/sec                              │
│                                                      │
│  Avec Query Cache :                                  │
│  • Chaque UPDATE invalide TOUT le cache              │
│  • 50 invalidations/sec                              │
│  • Cache inutilisable, constamment vidé              │
│  • Overhead pur sans bénéfice                        │
│                                                      │
│  Résultat : Performance PIRE qu'avec cache OFF       │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 2. Contention sur le mutex global

Le query cache utilise un **mutex global unique** pour toutes les opérations.

```
┌──────────────────────────────────────────────────┐
│  ARCHITECTURE DU QUERY CACHE                     │
├──────────────────────────────────────────────────┤
│                                                  │
│  Thread 1 ──┐                                    │
│  Thread 2 ──┤                                    │
│  Thread 3 ──┤──→ [MUTEX GLOBAL] ──→ [Cache]      │
│  Thread 4 ──┤                                    │
│  ...        │                                    │
│  Thread 64 ─┘                                    │
│                                                  │
│  ↑ Tous les threads en ATTENTE                   │
│    Un seul thread peut accéder au cache          │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Sur serveurs modernes (32+ cores)** :

```sql
-- Monitoring de la contention
SHOW STATUS LIKE 'Qcache%';

/*
Variable_name                    Value
-------------------------------------------
Qcache_hits                      1000000
Qcache_inserts                   500000
Qcache_not_cached                200000
Qcache_lowmem_prunes             150000  ← Évictions forcées
Qcache_queries_in_cache          50000
Qcache_free_memory               8388608
Qcache_total_blocks              100234
*/

-- Problème : Sur 32 cores, 31 threads ATTENDENT pendant qu'1 thread accède au cache
-- Résultat : Scalabilité inverse (plus de cores = pire performance)
```

**Benchmark réel** (serveur 32 cores) :

| Configuration | Throughput (queries/sec) |
|--------------|--------------------------|
| Query Cache OFF | 45,000 |
| Query Cache ON (128 MB) | 12,000 |
| **Performance** | **-73%** ⚠️ |

### 3. Matching exact uniquement

Le query cache est **extrêmement strict** sur le matching :

```sql
-- Ces requêtes sont TOUTES considérées DIFFÉRENTES :

SELECT * FROM users WHERE id = 123;
SELECT * FROM users WHERE id=123;        -- Espace différent
SELECT * FROM users where id = 123;      -- Casse différente
SELECT * FROM users WHERE id = 123 ;     -- Espace en fin
select * from users where id = 123;      -- Tout en minuscules

-- Chacune créé une entrée SÉPARÉE dans le cache !
```

**Avec des requêtes paramétrées** (pattern moderne) :

```sql
-- Application génère :
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 3;
-- ... 
SELECT * FROM users WHERE id = 10000;

-- Résultat : 10,000 entrées DIFFÉRENTES dans le cache
-- Aucune réutilisation, cache poll polluted
```

### 4. Overhead pour chaque requête

Même quand le cache est vide ou inefficace, **chaque requête paie le coût** de vérification.

```
Cycle de vie d'une requête avec Query Cache ON :

1. Hash de la requête SQL         (coût CPU)
2. Lock du mutex global            (contention)
3. Lookup dans la hash table       (coût CPU + mémoire)
4. Miss → Unlock mutex             
5. Exécuter la requête             (travail normal)
6. Lock mutex pour stocker         (contention)
7. Stocker dans cache              (coût mémoire)
8. Unlock mutex                    

VS Query Cache OFF :

1. Exécuter la requête             (travail normal)

→ 8 étapes vs 1 étape, même quand cache inutilisable
```

### 5. Fragmentation mémoire

Le query cache souffre de **fragmentation importante** :

```sql
-- Vérifier la fragmentation
SHOW STATUS LIKE 'Qcache_free_blocks';
/*
Variable_name          Value
---------------------------------
Qcache_free_blocks     15234    ← Beaucoup de petits blocs
*/

-- Si Qcache_free_blocks > 1000 : Fragmentation sévère
-- Même avec de la mémoire "libre", impossible d'allouer de nouvelles entrées
```

**Défragmentation manuelle** (impact performance) :

```sql
FLUSH QUERY CACHE;  -- Bloque TOUTES les requêtes pendant la défragmentation
-- Sur cache de 1 GB, peut prendre 5-10 secondes
-- = 5-10 secondes de blocage complet de la base
```

---

## Quand le Query Cache PEUT fonctionner

Il existe des cas **très spécifiques** où le query cache peut apporter un bénéfice :

### Scénarios théoriques favorables

```
✓ Tables 100% READ-ONLY (jamais d'UPDATE/INSERT/DELETE)
✓ Requêtes IDENTIQUES répétées en boucle
✓ Dataset petit (< 100 MB total)
✓ Serveur mono-core ou dual-core (pas de contention)
✓ Application qui ne peut pas implémenter son propre cache

Exemple : Site vitrine statique avec catalogue produits figé
```

**En pratique** :
- Ces conditions sont **extrêmement rares** en production moderne
- Même dans ces cas, un cache applicatif est plus efficace
- Le jeu n'en vaut presque jamais la chandelle

---

## Mesurer l'impact du Query Cache

### Métriques à surveiller

```sql
-- Dashboard complet Query Cache
SELECT 
    -- Configuration
    @@query_cache_type as cache_type,
    @@query_cache_size / 1024 / 1024 as cache_size_mb,
    
    -- Utilisation
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_queries_in_cache') as queries_cached,
    
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_free_memory') / 1024 / 1024 as free_mb,
    
    -- Efficacité
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_hits') as hits,
    
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_inserts') as inserts,
    
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_not_cached') as not_cached,
    
    -- Problèmes
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_lowmem_prunes') as evictions,
    
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_free_blocks') as free_blocks,
    
    -- Hit rate
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Qcache_hits') /
        NULLIF(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Qcache_hits') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Qcache_inserts') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Qcache_not_cached'),
            0
        ) * 100, 2
    ) as hit_rate_pct;
```

### Interprétation des métriques

```
┌─────────────────────────────────────────────────────┐
│  SIGNES QUE LE QUERY CACHE EST CONTRE-PRODUCTIF     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ⚠️ Qcache_lowmem_prunes élevé                      │
│     → Cache trop petit, évictions constantes        │
│                                                     │
│  ⚠️ Qcache_free_blocks > 1000                       │
│     → Fragmentation sévère                          │
│                                                     │
│  ⚠️ Hit rate < 30%                                  │
│     → Cache inefficace, plus de misses que de hits  │
│                                                     │
│  ⚠️ Qcache_inserts ≈ Qcache_lowmem_prunes           │
│     → Turnover complet, cache inutile               │
│                                                     │
│  ⚠️ Threads_running élevé avec QC ON                │
│     → Contention mutex probable                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Benchmark avant/après désactivation

```sql
-- 1. Établir baseline avec QC ON
CALL collect_baseline();  -- Procédure de la section 15.1

-- 2. Désactiver Query Cache
SET GLOBAL query_cache_type = OFF;
SET GLOBAL query_cache_size = 0;

-- 3. Attendre 30 minutes (warm-up)

-- 4. Nouvelle baseline
CALL collect_baseline();

-- 5. Comparer
SELECT 
    b1.metric_name,
    b1.metric_value as qc_on,
    b2.metric_value as qc_off,
    ROUND(
        (CAST(b2.metric_value AS DECIMAL(20,2)) - CAST(b1.metric_value AS DECIMAL(20,2))) /
        NULLIF(CAST(b1.metric_value AS DECIMAL(20,2)), 0) * 100,
        2
    ) as change_pct
FROM performance_baselines b1
JOIN performance_baselines b2 ON b1.metric_name = b2.metric_name
WHERE b1.collected_at = (SELECT MAX(collected_at) FROM performance_baselines WHERE collected_at < NOW() - INTERVAL 1 HOUR)
AND b2.collected_at = (SELECT MAX(collected_at) FROM performance_baselines)
AND b1.metric_name IN ('queries_per_second', 'Threads_running', 'Queries')
ORDER BY ABS(change_pct) DESC;
```

**Résultats typiques** (production OLTP) :

```
metric_name            qc_on    qc_off   change_pct
--------------------------------------------------------
queries_per_second     3200     8500     +165%     ✓
Threads_running        24       12       -50%      ✓
avg_query_time_ms      15       6        -60%      ✓
```

---

## Désactivation du Query Cache

### Méthode recommandée

```ini
# /etc/mysql/my.cnf ou /etc/my.cnf.d/server.cnf

[mariadb]
# Désactivation complète du Query Cache
query_cache_type = OFF
query_cache_size = 0

# OU (équivalent)
query_cache_type = 0
query_cache_size = 0
```

**Variables expliquées** :

```sql
-- query_cache_type : Contrôle le comportement
SELECT @@query_cache_type;
/*
Valeurs possibles :
  0 ou OFF    : Désactivé (recommandé)
  1 ou ON     : Activé pour toutes requêtes SELECT
  2 ou DEMAND : Activé uniquement si SQL_CACHE explicite
*/

-- query_cache_size : Taille allouée
SELECT @@query_cache_size;
/*
  0     : Aucune mémoire (recommandé)
  > 0   : Taille en bytes (déconseillé)
*/
```

### Désactivation dynamique (sans restart)

```sql
-- En session pour test
SET SESSION query_cache_type = OFF;

-- Globalement (toutes nouvelles connexions)
SET GLOBAL query_cache_type = OFF;
SET GLOBAL query_cache_size = 0;

-- Vérifier
SHOW VARIABLES LIKE 'query_cache%';
/*
query_cache_type            OFF
query_cache_size            0
query_cache_limit           1048576
query_cache_min_res_unit    4096
query_cache_wlock_invalidate OFF
*/

-- Vider le cache existant
FLUSH QUERY CACHE;
-- ou
RESET QUERY CACHE;
```

⚠️ **Important** : Les changements dynamiques s'appliquent aux **nouvelles connexions**. Les connexions existantes gardent leur configuration de session.

```sql
-- Forcer l'application immédiate (nécessite redémarrage des connexions)
-- Méthode 1 : Attendre que les connexions se terminent naturellement
-- Méthode 2 : Restart MariaDB (fenêtre de maintenance)
sudo systemctl restart mariadb
```

### Vérification post-désactivation

```sql
-- Confirmer que le cache n'est plus utilisé
SHOW STATUS LIKE 'Qcache%';
/*
Si query_cache_size = 0, toutes les métriques Qcache_* sont figées
*/

-- Surveiller les performances
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Queries') / 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Uptime') as queries_per_sec,
    
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_running') as threads_running;

-- Objectif : queries_per_sec ↑, threads_running ↓
```

---

## Alternatives modernes au Query Cache

### 1. Cache applicatif (recommandé)

**Le cache doit être dans l'application**, pas dans la base de données.

#### Redis / Memcached

```python
# Exemple Python avec Redis
import redis
import mysql.connector
import json

redis_client = redis.Redis(host='localhost', port=6379, db=0)
db = mysql.connector.connect(host='localhost', user='app', password='***', database='myapp')

def get_user(user_id):
    # 1. Vérifier Redis
    cache_key = f"user:{user_id}"
    cached = redis_client.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss → Query DB
    cursor = db.cursor(dictionary=True)
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    user = cursor.fetchone()
    
    # 3. Store in cache (TTL 5 minutes)
    if user:
        redis_client.setex(cache_key, 300, json.dumps(user))
    
    return user
```

**Avantages** :
- ✅ Invalidation granulaire (par clé, pas toute la table)
- ✅ Pas de contention sur la base de données
- ✅ Scalabilité horizontale (cluster Redis)
- ✅ TTL flexible par type de donnée
- ✅ Patterns avancés (cache-aside, write-through, etc.)

#### Cache in-process (simple)

```java
// Java avec Caffeine
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

Cache<Integer, User> userCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .build();

public User getUser(int userId) {
    return userCache.get(userId, id -> {
        // Cache miss → load from database
        return userRepository.findById(id);
    });
}
```

### 2. ProxySQL Query Cache

ProxySQL offre un **query cache intelligent** sans les problèmes du query cache natif.

```sql
-- Configuration ProxySQL
INSERT INTO mysql_query_rules (
    rule_id,
    active,
    match_pattern,
    cache_ttl,
    apply
) VALUES (
    1,
    1,
    '^SELECT.*FROM products WHERE category_id',  -- Regex pattern
    5000,  -- TTL 5 secondes
    1
);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

**Avantages ProxySQL** :
- ✅ Cache par pattern de requête (regex)
- ✅ TTL configurables par requête
- ✅ Invalidation intelligente
- ✅ Pas de contention sur MariaDB
- ✅ Metrics détaillées

### 3. Varnish / CDN (pour API web)

Pour les applications web publiques :

```nginx
# Varnish VCL
vcl 4.0;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}

sub vcl_recv {
    # Cache GET requests vers /api/products
    if (req.method == "GET" && req.url ~ "^/api/products") {
        return (hash);
    }
}

sub vcl_backend_response {
    # TTL 60 secondes
    if (bereq.url ~ "^/api/products") {
        set beresp.ttl = 60s;
    }
}
```

### 4. Materialized Views (pour analytics)

Pour les requêtes analytiques complexes :

```sql
-- Table de résultats précalculés
CREATE TABLE daily_sales_summary (
    sale_date DATE PRIMARY KEY,
    total_amount DECIMAL(15,2),
    order_count INT,
    unique_customers INT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Rafraîchissement périodique (event)
CREATE EVENT refresh_daily_sales
ON SCHEDULE EVERY 5 MINUTE
DO
    INSERT INTO daily_sales_summary (sale_date, total_amount, order_count, unique_customers)
    SELECT 
        DATE(order_date),
        SUM(amount),
        COUNT(*),
        COUNT(DISTINCT customer_id)
    FROM orders
    WHERE order_date >= CURDATE() - INTERVAL 1 DAY
    GROUP BY DATE(order_date)
    ON DUPLICATE KEY UPDATE
        total_amount = VALUES(total_amount),
        order_count = VALUES(order_count),
        unique_customers = VALUES(unique_customers),
        updated_at = CURRENT_TIMESTAMP;
```

### 5. Read Replicas (pour load distribution)

Distribuer les lectures sur des replicas :

```
┌────────────────────────────────────────┐
│         APPLICATION                    │
└────────────────┬───────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
   [PRIMARY]         [REPLICA 1]
   (Writes)          [REPLICA 2]  ← Reads
                     [REPLICA 3]
```

**ProxySQL pour routing** :

```sql
-- Writes vers primary
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (10, 'primary.db', 3306);

-- Reads vers replicas
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (20, 'replica1.db', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (20, 'replica2.db', 3306);

-- Routing rules
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES 
    (1, 1, '^SELECT.*FOR UPDATE', 10, 1),  -- SELECT FOR UPDATE → primary
    (2, 1, '^SELECT', 20, 1);               -- SELECT → replicas
```

---

## Matrice de décision : Quelle alternative choisir ?

```
┌──────────────────────────────────────────────────────────────┐
│  CAS D'USAGE                    SOLUTION RECOMMANDÉE         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Application web (sessions,    → Redis/Memcached             │
│  données utilisateur)          → TTL court (1-5 min)         │
│                                                              │
│  API REST publique             → Varnish / CDN               │
│  (données read-mostly)         → TTL moyen (5-60 min)        │
│                                                              │
│  Dashboard analytics           → Materialized views          │
│  (rapports complexes)          → Rafraîchissement périodique │
│                                                              │
│  OLTP haute volumétrie         → Read replicas + ProxySQL    │
│  (e-commerce, SaaS)            → Load balancing reads        │
│                                                              │
│  Microservices                 → Cache in-process            │
│  (données locales)             → Caffeine, Guava (Java)      │
│                                → node-cache (Node.js)        │
│                                                              │
│  Requêtes dynamiques           → ProxySQL query cache        │
│  (patterns répétitifs)         → Cache par pattern regex     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Migration : Plan de transition

### Phase 1 : Audit (1-2 jours)

```sql
-- 1. Vérifier l'utilisation actuelle
SHOW VARIABLES LIKE 'query_cache%';
SHOW STATUS LIKE 'Qcache%';

-- 2. Analyser l'efficacité
SELECT 
    -- Hit rate actuel
    ROUND(
        (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
         WHERE VARIABLE_NAME = 'Qcache_hits') /
        NULLIF(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Qcache_hits') +
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Qcache_inserts'),
            0
        ) * 100, 2
    ) as current_hit_rate_pct,
    
    -- Taille utilisée
    (@@query_cache_size - 
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
      WHERE VARIABLE_NAME = 'Qcache_free_memory')) / 1024 / 1024 as used_mb;

-- 3. Identifier les requêtes les plus fréquentes
-- (Ces requêtes devront être cachées autrement)
SELECT 
    DIGEST_TEXT,
    COUNT_STAR as execution_count,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 3) as avg_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE 'SELECT%'
ORDER BY COUNT_STAR DESC
LIMIT 20;
```

### Phase 2 : Implémentation du nouveau cache (1-2 semaines)

```
1. Choisir la solution (Redis recommandé pour la plupart des cas)

2. Déployer infrastructure cache
   - Redis cluster (3+ nodes pour HA)
   - Monitoring (Redis Sentinel)

3. Instrumenter l'application
   - Wrapper de cache autour des requêtes fréquentes
   - Configuration TTL par type de donnée
   - Métriques (hit rate, latency)

4. Tests en staging
   - Load testing
   - Vérification invalidation
   - Monitoring des performances
```

### Phase 3 : Bascule (1 jour)

```sql
-- Environnement staging
-- 1. Désactiver query cache
SET GLOBAL query_cache_type = OFF;
SET GLOBAL query_cache_size = 0;

-- 2. Monitoring intensif pendant 24-48h
-- Vérifier :
--   - Throughput (doit être égal ou supérieur)
--   - Latency (doit être égale ou inférieure)
--   - CPU usage (peut augmenter légèrement)
--   - Cache hit rate applicatif

-- 3. Si tout OK, appliquer en production
```

### Phase 4 : Nettoyage (après 1 semaine)

```ini
# Supprimer définitivement de la config
[mariadb]
# query_cache_type = OFF  ← Supprimer la ligne
# query_cache_size = 0    ← Supprimer la ligne

# La valeur par défaut dans MariaDB 11.8 est déjà OFF
```

---

## Études de cas réels

### Cas 1 : E-commerce (500k requêtes/min)

**Avant** :
- Query Cache ON (512 MB)
- Hit rate : 15%
- Qcache_lowmem_prunes : 50,000/min
- Throughput : 8,000 queries/sec

**Après** (Redis + désactivation QC) :
- Query Cache OFF
- Redis hit rate : 85%
- Throughput : 23,000 queries/sec
- **Amélioration : +187%** ✅

### Cas 2 : SaaS B2B (Dashboard analytics)

**Avant** :
- Query Cache ON (1 GB)
- Requêtes complexes (20-30s)
- Hit rate : 5% (données constamment mises à jour)

**Après** (Materialized views) :
- Query Cache OFF
- Vues rafraîchies toutes les 5 minutes
- Requêtes < 1 seconde
- **Amélioration : 95% réduction latence** ✅

### Cas 3 : API publique (100k users/jour)

**Avant** :
- Query Cache ON (256 MB)
- Contention mutex sévère (32 cores)
- Threads_running moyen : 45

**Après** (Varnish + ProxySQL) :
- Query Cache OFF
- Varnish devant l'API (cache HTTP)
- ProxySQL pour query routing
- Threads_running moyen : 8
- **Amélioration : 82% réduction load DB** ✅

---

## Mythes et idées reçues

### ❌ Mythe 1 : "Plus de query cache = meilleures performances"

**Réalité** : Plus de query cache = plus de contention, plus de fragmentation, **pires** performances.

### ❌ Mythe 2 : "Mon hit rate est à 50%, c'est bien"

**Réalité** : Un hit rate de 50% signifie que 50% des requêtes paient le coût du cache pour rien. Désactivez.

### ❌ Mythe 3 : "Je dois garder QC pour les tables read-only"

**Réalité** : Un cache applicatif sera toujours plus performant, même pour read-only.

### ❌ Mythe 4 : "C'est un paramètre à tuner finement"

**Réalité** : C'est un paramètre à **DÉSACTIVER** complètement. Il n'y a pas de "bon tuning" possible.

### ❌ Mythe 5 : "C'est compliqué de cacher dans l'app"

**Réalité** : Redis + 10 lignes de code >> Query cache. Plus simple ET plus performant.

---

## ✅ Points clés à retenir

- 🚫 **Query Cache = legacy dangereux** : À désactiver systématiquement en production moderne
- 📉 **Problèmes fondamentaux** : Invalidation cascade, mutex global, scalabilité inverse
- ⚠️ **MySQL 8.0 l'a supprimé** : Oracle a reconnu que c'était une mauvaise idée
- 📊 **MariaDB 11.8** : Toujours présent mais déconseillé officiellement
- 🎯 **Configuration recommandée** : `query_cache_type = OFF`, `query_cache_size = 0`
- ✅ **Alternatives supérieures** : Redis, Memcached, ProxySQL, cache applicatif
- 🔧 **Migration simple** : Désactivation sans risque dans 99% des cas
- 📈 **Gain typique** : +50% à +200% de throughput après désactivation
- 💡 **Principe** : Le cache appartient à l'application, pas à la base de données
- 🎓 **Leçon** : Se méfier des "best practices" anciennes trouvées en ligne

---

## 🔗 Ressources et références

### Documentation officielle

- [📖 MariaDB Query Cache (deprecated)](https://mariadb.com/kb/en/query-cache/)
- [📖 MySQL 8.0 Query Cache Removal](https://dev.mysql.com/doc/refman/8.0/en/query-cache.html)
- [📖 MariaDB Performance Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)

### Articles de référence

- [Why MySQL's Query Cache Is Bad for Performance](https://www.percona.com/blog/2015/06/18/mysql-query-cache-worst-enemy/)
- [MySQL Query Cache: The Good, The Bad, and The Ugly](https://scalegrid.io/blog/mysql-query-cache/)
- [How to Disable Query Cache in MySQL/MariaDB](https://dba.stackexchange.com/questions/150213/)

### Alternatives

- [Redis Documentation](https://redis.io/documentation)
- [ProxySQL Documentation](https://proxysql.com/documentation/)
- [Varnish Cache](https://varnish-cache.org/docs/)

---

## ➡️ Section suivante

**[15.4 Configuration I/O et disques](/15-performance-tuning/04-configuration-io-disques.md)** : Maintenant que nous avons éliminé le query cache, optimisons les performances I/O avec les paramètres modernes pour SSD/NVMe, incluant les nouveautés MariaDB 11.8.

---

*Le Query Cache est un exemple parfait de fonctionnalité qui semblait bonne sur le papier mais s'est révélée nuisible en pratique. La désactiver est l'une des optimisations les plus simples et les plus impactantes que vous puissiez faire.*

⏭️ [Configuration I/O et disques](/15-performance-tuning/04-configuration-io-disques.md)
