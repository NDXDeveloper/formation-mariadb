🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 Maintenance des tables

> **Niveau** : Avancé  
> **Durée estimée** : 2-3 heures  
> **Prérequis** :
> - Sections 11.1-11.5 (Configuration, logs)
> - Connaissance des moteurs de stockage (Chapitre 7)
> - Compréhension de l'indexation (Chapitre 5)
> - Expérience en administration système

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le rôle et l'importance de la maintenance régulière des tables
- **Utiliser** les quatre commandes de maintenance (OPTIMIZE, ANALYZE, CHECK, REPAIR)
- **Planifier** une stratégie de maintenance adaptée à votre charge de travail
- **Identifier** quand et comment effectuer chaque type de maintenance
- **Mesurer** l'impact de la maintenance sur les performances
- **Automatiser** les tâches de maintenance récurrentes
- **Gérer** la maintenance en production sans interruption de service

---

## Introduction

La **maintenance des tables** est une composante essentielle de l'administration MariaDB souvent négligée, mais qui a un **impact direct** sur :

- ⚡ **Performances** : Fragmentation → requêtes lentes
- 📊 **Optimiseur de requêtes** : Statistiques obsolètes → mauvais plans d'exécution
- 💾 **Espace disque** : Fragmentation → gaspillage de stockage
- 🔍 **Intégrité** : Corruption silencieuse → perte de données
- 🎯 **Stabilité** : Problèmes accumulés → incidents majeurs

### Pourquoi la maintenance est critique

```
Sans maintenance régulière:
    ❌ Fragmentation croissante (DELETE, UPDATE)
    ❌ Statistiques obsolètes (optimiseur inefficace)
    ❌ Index dégradés (performance réduite)
    ❌ Corruption non détectée (perte de données)
    ❌ Espace disque gaspillé (coût infrastructure)

Avec maintenance régulière:
    ✅ Tables compactes et optimisées
    ✅ Statistiques à jour (requêtes optimales)
    ✅ Index efficaces
    ✅ Détection précoce des problèmes
    ✅ Utilisation optimale des ressources
```

💡 **Principe fondamental** : La maintenance n'est pas une option, c'est une **nécessité opérationnelle** pour tout système en production.

---

## Les quatre piliers de la maintenance

MariaDB fournit quatre commandes principales pour maintenir la santé des tables :

### Vue d'ensemble comparative

| Commande | Rôle principal | Fréquence | Impact performance | Verrouillage |
|----------|----------------|-----------|-------------------|--------------|
| **OPTIMIZE** | Défragmentation, récupération espace | Mensuelle | ⚠️ Élevé | ⚠️ Table lock (InnoDB: rebuild) |
| **ANALYZE** | Mise à jour statistiques optimiseur | Hebdomadaire | ✅ Faible | ✅ Read lock court |
| **CHECK** | Vérification intégrité | Quotidienne | ✅ Faible-Moyen | ✅ Read lock |
| **REPAIR** | Réparation corruption | Ad-hoc (urgence) | ⚠️ Très élevé | ❌ Table lock exclusif |

### OPTIMIZE TABLE

**Objectif** : Défragmenter et récupérer l'espace perdu.

```sql
OPTIMIZE TABLE orders;
```

**Quand l'utiliser** :
- ✅ Après de nombreux DELETE ou UPDATE
- ✅ Tables avec fragmentation > 20%
- ✅ Récupération d'espace disque
- ✅ Amélioration performance scans de table

**Impact** :
- 🔒 **InnoDB** : Reconstruction complète de la table (ALTER TABLE)
- 🔒 **MyISAM** : Réorganisation des fichiers .MYD et .MYI
- ⏱️ **Durée** : Proportionnelle à la taille de la table

💡 **Attention InnoDB** : `OPTIMIZE TABLE` sur InnoDB = `ALTER TABLE ... ENGINE=InnoDB` (très lourd).

### ANALYZE TABLE

**Objectif** : Mettre à jour les **statistiques de distribution** des données pour l'optimiseur.

```sql
ANALYZE TABLE products;
```

**Quand l'utiliser** :
- ✅ Après importations massives
- ✅ Après modifications importantes (> 10% des lignes)
- ✅ Quand les requêtes utilisent de mauvais index
- ✅ Régulièrement (hebdomadaire ou quotidien)

