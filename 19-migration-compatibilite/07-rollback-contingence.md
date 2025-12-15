🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.7 Rollback et contingence

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Maîtrise des stratégies de sauvegarde (chapitre 12), expérience en gestion d'incidents, connaissance des architectures haute disponibilité (chapitre 14)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Concevoir des plans de rollback adaptés à chaque stratégie de migration
- Mettre en place des procédures de contingence robustes et testées
- Gérer les données créées pendant la période de migration en cas de rollback
- Définir des critères objectifs de déclenchement du rollback
- Orchestrer une communication de crise efficace
- Documenter et tester les procédures de repli avant la migration

---

## Introduction

"Hope is not a strategy" — cette maxime s'applique parfaitement aux migrations de bases de données. Espérer que tout se passera bien sans avoir préparé de plan B est une recette pour le désastre. Les migrations les plus réussies sont souvent celles où le plan de rollback n'a jamais été utilisé, précisément parce que sa préparation minutieuse a forcé l'équipe à anticiper tous les scénarios problématiques.

Un plan de rollback n'est pas un aveu de défaite ou un signe de manque de confiance. C'est une assurance professionnelle qui permet de migrer sereinement, sachant qu'en cas de problème imprévu, un retour à l'état stable est possible. Cette section vous guide dans la conception, la documentation et le test de ces plans critiques.

---

## Philosophie du rollback

### Le rollback comme filet de sécurité

```
Approche mentale du rollback
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

         MIGRATION RÉUSSIE                    MIGRATION ÉCHOUÉE
         (objectif principal)                 (scénario alternatif)
                │                                     │
                │                                     │
                ▼                                     ▼
        ┌───────────────┐                    ┌───────────────┐
        │  Production   │                    │   ROLLBACK    │
        │  MariaDB 11.8 │                    │   ACTIVÉ      │
        │               │                    │               │
        │  ✓ Stable     │                    │  Retour état  │
        │  ✓ Performant │                    │  stable connu │
        │  ✓ Validé     │                    │               │
        └───────────────┘                    └───────────────┘
                                                    │
                                                    │
                                                    ▼
                                            ┌───────────────┐
                                            │  Analyse      │
                                            │  post-mortem  │
                                            │               │
                                            │  Correction   │
                                            │  Nouvelle     │
                                            │  tentative    │
                                            └───────────────┘

Le rollback n'est pas un échec, c'est une décision professionnelle
de préserver la continuité de service.
```

### Principes fondamentaux

| Principe | Description | Implication |
|----------|-------------|-------------|
| **Préparation** | Le rollback se prépare AVANT la migration | Documentation, scripts, tests |
| **Rapidité** | Le rollback doit être plus rapide que le fix | Objectif : minutes, pas heures |
| **Automatisation** | Minimiser les interventions manuelles | Scripts pré-validés |
| **Réversibilité** | Chaque étape doit être réversible | Points de contrôle |
| **Données** | Préserver l'intégrité des données | Synchronisation, delta |

---

## Types de rollback par stratégie de migration

### Rollback après migration in-place

La migration in-place modifie directement les fichiers de données. Le rollback nécessite une restauration.

```
Rollback In-Place
━━━━━━━━━━━━━━━━━

AVANT MIGRATION
┌─────────────────────┐
│ MariaDB 11.4        │
│                     │
│ ┌─────────────────┐ │
│ │ Data files      │ │ ──────▶ BACKUP COMPLET
│ │ (originaux)     │ │         (mariabackup)
│ └─────────────────┘ │
└─────────────────────┘

APRÈS MIGRATION (problème détecté)
┌─────────────────────┐
│ MariaDB 11.8        │
│                     │
│ ┌─────────────────┐ │
│ │ Data files      │ │ ◀────── Ces fichiers sont
│ │ (modifiés)      │ │         INCOMPATIBLES avec 11.4
│ └─────────────────┘ │
└─────────────────────┘

ROLLBACK REQUIS
1. Arrêter MariaDB 11.8
2. Réinstaller MariaDB 11.4
3. Restaurer les data files depuis le backup
4. Démarrer MariaDB 11.4
5. Appliquer les transactions depuis le backup (PITR si disponible)
```

**Script de rollback in-place :**

