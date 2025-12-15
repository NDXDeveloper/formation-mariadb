🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.9 Migration System-Versioned Tables (changement format timestamp 11.8) 🆕

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Maîtrise des tables temporelles MariaDB (chapitre 8), expérience avec les migrations de schéma, compréhension des formats de stockage internes

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre le changement de format de timestamp dans MariaDB 11.8 pour les System-Versioned Tables
- Évaluer l'impact de ce changement sur vos tables temporelles existantes
- Planifier et exécuter la migration des tables versionnées vers le nouveau format
- Préserver l'intégrité de l'historique pendant la migration
- Gérer la compatibilité applicative avec le nouveau format
- Optimiser les performances des tables migrées

---

## Introduction

Les System-Versioned Tables (tables temporelles) constituent l'une des fonctionnalités les plus puissantes de MariaDB pour l'audit, la conformité réglementaire et l'analyse historique. Elles permettent de conserver automatiquement l'historique de toutes les modifications de données.

MariaDB 11.8 introduit un changement fondamental dans le format de stockage des timestamps utilisés pour le versioning : le passage d'un format 32 bits à un format 64 bits. Ce changement résout le fameux "problème de l'année 2038" et étend la plage temporelle supportée jusqu'à l'année 2106.

Bien que ce changement soit transparent pour les nouvelles tables, les tables versionnées existantes nécessitent une migration pour bénéficier du nouveau format. Cette section vous guide à travers ce processus critique.

---

## Comprendre le changement de format

### Le problème de l'année 2038