**Impact** :
- ✅ **Très faible** : Lecture des index et calcul statistiques
- ⏱️ **Rapide** : Quelques secondes même pour grandes tables
- 🔓 **Minimal** : Read lock très court

**Exemples de statistiques** :
- Cardinalité des index
- Distribution des valeurs (histogrammes)
- Nombre de lignes
- Longueur moyenne des lignes

### CHECK TABLE

**Objectif** : Vérifier l'**intégrité physique et logique** des tables.

```sql
CHECK TABLE users;
```

**Quand l'utiliser** :
- ✅ Quotidiennement (automatisé)
- ✅ Après un crash serveur
- ✅ Avant/après migrations importantes
- ✅ En cas de comportements suspects

**Niveaux de vérification** :
- `QUICK` : Vérification rapide (index uniquement)
- `FAST` : Tables non fermées proprement
- `CHANGED` : Tables modifiées depuis dernier check
- `MEDIUM` : Vérification standard (défaut)
- `EXTENDED` : Vérification exhaustive (très long)

### REPAIR TABLE

**Objectif** : **Réparer** une table corrompue.

```sql
REPAIR TABLE corrupted_table;
```

**Quand l'utiliser** :
- ⚠️ **UNIQUEMENT** en cas de corruption détectée
- ⚠️ Après crash système/disque
- ⚠️ Quand CHECK TABLE rapporte des erreurs

**Limitations** :
- ❌ **InnoDB** : Non supporté (utiliser `innodb_force_recovery`)
- ✅ **MyISAM/Aria** : Supporté
- 🔒 **Verrouillage exclusif** : Table inaccessible

⚠️ **DANGER** : REPAIR ne garantit **pas** la récupération complète. Toujours restaurer depuis backup si possible.

---

## Détection des besoins de maintenance

### 1. Fragmentation des tables

La **fragmentation** se produit quand les données ne sont plus stockées de manière contiguë.

#### Causes de fragmentation

```sql
-- DELETE crée des "trous"
DELETE FROM logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 30 DAY);
-- Résultat: Espace vide dans les pages de données

-- UPDATE augmentant la taille de ligne
UPDATE articles SET content = CONCAT(content, ' [UPDATED]');
-- Résultat: Ligne déplacée vers nouvelle page, ancien espace perdu
```

#### Mesure de la fragmentation

```sql
-- Requête de détection de fragmentation
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ENGINE,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb,
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND DATA_FREE > 0
ORDER BY fragmentation_pct DESC;
```

**Sortie exemple** :

```
+---------------+-------------+--------+---------+----------+---------+-------------------+
| TABLE_SCHEMA  | TABLE_NAME  | ENGINE | data_mb | index_mb | free_mb | fragmentation_pct |
+---------------+-------------+--------+---------+----------+---------+-------------------+
| ecommerce     | order_items | InnoDB | 1250.00 | 450.00   | 350.00  | 17.07             |
| ecommerce     | logs        | InnoDB | 5600.00 | 120.00   | 1200.00 | 17.22             |
| analytics     | events      | InnoDB | 8900.00 | 1100.00  | 500.00  | 4.76              |
+---------------+-------------+--------+---------+----------+---------+-------------------+
```

**Interprétation** :
- **< 5%** : Fragmentation négligeable
- **5-15%** : Fragmentation modérée → planifier OPTIMIZE
- **> 15%** : Fragmentation importante → OPTIMIZE urgent
- **> 30%** : Fragmentation critique → OPTIMIZE immédiat

### 2. Statistiques obsolètes

Les **statistiques** permettent à l'optimiseur de choisir le meilleur plan d'exécution.

#### Détection de statistiques obsolètes

```sql
-- Vérifier la date de dernière ANALYZE
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    UPDATE_TIME,
    TIMESTAMPDIFF(DAY, UPDATE_TIME, NOW()) AS days_since_update
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
    AND UPDATE_TIME IS NOT NULL
ORDER BY days_since_update DESC;
```

**Indicateurs de statistiques obsolètes** :
- ✅ Plans d'exécution sous-optimaux (EXPLAIN montre full scan au lieu d'index)
- ✅ Requêtes soudainement plus lentes sans changement de code
- ✅ Cardinalité incorrecte dans `SHOW INDEX`
- ✅ Modifications massives de données (> 10% des lignes)