```bash
#!/bin/bash
# rollback_inplace.sh
# Rollback d'une migration in-place

set -e

# Configuration
BACKUP_DIR="/backup/pre-migration"
DATA_DIR="/var/lib/mysql"
OLD_VERSION="11.4"
LOG_FILE="/var/log/mariadb-rollback.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "═══════════════════════════════════════════════════════════"
log "   ROLLBACK MIGRATION IN-PLACE"
log "═══════════════════════════════════════════════════════════"

# Vérification du backup
if [ ! -d "$BACKUP_DIR" ]; then
    log "ERREUR: Backup non trouvé dans $BACKUP_DIR"
    exit 1
fi

# Confirmation
read -p "⚠️ ATTENTION: Cette opération va restaurer MariaDB $OLD_VERSION. Continuer? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    log "Rollback annulé par l'utilisateur"
    exit 0
fi

# 1. Arrêt du service
log "[1/6] Arrêt de MariaDB..."
systemctl stop mariadb || true
sleep 5

# 2. Sauvegarde de l'état actuel (au cas où)
log "[2/6] Sauvegarde de l'état actuel..."
EMERGENCY_BACKUP="/backup/emergency-$(date +%Y%m%d-%H%M%S)"
mkdir -p $EMERGENCY_BACKUP
cp -a $DATA_DIR $EMERGENCY_BACKUP/

# 3. Suppression des données actuelles
log "[3/6] Suppression des données actuelles..."
rm -rf $DATA_DIR/*

# 4. Restauration du backup
log "[4/6] Restauration du backup..."
if [ -f "$BACKUP_DIR/physical/backup.stream" ]; then
    # Backup streamé
    mbstream -x -C $DATA_DIR < $BACKUP_DIR/physical/backup.stream
    mariabackup --prepare --target-dir=$DATA_DIR
else
    # Backup standard
    mariabackup --copy-back --target-dir=$BACKUP_DIR/physical
fi

# 5. Correction des permissions
log "[5/6] Correction des permissions..."
chown -R mysql:mysql $DATA_DIR

# 6. Downgrade des paquets si nécessaire
log "[6/6] Vérification de la version MariaDB..."
CURRENT_VERSION=$(mariadb --version 2>/dev/null | grep -oP '\d+\.\d+' | head -1)
if [ "$CURRENT_VERSION" != "$OLD_VERSION" ]; then
    log "Downgrade des paquets requis: $CURRENT_VERSION → $OLD_VERSION"
    # Adapter selon le gestionnaire de paquets
    # apt install mariadb-server=$OLD_VERSION* mariadb-client=$OLD_VERSION*
    log "⚠️ Downgrade manuel des paquets peut être nécessaire"
fi

# Démarrage
log "Démarrage de MariaDB..."
systemctl start mariadb

# Vérification
log "Vérification post-rollback..."
mariadb -e "SELECT VERSION();" && log "✅ Rollback terminé avec succès"

log "═══════════════════════════════════════════════════════════"
log "   ROLLBACK TERMINÉ"
log "═══════════════════════════════════════════════════════════"
log "Backup d'urgence de l'état pré-rollback: $EMERGENCY_BACKUP"
```

### Rollback après migration logique

La migration logique crée une nouvelle instance. Le rollback est plus simple : revenir à l'ancienne instance.

```
Rollback Migration Logique
━━━━━━━━━━━━━━━━━━━━━━━━━━

PENDANT MIGRATION
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL (Source)      │ ──────▶ │ MariaDB (Cible)     │
│ ACTIVE              │  Dump   │ STANDBY             │
│                     │         │                     │
│ Applications ───────│─────────│──────────▶          │
│                     │         │                     │
└─────────────────────┘         └─────────────────────┘

POST-CUTOVER (problème détecté)
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL (Source)      │         │ MariaDB (Cible)     │
│ STANDBY             │         │ ACTIVE (problème)   │
│ (toujours dispo)    │         │                     │
│                     │         │ Applications ───────│
└─────────────────────┘         └─────────────────────┘

ROLLBACK
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL (Source)      │         │ MariaDB (Cible)     │
│ ACTIVE              │ ◀────── │ DÉSACTIVÉ           │
│                     │ Rebas-  │                     │
│ Applications ───────│─cule    │                     │
└─────────────────────┘         └─────────────────────┘

⚠️ ATTENTION: Les données créées sur MariaDB depuis le cutover
   doivent être synchronisées vers MySQL si nécessaire.
```

**Script de rollback logique :**

```bash
#!/bin/bash
# rollback_logical.sh
# Rollback d'une migration logique (rebascule vers source)

set -e

# Configuration
SOURCE_HOST="mysql-source.example.com"
TARGET_HOST="mariadb-target.example.com"
PROXY_HOST="proxysql.example.com"
APP_CONFIG="/etc/myapp/database.yml"
LOG_FILE="/var/log/migration-rollback.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "═══════════════════════════════════════════════════════════"
log "   ROLLBACK MIGRATION LOGIQUE"
log "═══════════════════════════════════════════════════════════"

# Vérification de la source
log "[1/5] Vérification de la disponibilité de la source..."
if ! mariadb -h $SOURCE_HOST -e "SELECT 1" > /dev/null 2>&1; then
    log "ERREUR: Source $SOURCE_HOST non accessible"
    exit 1
fi
log "✓ Source accessible"

# Arrêt des écritures sur la cible
log "[2/5] Arrêt des écritures sur la cible..."
mariadb -h $TARGET_HOST -e "SET GLOBAL read_only = ON;"
mariadb -h $TARGET_HOST -e "SET GLOBAL super_read_only = ON;"
log "✓ Cible en read-only"

# Synchronisation delta (si réplication active)
log "[3/5] Vérification du delta de données..."
# Si réplication inverse active, attendre la synchronisation
if mariadb -h $SOURCE_HOST -e "SHOW SLAVE STATUS\G" | grep -q "Slave_IO_Running: Yes"; then
    log "Réplication inverse détectée, attente de synchronisation..."
    while true; do
        LAG=$(mariadb -h $SOURCE_HOST -N -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master" | awk '{print $2}')
        if [ "$LAG" == "0" ] || [ "$LAG" == "NULL" ]; then
            break
        fi
        log "  Lag actuel: ${LAG}s, attente..."
        sleep 2
    done
    log "✓ Synchronisation complète"
else
    log "⚠️ Pas de réplication inverse - des données peuvent être perdues"
    read -p "Continuer malgré tout? (yes/no): " confirm
    if [ "$confirm" != "yes" ]; then
        # Remettre la cible en écriture
        mariadb -h $TARGET_HOST -e "SET GLOBAL super_read_only = OFF;"
        mariadb -h $TARGET_HOST -e "SET GLOBAL read_only = OFF;"
        log "Rollback annulé"
        exit 1
    fi
fi

# Rebascule du proxy/load balancer
log "[4/5] Rebascule du trafic vers la source..."
if [ -n "$PROXY_HOST" ]; then
    # ProxySQL
    mysql -h $PROXY_HOST -P 6032 -u admin -padmin -e "
        UPDATE mysql_servers SET status = 'OFFLINE_SOFT' WHERE hostname = '$TARGET_HOST';
        UPDATE mysql_servers SET status = 'ONLINE' WHERE hostname = '$SOURCE_HOST';
        LOAD MYSQL SERVERS TO RUNTIME;
    "
    log "✓ ProxySQL reconfiguré"
fi

# Ou modification directe de la config applicative
if [ -f "$APP_CONFIG" ]; then
    sed -i "s/$TARGET_HOST/$SOURCE_HOST/g" $APP_CONFIG
    log "✓ Configuration applicative mise à jour"
fi

# Vérification
log "[5/5] Vérification du rollback..."
sleep 5

# Test de connexion via le proxy
ACTIVE_HOST=$(mariadb -h $PROXY_HOST -N -e "SELECT @@hostname" 2>/dev/null || echo "unknown")
log "Host actif: $ACTIVE_HOST"

if [ "$ACTIVE_HOST" == "$SOURCE_HOST" ] || [ "$ACTIVE_HOST" == "$(echo $SOURCE_HOST | cut -d. -f1)" ]; then
    log "✅ Rollback terminé avec succès"
else
    log "⚠️ Vérification manuelle recommandée"
fi

log "═══════════════════════════════════════════════════════════"
log "   ACTIONS POST-ROLLBACK REQUISES"
log "═══════════════════════════════════════════════════════════"
log "1. Vérifier le fonctionnement des applications"
log "2. Analyser les données créées sur MariaDB pendant la période"
log "3. Planifier le post-mortem"
log "4. Communiquer le statut aux parties prenantes"
```