```
Le problème Year 2038
━━━━━━━━━━━━━━━━━━━━━

Format TIMESTAMP 32 bits (ancien)
┌─────────────────────────────────────────────────────────────────┐
│ Plage : 1970-01-01 00:00:01 UTC → 2038-01-19 03:14:07 UTC       │
│                                                                 │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ 32 bits = 2^31 - 1 = 2 147 483 647 secondes depuis epoch   │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│ Timeline:                                                       │
│ 1970────────────────2024─────────2038                           │
│   │                    │           ▲                            │
│   │◀────── 54 ans ────▶│◀─ 14 ────▶│                             │
│   │      utilisés      │   ans     │ OVERFLOW!                   │
│                                    │                             │
│                                    └─ Fin du support 32 bits     │
└─────────────────────────────────────────────────────────────────┘

Format TIMESTAMP 64 bits (MariaDB 11.8+)
┌─────────────────────────────────────────────────────────────────┐
│ Plage : 1970-01-01 → 2106-02-07 (et au-delà théoriquement)      │
│                                                                 │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ 64 bits = Support de dates bien au-delà de 2038            │  │
│ │ Précision microseconde préservée                           │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│ Timeline:                                                       │
│ 1970────────────────2024──────────2038──────────2106────────▶   │
│   │                    │            │             │             │
│   │◀────────────────── Support étendu ───────────▶│             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Impact sur les System-Versioned Tables

Les System-Versioned Tables utilisent deux colonnes spéciales générées automatiquement :

```sql
-- Structure interne d'une table versionnée
CREATE TABLE contracts (
    id INT PRIMARY KEY,
    client_name VARCHAR(100),
    amount DECIMAL(10,2),
    status VARCHAR(20),
    
    -- Colonnes de versioning (générées automatiquement ou explicites)
    row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START INVISIBLE,
    row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END INVISIBLE,
    
    PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;
```

**Avant MariaDB 11.8 :**
- `row_start` et `row_end` : Format TIMESTAMP 32 bits
- Plage maximale : jusqu'au 19 janvier 2038
- Impact : Impossible de versionner des données avec des dates futures > 2038

**À partir de MariaDB 11.8 :**
- `row_start` et `row_end` : Format TIMESTAMP 64 bits
- Plage étendue : jusqu'en 2106 et au-delà
- Nouvelles tables : Automatiquement au nouveau format
- Tables existantes : Nécessitent une migration

### Détection du format actuel

```sql
-- ═══════════════════════════════════════════════════════════
-- DÉTECTION DU FORMAT DE TIMESTAMP DES TABLES VERSIONNÉES
-- ═══════════════════════════════════════════════════════════

-- Identifier toutes les tables versionnées
SELECT 
    table_schema,
    table_name,
    create_options,
    engine
FROM information_schema.tables
WHERE create_options LIKE '%versioning%'
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY table_schema, table_name;

-- Vérifier le format des colonnes de versioning
SELECT 
    c.table_schema,
    c.table_name,
    c.column_name,
    c.column_type,
    c.extra,
    c.generation_expression
FROM information_schema.columns c
JOIN information_schema.tables t 
    ON c.table_schema = t.table_schema 
    AND c.table_name = t.table_name
WHERE t.create_options LIKE '%versioning%'
  AND (c.column_name LIKE '%row_start%' 
       OR c.column_name LIKE '%row_end%'
       OR c.extra LIKE '%GENERATED%ROW%')
  AND c.table_schema NOT IN ('mysql', 'information_schema', 'performance_schema')
ORDER BY c.table_schema, c.table_name, c.column_name;

-- Vérifier la version de MariaDB et les capacités
SELECT VERSION() AS mariadb_version;

-- Test de capacité TIMESTAMP étendu
SELECT 
    TIMESTAMP'2050-01-01 00:00:00' AS future_timestamp,
    UNIX_TIMESTAMP(TIMESTAMP'2050-01-01 00:00:00') AS unix_value;
-- Si unix_value > 2147483647, le format 64 bits est supporté
```

---

## Analyse d'impact pré-migration

### Script d'audit des tables versionnées

```python
#!/usr/bin/env python3
# audit_versioned_tables.py
# Audit complet des tables versionnées avant migration vers 11.8

import mariadb
from dataclasses import dataclass
from typing import List, Dict, Optional
from datetime import datetime
import json

@dataclass
class VersionedTableInfo:
    """Informations sur une table versionnée"""
    schema: str
    table_name: str
    engine: str
    row_count: int
    history_row_count: int
    data_size_mb: float
    index_size_mb: float
    row_start_column: str
    row_end_column: str
    oldest_history: Optional[datetime]
    newest_history: Optional[datetime]
    has_partitions: bool
    partition_type: Optional[str]
    estimated_migration_time_minutes: float


class VersionedTableAuditor:
    """Auditeur de tables versionnées"""
    
    def __init__(self, config: Dict):
        self.config = config
        self.conn = None
        self.tables: List[VersionedTableInfo] = []
    
    def connect(self):
        self.conn = mariadb.connect(**self.config)
    
    def disconnect(self):
        if self.conn:
            self.conn.close()
    
    def find_versioned_tables(self) -> List[Dict]:
        """Trouve toutes les tables versionnées"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT 
                table_schema,
                table_name,
                engine,
                table_rows,
                data_length,
                index_length,
                create_options
            FROM information_schema.tables
            WHERE create_options LIKE '%versioning%'
              AND table_schema NOT IN ('mysql', 'information_schema', 
                                       'performance_schema', 'sys')
        """)
        tables = cursor.fetchall()
        cursor.close()
        return tables
    
    def get_versioning_columns(self, schema: str, table: str) -> Dict:
        """Identifie les colonnes de versioning"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT column_name, extra
            FROM information_schema.columns
            WHERE table_schema = %s
              AND table_name = %s
              AND extra LIKE '%GENERATED%'
        """, (schema, table))
        
        columns = cursor.fetchall()
        cursor.close()
        
        result = {'row_start': None, 'row_end': None}
        for col in columns:
            if 'ROW START' in col['extra'].upper():
                result['row_start'] = col['column_name']
            elif 'ROW END' in col['extra'].upper():
                result['row_end'] = col['column_name']
        
        return result
    
    def get_history_stats(self, schema: str, table: str) -> Dict:
        """Obtient les statistiques d'historique"""
        cursor = self.conn.cursor(dictionary=True)
        
        # Compter les lignes historiques
        try:
            cursor.execute(f"""
                SELECT COUNT(*) as history_count
                FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL
                WHERE row_end < '2038-01-19 03:14:07'
            """)
            history_count = cursor.fetchone()['history_count']
        except:
            history_count = 0
        
        # Plage temporelle de l'historique
        try:
            cursor.execute(f"""
                SELECT 
                    MIN(row_start) as oldest,
                    MAX(row_end) as newest
                FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL
                WHERE row_end < '2038-01-19 03:14:07'
            """)
            time_range = cursor.fetchone()
        except:
            time_range = {'oldest': None, 'newest': None}
        
        cursor.close()
        
        return {
            'history_count': history_count,
            'oldest': time_range['oldest'],
            'newest': time_range['newest']
        }
    
    def check_partitions(self, schema: str, table: str) -> Dict:
        """Vérifie le partitionnement"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT 
                partition_method,
                partition_expression,
                COUNT(*) as partition_count
            FROM information_schema.partitions
            WHERE table_schema = %s
              AND table_name = %s
              AND partition_name IS NOT NULL
            GROUP BY partition_method, partition_expression
        """, (schema, table))
        
        result = cursor.fetchone()
        cursor.close()
        
        if result:
            return {
                'has_partitions': True,
                'method': result['partition_method'],
                'count': result['partition_count']
            }
        return {'has_partitions': False, 'method': None, 'count': 0}
    
    def estimate_migration_time(self, row_count: int, data_size_mb: float) -> float:
        """Estime le temps de migration en minutes"""
        # Estimation basée sur ~100K rows/minute ou 1GB/minute
        time_by_rows = row_count / 100000
        time_by_size = data_size_mb / 1024
        return max(time_by_rows, time_by_size, 1)  # Minimum 1 minute
    
    def audit_table(self, table_info: Dict) -> VersionedTableInfo:
        """Audit complet d'une table"""
        schema = table_info['table_schema']
        table = table_info['table_name']
        
        versioning_cols = self.get_versioning_columns(schema, table)
        history_stats = self.get_history_stats(schema, table)
        partition_info = self.check_partitions(schema, table)
        
        data_size_mb = (table_info['data_length'] or 0) / (1024 * 1024)
        index_size_mb = (table_info['index_length'] or 0) / (1024 * 1024)
        
        return VersionedTableInfo(
            schema=schema,
            table_name=table,
            engine=table_info['engine'],
            row_count=table_info['table_rows'] or 0,
            history_row_count=history_stats['history_count'],
            data_size_mb=data_size_mb,
            index_size_mb=index_size_mb,
            row_start_column=versioning_cols['row_start'],
            row_end_column=versioning_cols['row_end'],
            oldest_history=history_stats['oldest'],
            newest_history=history_stats['newest'],
            has_partitions=partition_info['has_partitions'],
            partition_type=partition_info['method'],
            estimated_migration_time_minutes=self.estimate_migration_time(
                table_info['table_rows'] or 0, data_size_mb
            )
        )
    
    def run_audit(self) -> List[VersionedTableInfo]:
        """Exécute l'audit complet"""
        self.connect()
        
        print("=" * 70)
        print("AUDIT DES TABLES VERSIONNÉES")
        print("=" * 70)
        
        tables = self.find_versioned_tables()
        print(f"\nTables versionnées trouvées: {len(tables)}")
        print("-" * 70)
        
        for table_info in tables:
            print(f"\nAnalyse de {table_info['table_schema']}.{table_info['table_name']}...")
            audited = self.audit_table(table_info)
            self.tables.append(audited)
            
            print(f"  Lignes actuelles: {audited.row_count:,}")
            print(f"  Lignes historiques: {audited.history_row_count:,}")
            print(f"  Taille données: {audited.data_size_mb:.2f} MB")
            print(f"  Partitionné: {'Oui (' + audited.partition_type + ')' if audited.has_partitions else 'Non'}")
            print(f"  Temps migration estimé: {audited.estimated_migration_time_minutes:.1f} min")
        
        self.disconnect()
        
        # Résumé
        total_rows = sum(t.row_count for t in self.tables)
        total_history = sum(t.history_row_count for t in self.tables)
        total_size = sum(t.data_size_mb for t in self.tables)
        total_time = sum(t.estimated_migration_time_minutes for t in self.tables)
        
        print("\n" + "=" * 70)
        print("RÉSUMÉ")
        print("=" * 70)
        print(f"Total tables versionnées: {len(self.tables)}")
        print(f"Total lignes actuelles: {total_rows:,}")
        print(f"Total lignes historiques: {total_history:,}")
        print(f"Taille totale: {total_size:.2f} MB ({total_size/1024:.2f} GB)")
        print(f"Temps migration total estimé: {total_time:.1f} min ({total_time/60:.1f} h)")
        
        return self.tables
    
    def export_report(self, filepath: str):
        """Exporte le rapport en JSON"""
        report = {
            'audit_date': datetime.now().isoformat(),
            'tables': [
                {
                    'schema': t.schema,
                    'table': t.table_name,
                    'engine': t.engine,
                    'row_count': t.row_count,
                    'history_row_count': t.history_row_count,
                    'data_size_mb': t.data_size_mb,
                    'partitioned': t.has_partitions,
                    'estimated_time_min': t.estimated_migration_time_minutes
                }
                for t in self.tables
            ]
        }
        
        with open(filepath, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"\nRapport exporté: {filepath}")