#### Exemple d'impact

```sql
-- Avant ANALYZE : mauvais plan (full scan)
EXPLAIN SELECT * FROM products WHERE category_id = 5;
-- rows: 1000000 (estimation incorrecte)

-- Après importation de données + ANALYZE
ANALYZE TABLE products;

-- Après ANALYZE : bon plan (index utilisé)
EXPLAIN SELECT * FROM products WHERE category_id = 5;
-- rows: 150 (estimation correcte)
```

### 3. Corruption de tables

#### Signes de corruption

- ❌ Erreurs SQL aléatoires : "Table is marked as crashed"
- ❌ Résultats incohérents (nombre de lignes varie)
- ❌ Crashs serveur répétés
- ❌ Warnings dans l'error log

#### Vérification proactive

```sql
-- CHECK toutes les tables d'une base
SELECT
    CONCAT('CHECK TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';') AS check_command
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'ecommerce'
    AND ENGINE IN ('MyISAM', 'Aria', 'InnoDB');
```

---

## Stratégies de maintenance par moteur

### InnoDB (moteur par défaut)

#### Caractéristiques

- ✅ **ANALYZE** : Très efficace et rapide
- ⚠️ **OPTIMIZE** : Reconstruction complète (ALTER TABLE)
- ❌ **REPAIR** : Non supporté

#### Stratégie InnoDB

```sql
-- Maintenance hebdomadaire
ANALYZE TABLE orders;
ANALYZE TABLE products;
ANALYZE TABLE customers;

-- Maintenance mensuelle (fenêtre de maintenance)
-- OPTIMIZE uniquement si fragmentation > 15%
OPTIMIZE TABLE logs;  -- Attention: très lourd !

-- Alternative à OPTIMIZE pour InnoDB
ALTER TABLE logs ENGINE=InnoDB;  -- Équivalent
```

#### Optimisation InnoDB spécifique

```sql
-- Vérifier le tablespace
SELECT
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb
FROM information_schema.TABLES
WHERE ENGINE = 'InnoDB'
    AND TABLE_SCHEMA = 'ecommerce';

-- Libérer l'espace dans le tablespace système (nécessite dump/restore)
-- Ou utiliser innodb_file_per_table (défaut depuis MariaDB 5.6)
```

### MyISAM / Aria

#### Caractéristiques

- ✅ **OPTIMIZE** : Efficace et rapide
- ✅ **ANALYZE** : Rapide
- ✅ **CHECK** : Plusieurs niveaux
- ✅ **REPAIR** : Supporté

#### Stratégie MyISAM/Aria

```sql
-- Maintenance hebdomadaire
CHECK TABLE legacy_table MEDIUM;
ANALYZE TABLE legacy_table;

-- Maintenance mensuelle
OPTIMIZE TABLE legacy_table;

-- En cas de corruption
REPAIR TABLE legacy_table;
```

---

## Impact et verrouillage

### Tableau des verrouillages

| Opération | InnoDB | MyISAM | Lecture autorisée | Écriture autorisée |
|-----------|--------|--------|-------------------|-------------------|
| **ANALYZE** | Read lock court | Read lock | ✅ Oui (après lock) | ❌ Non (pendant lock) |
| **CHECK** | Read lock | Read lock | ✅ Oui | ❌ Non |
| **OPTIMIZE** | Table rebuild | Write lock | ❌ Non | ❌ Non |
| **REPAIR** | N/A | Write lock | ❌ Non | ❌ Non |

### Mesure de l'impact

```sql
-- Activer le profiling
SET profiling = 1;

-- Exécuter la maintenance
ANALYZE TABLE products;

-- Mesurer le temps
SHOW PROFILES;

-- Désactiver
SET profiling = 0;
```

**Exemple de durée** :

| Table | Lignes | Taille | ANALYZE | OPTIMIZE (InnoDB) |
|-------|--------|--------|---------|-------------------|
| 10K lignes | 10,000 | 5 MB | < 1s | ~2s |
| 1M lignes | 1,000,000 | 500 MB | ~5s | ~60s |
| 10M lignes | 10,000,000 | 5 GB | ~30s | ~10min |
| 100M lignes | 100,000,000 | 50 GB | ~5min | ~2h |