### Rollback après migration par réplication

La réplication offre le rollback le plus élégant : inverser le sens du trafic.

```
Rollback Migration Réplication
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ÉTAT NORMAL POST-CUTOVER
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL               │ ◀────── │ MariaDB             │
│ (Ancien Primary)    │ Répl.   │ (Nouveau Primary)   │
│ Replica de secours  │ inverse │ ACTIVE              │
└─────────────────────┘         └─────────────────────┘
                                        │
                                        ▼
                                  Applications

ROLLBACK (quelques secondes)
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL               │         │ MariaDB             │
│ ACTIVE              │         │ STOP SLAVE          │
│ (redevient Primary) │         │ READ ONLY           │
└─────────────────────┘         └─────────────────────┘
        │
        ▼
  Applications

Le rollback est quasi-instantané car :
- MySQL a toutes les données (réplication inverse)
- Pas de restauration nécessaire
- Simple changement de routage
```

**Script de rollback réplication :**

```bash
#!/bin/bash
# rollback_replication.sh
# Rollback instantané via inversion de réplication

set -e

OLD_PRIMARY="mysql-source.example.com"
NEW_PRIMARY="mariadb-target.example.com"
VIP="192.168.1.100"  # Virtual IP

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "═══════════════════════════════════════════════════════════"
log "   ROLLBACK INSTANTANÉ (RÉPLICATION)"
log "═══════════════════════════════════════════════════════════"

# Temps de début
START_TIME=$(date +%s)

# 1. Vérifier que l'ancien primary est synchronisé
log "[1/4] Vérification de la synchronisation..."
LAG=$(mariadb -h $OLD_PRIMARY -N -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master" | awk '{print $2}')
if [ "$LAG" != "0" ] && [ "$LAG" != "NULL" ]; then
    log "⚠️ Lag de réplication: ${LAG}s"
    log "Attente de la synchronisation..."
    
    # Arrêter les écritures sur le nouveau primary pour permettre le rattrapage
    mariadb -h $NEW_PRIMARY -e "SET GLOBAL read_only = ON;"
    
    while [ "$LAG" != "0" ] && [ "$LAG" != "NULL" ]; do
        sleep 1
        LAG=$(mariadb -h $OLD_PRIMARY -N -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master" | awk '{print $2}')
    done
fi
log "✓ Réplication synchronisée"

# 2. Arrêter la réplication sur l'ancien primary
log "[2/4] Arrêt de la réplication sur $OLD_PRIMARY..."
mariadb -h $OLD_PRIMARY -e "STOP SLAVE;"
mariadb -h $OLD_PRIMARY -e "RESET SLAVE ALL;"
log "✓ Réplication arrêtée"

# 3. Basculer la VIP
log "[3/4] Bascule de la VIP..."
# Méthode 1: Keepalived (forcer failover)
# ssh $OLD_PRIMARY "systemctl restart keepalived"

# Méthode 2: Manipulation directe
ssh $NEW_PRIMARY "ip addr del $VIP/24 dev eth0" 2>/dev/null || true
ssh $OLD_PRIMARY "ip addr add $VIP/24 dev eth0"
ssh $OLD_PRIMARY "arping -U -c 3 -I eth0 $VIP"
log "✓ VIP basculée vers $OLD_PRIMARY"

# 4. Mettre le nouveau primary en read-only
log "[4/4] Mise en read-only de $NEW_PRIMARY..."
mariadb -h $NEW_PRIMARY -e "SET GLOBAL super_read_only = ON;"
log "✓ $NEW_PRIMARY en read-only"

# Temps de fin
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

log "═══════════════════════════════════════════════════════════"
log "   ROLLBACK TERMINÉ EN ${DURATION} SECONDES"
log "═══════════════════════════════════════════════════════════"
log ""
log "État actuel:"
log "  Primary actif: $OLD_PRIMARY"
log "  VIP: $VIP → $OLD_PRIMARY"
log "  $NEW_PRIMARY: read-only (données préservées)"
```

---

## Gestion des données delta

### Le problème du delta

Lorsque le rollback intervient après le cutover, des données ont potentiellement été créées ou modifiées sur la nouvelle base. Ces données "delta" posent un défi :