# Utilisation
if __name__ == '__main__':
    config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'root',
        'password': 'password',
        'database': 'information_schema'
    }
    
    auditor = VersionedTableAuditor(config)
    tables = auditor.run_audit()
    auditor.export_report('versioned_tables_audit.json')
```

---

## Stratégies de migration

### Stratégie 1 : Migration par ALTER TABLE

La méthode la plus simple pour les tables de taille modérée.

```sql
-- ═══════════════════════════════════════════════════════════
-- MIGRATION PAR ALTER TABLE (REBUILD)
-- ═══════════════════════════════════════════════════════════

-- Cette méthode reconstruit la table avec le nouveau format
-- Avantage : Simple
-- Inconvénient : Lock pendant la durée (downtime)

-- 1. Vérifier l'état initial
SHOW CREATE TABLE contracts\G

-- 2. Forcer la reconstruction avec le nouveau format
ALTER TABLE contracts ENGINE=InnoDB;

-- 3. Vérifier le nouveau format
-- Le format sera automatiquement mis à jour si MariaDB est en 11.8+

-- Alternative avec ALGORITHM=COPY (force une copie complète)
ALTER TABLE contracts ENGINE=InnoDB, ALGORITHM=COPY;
```

**Temps estimé par taille de table :**

| Taille | Lignes | Temps estimé |
|--------|--------|--------------|
| < 100 MB | < 100K | < 1 minute |
| 100 MB - 1 GB | 100K - 1M | 1-10 minutes |
| 1 GB - 10 GB | 1M - 10M | 10-60 minutes |
| > 10 GB | > 10M | > 1 heure |

### Stratégie 2 : Migration avec pt-online-schema-change

Pour les tables volumineuses nécessitant une migration sans downtime.

```bash
#!/bin/bash
# migrate_versioned_ptosc.sh
# Migration zero-downtime avec pt-online-schema-change

TABLE_SCHEMA="$1"
TABLE_NAME="$2"
DB_HOST="${3:-localhost}"
DB_USER="${4:-root}"
DB_PASS="${5:-}"

if [ -z "$TABLE_SCHEMA" ] || [ -z "$TABLE_NAME" ]; then
    echo "Usage: $0 <schema> <table> [host] [user] [password]"
    exit 1
fi

echo "═══════════════════════════════════════════════════════════"
echo "   MIGRATION VERSIONED TABLE - pt-online-schema-change"
echo "═══════════════════════════════════════════════════════════"
echo "Table: ${TABLE_SCHEMA}.${TABLE_NAME}"
echo ""