---

## Planification de la maintenance

### Maintenance préventive (schedule recommandé)

#### Quotidienne

```sql
-- CHECK rapide des tables critiques
CHECK TABLE orders FAST;
CHECK TABLE payments FAST;
CHECK TABLE users FAST;
```

```bash
# Script cron quotidien
#!/bin/bash
# /etc/cron.daily/mariadb-check

mariadb -e "
    SELECT CONCAT('CHECK TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ' FAST;')
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = 'ecommerce'
    AND ENGINE = 'InnoDB'
" -sN | mariadb
```

#### Hebdomadaire

```sql
-- ANALYZE toutes les tables actives
ANALYZE TABLE orders;
ANALYZE TABLE order_items;
ANALYZE TABLE products;
ANALYZE TABLE customers;
ANALYZE TABLE invoices;

-- CHECK approfondi
CHECK TABLE orders MEDIUM;
```

```bash
# Script cron hebdomadaire (dimanche 2h du matin)
# /etc/cron.weekly/mariadb-analyze

mariadb -e "
    SELECT CONCAT('ANALYZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';')
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = 'ecommerce'
" -sN | mariadb
```

#### Mensuelle

```sql
-- OPTIMIZE tables avec fragmentation > 10%
-- Uniquement pendant fenêtre de maintenance

SELECT
    CONCAT('OPTIMIZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';') AS optimize_cmd,
    ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS frag_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'ecommerce'
    AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) > 0.10;
```

### Maintenance réactive (ad-hoc)

#### Après importation massive

```sql
-- 1. Désactiver les index (optionnel, MyISAM uniquement)
ALTER TABLE products DISABLE KEYS;

-- 2. Import
LOAD DATA INFILE '/tmp/products.csv' INTO TABLE products;

-- 3. Réactiver et reconstruire index
ALTER TABLE products ENABLE KEYS;

-- 4. Mettre à jour statistiques
ANALYZE TABLE products;
```

#### Après crash serveur

```bash
#!/bin/bash
# Script post-crash

echo "Vérification des tables après crash..."

# 1. Check toutes les tables
mariadb -e "
    SELECT CONCAT('CHECK TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ' EXTENDED;')
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
" -sN | mariadb > /tmp/check_results.log

# 2. Identifier les corruptions
grep -i "error\|corrupt" /tmp/check_results.log

# 3. Notifier admin si corruption détectée
if grep -qi "corrupt" /tmp/check_results.log; then
    mail -s "URGENT: Tables corrompues détectées" dba@example.com < /tmp/check_results.log
fi
```

---

## Automatisation de la maintenance

### Script complet de maintenance automatique

```bash
#!/bin/bash
# /usr/local/bin/mariadb-maintenance.sh
# Maintenance automatique MariaDB

# Configuration
DB_NAME="ecommerce"
LOG_FILE="/var/log/mysql/maintenance-$(date +%Y%m%d).log"
FRAGMENTATION_THRESHOLD=15

# Fonction de log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Début maintenance MariaDB ==="

# 1. ANALYZE toutes les tables
log "Phase 1: ANALYZE TABLE"
mariadb -sN -e "
    SELECT CONCAT('ANALYZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';')
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB_NAME'
" | while read cmd; do
    log "Exécution: $cmd"
    mariadb -e "$cmd" 2>&1 | tee -a "$LOG_FILE"
done

# 2. CHECK tables
log "Phase 2: CHECK TABLE"
mariadb -sN -e "
    SELECT CONCAT('CHECK TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ' MEDIUM;')
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB_NAME'
" | while read cmd; do
    log "Exécution: $cmd"
    mariadb -e "$cmd" 2>&1 | tee -a "$LOG_FILE"
done

# 3. OPTIMIZE si fragmentation élevée
log "Phase 3: OPTIMIZE TABLE (si nécessaire)"
mariadb -sN -e "
    SELECT
        CONCAT('OPTIMIZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';') AS cmd,
        ROUND((DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100, 2) AS frag
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '$DB_NAME'
        AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100 > $FRAGMENTATION_THRESHOLD
" | while read cmd frag; do
    log "Fragmentation $frag% détectée: $cmd"
    mariadb -e "$cmd" 2>&1 | tee -a "$LOG_FILE"
done

log "=== Fin maintenance MariaDB ==="

# Envoyer rapport par email
mail -s "Rapport maintenance MariaDB $(date +%Y-%m-%d)" dba@example.com < "$LOG_FILE"
```