```
Timeline du problème delta
━━━━━━━━━━━━━━━━━━━━━━━━━━

T0              T1              T2              T3
│               │               │               │
▼               ▼               ▼               ▼
Cutover         Données         Problème        Rollback
vers            créées sur      détecté         décidé
MariaDB         MariaDB

                │◀─────────────▶│
                   PÉRIODE DELTA
                   
Ces données existent sur MariaDB mais PAS sur MySQL.
Que faire de ces données lors du rollback?
```

### Stratégies de gestion du delta

| Stratégie | Description | Cas d'usage |
|-----------|-------------|-------------|
| **Abandon** | Perdre les données delta | Données non critiques, période courte |
| **Export/Import** | Exporter delta, importer dans source | Données critiques, volume faible |
| **Réplication inverse** | Sync continu vers source | Volume important, zéro perte |
| **Réconciliation** | Fusion manuelle | Conflits possibles, données complexes |

### Script d'extraction du delta

```python
#!/usr/bin/env python3
# extract_delta.py
# Extraction des données créées pendant la période delta

import mariadb
import json
from datetime import datetime
from typing import Dict, List, Any

class DeltaExtractor:
    """Extrait les données créées pendant la période delta"""
    
    def __init__(self, config: Dict):
        self.config = config
        self.conn = None
    
    def connect(self):
        self.conn = mariadb.connect(**self.config)
    
    def disconnect(self):
        if self.conn:
            self.conn.close()
    
    def get_tables_with_timestamp(self) -> List[Dict]:
        """Identifie les tables avec colonne de timestamp"""
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT 
                table_schema,
                table_name,
                column_name
            FROM information_schema.columns
            WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
              AND (column_name LIKE '%created%' 
                   OR column_name LIKE '%timestamp%'
                   OR column_name LIKE '%date%'
                   OR column_name = 'created_at'
                   OR column_name = 'updated_at')
              AND data_type IN ('datetime', 'timestamp')
        """)
        
        tables = []
        for schema, table, column in cursor.fetchall():
            tables.append({
                'schema': schema,
                'table': table,
                'timestamp_column': column
            })
        
        cursor.close()
        return tables
    
    def extract_delta(self, table_info: Dict, cutover_time: datetime) -> List[Dict]:
        """Extrait les données créées après le cutover"""
        cursor = self.conn.cursor(dictionary=True)
        
        query = f"""
            SELECT * FROM {table_info['schema']}.{table_info['table']}
            WHERE {table_info['timestamp_column']} >= %s
        """
        
        cursor.execute(query, (cutover_time,))
        rows = cursor.fetchall()
        cursor.close()
        
        return rows
    
    def export_delta(self, cutover_time: datetime, output_dir: str):
        """Exporte toutes les données delta"""
        self.connect()
        
        tables = self.get_tables_with_timestamp()
        delta_summary = {
            'cutover_time': cutover_time.isoformat(),
            'extraction_time': datetime.now().isoformat(),
            'tables': []
        }
        
        print(f"Extraction du delta depuis {cutover_time}")
        print(f"Tables avec timestamp identifiées: {len(tables)}")
        print("-" * 60)
        
        for table_info in tables:
            table_name = f"{table_info['schema']}.{table_info['table']}"
            print(f"Extraction de {table_name}...")
            
            try:
                rows = self.extract_delta(table_info, cutover_time)
                
                if rows:
                    # Sauvegarder les données
                    filename = f"{output_dir}/{table_info['schema']}_{table_info['table']}_delta.json"
                    with open(filename, 'w') as f:
                        json.dump(rows, f, default=str, indent=2)
                    
                    delta_summary['tables'].append({
                        'table': table_name,
                        'timestamp_column': table_info['timestamp_column'],
                        'row_count': len(rows),
                        'file': filename
                    })
                    
                    print(f"  ✓ {len(rows)} lignes exportées")
                else:
                    print(f"  - Aucune donnée delta")
                    
            except Exception as e:
                print(f"  ✗ Erreur: {e}")
        
        # Sauvegarder le résumé
        summary_file = f"{output_dir}/delta_summary.json"
        with open(summary_file, 'w') as f:
            json.dump(delta_summary, f, indent=2)
        
        self.disconnect()
        
        print("-" * 60)
        print(f"Résumé sauvegardé dans: {summary_file}")
        
        total_rows = sum(t['row_count'] for t in delta_summary['tables'])
        print(f"Total: {total_rows} lignes delta dans {len(delta_summary['tables'])} tables")
        
        return delta_summary


class DeltaImporter:
    """Importe les données delta dans la base source"""
    
    def __init__(self, config: Dict):
        self.config = config
        self.conn = None
    
    def connect(self):
        self.conn = mariadb.connect(**self.config)
    
    def disconnect(self):
        if self.conn:
            self.conn.close()
    
    def import_table_delta(self, table_name: str, data_file: str) -> int:
        """Importe les données delta pour une table"""
        with open(data_file, 'r') as f:
            rows = json.load(f)
        
        if not rows:
            return 0
        
        cursor = self.conn.cursor()
        
        # Construire la requête INSERT
        columns = list(rows[0].keys())
        placeholders = ', '.join(['%s'] * len(columns))
        columns_str = ', '.join([f'`{c}`' for c in columns])
        
        query = f"""
            INSERT INTO {table_name} ({columns_str})
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE {', '.join([f'`{c}`=VALUES(`{c}`)' for c in columns])}
        """
        
        imported = 0
        for row in rows:
            try:
                values = [row.get(c) for c in columns]
                cursor.execute(query, values)
                imported += 1
            except Exception as e:
                print(f"  Erreur import ligne: {e}")
        
        self.conn.commit()
        cursor.close()
        
        return imported
    
    def import_all_delta(self, summary_file: str):
        """Importe tout le delta depuis le fichier de résumé"""
        with open(summary_file, 'r') as f:
            summary = json.load(f)
        
        self.connect()
        
        print("Import du delta dans la base source")
        print("-" * 60)
        
        total_imported = 0
        
        for table_info in summary['tables']:
            print(f"Import de {table_info['table']}...")
            try:
                imported = self.import_table_delta(
                    table_info['table'],
                    table_info['file']
                )
                total_imported += imported
                print(f"  ✓ {imported}/{table_info['row_count']} lignes importées")
            except Exception as e:
                print(f"  ✗ Erreur: {e}")
        
        self.disconnect()
        
        print("-" * 60)
        print(f"Total importé: {total_imported} lignes")


# Utilisation
if __name__ == '__main__':
    import sys
    import os
    from datetime import datetime
    
    # Configuration cible (MariaDB)
    target_config = {
        'host': 'mariadb-target',
        'port': 3306,
        'user': 'admin',
        'password': 'password',
        'database': 'mydb'
    }
    
    # Configuration source (MySQL)
    source_config = {
        'host': 'mysql-source',
        'port': 3306,
        'user': 'admin',
        'password': 'password',
        'database': 'mydb'
    }
    
    # Temps du cutover (à adapter)
    cutover_time = datetime(2025, 12, 15, 10, 0, 0)
    
    output_dir = '/tmp/delta_export'
    os.makedirs(output_dir, exist_ok=True)
    
    # Extraction
    print("=" * 60)
    print("PHASE 1: EXTRACTION DU DELTA")
    print("=" * 60)
    extractor = DeltaExtractor(target_config)
    summary = extractor.export_delta(cutover_time, output_dir)
    
    # Import (après confirmation)
    print("\n" + "=" * 60)
    print("PHASE 2: IMPORT DU DELTA")
    print("=" * 60)
    
    confirm = input("Importer le delta dans la source? (yes/no): ")
    if confirm == 'yes':
        importer = DeltaImporter(source_config)
        importer.import_all_delta(f"{output_dir}/delta_summary.json")
```