# Vérification que la table est versionnée
IS_VERSIONED=$(mariadb -h $DB_HOST -u $DB_USER -p$DB_PASS -N -e "
    SELECT COUNT(*) FROM information_schema.tables 
    WHERE table_schema='$TABLE_SCHEMA' 
      AND table_name='$TABLE_NAME'
      AND create_options LIKE '%versioning%'
")

if [ "$IS_VERSIONED" != "1" ]; then
    echo "ERREUR: La table n'est pas versionnée ou n'existe pas"
    exit 1
fi

echo "✓ Table versionnée confirmée"
echo ""

# Obtenir les stats de la table
STATS=$(mariadb -h $DB_HOST -u $DB_USER -p$DB_PASS -N -e "
    SELECT 
        table_rows,
        ROUND(data_length/1024/1024, 2) as data_mb
    FROM information_schema.tables
    WHERE table_schema='$TABLE_SCHEMA' AND table_name='$TABLE_NAME'
")

ROWS=$(echo "$STATS" | awk '{print $1}')
SIZE_MB=$(echo "$STATS" | awk '{print $2}')

echo "Statistiques:"
echo "  Lignes: $ROWS"
echo "  Taille: ${SIZE_MB} MB"
echo ""

# Estimation du temps
EST_MINUTES=$(echo "scale=0; $ROWS / 100000 + 1" | bc)
echo "Temps estimé: ~${EST_MINUTES} minutes"
echo ""

read -p "Lancer la migration? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
    echo "Migration annulée"
    exit 0
fi

# Exécution de pt-online-schema-change
# L'ALTER "ENGINE=InnoDB" force une reconstruction complète
echo ""
echo "Démarrage de la migration..."
echo ""

pt-online-schema-change \
    --alter "ENGINE=InnoDB" \
    --host=$DB_HOST \
    --user=$DB_USER \
    --password=$DB_PASS \
    --execute \
    --progress=time,30 \
    --max-lag=5 \
    --chunk-size=1000 \
    --critical-load="Threads_running=50" \
    --set-vars="innodb_lock_wait_timeout=5" \
    --preserve-triggers \
    D=$TABLE_SCHEMA,t=$TABLE_NAME

EXIT_CODE=$?

echo ""
if [ $EXIT_CODE -eq 0 ]; then
    echo "✅ Migration terminée avec succès"
    
    # Vérification post-migration
    echo ""
    echo "Vérification post-migration:"
    mariadb -h $DB_HOST -u $DB_USER -p$DB_PASS -e "
        SELECT 
            'Lignes actuelles' as metric,
            COUNT(*) as value
        FROM ${TABLE_SCHEMA}.${TABLE_NAME}
        UNION ALL
        SELECT 
            'Lignes historiques',
            COUNT(*)
        FROM ${TABLE_SCHEMA}.${TABLE_NAME} FOR SYSTEM_TIME ALL
        WHERE row_end < '2038-01-19'
    "
else
    echo "❌ Migration échouée (code: $EXIT_CODE)"
    exit 1
fi
```

### Stratégie 3 : Migration par réplication

Pour les environnements avec réplicas, la migration peut être faite via upgrade du replica.

```
Migration via Réplication
━━━━━━━━━━━━━━━━━━━━━━━━

PHASE 1: Upgrade du Replica
┌─────────────────────┐         ┌─────────────────────┐
│  Source             │         │  Replica            │
│  MariaDB 10.6       │ ──────▶ │  MariaDB 11.8       │
│                     │ Répl.   │                     │
│  Format ancien      │         │  Format NOUVEAU     │
│  (32 bits)          │         │  (64 bits)          │
└─────────────────────┘         └─────────────────────┘

La réplication convertit automatiquement le format lors de l'écriture
sur le replica 11.8.

PHASE 2: Promotion du Replica
┌─────────────────────┐         ┌─────────────────────┐
│  Ancien Primary     │         │  Nouveau Primary    │
│  MariaDB 10.6       │         │  MariaDB 11.8       │
│  (désactivé)        │         │  (promu)            │
└─────────────────────┘         └─────────────────────┘

Les tables versionnées sont maintenant au nouveau format.
```

```sql
-- Vérification sur le replica 11.8 après synchronisation
-- Les données historiques sont automatiquement converties

-- Sur le replica MariaDB 11.8
SELECT 
    table_name,
    engine,
    create_options
FROM information_schema.tables
WHERE table_schema = 'mydb'
  AND create_options LIKE '%versioning%';

-- Vérifier qu'on peut insérer des dates > 2038
-- (Cela confirme le nouveau format)
SET @test_future = TIMESTAMP'2050-01-01 00:00:00';
SELECT @test_future;
-- Devrait retourner '2050-01-01 00:00:00' sans erreur
```

### Stratégie 4 : Export/Import avec préservation de l'historique

Pour les migrations complexes ou cross-platform.

```sql
-- ═══════════════════════════════════════════════════════════
-- MIGRATION PAR EXPORT/IMPORT AVEC HISTORIQUE
-- ═══════════════════════════════════════════════════════════

-- ÉTAPE 1: Exporter les données actuelles ET l'historique

-- Créer une table temporaire avec TOUTES les versions
CREATE TABLE contracts_export AS
SELECT 
    id,
    client_name,
    amount,
    status,
    row_start,
    row_end,
    CASE 
        WHEN row_end = '2038-01-19 03:14:07.999999' THEN 'CURRENT'
        ELSE 'HISTORY'
    END as version_type
FROM contracts FOR SYSTEM_TIME ALL;

-- Vérifier l'export
SELECT version_type, COUNT(*) FROM contracts_export GROUP BY version_type;


-- ÉTAPE 2: Sur MariaDB 11.8 cible, créer la nouvelle structure

-- Supprimer le versioning temporairement pour import manuel
CREATE TABLE contracts_new (
    id INT PRIMARY KEY,
    client_name VARCHAR(100),
    amount DECIMAL(10,2),
    status VARCHAR(20),
    row_start TIMESTAMP(6) NOT NULL DEFAULT '1970-01-01 00:00:01',
    row_end TIMESTAMP(6) NOT NULL DEFAULT '2106-02-07 06:28:15'
);


-- ÉTAPE 3: Importer les données

-- Importer les lignes actuelles
INSERT INTO contracts_new (id, client_name, amount, status, row_start, row_end)
SELECT id, client_name, amount, status, row_start, '2106-02-07 06:28:15.999999'
FROM contracts_export WHERE version_type = 'CURRENT';

-- Importer l'historique
INSERT INTO contracts_new (id, client_name, amount, status, row_start, row_end)
SELECT id, client_name, amount, status, row_start, row_end
FROM contracts_export WHERE version_type = 'HISTORY';


-- ÉTAPE 4: Convertir en table versionnée

-- D'abord, ajouter les contraintes de période
ALTER TABLE contracts_new
    ADD PERIOD FOR SYSTEM_TIME (row_start, row_end);

-- Ensuite, activer le versioning
ALTER TABLE contracts_new
    ADD SYSTEM VERSIONING;

-- Renommer les tables
RENAME TABLE contracts TO contracts_old,
             contracts_new TO contracts;


-- ÉTAPE 5: Vérification
SELECT 'Lignes actuelles' as type, COUNT(*) as count FROM contracts
UNION ALL
SELECT 'Historique total', COUNT(*) FROM contracts FOR SYSTEM_TIME ALL
UNION ALL
SELECT 'Historique only', COUNT(*) FROM contracts FOR SYSTEM_TIME ALL 
WHERE row_end < '2038-01-19';
```

---

## Script de migration complet

```python
#!/usr/bin/env python3
# migrate_versioned_tables.py
# Migration complète des tables versionnées vers MariaDB 11.8

import mariadb
import time
from datetime import datetime
from typing import Dict, List, Optional
from dataclasses import dataclass
import subprocess
import sys

@dataclass
class MigrationResult:
    """Résultat de migration d'une table"""
    schema: str
    table: str
    success: bool
    method: str
    duration_seconds: float
    rows_migrated: int
    history_preserved: bool
    error_message: Optional[str] = None


class VersionedTableMigrator:
    """Migre les tables versionnées vers le nouveau format 11.8"""
    
    def __init__(self, config: Dict):
        self.config = config
        self.conn = None
        self.results: List[MigrationResult] = []
    
    def log(self, message: str, level: str = "INFO"):
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print(f"[{timestamp}] [{level}] {message}")
    
    def connect(self):
        self.conn = mariadb.connect(**self.config)
    
    def disconnect(self):
        if self.conn:
            self.conn.close()
    
    def verify_mariadb_version(self) -> bool:
        """Vérifie que MariaDB est en version 11.8+"""
        cursor = self.conn.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()[0]
        cursor.close()
        
        self.log(f"Version MariaDB: {version}")
        
        # Extraire le numéro de version majeur.mineur
        parts = version.split('.')
        major = int(parts[0])
        minor = int(parts[1].split('-')[0])
        
        if major < 11 or (major == 11 and minor < 8):
            self.log(f"Version {version} < 11.8. Migration du format non nécessaire.", "WARNING")
            return False
        
        return True
    
    def get_versioned_tables(self) -> List[Dict]:
        """Récupère la liste des tables versionnées"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT 
                table_schema,
                table_name,
                engine,
                table_rows,
                ROUND(data_length/1024/1024, 2) as size_mb
            FROM information_schema.tables
            WHERE create_options LIKE '%versioning%'
              AND table_schema NOT IN ('mysql', 'information_schema', 
                                       'performance_schema', 'sys')
            ORDER BY data_length DESC
        """)
        tables = cursor.fetchall()
        cursor.close()
        return tables
    
    def backup_table(self, schema: str, table: str) -> bool:
        """Crée un backup de la table avant migration"""
        backup_name = f"{table}_backup_{datetime.now().strftime('%Y%m%d%H%M%S')}"
        
        cursor = self.conn.cursor()
        try:
            # Backup de la structure et des données actuelles
            cursor.execute(f"""
                CREATE TABLE `{schema}`.`{backup_name}` AS
                SELECT * FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL
            """)
            self.conn.commit()
            self.log(f"Backup créé: {schema}.{backup_name}")
            return True
        except Exception as e:
            self.log(f"Erreur backup: {e}", "ERROR")
            return False
        finally:
            cursor.close()
    
    def migrate_alter_table(self, schema: str, table: str) -> MigrationResult:
        """Migration par ALTER TABLE (avec lock)"""
        self.log(f"Migration ALTER TABLE: {schema}.{table}")
        
        start_time = time.time()
        cursor = self.conn.cursor()
        
        try:
            # Compter les lignes avant
            cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}`")
            rows_before = cursor.fetchone()[0]
            
            cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL")
            history_before = cursor.fetchone()[0]
            
            # ALTER TABLE pour forcer la reconstruction
            cursor.execute(f"ALTER TABLE `{schema}`.`{table}` ENGINE=InnoDB, ALGORITHM=COPY")
            self.conn.commit()
            
            # Compter les lignes après
            cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}`")
            rows_after = cursor.fetchone()[0]
            
            cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL")
            history_after = cursor.fetchone()[0]
            
            duration = time.time() - start_time
            
            return MigrationResult(
                schema=schema,
                table=table,
                success=True,
                method="ALTER TABLE",
                duration_seconds=duration,
                rows_migrated=rows_after,
                history_preserved=(history_before == history_after)
            )
            
        except Exception as e:
            return MigrationResult(
                schema=schema,
                table=table,
                success=False,
                method="ALTER TABLE",
                duration_seconds=time.time() - start_time,
                rows_migrated=0,
                history_preserved=False,
                error_message=str(e)
            )
        finally:
            cursor.close()
    
    def migrate_pt_osc(self, schema: str, table: str) -> MigrationResult:
        """Migration avec pt-online-schema-change (zero-downtime)"""
        self.log(f"Migration pt-osc: {schema}.{table}")
        
        start_time = time.time()
        
        # Compter les lignes avant
        cursor = self.conn.cursor()
        cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL")
        history_before = cursor.fetchone()[0]
        cursor.close()
        
        # Construire la commande pt-osc
        cmd = [
            "pt-online-schema-change",
            f"--host={self.config['host']}",
            f"--port={self.config.get('port', 3306)}",
            f"--user={self.config['user']}",
            f"--password={self.config['password']}",
            "--alter=ENGINE=InnoDB",
            "--execute",
            "--progress=time,30",
            "--max-lag=10",
            "--chunk-size=1000",
            "--preserve-triggers",
            f"D={schema},t={table}"
        ]
        
        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=7200)
            
            duration = time.time() - start_time
            
            if result.returncode == 0:
                # Vérifier l'historique après
                cursor = self.conn.cursor()
                cursor.execute(f"SELECT COUNT(*) FROM `{schema}`.`{table}` FOR SYSTEM_TIME ALL")
                history_after = cursor.fetchone()[0]
                cursor.close()
                
                return MigrationResult(
                    schema=schema,
                    table=table,
                    success=True,
                    method="pt-online-schema-change",
                    duration_seconds=duration,
                    rows_migrated=history_after,
                    history_preserved=(history_before == history_after)
                )
            else:
                return MigrationResult(
                    schema=schema,
                    table=table,
                    success=False,
                    method="pt-online-schema-change",
                    duration_seconds=duration,
                    rows_migrated=0,
                    history_preserved=False,
                    error_message=result.stderr
                )
                
        except subprocess.TimeoutExpired:
            return MigrationResult(
                schema=schema,
                table=table,
                success=False,
                method="pt-online-schema-change",
                duration_seconds=time.time() - start_time,
                rows_migrated=0,
                history_preserved=False,
                error_message="Timeout après 2 heures"
            )
        except Exception as e:
            return MigrationResult(
                schema=schema,
                table=table,
                success=False,
                method="pt-online-schema-change",
                duration_seconds=time.time() - start_time,
                rows_migrated=0,
                history_preserved=False,
                error_message=str(e)
            )
    
    def choose_migration_method(self, table_info: Dict) -> str:
        """Choisit la méthode de migration appropriée"""
        size_mb = table_info.get('size_mb', 0)
        rows = table_info.get('table_rows', 0)
        
        # Tables petites : ALTER TABLE simple
        if size_mb < 100 and rows < 100000:
            return "alter"
        
        # Tables moyennes à grandes : pt-online-schema-change
        return "pt_osc"
    
    def migrate_table(self, table_info: Dict, method: Optional[str] = None) -> MigrationResult:
        """Migre une table avec la méthode appropriée"""
        schema = table_info['table_schema']
        table = table_info['table_name']
        
        # Choisir la méthode si non spécifiée
        if method is None:
            method = self.choose_migration_method(table_info)
        
        self.log(f"\n{'='*60}")
        self.log(f"Migration de {schema}.{table}")
        self.log(f"Méthode: {method}")
        self.log(f"Taille: {table_info.get('size_mb', 0)} MB, Lignes: {table_info.get('table_rows', 0)}")
        
        # Backup optionnel
        # self.backup_table(schema, table)
        
        # Migration
        if method == "alter":
            result = self.migrate_alter_table(schema, table)
        else:
            result = self.migrate_pt_osc(schema, table)
        
        self.results.append(result)
        
        # Afficher le résultat
        if result.success:
            self.log(f"✅ Succès en {result.duration_seconds:.1f}s")
            self.log(f"   Lignes migrées: {result.rows_migrated}")
            self.log(f"   Historique préservé: {'Oui' if result.history_preserved else 'NON!'}")
        else:
            self.log(f"❌ Échec: {result.error_message}", "ERROR")
        
        return result
    
    def migrate_all(self, method: Optional[str] = None, dry_run: bool = False):
        """Migre toutes les tables versionnées"""
        self.connect()
        
        # Vérifier la version
        if not self.verify_mariadb_version():
            self.log("Migration annulée: version MariaDB incompatible", "ERROR")
            return
        
        tables = self.get_versioned_tables()
        
        self.log("=" * 70)
        self.log(f"MIGRATION DES TABLES VERSIONNÉES VERS FORMAT 11.8")
        self.log("=" * 70)
        self.log(f"Tables à migrer: {len(tables)}")
        
        if dry_run:
            self.log("\n*** MODE DRY-RUN - Aucune modification ***\n")
            for t in tables:
                chosen_method = method or self.choose_migration_method(t)
                self.log(f"  {t['table_schema']}.{t['table_name']} → {chosen_method}")
            return
        
        # Migration
        for table_info in tables:
            self.migrate_table(table_info, method)
        
        self.disconnect()
        
        # Rapport final
        self.print_summary()
    
    def print_summary(self):
        """Affiche le résumé de la migration"""
        print("\n" + "=" * 70)
        print("RÉSUMÉ DE LA MIGRATION")
        print("=" * 70)
        
        success = [r for r in self.results if r.success]
        failed = [r for r in self.results if not r.success]
        
        print(f"Total tables: {len(self.results)}")
        print(f"Succès: {len(success)}")
        print(f"Échecs: {len(failed)}")
        
        total_duration = sum(r.duration_seconds for r in self.results)
        print(f"Durée totale: {total_duration:.1f}s ({total_duration/60:.1f} min)")
        
        # Vérification de l'historique
        history_issues = [r for r in success if not r.history_preserved]
        if history_issues:
            print(f"\n⚠️ ATTENTION: {len(history_issues)} tables avec problème d'historique:")
            for r in history_issues:
                print(f"   - {r.schema}.{r.table}")
        
        if failed:
            print(f"\n❌ Tables en échec:")
            for r in failed:
                print(f"   - {r.schema}.{r.table}: {r.error_message}")