### Configuration cron

```bash
# /etc/cron.d/mariadb-maintenance

# Maintenance quotidienne (3h du matin)
0 3 * * * mysql /usr/local/bin/mariadb-maintenance.sh

# Alternative: systemd timer
# /etc/systemd/system/mariadb-maintenance.timer
```

### Systemd timer (alternative moderne)

```ini
# /etc/systemd/system/mariadb-maintenance.timer
[Unit]
Description=MariaDB Maintenance Timer

[Timer]
OnCalendar=daily
OnCalendar=03:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/mariadb-maintenance.service
[Unit]
Description=MariaDB Maintenance Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mariadb-maintenance.sh
User=mysql
```

```bash
# Activer le timer
sudo systemctl enable mariadb-maintenance.timer
sudo systemctl start mariadb-maintenance.timer

# Vérifier
sudo systemctl list-timers
```

---

## Maintenance en production sans downtime

### Stratégies pour minimiser l'impact

#### 1. Fenêtre de maintenance

```bash
# Planifier pendant les heures creuses
# Ex: 2h-4h du matin pour e-commerce B2C
0 2 * * 0 mysql /usr/local/bin/mariadb-optimize-weekly.sh
```

#### 2. Maintenance table par table

```sql
-- Au lieu de tout faire d'un coup
OPTIMIZE TABLE table1;
-- Pause...
OPTIMIZE TABLE table2;
-- Pause...
```

#### 3. Utiliser pt-online-schema-change (Percona)

```bash
# OPTIMIZE sans bloquer les écritures
pt-online-schema-change \
    --alter "ENGINE=InnoDB" \
    --execute \
    h=localhost,D=ecommerce,t=orders
```

#### 4. Réplication : Maintenance sur slave d'abord

```bash
# 1. Maintenance sur slave
# 2. Promouvoir slave en master (failover)
# 3. Maintenance sur ancien master (maintenant slave)
# 4. Failback si nécessaire
```

---

## Monitoring et métriques

### Métriques clés à surveiller

```sql
-- 1. Taille totale des bases
SELECT
    TABLE_SCHEMA,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA
ORDER BY total_mb DESC;

-- 2. Top 10 tables par taille
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
    TABLE_ROWS
FROM information_schema.TABLES
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 10;

-- 3. Fragmentation globale
SELECT
    ROUND(SUM(DATA_FREE) / 1024 / 1024, 2) AS total_free_mb,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_used_mb,
    ROUND((SUM(DATA_FREE) / (SUM(DATA_LENGTH + INDEX_LENGTH + DATA_FREE))) * 100, 2) AS global_fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys');

-- 4. Tables non analysées depuis > 7 jours
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    UPDATE_TIME,
    TIMESTAMPDIFF(DAY, UPDATE_TIME, NOW()) AS days_old
FROM information_schema.TABLES
WHERE UPDATE_TIME IS NOT NULL
    AND TIMESTAMPDIFF(DAY, UPDATE_TIME, NOW()) > 7
ORDER BY days_old DESC;
```

### Alerting

```bash
#!/bin/bash
# Script d'alerte fragmentation

FRAGMENTATION_THRESHOLD=20

FRAGMENTED=$(mariadb -sN -e "
    SELECT COUNT(*)
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = 'ecommerce'
        AND (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH + DATA_FREE)) * 100 > $FRAGMENTATION_THRESHOLD
")

if [ "$FRAGMENTED" -gt 0 ]; then
    echo "ALERTE: $FRAGMENTED tables avec fragmentation > ${FRAGMENTATION_THRESHOLD}%" | \
        mail -s "MariaDB Fragmentation Alert" dba@example.com
fi
```

---

## Bonnes pratiques de maintenance

### ✅ À FAIRE

