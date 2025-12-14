🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D. Configuration de Référence par Cas d'Usage

> **Type** : Annexe de référence pratique  
> **Public cible** : DBA, Administrateurs système, DevOps  
> **Utilisation** : Templates de configuration prêts à l'emploi

---

## 🎯 Objectif de cette annexe

Cette annexe fournit des **configurations de référence optimisées** pour MariaDB 11.8 LTS selon différents cas d'usage. Chaque configuration est accompagnée d'explications détaillées sur les paramètres clés et leurs valeurs recommandées.

Ces templates constituent un **point de départ solide** pour vos déploiements en production, mais doivent être adaptés à votre contexte spécifique (matériel, volumétrie, contraintes métier).

---

## 📋 Cas d'usage couverts

Cette annexe propose des configurations optimisées pour **4 scénarios types** :

| Cas d'usage | Description | Section |
|-------------|-------------|---------|
| **OLTP** | Haute concurrence, faible latence, transactions courtes | [D.1](01-configuration-oltp.md) |
| **OLAP** | Analytique, requêtes complexes, agrégations massives | [D.2](02-configuration-olap.md) |
| **Mixed Workload** | Hybride transactionnel + analytique | [D.3](03-configuration-mixed-workload.md) |
| **Développement Local** | Environnement de dev, ressources limitées | [D.4](04-configuration-developpement-local.md) |

---

## 🔧 Structure des configurations

Chaque section de cette annexe suit la même structure :

### 1. Profil du cas d'usage
- Caractéristiques de la charge de travail
- Métriques clés à optimiser
- Contraintes typiques

### 2. Configuration my.cnf complète
- Fichier de configuration commenté
- Organisation par catégories logiques
- Valeurs optimisées pour le cas d'usage

### 3. Explications des paramètres clés
- Justification des choix de configuration
- Impact sur les performances
- Alternatives et compromis

### 4. Recommandations matérielles
- Spécifications CPU, RAM, disques
- Architecture réseau si pertinent

### 5. Points de vigilance
- Limitations à connaître
- Métriques à monitorer
- Ajustements courants en production

---

## 💡 Principes généraux de configuration

### Méthodologie d'optimisation

1. **Partir d'une baseline solide** : Utiliser les templates de cette annexe
2. **Mesurer avant d'optimiser** : Établir des métriques de référence
3. **Ajuster progressivement** : Un paramètre à la fois, observer l'impact
4. **Monitorer en continu** : Utiliser Performance Schema et métriques système
5. **Documenter les changements** : Tracer qui, quand, pourquoi

### Facteurs à prendre en compte

#### 1. **Ressources matérielles**
```bash
# RAM disponible
free -h

# CPU et cœurs
lscpu | grep -E "^CPU\(s\)|^Model name"

# Disques (type, IOPS, débit)
lsblk -d -o name,rota,size,type
# rota=0 → SSD/NVMe, rota=1 → HDD
```

#### 2. **Charge de travail**
- **OLTP** : Nombreuses transactions courtes, lectures/écritures équilibrées
- **OLAP** : Requêtes longues, agrégations, scans de tables, majoritairement lectures
- **Mixed** : Combinaison des deux, requiert compromis
- **Dev** : Ressources limitées, rapidité de démarrage, facilité debug

#### 3. **Volumétrie et croissance**
- Taille actuelle des données
- Taux de croissance prévu
- Nombre de connexions simultanées
- Patterns d'accès (pics, saisonnalité)

#### 4. **Contraintes de disponibilité**
- SLA (Service Level Agreement)
- Tolérance aux pannes
- Fenêtres de maintenance
- Besoins de réplication/HA

---

## 🎨 Catégories de paramètres

Les configurations de cette annexe organisent les paramètres par catégories fonctionnelles :

### 📊 Mémoire

**Paramètres essentiels :**
- `innodb_buffer_pool_size` : Cache des données InnoDB (le plus critique)
- `innodb_buffer_pool_instances` : Parallélisme du buffer pool
- `innodb_log_buffer_size` : Buffer des logs de transaction
- `key_buffer_size` : Cache pour MyISAM (si utilisé)
- `max_allowed_packet` : Taille max des paquets réseau