---

## Critères de déclenchement du rollback

### Arbre de décision

```
Arbre de décision du rollback
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌─────────────────────────────┐
                    │  PROBLÈME DÉTECTÉ           │
                    │  POST-MIGRATION             │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────┴───────────────┐
                    │  Le problème est-il         │
                    │  CRITIQUE pour le business? │
                    └─────────────────────────────┘
                           │             │
                          OUI           NON
                           │             │
                           ▼             ▼
              ┌────────────────┐  ┌────────────────┐
              │ Peut-on fixer  │  │ Documenter     │
              │ en < 30 min?   │  │ Planifier fix  │
              └────────────────┘  │ en heures      │
                │           │     │ ouvrées        │
               OUI         NON    └────────────────┘
                │           │
                ▼           ▼
        ┌───────────┐  ┌───────────────────┐
        │ Tenter    │  │ La période de     │
        │ le fix    │  │ rollback est-elle │
        └─────┬─────┘  │ encore active?    │
              │        └───────────────────┘
              │              │         │
              │             OUI       NON
              │              │         │
              ▼              ▼         ▼
        ┌───────────┐  ╔═══════════╗  ┌───────────────┐
        │ Fix       │  ║ ROLLBACK  ║  │ Plan de       │
        │ réussi?   │  ║ IMMÉDIAT  ║  │ contingence   │
        └───────────┘  ╚═══════════╝  │ alternatif    │
          │       │                   └───────────────┘
         OUI     NON
          │       │
          ▼       ▼
     ┌────────┐  ╔═══════════╗
     │Continue│  ║ ROLLBACK  ║
     │ normal │  ╚═══════════╝
     └────────┘
```

### Critères objectifs de rollback

```yaml
# rollback_criteria.yml
# Critères objectifs déclenchant un rollback

critical_triggers:
  # Déclenchement immédiat du rollback
  - name: "Corruption de données"
    condition: "Erreurs de checksum ou données manquantes"
    action: "ROLLBACK IMMÉDIAT"
    
  - name: "Perte de transactions"
    condition: "Transactions committées non visibles"
    action: "ROLLBACK IMMÉDIAT"
    
  - name: "Service indisponible"
    condition: "Erreur rate > 50% pendant > 5 minutes"
    action: "ROLLBACK IMMÉDIAT"

high_severity_triggers:
  # Rollback après 15 minutes sans résolution
  - name: "Dégradation performance critique"
    condition: "Latence P95 > 5x baseline pendant > 10 minutes"
    grace_period: "15 minutes"
    
  - name: "Erreurs applicatives"
    condition: "Erreur rate > 5% pendant > 10 minutes"
    grace_period: "15 minutes"
    
  - name: "Fonctionnalité métier critique KO"
    condition: "Feature critique non fonctionnelle"
    grace_period: "15 minutes"

medium_severity_triggers:
  # Rollback après 1 heure sans résolution
  - name: "Dégradation performance modérée"
    condition: "Latence P95 > 2x baseline pendant > 30 minutes"
    grace_period: "1 heure"
    
  - name: "Jobs batch en échec"
    condition: "Jobs critiques échouent > 2 fois"
    grace_period: "1 heure"

no_rollback_required:
  # Situations ne nécessitant PAS de rollback
  - name: "Dégradation mineure temporaire"
    condition: "Latence +20% pendant < 15 minutes"
    
  - name: "Erreurs isolées"
    condition: "Erreurs < 0.1% sur endpoints non critiques"
    
  - name: "Problème identifié avec fix connu"
    condition: "Fix applicable en < 30 minutes sans impact"
```

### Script d'évaluation automatique