1. **Planifier la maintenance** : Automatiser avec cron/systemd
2. **ANALYZE régulièrement** : Au minimum hebdomadaire
3. **CHECK quotidiennement** : Mode FAST pour détection précoce
4. **Surveiller la fragmentation** : Alertes si > 15%
5. **Tester les backups** : Avant toute OPTIMIZE/REPAIR
6. **Logger les opérations** : Traçabilité complète
7. **Fenêtre de maintenance** : Heures creuses pour OPTIMIZE
8. **Monitoring post-maintenance** : Vérifier amélioration performance

### ❌ À ÉVITER

1. **OPTIMIZE en production** sans planification
2. **REPAIR sans backup** récent
3. **Ignorer les warnings** de CHECK TABLE
4. **Maintenance synchrone** de toutes les tables
5. **Pas de monitoring** de l'impact
6. **OPTIMIZE systématique** sans mesure de fragmentation
7. **Négliger les statistiques** (ANALYZE)
8. **Maintenance manuelle** uniquement (pas d'automatisation)

---

## Checklist de maintenance

### Quotidienne (automatisée)

- [ ] CHECK TABLE FAST sur tables critiques
- [ ] Vérifier error log pour corruption
- [ ] Surveiller fragmentation globale
- [ ] Vérifier espace disque disponible

### Hebdomadaire (automatisée)

- [ ] ANALYZE TABLE toutes les tables actives
- [ ] CHECK TABLE MEDIUM tables critiques
- [ ] Analyser slow query log
- [ ] Rapport fragmentation par table

### Mensuelle (fenêtre maintenance)

- [ ] OPTIMIZE tables fragmentées > 15%
- [ ] CHECK TABLE EXTENDED tables critiques
- [ ] Test restauration PITR
- [ ] Audit sécurité et privilèges
- [ ] Revue performance indexes

### Trimestrielle

- [ ] Audit complet schéma (tables inutilisées, etc.)
- [ ] Revue stratégie partitionnement
- [ ] Test failover haute disponibilité
- [ ] Revue et mise à jour scripts maintenance

---

## ✅ Points clés à retenir

- **4 commandes** : OPTIMIZE (défragmentation), ANALYZE (statistiques), CHECK (intégrité), REPAIR (réparation)
- **Fragmentation** : > 15% = OPTIMIZE nécessaire
- **Statistiques** : ANALYZE hebdomadaire minimum, après imports massifs
- **Intégrité** : CHECK quotidien (FAST), CHECK approfondi hebdomadaire
- **InnoDB** : OPTIMIZE = reconstruction complète (très lourd)
- **MyISAM/Aria** : Toutes les commandes supportées
- **Automatisation** : cron ou systemd timer indispensable
- **Production** : Fenêtre de maintenance pour OPTIMIZE
- **Monitoring** : Surveiller fragmentation, statistiques obsolètes
- **Backup** : TOUJOURS avant REPAIR ou OPTIMIZE critique
- **pt-online-schema-change** : Alternative sans downtime pour OPTIMIZE
- **Logs** : Tracer toutes les opérations de maintenance

---

## 🔗 Ressources et références

- [📖 Documentation officielle - OPTIMIZE TABLE](https://mariadb.com/kb/en/optimize-table/)
- [📖 Documentation officielle - ANALYZE TABLE](https://mariadb.com/kb/en/analyze-table/)
- [📖 Documentation officielle - CHECK TABLE](https://mariadb.com/kb/en/check-table/)
- [📖 Documentation officielle - REPAIR TABLE](https://mariadb.com/kb/en/repair-table/)
- [📖 InnoDB Table and Index Structures](https://mariadb.com/kb/en/innodb-table-and-index-structures/)
- [🔧 Percona Toolkit - pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html)

---

## ➡️ Sections suivantes

- **11.6.1 OPTIMIZE TABLE** : Détails approfondis, stratégies par moteur
- **11.6.2 ANALYZE TABLE** : Statistiques, persistance, histogrammes
- **11.6.3 CHECK TABLE et REPAIR TABLE** : Vérification intégrité, réparation

---

**💡 Conseil final** : La maintenance n'est pas un coût, c'est un **investissement** dans la santé et la performance de votre infrastructure. Automatisez-la aujourd'hui, remerciez-vous demain ! 🔧⚡

⏭️ [OPTIMIZE TABLE](/11-administration-configuration/06.1-optimize-table.md)