# Point d'entrée
if __name__ == '__main__':
    import argparse
    
    parser = argparse.ArgumentParser(description='Migration des tables versionnées vers MariaDB 11.8')
    parser.add_argument('--host', default='localhost', help='Host MariaDB')
    parser.add_argument('--port', type=int, default=3306, help='Port MariaDB')
    parser.add_argument('--user', default='root', help='Utilisateur')
    parser.add_argument('--password', default='', help='Mot de passe')
    parser.add_argument('--method', choices=['alter', 'pt_osc'], help='Méthode de migration')
    parser.add_argument('--dry-run', action='store_true', help='Mode simulation')
    
    args = parser.parse_args()
    
    config = {
        'host': args.host,
        'port': args.port,
        'user': args.user,
        'password': args.password
    }
    
    migrator = VersionedTableMigrator(config)
    migrator.migrate_all(method=args.method, dry_run=args.dry_run)
```

---

## Vérification post-migration

### Tests de validation

```sql
-- ═══════════════════════════════════════════════════════════
-- VALIDATION POST-MIGRATION DES TABLES VERSIONNÉES
-- ═══════════════════════════════════════════════════════════

-- 1. Vérifier que le nouveau format est actif
-- Test : Insertion d'une date > 2038

-- Créer une ligne de test
INSERT INTO contracts (id, client_name, amount, status)
VALUES (99999, 'Test Migration', 1000.00, 'test');