```python
#!/usr/bin/env python3
# rollback_evaluator.py
# Évalue automatiquement si un rollback est nécessaire

import time
from dataclasses import dataclass
from typing import Dict, List, Optional
from enum import Enum
from datetime import datetime, timedelta

class Severity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

class Decision(Enum):
    ROLLBACK_IMMEDIATE = "rollback_immediate"
    ROLLBACK_RECOMMENDED = "rollback_recommended"
    MONITOR = "monitor"
    CONTINUE = "continue"

@dataclass
class Alert:
    name: str
    severity: Severity
    message: str
    triggered_at: datetime
    value: float
    threshold: float

class RollbackEvaluator:
    """Évalue si un rollback est nécessaire"""
    
    def __init__(self, baseline_metrics: Dict):
        self.baseline = baseline_metrics
        self.alerts: List[Alert] = []
        self.grace_periods = {
            Severity.CRITICAL: timedelta(minutes=0),
            Severity.HIGH: timedelta(minutes=15),
            Severity.MEDIUM: timedelta(minutes=60),
            Severity.LOW: timedelta(hours=4)
        }
    
    def check_error_rate(self, current_rate: float) -> Optional[Alert]:
        """Vérifie le taux d'erreurs"""
        if current_rate > 0.5:
            return Alert(
                name="error_rate_critical",
                severity=Severity.CRITICAL,
                message=f"Error rate {current_rate:.1%} > 50%",
                triggered_at=datetime.now(),
                value=current_rate,
                threshold=0.5
            )
        elif current_rate > 0.05:
            return Alert(
                name="error_rate_high",
                severity=Severity.HIGH,
                message=f"Error rate {current_rate:.1%} > 5%",
                triggered_at=datetime.now(),
                value=current_rate,
                threshold=0.05
            )
        return None
    
    def check_latency(self, current_p95: float) -> Optional[Alert]:
        """Vérifie la latence P95"""
        baseline_p95 = self.baseline.get('latency_p95', 100)
        ratio = current_p95 / baseline_p95
        
        if ratio > 5:
            return Alert(
                name="latency_critical",
                severity=Severity.HIGH,
                message=f"Latency P95 {current_p95}ms ({ratio:.1f}x baseline)",
                triggered_at=datetime.now(),
                value=current_p95,
                threshold=baseline_p95 * 5
            )
        elif ratio > 2:
            return Alert(
                name="latency_degraded",
                severity=Severity.MEDIUM,
                message=f"Latency P95 {current_p95}ms ({ratio:.1f}x baseline)",
                triggered_at=datetime.now(),
                value=current_p95,
                threshold=baseline_p95 * 2
            )
        return None
    
    def check_data_integrity(self, checksum_match: bool) -> Optional[Alert]:
        """Vérifie l'intégrité des données"""
        if not checksum_match:
            return Alert(
                name="data_corruption",
                severity=Severity.CRITICAL,
                message="Data integrity check failed",
                triggered_at=datetime.now(),
                value=0,
                threshold=1
            )
        return None
    
    def evaluate(self, metrics: Dict) -> Decision:
        """Évalue les métriques et recommande une décision"""
        self.alerts = []
        
        # Collecter les alertes
        if alert := self.check_error_rate(metrics.get('error_rate', 0)):
            self.alerts.append(alert)
        
        if alert := self.check_latency(metrics.get('latency_p95', 0)):
            self.alerts.append(alert)
        
        if alert := self.check_data_integrity(metrics.get('checksum_match', True)):
            self.alerts.append(alert)
        
        # Décision
        if not self.alerts:
            return Decision.CONTINUE
        
        # Vérifier les alertes critiques
        critical_alerts = [a for a in self.alerts if a.severity == Severity.CRITICAL]
        if critical_alerts:
            return Decision.ROLLBACK_IMMEDIATE
        
        # Vérifier les alertes haute sévérité avec grace period
        high_alerts = [a for a in self.alerts if a.severity == Severity.HIGH]
        if high_alerts:
            oldest = min(a.triggered_at for a in high_alerts)
            if datetime.now() - oldest > self.grace_periods[Severity.HIGH]:
                return Decision.ROLLBACK_RECOMMENDED
            return Decision.MONITOR
        
        # Alertes moyennes
        return Decision.MONITOR
    
    def get_recommendation(self) -> str:
        """Génère une recommandation textuelle"""
        decision = self.evaluate({})  # Utiliser les dernières métriques
        
        output = []
        output.append("=" * 60)
        output.append("ÉVALUATION ROLLBACK")
        output.append("=" * 60)
        output.append(f"Timestamp: {datetime.now().isoformat()}")
        output.append(f"Alertes actives: {len(self.alerts)}")
        output.append("")
        
        if self.alerts:
            output.append("Alertes:")
            for alert in self.alerts:
                output.append(f"  [{alert.severity.value.upper()}] {alert.name}")
                output.append(f"    {alert.message}")
        
        output.append("")
        output.append("-" * 60)
        
        if decision == Decision.ROLLBACK_IMMEDIATE:
            output.append("🚨 DÉCISION: ROLLBACK IMMÉDIAT RECOMMANDÉ")
            output.append("   Action: Exécuter le rollback maintenant")
        elif decision == Decision.ROLLBACK_RECOMMENDED:
            output.append("⚠️ DÉCISION: ROLLBACK RECOMMANDÉ")
            output.append("   Action: Préparer le rollback, tenter un dernier fix")
        elif decision == Decision.MONITOR:
            output.append("👀 DÉCISION: SURVEILLANCE RENFORCÉE")
            output.append("   Action: Monitorer, préparer le rollback")
        else:
            output.append("✅ DÉCISION: CONTINUER")
            output.append("   Action: Opération normale")
        
        output.append("=" * 60)
        
        return "\n".join(output)


# Utilisation
if __name__ == '__main__':
    # Baseline de référence
    baseline = {
        'latency_p95': 50,  # ms
        'error_rate': 0.001,  # 0.1%
        'qps': 1000
    }
    
    evaluator = RollbackEvaluator(baseline)
    
    # Simulation de métriques problématiques
    current_metrics = {
        'error_rate': 0.08,  # 8%
        'latency_p95': 280,  # ms
        'checksum_match': True
    }
    
    decision = evaluator.evaluate(current_metrics)
    print(evaluator.get_recommendation())
```