**Règles générales :**
- OLTP : 60-70% RAM pour buffer pool
- OLAP : 70-80% RAM pour buffer pool
- Dev : 256M-1G selon ressources

### 💾 Disques et I/O

**Paramètres essentiels :**
- `innodb_io_capacity` : IOPS disponibles (100-200 HDD, 2000-5000 SSD, 10000+ NVMe)
- `innodb_io_capacity_max` : IOPS max en cas de flush intensif
- `innodb_flush_method` : Méthode de flush (O_DIRECT pour production)
- `innodb_flush_log_at_trx_commit` : Durabilité vs performance
- `sync_binlog` : Synchronisation des binary logs

**🆕 MariaDB 11.8 :**
- `innodb_alter_copy_bulk` : Optimisation construction d'index
- Cost optimizer amélioré pour SSD

### 🔄 Logs et durabilité

**Balance sécurité/performance :**

```ini
# Maximum durabilité (ACID strict)
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

# Performance maximale (risque perte 1s de transactions)
innodb_flush_log_at_trx_commit = 2
sync_binlog = 0

# Compromis (production OLTP typique)
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

### 🔗 Connexions et threads

**Paramètres essentiels :**
- `max_connections` : Nombre max de connexions simultanées
- `thread_cache_size` : Cache de threads pour réutilisation
- `thread_pool_size` : Taille du pool de threads (si activé)
- `table_open_cache` : Cache des descripteurs de tables
- `table_definition_cache` : Cache des définitions de tables

### 🔍 Query Cache (⚠️ Déprécié)

```ini
# NE PAS UTILISER en MariaDB 11.8
# query_cache_type = 0
# query_cache_size = 0

# Le query cache est désactivé par défaut et déprécié
# Utiliser plutôt :
# - Cache applicatif (Redis, Memcached)
# - InnoDB Buffer Pool bien dimensionné
# - Vues matérialisées (workarounds)
```

### 🌍 Charset et collation

**🆕 MariaDB 11.8 - utf8mb4 par défaut :**

```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci  # Ou uca1400_ai_ci (UCA 14.0.0)

# Support complet Unicode, emojis, langues asiatiques
# UCA 14.0.0 : Meilleur tri multilingue
```

### ⏰ Timestamps

**🆕 MariaDB 11.8 - Extension TIMESTAMP 2106 :**

```ini
# Résolution du problème Y2038
# TIMESTAMP supporte maintenant jusqu'en 2106
# Pas de configuration spéciale nécessaire
# Automatiquement actif en 11.8
```

---

## 🧮 Calcul des ressources mémoire

### Formule de base

```
RAM Totale Requise ≈ 
    innodb_buffer_pool_size
  + (max_connections × (read_buffer_size + sort_buffer_size + join_buffer_size))
  + thread_cache_size × (256K)
  + key_buffer_size (si MyISAM)
  + 1-2 GB (OS + autres processus)
```

### Exemple OLTP - Serveur 16GB RAM

```ini
innodb_buffer_pool_size = 10G     # 62.5% RAM
max_connections = 200
read_buffer_size = 256K
sort_buffer_size = 512K
join_buffer_size = 256K
thread_cache_size = 100
key_buffer_size = 32M

# Calcul approximatif :
# 10GB (buffer pool)
# + 200 × (256K + 512K + 256K) ≈ 200MB
# + 100 × 256K ≈ 25MB
# + 32MB (key buffer)
# + 1.5GB (OS)
# ───────────────
# ≈ 11.8GB / 16GB → OK avec marge
```

---

## 📈 Monitoring post-déploiement

Après avoir appliqué une configuration, surveiller ces métriques :

### Performance Schema - Métriques clés

```sql
-- Utilisation du buffer pool
SELECT 
    CONCAT(ROUND(100.0 * pages_data / pages_total, 2), '%') AS pct_data,
    CONCAT(ROUND(100.0 * pages_dirty / pages_total, 2), '%') AS pct_dirty,
    CONCAT(ROUND(100.0 * pages_free / pages_total, 2), '%') AS pct_free