-- Vérifier les timestamps générés
SELECT 
    id,
    row_start,
    row_end,
    CASE 
        WHEN row_end > '2038-01-19 03:14:07' THEN 'FORMAT 64 BITS ✓'
        ELSE 'FORMAT 32 BITS (ancien)'
    END as format_check
FROM contracts 
WHERE id = 99999;

-- Nettoyer
DELETE FROM contracts WHERE id = 99999;


-- 2. Vérifier l'intégrité de l'historique

-- Compter les versions par enregistrement
SELECT 
    id,
    COUNT(*) as version_count,
    MIN(row_start) as first_version,
    MAX(row_start) as last_version
FROM contracts FOR SYSTEM_TIME ALL
GROUP BY id
ORDER BY version_count DESC
LIMIT 10;


-- 3. Vérifier les requêtes temporelles

-- AS OF (point dans le temps)
SELECT * FROM contracts 
FOR SYSTEM_TIME AS OF TIMESTAMP'2024-06-15 12:00:00'
LIMIT 5;

-- BETWEEN (plage temporelle)
SELECT * FROM contracts 
FOR SYSTEM_TIME BETWEEN 
    TIMESTAMP'2024-01-01 00:00:00' AND TIMESTAMP'2024-12-31 23:59:59'
LIMIT 10;