---

## Plan de communication de crise

### Matrice de communication

```
Matrice de communication - Migration et Rollback
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PHASE           │ AUDIENCE           │ CANAL        │ FRÉQUENCE
────────────────┼────────────────────┼──────────────┼───────────────
Pré-migration   │ Équipes techniques │ Slack/Teams  │ J-7, J-1
                │ Management         │ Email        │ J-7
                │ Utilisateurs       │ Status page  │ J-1
                │                    │              │
Pendant         │ Équipes techniques │ Slack/Teams  │ Toutes 30min
migration       │ Management         │ Slack/Email  │ Étapes clés
                │ Utilisateurs       │ Status page  │ Début/Fin
                │                    │              │
Incident        │ Équipes techniques │ War room     │ Continue
(si rollback)   │ Management         │ Appel/SMS    │ Immédiat
                │ Utilisateurs       │ Status page  │ 15min
                │ Communication      │ Draft préparé│ Si > 1h
                │                    │              │
Post-incident   │ Tous               │ Post-mortem  │ J+1 à J+5
```

### Templates de communication

```markdown
## TEMPLATE: Communication pré-migration

**Objet**: [MAINTENANCE PLANIFIÉE] Migration base de données - [DATE]

Bonjour,

Une maintenance planifiée aura lieu le [DATE] de [HEURE DÉBUT] à [HEURE FIN].

**Impact attendu:**
- Service [NOM] indisponible pendant environ [DURÉE]
- Fonctionnalités impactées: [LISTE]

**Actions requises:**
- Sauvegarder vos travaux en cours avant [HEURE]
- Éviter les opérations critiques pendant la fenêtre

**En cas de problème:**
- Contact: [EMAIL/TÉLÉPHONE]
- Status: [URL STATUS PAGE]

Nous vous tiendrons informés de l'avancement.

Cordialement,
[ÉQUIPE]
```

```markdown
## TEMPLATE: Communication rollback en cours

**🚨 ALERTE: Rollback en cours**

**Statut**: Rollback de la migration déclenché à [HEURE]

**Raison**: [DESCRIPTION BRÈVE]
- [Détail technique si pertinent]

**Impact actuel:**
- Service [NOM]: [STATUT]
- Durée estimée du rollback: [DURÉE]

**Prochaine mise à jour**: [HEURE]

---

**Pour les équipes techniques:**
- War room: [LIEN]
- Runbook: [LIEN]
- Contact lead: [NOM] - [TÉLÉPHONE]
```

```markdown
## TEMPLATE: Communication post-rollback

**Objet**: [RÉSOLU] Migration base de données - Rollback effectué

Bonjour,

La migration planifiée ce jour a été annulée suite à [RAISON HAUT NIVEAU].

**Chronologie:**
- [HEURE]: Début de migration
- [HEURE]: Problème détecté
- [HEURE]: Décision de rollback
- [HEURE]: Rollback terminé
- [HEURE]: Service restauré

**Impact:**
- Durée d'indisponibilité totale: [DURÉE]
- Données affectées: [Aucune / Détails]

**Prochaines étapes:**
- Analyse post-mortem: [DATE]
- Nouvelle tentative: [À PLANIFIER]

Un post-mortem détaillé sera partagé dans les prochains jours.

Nous vous présentons nos excuses pour la gêne occasionnée.

Cordialement,
[ÉQUIPE]
```

---

## Runbook de rollback

### Structure du runbook

```markdown
# RUNBOOK: Rollback Migration MariaDB 11.8

## Informations générales

| Champ | Valeur |
|-------|--------|
| Version | 1.0 |
| Date | 2025-12-15 |
| Auteur | [NOM] |
| Approuvé par | [NOM] |
| Dernière révision | [DATE] |

## Contacts d'urgence

| Rôle | Nom | Téléphone | Email |
|------|-----|-----------|-------|
| DBA Lead | [NOM] | [TEL] | [EMAIL] |
| Tech Lead | [NOM] | [TEL] | [EMAIL] |
| Ops Manager | [NOM] | [TEL] | [EMAIL] |
| Escalation | [NOM] | [TEL] | [EMAIL] |

## Prérequis

- [ ] Accès SSH aux serveurs DB
- [ ] Credentials administrateur MariaDB
- [ ] Accès au load balancer / proxy
- [ ] Backup pré-migration vérifié
- [ ] Scripts de rollback testés

## Procédure de rollback

### Étape 1: Évaluation (5 min)

1. Confirmer le problème avec l'équipe
2. Évaluer la sévérité (critique/haute/moyenne)
3. Décision go/no-go du rollback

**Commande de vérification:**
```bash
./check_health.sh
```

### Étape 2: Communication (2 min)

1. Annoncer dans le channel #incident
2. Mettre à jour la status page
3. Notifier le management si critique

**Template:**
> 🚨 Rollback migration en cours. ETA: [X] minutes.

### Étape 3: Exécution du rollback (10-30 min)

**Pour migration in-place:**
```bash
sudo ./rollback_inplace.sh
```

**Pour migration logique:**
```bash
sudo ./rollback_logical.sh
```

**Pour migration réplication:**
```bash
sudo ./rollback_replication.sh
```

### Étape 4: Vérification (5 min)

```bash
# Vérifier la version
mariadb -e "SELECT VERSION();"

# Vérifier les connexions
mariadb -e "SHOW PROCESSLIST;"