FROM (
    SELECT 
        variable_value AS pages_data
    FROM information_schema.global_status 
    WHERE variable_name = 'Innodb_buffer_pool_pages_data'
) AS data,
(
    SELECT 
        variable_value AS pages_dirty
    FROM information_schema.global_status 
    WHERE variable_name = 'Innodb_buffer_pool_pages_dirty'
) AS dirty,
(
    SELECT 
        variable_value AS pages_free
    FROM information_schema.global_status 
    WHERE variable_name = 'Innodb_buffer_pool_pages_free'
) AS free,
(
    SELECT 
        variable_value AS pages_total
    FROM information_schema.global_status 
    WHERE variable_name = 'Innodb_buffer_pool_pages_total'
) AS total;

-- Taux de hit du buffer pool (objectif > 99%)
SELECT 
    ROUND(100.0 * (1 - (
        (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_reads') / 
        (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_read_requests')
    )), 4) AS buffer_pool_hit_rate_pct;

-- Connexions actives vs max
SELECT 
    (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Threads_connected') AS current_connections,
    @@max_connections AS max_connections,
    ROUND(100.0 * (SELECT variable_value FROM information_schema.global_status WHERE variable_name = 'Threads_connected') / @@max_connections, 2) AS pct_used;
```

### Métriques système à surveiller

```bash
# CPU usage
top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1

# Mémoire
free -h

# I/O Wait
iostat -x 1 5 | grep -A 1 "avg-cpu"

# Disk IOPS
iostat -x 1 5 | grep -E "(Device|sd|nvme)"
```

---

## ⚠️ Points de vigilance

### 1. Ne pas copier-coller aveuglément

❌ **Mauvaise pratique :**
```bash
# Copier directement sans adaptation
cp config-oltp.cnf /etc/mysql/my.cnf
systemctl restart mariadb
# DANGER : Peut crasher si RAM insuffisante !
```

✅ **Bonne pratique :**
```bash
# 1. Vérifier les ressources disponibles
free -h
lscpu

# 2. Adapter les valeurs à votre matériel
# 3. Tester sur environnement de dev/staging
# 4. Déployer progressivement en production
# 5. Monitorer pendant 24-48h minimum
```

### 2. Variables dynamiques vs statiques

```sql
-- Variables modifiables à chaud (dynamiques)
SET GLOBAL innodb_buffer_pool_size = 12G;  -- Possible depuis MariaDB 10.2.2

-- Variables nécessitant redémarrage (statiques)
-- innodb_buffer_pool_instances
-- innodb_log_file_size
-- → Modifier dans my.cnf + redémarrage
```

### 3. Ordre de priorité des optimisations

1. **Schema et indexes** (impact max, coût min)
2. **Requêtes SQL** (optimisation queries)
3. **Configuration serveur** (tuning my.cnf)
4. **Matériel** (scaling vertical/horizontal)

> 💡 **Règle d'or** : Une mauvaise requête avec 1000 index sur un serveur surpuissant sera toujours lente. Optimiser le code avant le hardware !

---

## 🔗 Structure de cette annexe

```
D. Configuration de Référence par Cas d'Usage/
│
├── README.md (ce fichier)
│   └── Introduction, principes généraux, méthodologie
│
├── D.1 - Configuration OLTP
│   └── Haute concurrence, faible latence
│       • my.cnf optimisé OLTP
│       • InnoDB tunning transactionnel
│       • Gestion des connexions
│
├── D.2 - Configuration OLAP
│   └── Data warehousing, analytics
│       • my.cnf optimisé OLAP
│       • Buffers pour grandes requêtes
│       • ColumnStore si applicable
│
├── D.3 - Configuration Mixed Workload
│   └── Hybride OLTP + OLAP
│       • Compromis équilibrés
│       • Read/Write splitting
│       • MaxScale integration
│
└── D.4 - Configuration Développement Local
    └── Dev, testing, CI/CD
        • Ressources minimales
        • Rapidité de démarrage
        • Logging détaillé pour debug
```

---

## 📖 Utilisation recommandée

### Pour un nouveau déploiement

1. **Identifier votre cas d'usage** dominant (OLTP, OLAP, Mixed, Dev)
2. **Consulter la section correspondante** (D.1, D.2, D.3 ou D.4)
3. **Copier le template my.cnf** fourni
4. **Adapter les valeurs** selon vos ressources matérielles
5. **Tester en dev/staging** avant production
6. **Déployer** et **monitorer** activement les 48 premières heures

### Pour optimiser une installation existante

1. **Analyser la charge actuelle** :
   ```sql
   -- Top requêtes lentes
   SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;
   
   -- Utilisation mémoire
   SHOW GLOBAL STATUS LIKE 'innodb_buffer_pool%';
   ```

2. **Identifier les goulots d'étranglement** (CPU, RAM, I/O, réseau)

3. **Comparer avec le template** du cas d'usage correspondant

4. **Ajuster progressivement** les paramètres critiques

5. **Mesurer l'impact** de chaque changement

---

## 🆕 Spécificités MariaDB 11.8 LTS

Les configurations de cette annexe intègrent les nouveautés et optimisations de MariaDB 11.8 :

### Sécurité par défaut
- **TLS activé** par défaut (peut être désactivé si besoin)
- **Plugin PARSEC** disponible pour authentification HSM
- **Privilèges granulaires** améliorés

### Performance
- **innodb_alter_copy_bulk** : Construction d'index optimisée
- **Cost optimizer SSD** : Meilleur choix de plans d'exécution
- **Optimistic ALTER TABLE** : Réduction lag réplication

### Unicode et charset
- **utf8mb4** par défaut avec **UCA 14.0.0**
- Meilleur support multilingue et emojis

### Timestamp étendu
- **Extension 2106** : Résolution problème Y2038
- Pas de configuration nécessaire, actif par défaut

### Nouveaux moteurs et fonctionnalités
- **MariaDB Vector** : Recherche vectorielle IA/RAG
- **S3 Engine** : Archivage données froides
- **Application Time Period Tables** : Gestion périodes temporelles

---

## ✅ Points clés à retenir

- 🎯 **Adapter, ne pas copier** : Chaque environnement est unique
- 📊 **Mesurer avant/après** : Metrics-driven optimization
- 🔄 **Itérer progressivement** : Petits changements, grande vigilance
- 📚 **Documenter** : Tracer tous les changements de configuration
- 🔍 **Monitorer en continu** : Les besoins évoluent, la config aussi
- 🆕 **Profiter de 11.8** : Nouvelles optimisations et fonctionnalités

---

## 🔗 Ressources complémentaires

### Documentation officielle MariaDB
- [Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [InnoDB System Variables](https://mariadb.com/kb/en/innodb-system-variables/)
- [Performance Tuning](https://mariadb.com/kb/en/optimization-and-tuning/)

### Outils d'analyse
- **MySQLTuner** : Script d'analyse de configuration
- **Percona Toolkit** : pt-mysql-summary, pt-variable-advisor
- **PMM (Percona Monitoring and Management)** : Monitoring complet

### Autres annexes utiles
- [Annexe E - Checklist de Performance](/annexes/checklist-performance/README.md)
- [Annexe C - Requêtes SQL de Référence](/annexes/requetes-sql-reference/README.md)
- [Annexe B - Commandes CLI Essentielles](/annexes/commandes-cli/README.md)

---

## ➡️ Sections suivantes

Consultez les configurations détaillées selon votre cas d'usage :

- **[D.1 - Configuration OLTP](01-configuration-oltp.md)** → Haute concurrence, transactions rapides
- **[D.2 - Configuration OLAP](02-configuration-olap.md)** → Analytics, requêtes complexes  
- **[D.3 - Configuration Mixed Workload](03-configuration-mixed-workload.md)** → Hybride transactionnel + analytique
- **[D.4 - Configuration Développement Local](04-configuration-developpement-local.md)** → Dev, testing, CI/CD

---

**MariaDB** : Version 11.8 LTS

⏭️ [OLTP (High concurrency, low latency)](/annexes/configuration-reference/01-configuration-oltp.md)