-- ALL (tout l'historique)
SELECT COUNT(*) as total_versions FROM contracts FOR SYSTEM_TIME ALL;


-- 4. Test de performance des requêtes temporelles

-- Mesurer le temps d'une requête typique
SET @start_time = NOW(6);

SELECT COUNT(*) FROM contracts 
FOR SYSTEM_TIME AS OF TIMESTAMP'2024-06-15 12:00:00';

SELECT TIMEDIFF(NOW(6), @start_time) as query_time;


-- 5. Vérifier les index sur les colonnes de versioning

SHOW INDEX FROM contracts WHERE Column_name IN ('row_start', 'row_end');
```

### Script de validation automatique

```bash
#!/bin/bash
# validate_versioned_migration.sh
# Validation post-migration des tables versionnées

DB_HOST="${1:-localhost}"
DB_USER="${2:-root}"
DB_PASS="${3:-}"
DB_NAME="${4:-mydb}"
TABLE="${5:-contracts}"

echo "═══════════════════════════════════════════════════════════"
echo "   VALIDATION POST-MIGRATION TABLE VERSIONNÉE"
echo "═══════════════════════════════════════════════════════════"
echo "Table: ${DB_NAME}.${TABLE}"
echo ""

# Fonction de requête
query() {
    mariadb -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME -N -e "$1" 2>/dev/null
}

# 1. Vérifier que la table est versionnée
echo "[1/5] Vérification du versioning..."
IS_VERSIONED=$(query "
    SELECT COUNT(*) FROM information_schema.tables 
    WHERE table_schema='$DB_NAME' 
      AND table_name='$TABLE'
      AND create_options LIKE '%versioning%'
")

if [ "$IS_VERSIONED" == "1" ]; then
    echo "  ✓ Table versionnée"
else
    echo "  ✗ Table NON versionnée!"
    exit 1
fi

# 2. Vérifier le format 64 bits
echo ""
echo "[2/5] Vérification du format timestamp..."

# Insérer une ligne de test
TEST_ID=999999999
query "INSERT INTO $TABLE (id, client_name, amount, status) VALUES ($TEST_ID, 'Test', 1, 'test')" 2>/dev/null

# Vérifier row_end
ROW_END=$(query "SELECT row_end FROM $TABLE WHERE id=$TEST_ID")
query "DELETE FROM $TABLE WHERE id=$TEST_ID"

if [[ "$ROW_END" > "2038-01-19" ]]; then
    echo "  ✓ Format 64 bits (row_end: $ROW_END)"
else
    echo "  ⚠ Format peut-être ancien (row_end: $ROW_END)"
fi

# 3. Compter les lignes
echo ""
echo "[3/5] Comptage des données..."
CURRENT=$(query "SELECT COUNT(*) FROM $TABLE")
HISTORY=$(query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME ALL WHERE row_end < '2038-01-19'")
TOTAL=$(query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME ALL")

echo "  Lignes actuelles: $CURRENT"
echo "  Lignes historiques: $HISTORY"
echo "  Total versions: $TOTAL"

# 4. Tester les requêtes temporelles
echo ""
echo "[4/5] Test des requêtes temporelles..."

# AS OF
AS_OF_RESULT=$(query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME AS OF NOW() - INTERVAL 1 DAY" 2>&1)
if [[ "$AS_OF_RESULT" =~ ^[0-9]+$ ]]; then
    echo "  ✓ FOR SYSTEM_TIME AS OF: OK"
else
    echo "  ✗ FOR SYSTEM_TIME AS OF: Erreur"
fi

# BETWEEN
BETWEEN_RESULT=$(query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME BETWEEN NOW() - INTERVAL 7 DAY AND NOW()" 2>&1)
if [[ "$BETWEEN_RESULT" =~ ^[0-9]+$ ]]; then
    echo "  ✓ FOR SYSTEM_TIME BETWEEN: OK"
else
    echo "  ✗ FOR SYSTEM_TIME BETWEEN: Erreur"
fi

# ALL
ALL_RESULT=$(query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME ALL" 2>&1)
if [[ "$ALL_RESULT" =~ ^[0-9]+$ ]]; then
    echo "  ✓ FOR SYSTEM_TIME ALL: OK"
else
    echo "  ✗ FOR SYSTEM_TIME ALL: Erreur"
fi

# 5. Performance
echo ""
echo "[5/5] Test de performance..."

# Temps d'une requête historique
START=$(date +%s.%N)
query "SELECT COUNT(*) FROM $TABLE FOR SYSTEM_TIME ALL" > /dev/null
END=$(date +%s.%N)
DURATION=$(echo "$END - $START" | bc)

echo "  Temps requête historique: ${DURATION}s"

if (( $(echo "$DURATION < 5" | bc -l) )); then
    echo "  ✓ Performance acceptable"
else
    echo "  ⚠ Performance peut être optimisée"
fi

echo ""
echo "═══════════════════════════════════════════════════════════"
echo "   VALIDATION TERMINÉE"
echo "═══════════════════════════════════════════════════════════"
```

---

## Gestion de la compatibilité applicative

### Impact sur les applications

```
Impact du changement de format
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRANSPARENT                          ATTENTION REQUISE
──────────────────────────────────   ──────────────────────────────────
✓ Requêtes SELECT normales           ⚠ Comparaisons avec littéraux
✓ INSERT, UPDATE, DELETE               2038-01-19 03:14:07
✓ FOR SYSTEM_TIME AS OF              ⚠ Stockage externe de row_end
✓ FOR SYSTEM_TIME BETWEEN            ⚠ Applications comparant row_end
✓ FOR SYSTEM_TIME ALL                ⚠ ETL/exports avec formats stricts
✓ Triggers et procédures             ⚠ Réplication vers versions < 11.8
```

### Adaptation du code applicatif

```python
# Exemple Python : Adaptation pour la compatibilité

import mariadb
from datetime import datetime

class VersionedTableClient:
    """Client adapté pour les tables versionnées MariaDB 11.8"""
    
    # Constantes pour les marqueurs de fin de validité
    # Ancien format (< 11.8)
    OLD_MAX_TIMESTAMP = datetime(2038, 1, 19, 3, 14, 7)
    
    # Nouveau format (11.8+)
    NEW_MAX_TIMESTAMP = datetime(2106, 2, 7, 6, 28, 15)
    
    def __init__(self, config):
        self.conn = mariadb.connect(**config)
        self.max_timestamp = self._detect_format()
    
    def _detect_format(self) -> datetime:
        """Détecte le format de timestamp utilisé"""
        cursor = self.conn.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()[0]
        cursor.close()
        
        # Parser la version
        parts = version.split('.')
        major = int(parts[0])
        minor = int(parts[1].split('-')[0])
        
        if major > 11 or (major == 11 and minor >= 8):
            return self.NEW_MAX_TIMESTAMP
        return self.OLD_MAX_TIMESTAMP
    
    def is_current_row(self, row_end: datetime) -> bool:
        """Vérifie si une ligne est la version courante"""
        # Compatible avec les deux formats
        return row_end >= self.max_timestamp - timedelta(seconds=1)
    
    def get_current_data(self, table: str, conditions: str = "1=1"):
        """Récupère les données actuelles (pas l'historique)"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute(f"""
            SELECT * FROM {table}
            WHERE {conditions}
            -- Pas besoin de FOR SYSTEM_TIME pour les données actuelles
        """)
        return cursor.fetchall()
    
    def get_data_at_time(self, table: str, point_in_time: datetime, conditions: str = "1=1"):
        """Récupère les données à un instant T"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute(f"""
            SELECT * FROM {table}
            FOR SYSTEM_TIME AS OF %s
            WHERE {conditions}
        """, (point_in_time,))
        return cursor.fetchall()
    
    def get_history(self, table: str, id_column: str, id_value, 
                    start_time: datetime = None, end_time: datetime = None):
        """Récupère l'historique d'un enregistrement"""
        cursor = self.conn.cursor(dictionary=True)
        
        if start_time and end_time:
            cursor.execute(f"""
                SELECT * FROM {table}
                FOR SYSTEM_TIME BETWEEN %s AND %s
                WHERE {id_column} = %s
                ORDER BY row_start
            """, (start_time, end_time, id_value))
        else:
            cursor.execute(f"""
                SELECT * FROM {table}
                FOR SYSTEM_TIME ALL
                WHERE {id_column} = %s
                ORDER BY row_start
            """, (id_value,))
        
        return cursor.fetchall()


# Utilisation
if __name__ == '__main__':
    config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'app_user',
        'password': 'password',
        'database': 'mydb'
    }
    
    client = VersionedTableClient(config)
    
    print(f"Format détecté: max_timestamp = {client.max_timestamp}")
    
    # Récupérer les données actuelles
    current = client.get_current_data('contracts', "status = 'active'")
    print(f"Contrats actifs actuels: {len(current)}")
    
    # Récupérer les données d'il y a 30 jours
    from datetime import timedelta
    past_date = datetime.now() - timedelta(days=30)
    historical = client.get_data_at_time('contracts', past_date)
    print(f"Contrats il y a 30 jours: {len(historical)}")
    
    # Historique d'un contrat spécifique
    history = client.get_history('contracts', 'id', 1)
    print(f"Versions du contrat #1: {len(history)}")
```

---

## Cas particuliers

### Tables partitionnées avec versioning

```sql
-- ═══════════════════════════════════════════════════════════
-- MIGRATION DE TABLES VERSIONNÉES PARTITIONNÉES
-- ═══════════════════════════════════════════════════════════

-- Les tables versionnées partitionnées nécessitent une attention
-- particulière car le partitionnement utilise souvent row_end

-- Exemple de structure existante
CREATE TABLE events_versioned (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    event_data JSON,
    row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
    row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (row_start, row_end),
    PRIMARY KEY (id, row_end)
) WITH SYSTEM VERSIONING
PARTITION BY SYSTEM_TIME (
    PARTITION p_history HISTORY,
    PARTITION p_current CURRENT
);

-- Vérifier les partitions
SELECT 
    partition_name,
    partition_description,
    table_rows
FROM information_schema.partitions
WHERE table_name = 'events_versioned';

-- MIGRATION : Reconstruire les partitions

-- 1. Exporter les données
CREATE TABLE events_backup AS 
SELECT * FROM events_versioned FOR SYSTEM_TIME ALL;

-- 2. Supprimer l'ancienne table
DROP TABLE events_versioned;

-- 3. Recréer avec le nouveau format (automatique en 11.8)
CREATE TABLE events_versioned (
    id BIGINT AUTO_INCREMENT,
    event_type VARCHAR(50),
    event_data JSON,
    row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
    row_end TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (row_start, row_end),
    PRIMARY KEY (id, row_end)
) WITH SYSTEM VERSIONING
PARTITION BY SYSTEM_TIME INTERVAL 1 MONTH (
    PARTITION p_history HISTORY,
    PARTITION p_current CURRENT
);

-- 4. Réimporter les données (complexe, nécessite manipulation)
-- Voir section "Export/Import avec préservation de l'historique"
```

### Tables avec colonnes de versioning explicites

```sql
-- Tables avec colonnes de versioning nommées explicitement
CREATE TABLE audit_log (
    id INT PRIMARY KEY,
    action VARCHAR(100),
    details TEXT,
    
    -- Colonnes explicitement nommées
    valid_from TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
    valid_to TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
    
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH SYSTEM VERSIONING;

-- La migration fonctionne de la même manière
ALTER TABLE audit_log ENGINE=InnoDB, ALGORITHM=COPY;

-- Les noms de colonnes sont préservés
SHOW CREATE TABLE audit_log\G
```

### Réplication vers des versions antérieures

```
Réplication cross-version avec tables versionnées
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                  ⚠️ ATTENTION ⚠️
                  
MariaDB 11.8 ──────────────▶ MariaDB 10.6
(format 64 bits)              (format 32 bits)

Les timestamps > 2038-01-19 seront TRONQUÉS sur le replica!

Solutions:
1. Upgrade le replica vers 11.8+
2. Éviter les dates > 2038 pendant la période de transition
3. Utiliser une réplication logique filtrée
```

```sql
-- Vérification avant de configurer la réplication
-- Sur le replica (cible)
SELECT VERSION();

-- Si < 11.8, avertissement
-- Les données avec row_end > 2038 ne seront pas correctement répliquées

-- Alternative : Filtrer les tables versionnées de la réplication
-- Et les synchroniser autrement
CHANGE REPLICATION FILTER 
    REPLICATE_IGNORE_TABLE = (mydb.versioned_table);
```

---

## Checklist de migration

```markdown
## CHECKLIST MIGRATION SYSTEM-VERSIONED TABLES → MariaDB 11.8

### Pré-migration
- [ ] Audit des tables versionnées (script audit_versioned_tables.py)
- [ ] Identification du format actuel (32 bits vs 64 bits)
- [ ] Estimation du temps de migration par table
- [ ] Planification de la fenêtre de maintenance
- [ ] Backup complet de la base
- [ ] Test de la procédure en staging

### Validation de l'environnement
- [ ] MariaDB 11.8+ installé
- [ ] Espace disque suffisant (2x taille des tables)
- [ ] pt-online-schema-change disponible (si nécessaire)
- [ ] Scripts de migration testés

### Migration
- [ ] Backup des tables versionnées
- [ ] Migration table par table (ordre de criticité)
- [ ] Vérification du format après chaque table
- [ ] Vérification de l'historique préservé
- [ ] Tests des requêtes temporelles

### Post-migration
- [ ] Validation complète (script validate_versioned_migration.sh)
- [ ] Test des applications
- [ ] Monitoring des performances
- [ ] Mise à jour de la documentation
- [ ] Communication aux équipes

### Rollback (si nécessaire)
- [ ] Restauration depuis backup
- [ ] Ou rollback vers version précédente de MariaDB
```

---

## ✅ Points clés à retenir

- MariaDB 11.8 étend les timestamps de **32 bits à 64 bits**, résolvant le problème Year 2038
- Les **nouvelles tables versionnées** utilisent automatiquement le nouveau format
- Les **tables existantes** nécessitent une migration explicite (`ALTER TABLE ... ENGINE=InnoDB`)
- L'**historique est préservé** lors de la migration si effectuée correctement
- Utilisez **pt-online-schema-change** pour les tables volumineuses (zero-downtime)
- La **réplication vers des versions < 11.8** peut tronquer les timestamps > 2038
- **Testez exhaustivement** les requêtes temporelles après migration
- Les applications doivent être adaptées si elles comparent `row_end` avec des littéraux

---

## 🔗 Ressources et références

- [📖 MariaDB System-Versioned Tables](https://mariadb.com/kb/en/system-versioned-tables/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-release-notes/)
- [📖 Temporal Tables SQL:2011](https://mariadb.com/kb/en/temporal-data-tables/)
- [📖 pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [📖 Year 2038 Problem](https://en.wikipedia.org/wiki/Year_2038_problem)

---

## ➡️ Chapitre suivant

Félicitations ! Vous avez terminé le **Chapitre 19 : Migration et Compatibilité**. Le chapitre suivant, **Chapitre 20 : Écosystème et intégration**, explorera l'intégration de MariaDB avec les outils et technologies modernes : conteneurisation, Kubernetes, cloud providers, et monitoring avancé.

⏭️ [Cas d'Usage et Architectures](/20-cas-usage-architectures/README.md)