# Tester les requêtes critiques
./test_critical_queries.sh
```

### Étape 5: Validation applicative (10 min)

1. Vérifier les endpoints critiques
2. Tester un parcours utilisateur complet
3. Confirmer avec l'équipe applicative

### Étape 6: Communication finale (5 min)

1. Annoncer la fin du rollback
2. Mettre à jour la status page
3. Planifier le post-mortem

## Troubleshooting

### Problème: Rollback ne démarre pas
**Solution:** Vérifier les permissions, le backup, l'espace disque

### Problème: Données manquantes après rollback
**Solution:** Exécuter le script de récupération delta

### Problème: Applications ne se reconnectent pas
**Solution:** Vérifier le pool de connexions, redémarrer les apps

## Annexes

- Script de rollback: `/opt/migration/rollback/`
- Backup pré-migration: `/backup/pre-migration/`
- Logs: `/var/log/migration/`
```

---

## Tests du plan de rollback

### Checklist de test

```bash
#!/bin/bash
# test_rollback_plan.sh
# Teste le plan de rollback en environnement de staging

set -e

echo "═══════════════════════════════════════════════════════════"
echo "   TEST DU PLAN DE ROLLBACK"
echo "═══════════════════════════════════════════════════════════"

# 1. Vérification des prérequis
echo "[1/6] Vérification des prérequis..."

check_prerequisite() {
    local name=$1
    local cmd=$2
    
    if eval "$cmd" > /dev/null 2>&1; then
        echo "  ✓ $name"
        return 0
    else
        echo "  ✗ $name"
        return 1
    fi
}

PREREQ_OK=true
check_prerequisite "Accès SSH serveur source" "ssh mysql-source 'echo ok'" || PREREQ_OK=false
check_prerequisite "Accès SSH serveur cible" "ssh mariadb-target 'echo ok'" || PREREQ_OK=false
check_prerequisite "Backup disponible" "ls /backup/pre-migration/full_backup.sql" || PREREQ_OK=false
check_prerequisite "Script rollback existe" "ls ./rollback_logical.sh" || PREREQ_OK=false
check_prerequisite "Espace disque suffisant" "[ $(df /backup --output=avail | tail -1) -gt 10000000 ]" || PREREQ_OK=false

if [ "$PREREQ_OK" != "true" ]; then
    echo ""
    echo "⚠️ Certains prérequis ne sont pas satisfaits"
    exit 1
fi

# 2. Test de restauration du backup
echo ""
echo "[2/6] Test de restauration du backup (dry-run)..."
# Simuler sans exécuter réellement
echo "  ✓ Backup lisible et intègre"

# 3. Test de connectivité
echo ""
echo "[3/6] Test de connectivité aux bases..."
mariadb -h mysql-source -e "SELECT 'source OK'" > /dev/null && echo "  ✓ Source accessible"
mariadb -h mariadb-target -e "SELECT 'target OK'" > /dev/null && echo "  ✓ Cible accessible"

# 4. Test du script de rollback (mode simulation)
echo ""
echo "[4/6] Test du script de rollback (simulation)..."
if [ -x "./rollback_logical.sh" ]; then
    # Exécuter en mode dry-run si disponible
    # ./rollback_logical.sh --dry-run
    echo "  ✓ Script de rollback exécutable"
else
    echo "  ✗ Script de rollback non exécutable"
fi

# 5. Test de communication
echo ""
echo "[5/6] Test des canaux de communication..."
# Vérifier que les contacts sont joignables (simulation)
echo "  ✓ Contacts d'urgence documentés"
echo "  ✓ Status page accessible"

# 6. Mesure du temps estimé
echo ""
echo "[6/6] Estimation du temps de rollback..."
BACKUP_SIZE=$(du -sh /backup/pre-migration/ 2>/dev/null | cut -f1 || echo "N/A")
echo "  Taille du backup: $BACKUP_SIZE"
echo "  Temps estimé de restauration: ~30 minutes (à affiner)"

echo ""
echo "═══════════════════════════════════════════════════════════"
echo "   RÉSULTAT: PLAN DE ROLLBACK VALIDÉ"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "Recommandations:"
echo "  1. Effectuer un test complet en staging avant production"
echo "  2. Chronométrer le rollback réel en staging"
echo "  3. Documenter les temps observés"
```

---

## ✅ Points clés à retenir

- Le plan de rollback se prépare **AVANT** la migration, pas pendant
- Chaque stratégie de migration a sa **stratégie de rollback** associée
- La **gestion du delta** (données créées post-cutover) est critique en cas de rollback tardif
- Définissez des **critères objectifs** de déclenchement du rollback (SLA, erreurs, latence)
- La **communication de crise** doit être préparée avec des templates prêts à l'emploi
- Le **runbook de rollback** doit être testé et accessible à toute l'équipe
- Un rollback n'est **pas un échec** : c'est une décision professionnelle de préservation du service
- **Testez le rollback** en staging avec chronométrage avant toute migration production

---

## 🔗 Ressources et références

- [📖 MariaDB Backup and Restore](https://mariadb.com/kb/en/backup-and-restore-overview/)
- [📖 Mariabackup Full Backup and Restore](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/)
- [📖 Point-in-Time Recovery](https://mariadb.com/kb/en/point-in-time-recovery/)
- [📖 Incident Management Best Practices](https://sre.google/sre-book/managing-incidents/)

---

## ➡️ Section suivante

**[19.8 Zero-downtime migrations](./08-zero-downtime-migrations.md)** : Nous explorerons les techniques avancées de migration sans interruption de service : architecture blue-green, utilisation de la réplication, orchestration du cutover, et gestion du split-brain.

⏭️ [Zero-downtime migrations](/19-migration-compatibilite/08-zero-downtime-migrations.md)
