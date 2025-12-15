🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.8 Zero-downtime migrations

> **Niveau** : Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : Maîtrise de la réplication MariaDB (chapitre 14), expérience avec les load balancers et proxies, connaissance des stratégies de rollback (section 19.7)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Concevoir et implémenter des migrations sans interruption de service
- Maîtriser les architectures blue-green et canary pour les migrations de bases de données
- Orchestrer un cutover avec un temps d'indisponibilité minimal (secondes)
- Utiliser la réplication comme vecteur de migration zero-downtime
- Prévenir et gérer les situations de split-brain
- Choisir les outils adaptés (pt-online-schema-change, gh-ost, ProxySQL)
- Appliquer ces techniques à des scénarios réels de migration MariaDB 11.8

---

## Introduction

Dans l'économie numérique actuelle, chaque seconde d'indisponibilité a un coût. Pour un site e-commerce générant 100 000 € par heure, une migration de 4 heures représente potentiellement 400 000 € de pertes. Au-delà de l'aspect financier, l'indisponibilité érode la confiance des utilisateurs et peut avoir des répercussions durables sur la réputation.

Les migrations zero-downtime répondent à cette exigence en permettant de faire évoluer l'infrastructure de base de données tout en maintenant le service opérationnel. Ces techniques, longtemps réservées aux géants du web, sont aujourd'hui accessibles grâce à des outils matures et des architectures bien documentées.

Cette section présente les patterns, outils et méthodologies pour réaliser des migrations MariaDB sans interruption de service perceptible par les utilisateurs.

---

## Définition et objectifs

### Qu'est-ce qu'une migration zero-downtime ?

```
Spectrum du downtime de migration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MIGRATION TRADITIONNELLE
┌─────────────────────────────────────────────────────────────────┐
│                        SERVICE INDISPONIBLE                     │
│  ████████████████████████████████████████████████████████████   │
│  │◀─────────────── 2-8 heures ────────────────▶│                │
└─────────────────────────────────────────────────────────────────┘

MIGRATION LOW-DOWNTIME
┌─────────────────────────────────────────────────────────────────┐
│  Service OK     │ DOWN │           Service OK                   │
│  ██████████████ │ ░░░░ │ ████████████████████████████████████   │
│                 │◀────▶│                                        │
│                  5-30 min                                       │
└─────────────────────────────────────────────────────────────────┘

MIGRATION ZERO-DOWNTIME
┌─────────────────────────────────────────────────────────────────┐
│  Service OK (source)  │ Cutover │   Service OK (cible)          │
│  █████████████████████│░░│██████████████████████████████████    │
│                       │◀▶│                                      │
│                      < 30 sec (souvent < 5 sec)                 │
└─────────────────────────────────────────────────────────────────┘

MIGRATION TRUE ZERO-DOWNTIME (avec dual-write)
┌─────────────────────────────────────────────────────────────────┐
│              SERVICE TOUJOURS DISPONIBLE                        │
│  ████████████████████████████████████████████████████████████   │
│  Aucune interruption perceptible par les utilisateurs           │
└─────────────────────────────────────────────────────────────────┘
```

### Niveaux de zero-downtime

| Niveau | Downtime | Perception utilisateur | Complexité |
|--------|----------|------------------------|------------|
| **Near-zero** | < 30 secondes | Brève interruption | Modérée |
| **Zero-downtime** | < 5 secondes | Requêtes en cours échouent | Élevée |
| **True zero** | 0 seconde | Aucune interruption | Très élevée |
| **Transparent** | 0 seconde + retry | Transparent total | Maximale |

---

## Architectures de migration zero-downtime

### Pattern 1 : Blue-Green Database

L'architecture blue-green maintient deux environnements complets et bascule le trafic instantanément.

```
Architecture Blue-Green
━━━━━━━━━━━━━━━━━━━━━━━

PHASE 1: État initial
                    ┌───────────────────┐
                    │   Load Balancer   │
                    │    (HAProxy)      │
                    └─────────┬─────────┘
                              │ 100%
                              ▼
                    ┌───────────────────┐
                    │      BLUE         │
                    │   MySQL 5.7       │
                    │    (Active)       │
                    └───────────────────┘
                    
                    ┌───────────────────┐
                    │      GREEN        │
                    │  MariaDB 11.8     │
                    │   (Standby)       │
                    └───────────────────┘


PHASE 2: Synchronisation
                    ┌───────────────────┐
                    │   Load Balancer   │
                    └─────────┬─────────┘
                              │ 100%
                              ▼
                    ┌───────────────────┐
                    │      BLUE         │──────────┐
                    │   MySQL 5.7       │ Réplication
                    │    (Active)       │          │
                    └───────────────────┘          │
                                                   ▼
                    ┌───────────────────┐
                    │      GREEN        │
                    │  MariaDB 11.8     │
                    │   (Replica)       │
                    └───────────────────┘


PHASE 3: Cutover
                    ┌───────────────────┐
                    │   Load Balancer   │
                    └─────────┬─────────┘
                              │ 100%
                              ▼
                    ┌───────────────────┐
                    │      GREEN        │
                    │  MariaDB 11.8     │
                    │    (Active)       │
                    └───────────────────┘
                              │
                              │ Réplication inverse
                              ▼              (optionnel)
                    ┌───────────────────┐
                    │      BLUE         │
                    │   MySQL 5.7       │
                    │   (Standby)       │
                    └───────────────────┘
```

**Avantages :**
- Rollback instantané (rebascule)
- Test complet avant cutover
- Isolation des environnements

**Inconvénients :**
- Double infrastructure
- Synchronisation des données complexe
- Coût élevé

### Pattern 2 : Migration par réplication chaînée

La réplication permet une migration progressive avec synchronisation continue.

```
Migration par Réplication
━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌─────────────────────────────────────────┐
                    │              Applications               │
                    └───────────────────┬─────────────────────┘
                                        │
                    ┌───────────────────┴─────────────────────┐
                    │              ProxySQL                   │
                    │         (Routage intelligent)           │
                    └───────────────────┬─────────────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              │                         │                         │
              ▼                         ▼                         ▼
    ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
    │  MySQL Primary  │──────▶│ MariaDB Replica │──────▶│ MariaDB Replica │
    │    (Source)     │ Binlog│    (Target 1)   │ Binlog│    (Target 2)   │
    └─────────────────┘       └─────────────────┘       └─────────────────┘
              │                         │                         │
              │ Writes                  │ Reads (progressive)     │
              │                         │                         │
              └─────────────────────────┴─────────────────────────┘
                         Traffic progressif vers MariaDB
```

### Pattern 3 : Strangler Fig (Migration progressive)

Inspiré du pattern applicatif, cette approche migre les données table par table ou service par service.

```
Pattern Strangler Fig
━━━━━━━━━━━━━━━━━━━━━

PHASE 1: Quelques tables migrées
┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                      │
├─────────────────────────────────────────────────────────────┤
│                     Data Access Layer                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  users_dao  │  │ orders_dao  │  │products_dao │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  MariaDB  │    │   MySQL   │    │   MySQL   │
    │  (migré)  │    │ (legacy)  │    │ (legacy)  │
    └───────────┘    └───────────┘    └───────────┘

PHASE 2: Plus de tables migrées
          │                │                │
          ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  MariaDB  │    │  MariaDB  │    │   MySQL   │
    │  (migré)  │    │  (migré)  │    │ (legacy)  │
    └───────────┘    └───────────┘    └───────────┘

PHASE FINALE: Migration complète
          │                │                │
          ▼                ▼                ▼
    ┌─────────────────────────────────────────────┐
    │              MariaDB 11.8 (complet)         │
    └─────────────────────────────────────────────┘
```

---

## Mise en place de la réplication cross-version

### Configuration MySQL → MariaDB

```sql
-- ═══════════════════════════════════════════════════════════
-- CONFIGURATION RÉPLICATION MySQL 5.7/8.0 → MariaDB 11.8
-- ═══════════════════════════════════════════════════════════

-- SUR MYSQL (SOURCE)
-- -------------------

-- 1. Vérifier que le binlog est activé
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';  -- Doit être ROW ou MIXED

-- 2. Créer l'utilisateur de réplication
CREATE USER 'repl_mariadb'@'%' IDENTIFIED BY 'SecureRepl!Pass123';
GRANT REPLICATION SLAVE ON *.* TO 'repl_mariadb'@'%';
FLUSH PRIVILEGES;

-- 3. Obtenir la position du binlog
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
-- Noter: File et Position
-- Exemple: mysql-bin.000042, 12345678

-- 4. Faire le dump initial (dans un autre terminal)
-- mysqldump --all-databases --single-transaction --master-data=2 \
--   --routines --triggers --events > full_dump.sql

-- 5. Libérer le lock
UNLOCK TABLES;
```

```sql
-- SUR MARIADB (CIBLE)
-- --------------------

-- 1. Importer le dump initial
-- mariadb < full_dump.sql

-- 2. Configurer la réplication
CHANGE MASTER TO
    MASTER_HOST = 'mysql-source.example.com',
    MASTER_PORT = 3306,
    MASTER_USER = 'repl_mariadb',
    MASTER_PASSWORD = 'SecureRepl!Pass123',
    MASTER_LOG_FILE = 'mysql-bin.000042',
    MASTER_LOG_POS = 12345678,
    MASTER_SSL = 1;  -- TLS recommandé

-- 3. Démarrer la réplication
START SLAVE;

-- 4. Vérifier le statut
SHOW SLAVE STATUS\G

-- Points à vérifier :
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0 (ou proche de 0)
```

### Script de monitoring de la réplication

```bash
#!/bin/bash
# monitor_replication.sh
# Monitoring de la réplication pendant la migration

MARIADB_HOST="${1:-localhost}"
MARIADB_USER="${2:-root}"
MARIADB_PASS="${3:-}"
ALERT_LAG_SECONDS=60
CHECK_INTERVAL=5

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

check_replication() {
    local status=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e "
        SHOW SLAVE STATUS\G" 2>/dev/null)
    
    local io_running=$(echo "$status" | grep "Slave_IO_Running:" | awk '{print $2}')
    local sql_running=$(echo "$status" | grep "Slave_SQL_Running:" | awk '{print $2}')
    local lag=$(echo "$status" | grep "Seconds_Behind_Master:" | awk '{print $2}')
    local last_error=$(echo "$status" | grep "Last_Error:" | cut -d: -f2-)
    
    # Statut
    if [ "$io_running" != "Yes" ] || [ "$sql_running" != "Yes" ]; then
        log "🔴 CRITICAL: Réplication arrêtée!"
        log "   IO Running: $io_running"
        log "   SQL Running: $sql_running"
        log "   Erreur: $last_error"
        return 2
    fi
    
    # Lag
    if [ "$lag" == "NULL" ]; then
        log "⚠️ WARNING: Lag indéterminé"
        return 1
    elif [ "$lag" -gt "$ALERT_LAG_SECONDS" ]; then
        log "⚠️ WARNING: Lag élevé: ${lag}s"
        return 1
    else
        log "✅ OK: Lag: ${lag}s"
        return 0
    fi
}

# Boucle de monitoring
log "═══════════════════════════════════════════════════════════"
log "   MONITORING RÉPLICATION"
log "═══════════════════════════════════════════════════════════"
log "Host: $MARIADB_HOST"
log "Seuil d'alerte: ${ALERT_LAG_SECONDS}s"
log "Intervalle: ${CHECK_INTERVAL}s"
log "───────────────────────────────────────────────────────────"

while true; do
    check_replication
    sleep $CHECK_INTERVAL
done
```

---

## Outils de migration online

### pt-online-schema-change (Percona Toolkit)

`pt-online-schema-change` permet de modifier le schéma sans bloquer les écritures.

```bash
#!/bin/bash
# Utilisation de pt-online-schema-change

# Installation
# apt install percona-toolkit  # Debian/Ubuntu
# dnf install percona-toolkit  # RHEL/CentOS

# Exemple : Ajouter une colonne sans downtime
pt-online-schema-change \
    --alter "ADD COLUMN status VARCHAR(50) DEFAULT 'active'" \
    --host=localhost \
    --user=root \
    --password=secret \
    --execute \
    --progress=time,30 \
    --max-lag=10 \
    --check-slave-lag=mariadb-replica \
    --chunk-size=1000 \
    --critical-load="Threads_running=100" \
    D=mydb,t=large_table

# Options importantes :
# --execute          : Exécuter (sans = dry-run)
# --max-lag          : Pause si lag réplication > N secondes
# --chunk-size       : Nombre de lignes par batch
# --critical-load    : Arrêt si charge critique atteinte
# --progress         : Affichage progression
```

**Fonctionnement interne :**

```
Fonctionnement pt-online-schema-change
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Création table temporaire avec nouveau schéma
   original_table ──▶ _original_table_new (avec ALTER)

2. Création des triggers pour synchronisation
   INSERT → INSERT dans _new
   UPDATE → UPDATE dans _new  
   DELETE → DELETE dans _new

3. Copie des données par chunks
   ┌─────────────────────────────────────────────────────┐
   │ SELECT * FROM original_table                        │
   │ WHERE id BETWEEN 1 AND 1000                         │
   │ INSERT INTO _original_table_new ...                 │
   │                                                     │
   │ ... répété par chunks jusqu'à la fin                │
   └─────────────────────────────────────────────────────┘

4. Échange atomique des tables
   RENAME TABLE original_table TO _original_table_old,
                _original_table_new TO original_table;

5. Suppression de l'ancienne table
   DROP TABLE _original_table_old;
```

### gh-ost (GitHub Online Schema Transformation)

`gh-ost` est une alternative à pt-osc, sans triggers, utilisant le binlog.

```bash
#!/bin/bash
# Utilisation de gh-ost

# Installation
# wget https://github.com/github/gh-ost/releases/latest/download/gh-ost-binary-linux-amd64.tar.gz
# tar xzf gh-ost-binary-linux-amd64.tar.gz
# mv gh-ost /usr/local/bin/

# Exemple : Migration de colonne
gh-ost \
    --host=localhost \
    --port=3306 \
    --user=root \
    --password=secret \
    --database=mydb \
    --table=large_table \
    --alter="ADD COLUMN new_field INT DEFAULT 0" \
    --execute \
    --allow-on-master \
    --chunk-size=1000 \
    --max-lag-millis=1500 \
    --throttle-control-replicas="mariadb-replica:3306" \
    --panic-flag-file=/tmp/gh-ost-panic \
    --postpone-cut-over-flag-file=/tmp/gh-ost-postpone

# Options clés :
# --execute              : Exécuter (sinon dry-run)
# --allow-on-master      : Exécuter sur le master directement
# --max-lag-millis       : Seuil de lag en ms
# --panic-flag-file      : Créer ce fichier pour arrêter immédiatement
# --postpone-cut-over-flag-file : Bloquer le cutover tant que le fichier existe
```

**Avantages de gh-ost vs pt-osc :**

| Aspect | pt-online-schema-change | gh-ost |
|--------|------------------------|--------|
| **Méthode** | Triggers | Binlog |
| **Impact sur source** | Modéré (triggers) | Faible |
| **Contrôle cutover** | Automatique | Manuel possible |
| **Pause/Reprise** | Limitée | Complète |
| **Rollback** | Difficile | Plus facile |
| **Complexité** | Modérée | Plus élevée |

### Script de migration online avec gh-ost

```bash
#!/bin/bash
# zero_downtime_alter.sh
# Migration de schéma zero-downtime avec gh-ost

set -e

# Configuration
DB_HOST="localhost"
DB_PORT="3306"
DB_USER="root"
DB_PASS="secret"
DATABASE="mydb"
TABLE="$1"
ALTER="$2"

PANIC_FILE="/tmp/gh-ost-panic-${TABLE}"
POSTPONE_FILE="/tmp/gh-ost-postpone-${TABLE}"
LOG_DIR="/var/log/gh-ost"

# Validation
if [ -z "$TABLE" ] || [ -z "$ALTER" ]; then
    echo "Usage: $0 <table> <alter_statement>"
    echo "Example: $0 users 'ADD COLUMN email_verified BOOLEAN DEFAULT FALSE'"
    exit 1
fi

mkdir -p $LOG_DIR

echo "═══════════════════════════════════════════════════════════"
echo "   MIGRATION ZERO-DOWNTIME AVEC GH-OST"
echo "═══════════════════════════════════════════════════════════"
echo "Table: ${DATABASE}.${TABLE}"
echo "Alter: $ALTER"
echo ""

# Nettoyage des fichiers de contrôle précédents
rm -f $PANIC_FILE $POSTPONE_FILE

# Création du fichier de postpone pour contrôle manuel du cutover
touch $POSTPONE_FILE
echo "📌 Fichier de postpone créé: $POSTPONE_FILE"
echo "   Supprimer ce fichier pour déclencher le cutover"
echo ""

# Dry-run d'abord
echo "[1/3] Dry-run..."
gh-ost \
    --host=$DB_HOST \
    --port=$DB_PORT \
    --user=$DB_USER \
    --password=$DB_PASS \
    --database=$DATABASE \
    --table=$TABLE \
    --alter="$ALTER" \
    --chunk-size=1000 \
    --max-lag-millis=1500 \
    --verbose 2>&1 | tee $LOG_DIR/gh-ost-dryrun-${TABLE}.log

echo ""
read -p "Dry-run OK. Lancer la migration réelle? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "Migration annulée"
    rm -f $POSTPONE_FILE
    exit 0
fi

# Exécution réelle
echo ""
echo "[2/3] Migration en cours..."
echo "      Pour arrêter d'urgence: touch $PANIC_FILE"
echo "      Pour déclencher le cutover: rm $POSTPONE_FILE"
echo ""

gh-ost \
    --host=$DB_HOST \
    --port=$DB_PORT \
    --user=$DB_USER \
    --password=$DB_PASS \
    --database=$DATABASE \
    --table=$TABLE \
    --alter="$ALTER" \
    --execute \
    --allow-on-master \
    --chunk-size=1000 \
    --max-lag-millis=1500 \
    --panic-flag-file=$PANIC_FILE \
    --postpone-cut-over-flag-file=$POSTPONE_FILE \
    --initially-drop-ghost-table \
    --initially-drop-old-table \
    --verbose 2>&1 | tee $LOG_DIR/gh-ost-exec-${TABLE}.log &

GHOST_PID=$!

echo "gh-ost PID: $GHOST_PID"
echo ""

# Attendre que la copie soit terminée
echo "En attente de la fin de la copie des données..."
echo "Supprimez $POSTPONE_FILE quand prêt pour le cutover"
echo ""

wait $GHOST_PID
EXIT_CODE=$?

echo ""
echo "[3/3] Migration terminée"

if [ $EXIT_CODE -eq 0 ]; then
    echo "✅ Migration réussie!"
else
    echo "❌ Migration échouée (code: $EXIT_CODE)"
    echo "Consulter les logs: $LOG_DIR/gh-ost-exec-${TABLE}.log"
fi

# Nettoyage
rm -f $PANIC_FILE $POSTPONE_FILE

exit $EXIT_CODE
```

---

## Orchestration du cutover

### Procédure de cutover avec ProxySQL

```sql
-- ═══════════════════════════════════════════════════════════
-- ORCHESTRATION CUTOVER ZERO-DOWNTIME AVEC PROXYSQL
-- ═══════════════════════════════════════════════════════════

-- ÉTAT INITIAL
-- MySQL Source : hostgroup 10 (writer), hostgroup 20 (reader)
-- MariaDB Target : hostgroup 30 (replica, reader facultatif)

-- Phase 1: Vérification pré-cutover
-- ---------------------------------

-- Vérifier la santé des serveurs
SELECT * FROM mysql_servers;

-- Vérifier le lag de réplication (doit être 0)
SELECT hostgroup_id, hostname, port, status, 
       (SELECT variable_value FROM stats_mysql_global WHERE variable_name='Queries') as queries
FROM mysql_servers WHERE hostgroup_id IN (10, 30);


-- Phase 2: Préparer le cutover
-- ----------------------------

-- Réduire le max_connections temporairement pour drainer
UPDATE mysql_servers SET max_connections = 1 WHERE hostname = 'mysql-source' AND hostgroup_id = 10;
LOAD MYSQL SERVERS TO RUNTIME;

-- Attendre que les connexions se drainent (quelques secondes)
SELECT * FROM stats_mysql_connection_pool WHERE hostgroup IN (10);


-- Phase 3: Cutover (< 5 secondes)
-- -------------------------------

-- 3.1 Mettre MySQL source offline
UPDATE mysql_servers SET status = 'OFFLINE_SOFT' WHERE hostname = 'mysql-source';
LOAD MYSQL SERVERS TO RUNTIME;

-- 3.2 Attendre la synchronisation finale (lag = 0)
-- (vérifier manuellement ou via script)

-- 3.3 Promouvoir MariaDB
UPDATE mysql_servers SET hostgroup_id = 10, status = 'ONLINE', max_connections = 100 
WHERE hostname = 'mariadb-target' AND hostgroup_id = 30;
LOAD MYSQL SERVERS TO RUNTIME;

-- 3.4 Vérification immédiate
SELECT * FROM mysql_servers WHERE hostgroup_id IN (10, 20);


-- Phase 4: Post-cutover
-- ---------------------

-- Vérifier que le trafic passe bien
SELECT hostgroup, srv_host, Queries, Bytes_data_sent
FROM stats_mysql_connection_pool;

-- Ajuster les connexions
UPDATE mysql_servers SET max_connections = 200 WHERE hostname = 'mariadb-target';
LOAD MYSQL SERVERS TO RUNTIME;

-- Sauvegarder la configuration
SAVE MYSQL SERVERS TO DISK;
```

### Script d'orchestration automatique

```python
#!/usr/bin/env python3
# cutover_orchestrator.py
# Orchestration automatique du cutover zero-downtime

import time
import sys
from datetime import datetime
from typing import Dict, Optional
import mysql.connector

class CutoverOrchestrator:
    """Orchestre le cutover zero-downtime"""
    
    def __init__(self, config: Dict):
        self.source_config = config['source']
        self.target_config = config['target']
        self.proxy_config = config['proxy']
        self.max_lag_seconds = config.get('max_lag_seconds', 5)
        self.drain_timeout_seconds = config.get('drain_timeout', 30)
    
    def log(self, message: str, level: str = "INFO"):
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print(f"[{timestamp}] [{level}] {message}")
    
    def get_replication_lag(self) -> Optional[int]:
        """Obtient le lag de réplication sur la cible"""
        conn = mysql.connector.connect(**self.target_config)
        cursor = conn.cursor(dictionary=True)
        cursor.execute("SHOW SLAVE STATUS")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        
        if result:
            lag = result.get('Seconds_Behind_Master')
            return int(lag) if lag is not None else None
        return None
    
    def wait_for_sync(self, timeout: int = 300) -> bool:
        """Attend que la réplication soit synchronisée"""
        self.log(f"Attente synchronisation (timeout: {timeout}s)...")
        start = time.time()
        
        while time.time() - start < timeout:
            lag = self.get_replication_lag()
            
            if lag is None:
                self.log("Lag indéterminé, réplication peut-être arrêtée", "WARNING")
                return False
            
            if lag <= self.max_lag_seconds:
                self.log(f"✓ Synchronisé (lag: {lag}s)")
                return True
            
            self.log(f"Lag actuel: {lag}s, attente...")
            time.sleep(1)
        
        self.log(f"Timeout synchronisation après {timeout}s", "ERROR")
        return False
    
    def execute_proxy_command(self, command: str):
        """Exécute une commande sur ProxySQL"""
        conn = mysql.connector.connect(**self.proxy_config)
        cursor = conn.cursor()
        cursor.execute(command)
        conn.commit()
        cursor.close()
        conn.close()
    
    def drain_source(self) -> bool:
        """Draine les connexions de la source"""
        self.log("Drainage des connexions source...")
        
        # Réduire les connexions max
        self.execute_proxy_command("""
            UPDATE mysql_servers 
            SET max_connections = 1 
            WHERE hostname = '{}' AND hostgroup_id = 10
        """.format(self.source_config['host']))
        self.execute_proxy_command("LOAD MYSQL SERVERS TO RUNTIME")
        
        # Attendre le drainage
        start = time.time()
        while time.time() - start < self.drain_timeout_seconds:
            # Vérifier les connexions actives
            conn = mysql.connector.connect(**self.proxy_config)
            cursor = conn.cursor(dictionary=True)
            cursor.execute("""
                SELECT ConnUsed FROM stats_mysql_connection_pool 
                WHERE srv_host = '{}'
            """.format(self.source_config['host']))
            result = cursor.fetchone()
            cursor.close()
            conn.close()
            
            if result and result['ConnUsed'] <= 1:
                self.log("✓ Connexions drainées")
                return True
            
            time.sleep(1)
        
        self.log("Timeout drainage", "WARNING")
        return True  # Continuer quand même
    
    def stop_source_writes(self):
        """Arrête les écritures sur la source"""
        self.log("Arrêt des écritures sur la source...")
        
        # Mettre la source offline dans ProxySQL
        self.execute_proxy_command("""
            UPDATE mysql_servers 
            SET status = 'OFFLINE_SOFT' 
            WHERE hostname = '{}'
        """.format(self.source_config['host']))
        self.execute_proxy_command("LOAD MYSQL SERVERS TO RUNTIME")
        
        self.log("✓ Source offline")
    
    def stop_target_replication(self):
        """Arrête la réplication sur la cible"""
        self.log("Arrêt de la réplication sur la cible...")
        
        conn = mysql.connector.connect(**self.target_config)
        cursor = conn.cursor()
        cursor.execute("STOP SLAVE")
        cursor.close()
        conn.close()
        
        self.log("✓ Réplication arrêtée")
    
    def promote_target(self):
        """Promeut la cible en primary"""
        self.log("Promotion de la cible en primary...")
        
        # Activer la cible comme writer dans ProxySQL
        self.execute_proxy_command("""
            UPDATE mysql_servers 
            SET hostgroup_id = 10, status = 'ONLINE', max_connections = 200 
            WHERE hostname = '{}'
        """.format(self.target_config['host']))
        self.execute_proxy_command("LOAD MYSQL SERVERS TO RUNTIME")
        self.execute_proxy_command("SAVE MYSQL SERVERS TO DISK")
        
        self.log("✓ Cible promue")
    
    def verify_cutover(self) -> bool:
        """Vérifie que le cutover est effectif"""
        self.log("Vérification du cutover...")
        
        conn = mysql.connector.connect(**self.proxy_config)
        cursor = conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT srv_host, status, hostgroup 
            FROM stats_mysql_connection_pool 
            WHERE hostgroup = 10
        """)
        results = cursor.fetchall()
        cursor.close()
        conn.close()
        
        for row in results:
            if row['srv_host'] == self.target_config['host']:
                self.log(f"✓ Trafic routé vers {row['srv_host']}")
                return True
        
        self.log("Cutover non vérifié", "ERROR")
        return False
    
    def execute_cutover(self) -> bool:
        """Exécute le cutover complet"""
        self.log("═" * 60)
        self.log("   DÉBUT DU CUTOVER ZERO-DOWNTIME")
        self.log("═" * 60)
        
        cutover_start = time.time()
        
        try:
            # Phase 1: Pré-cutover
            self.log("\n[Phase 1] Pré-cutover")
            if not self.wait_for_sync(timeout=60):
                raise Exception("Synchronisation impossible")
            
            # Phase 2: Drainage
            self.log("\n[Phase 2] Drainage")
            self.drain_source()
            
            # Phase 3: Cutover (critique - mesurer le temps)
            self.log("\n[Phase 3] CUTOVER")
            cutover_moment = time.time()
            
            self.stop_source_writes()
            
            # Attente synchronisation finale
            time.sleep(2)  # Petit délai pour les dernières transactions
            
            if not self.wait_for_sync(timeout=30):
                raise Exception("Synchronisation finale échouée")
            
            self.stop_target_replication()
            self.promote_target()
            
            cutover_duration = time.time() - cutover_moment
            self.log(f"⏱️ Durée du cutover: {cutover_duration:.2f}s")
            
            # Phase 4: Vérification
            self.log("\n[Phase 4] Vérification")
            if not self.verify_cutover():
                raise Exception("Vérification échouée")
            
            total_duration = time.time() - cutover_start
            
            self.log("\n" + "═" * 60)
            self.log(f"   ✅ CUTOVER RÉUSSI EN {total_duration:.2f}s")
            self.log(f"   ⏱️ Downtime effectif: {cutover_duration:.2f}s")
            self.log("═" * 60)
            
            return True
            
        except Exception as e:
            self.log(f"\n❌ ERREUR: {e}", "ERROR")
            self.log("Rollback peut être nécessaire", "ERROR")
            return False


# Utilisation
if __name__ == '__main__':
    config = {
        'source': {
            'host': 'mysql-source',
            'port': 3306,
            'user': 'admin',
            'password': 'password'
        },
        'target': {
            'host': 'mariadb-target',
            'port': 3306,
            'user': 'admin',
            'password': 'password'
        },
        'proxy': {
            'host': 'proxysql',
            'port': 6032,
            'user': 'admin',
            'password': 'admin'
        },
        'max_lag_seconds': 5,
        'drain_timeout': 30
    }
    
    orchestrator = CutoverOrchestrator(config)
    success = orchestrator.execute_cutover()
    sys.exit(0 if success else 1)
```

---

## Gestion du split-brain

### Comprendre le split-brain

```
Situation de Split-Brain
━━━━━━━━━━━━━━━━━━━━━━━━

ÉTAT NORMAL
                    ┌─────────────────────┐
                    │    Load Balancer    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Primary (seul)   │
                    │    Accepte writes   │
                    └─────────────────────┘

SPLIT-BRAIN (DANGER!)
                    ┌─────────────────────┐
                    │    Applications     │
                    └───────┬─────┬───────┘
                            │     │
                    ┌───────┘     └───────┐
                    │                     │
                    ▼                     ▼
         ┌─────────────────┐   ┌─────────────────┐
         │   Primary A     │   │   Primary B     │
         │ (se croit seul) │   │ (se croit seul) │
         │                 │   │                 │
         │ Write: id=1     │   │ Write: id=1     │
         │ data='AAA'      │   │ data='BBB'      │
         └─────────────────┘   └─────────────────┘
         
         ⚠️ CONFLIT! Même ID, données différentes
         ⚠️ Données divergentes, réconciliation complexe
```

### Prévention du split-brain

```sql
-- ═══════════════════════════════════════════════════════════
-- MÉCANISMES DE PRÉVENTION DU SPLIT-BRAIN
-- ═══════════════════════════════════════════════════════════

-- 1. FENCING : Garantir qu'un seul primary peut écrire
-- --------------------------------------------------

-- Utiliser super_read_only sur tous les serveurs par défaut
-- Seul le primary actif a super_read_only = OFF

-- Sur tous les serveurs (configuration par défaut)
SET GLOBAL super_read_only = ON;
SET GLOBAL read_only = ON;

-- Sur le primary uniquement (après élection)
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;


-- 2. QUORUM : Décision basée sur la majorité
-- -----------------------------------------
-- Utiliser Galera Cluster ou Group Replication pour
-- garantir qu'un primary n'est élu que s'il a la majorité


-- 3. STONITH (Shoot The Other Node In The Head)
-- ---------------------------------------------
-- Si deux primaries sont détectés, un mécanisme externe
-- (Pacemaker, script) force l'arrêt de l'un d'eux
```

### Script de détection et résolution

```python
#!/usr/bin/env python3
# splitbrain_detector.py
# Détection et résolution des situations de split-brain

import mysql.connector
from typing import Dict, List, Optional
from datetime import datetime
import subprocess
import sys

class SplitBrainDetector:
    """Détecte et résout les situations de split-brain"""
    
    def __init__(self, servers: List[Dict]):
        self.servers = servers
        self.alerts = []
    
    def log(self, message: str, level: str = "INFO"):
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print(f"[{timestamp}] [{level}] {message}")
    
    def check_server_mode(self, server: Dict) -> Dict:
        """Vérifie le mode d'un serveur (read_only, super_read_only)"""
        try:
            conn = mysql.connector.connect(
                host=server['host'],
                port=server.get('port', 3306),
                user=server['user'],
                password=server['password']
            )
            cursor = conn.cursor(dictionary=True)
            
            cursor.execute("SELECT @@read_only as ro, @@super_read_only as sro, @@server_id as id")
            result = cursor.fetchone()
            
            cursor.close()
            conn.close()
            
            return {
                'host': server['host'],
                'reachable': True,
                'read_only': bool(result['ro']),
                'super_read_only': bool(result['sro']),
                'server_id': result['id'],
                'is_writable': not bool(result['ro']) and not bool(result['sro'])
            }
            
        except Exception as e:
            return {
                'host': server['host'],
                'reachable': False,
                'error': str(e)
            }
    
    def detect_split_brain(self) -> bool:
        """Détecte une situation de split-brain"""
        self.log("Vérification des serveurs...")
        
        writable_servers = []
        
        for server in self.servers:
            status = self.check_server_mode(server)
            
            if not status['reachable']:
                self.log(f"⚠️ {status['host']}: Non joignable - {status.get('error', 'Unknown')}", "WARNING")
                continue
            
            mode = "WRITABLE" if status['is_writable'] else "READ-ONLY"
            self.log(f"  {status['host']}: {mode}")
            
            if status['is_writable']:
                writable_servers.append(status)
        
        if len(writable_servers) > 1:
            self.log("", "")
            self.log("🚨 SPLIT-BRAIN DÉTECTÉ!", "CRITICAL")
            self.log(f"   {len(writable_servers)} serveurs en mode WRITABLE:", "CRITICAL")
            for srv in writable_servers:
                self.log(f"   - {srv['host']} (server_id: {srv['server_id']})", "CRITICAL")
            return True
        
        if len(writable_servers) == 0:
            self.log("⚠️ Aucun serveur writable détecté!", "WARNING")
        
        return False
    
    def resolve_split_brain(self, keep_server: str):
        """Résout le split-brain en gardant un seul primary"""
        self.log(f"Résolution: Conservation de {keep_server} comme primary")
        
        for server in self.servers:
            status = self.check_server_mode(server)
            
            if not status['reachable']:
                continue
            
            if server['host'] == keep_server:
                # Ce serveur reste le primary
                if not status['is_writable']:
                    self.log(f"Activation des écritures sur {server['host']}")
                    self._set_writable(server, True)
            else:
                # Les autres passent en read-only
                if status['is_writable']:
                    self.log(f"Passage en read-only de {server['host']}")
                    self._set_writable(server, False)
        
        self.log("✓ Split-brain résolu")
    
    def _set_writable(self, server: Dict, writable: bool):
        """Configure le mode writable d'un serveur"""
        conn = mysql.connector.connect(
            host=server['host'],
            port=server.get('port', 3306),
            user=server['user'],
            password=server['password']
        )
        cursor = conn.cursor()
        
        if writable:
            cursor.execute("SET GLOBAL super_read_only = OFF")
            cursor.execute("SET GLOBAL read_only = OFF")
        else:
            cursor.execute("SET GLOBAL read_only = ON")
            cursor.execute("SET GLOBAL super_read_only = ON")
        
        cursor.close()
        conn.close()
    
    def fence_server(self, server: Dict):
        """STONITH - Arrêt forcé d'un serveur"""
        self.log(f"🔫 STONITH: Arrêt forcé de {server['host']}", "WARNING")
        
        # Option 1: Arrêt via SSH
        try:
            subprocess.run(
                ["ssh", server['host'], "systemctl", "stop", "mariadb"],
                timeout=10,
                check=True
            )
            self.log(f"✓ {server['host']} arrêté via SSH")
        except Exception as e:
            self.log(f"Échec SSH: {e}", "ERROR")
            
            # Option 2: Arrêt via IPMI/iLO
            # subprocess.run(["ipmitool", "-H", server['ipmi'], "power", "off"])
            
            # Option 3: Désactivation réseau
            # self._disable_network(server)
    
    def monitor_loop(self, interval: int = 10):
        """Boucle de monitoring continu"""
        self.log("Démarrage du monitoring split-brain")
        
        import time
        while True:
            if self.detect_split_brain():
                # Alerte et attente d'intervention
                self.log("Intervention manuelle requise!", "CRITICAL")
                # En production: envoyer alerte PagerDuty, etc.
            
            time.sleep(interval)


# Utilisation
if __name__ == '__main__':
    servers = [
        {'host': 'mariadb-1', 'port': 3306, 'user': 'admin', 'password': 'password'},
        {'host': 'mariadb-2', 'port': 3306, 'user': 'admin', 'password': 'password'},
        {'host': 'mariadb-3', 'port': 3306, 'user': 'admin', 'password': 'password'},
    ]
    
    detector = SplitBrainDetector(servers)
    
    if len(sys.argv) > 1 and sys.argv[1] == '--monitor':
        detector.monitor_loop()
    else:
        if detector.detect_split_brain():
            print("\nPour résoudre, exécuter:")
            print(f"  python {sys.argv[0]} --resolve <hostname_a_garder>")
        
        if len(sys.argv) > 2 and sys.argv[1] == '--resolve':
            detector.resolve_split_brain(sys.argv[2])
```

---

## Spécificités MariaDB 11.8 🆕

### Migration des System-Versioned Tables

MariaDB 11.8 modifie le format de stockage des timestamps dans les tables temporelles. Une migration zero-downtime de ces tables nécessite une attention particulière.

```sql
-- ═══════════════════════════════════════════════════════════
-- MIGRATION ZERO-DOWNTIME DES SYSTEM-VERSIONED TABLES
-- ═══════════════════════════════════════════════════════════

-- Identification des tables system-versioned
SELECT 
    table_schema,
    table_name,
    create_options
FROM information_schema.tables
WHERE create_options LIKE '%VERSIONED%'
ORDER BY table_schema, table_name;

-- Exemple de sortie:
-- | app_db | contracts    | row_format=DYNAMIC with system versioning |
-- | app_db | audit_events | row_format=DYNAMIC with system versioning |

-- APPROCHE 1: Migration pendant la réplication
-- --------------------------------------------
-- Les tables sont automatiquement converties lors de la réplication
-- si la cible est en 11.8. Vérifier après migration:

SHOW CREATE TABLE contracts\G
-- Vérifier que ROW_START et ROW_END utilisent le nouveau format

-- APPROCHE 2: Rebuild explicite post-migration
-- --------------------------------------------
-- Si nécessaire, reconstruire les tables pour le nouveau format

-- Option A: ALTER TABLE simple (prend un lock)
ALTER TABLE contracts ENGINE=InnoDB;

-- Option B: pt-online-schema-change (zero-downtime)
-- pt-online-schema-change --alter "ENGINE=InnoDB" D=app_db,t=contracts --execute

-- Option C: gh-ost
-- gh-ost --alter "ENGINE=InnoDB" --database=app_db --table=contracts --execute
```

### Gestion du TLS par défaut

```bash
#!/bin/bash
# prepare_tls_migration.sh
# Prépare la migration vers MariaDB 11.8 avec TLS par défaut

# Vérifier les connexions actuelles sans TLS
mariadb -e "
SELECT 
    user, 
    host, 
    ssl_type,
    CASE WHEN ssl_type = '' THEN 'NO TLS' ELSE 'TLS' END as connection_security
FROM mysql.user 
WHERE user NOT IN ('mariadb.sys', 'mysql')
ORDER BY ssl_type, user;
"

# Générer des certificats si nécessaire
if [ ! -f /etc/mysql/ssl/server-cert.pem ]; then
    echo "Génération des certificats TLS..."
    
    mkdir -p /etc/mysql/ssl
    cd /etc/mysql/ssl
    
    # CA
    openssl genrsa 2048 > ca-key.pem
    openssl req -new -x509 -nodes -days 3650 -key ca-key.pem -out ca-cert.pem \
        -subj "/CN=MariaDB CA"
    
    # Server
    openssl req -newkey rsa:2048 -nodes -keyout server-key.pem -out server-req.pem \
        -subj "/CN=MariaDB Server"
    openssl x509 -req -in server-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem \
        -set_serial 01 -out server-cert.pem
    
    # Client
    openssl req -newkey rsa:2048 -nodes -keyout client-key.pem -out client-req.pem \
        -subj "/CN=MariaDB Client"
    openssl x509 -req -in client-req.pem -days 3650 -CA ca-cert.pem -CAkey ca-key.pem \
        -set_serial 02 -out client-cert.pem
    
    chown mysql:mysql *.pem
    chmod 600 *-key.pem
    
    echo "✓ Certificats générés"
fi

# Configuration MariaDB pour TLS
cat >> /etc/mysql/mariadb.conf.d/99-tls.cnf << 'EOF'
[mariadbd]
ssl_cert = /etc/mysql/ssl/server-cert.pem
ssl_key = /etc/mysql/ssl/server-key.pem
ssl_ca = /etc/mysql/ssl/ca-cert.pem

# Pour 11.8, TLS est activé par défaut
# Désactiver temporairement si nécessaire pendant la migration:
# require_secure_transport = OFF
EOF

echo "Configuration TLS ajoutée"
```

---

## Scénarios réels

### Scénario 1 : E-commerce haute disponibilité

**Contexte :** Site e-commerce avec 50 000 utilisateurs actifs, 500 transactions/minute, SLA 99.95%.

```
Architecture E-commerce
━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────┐
│                     CDN + WAF                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                   Load Balancer (HAProxy)                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
       ┌──────┴──────┐             ┌──────┴──────┐
       │   App x3    │             │   App x3    │
       │  (Zone A)   │             │  (Zone B)   │
       └──────┬──────┘             └──────┬──────┘
              │                           │
              └─────────────┬─────────────┘
                            │
              ┌─────────────┴─────────────┐
              │         ProxySQL          │
              └─────────────┬─────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  MySQL Primary  │ │ MySQL Replica 1 │ │ MySQL Replica 2 │
│   (Zone A)      │ │   (Zone B)      │ │   (Zone B)      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

**Stratégie de migration :**

```yaml
# Plan de migration E-commerce
phases:
  1_preparation:
    duration: "2 semaines"
    tasks:
      - Déployer MariaDB 11.8 en replica
      - Configurer la réplication depuis MySQL
      - Tests de charge sur replica
      - Formation équipes
  
  2_canary:
    duration: "1 semaine"
    tasks:
      - Router 5% du trafic lecture vers MariaDB
      - Monitoring intensif
      - Validation métriques (latence, erreurs)
  
  3_progressive:
    duration: "1 semaine"
    tasks:
      - Augmenter progressivement: 10%, 25%, 50%
      - Validation à chaque étape
  
  4_cutover:
    duration: "30 minutes (fenêtre)"
    tasks:
      - Cutover des écritures vers MariaDB
      - Downtime cible: < 10 secondes
    
    cutover_procedure:
      - "00:00 - Annonce maintenance"
      - "00:05 - Drainage connexions MySQL"
      - "00:06 - SET read_only=ON sur MySQL"
      - "00:07 - Attente sync (lag=0)"
      - "00:08 - STOP SLAVE sur MariaDB"
      - "00:08 - Promotion MariaDB via ProxySQL"
      - "00:09 - Vérification trafic"
      - "00:10 - Fin cutover"
  
  5_validation:
    duration: "24 heures"
    tasks:
      - Monitoring 24/7
      - MySQL en standby pour rollback
      - Validation business

rollback_trigger:
  - "Erreur rate > 1%"
  - "Latence P95 > 500ms"
  - "Transactions échouées > 10/min"
```

**Résultat obtenu :**
- Downtime effectif : 7 secondes
- Aucune transaction perdue
- Performance améliorée de 15% (optimizer MariaDB)

### Scénario 2 : SaaS multi-tenant

**Contexte :** Application SaaS B2B, 200 tenants, base de 2 TB, migrations de schéma fréquentes.

```python
#!/usr/bin/env python3
# saas_migration_orchestrator.py
# Migration tenant par tenant pour SaaS

from typing import List, Dict
import time

class SaaSMigrationOrchestrator:
    """Orchestre la migration tenant par tenant"""
    
    def __init__(self, tenants: List[Dict], source_config: Dict, target_config: Dict):
        self.tenants = tenants
        self.source = source_config
        self.target = target_config
        self.migrated = []
        self.failed = []
    
    def migrate_tenant(self, tenant: Dict) -> bool:
        """Migre un tenant individuel"""
        tenant_id = tenant['id']
        database = tenant['database']
        
        print(f"\n[Tenant {tenant_id}] Début migration de {database}")
        
        try:
            # 1. Créer la réplication pour ce tenant
            self._setup_tenant_replication(tenant)
            
            # 2. Attendre synchronisation
            self._wait_tenant_sync(tenant)
            
            # 3. Cutover du tenant (< 5s)
            self._cutover_tenant(tenant)
            
            # 4. Vérification
            if self._verify_tenant(tenant):
                self.migrated.append(tenant)
                print(f"[Tenant {tenant_id}] ✓ Migration réussie")
                return True
            else:
                raise Exception("Vérification échouée")
                
        except Exception as e:
            print(f"[Tenant {tenant_id}] ✗ Erreur: {e}")
            self._rollback_tenant(tenant)
            self.failed.append({'tenant': tenant, 'error': str(e)})
            return False
    
    def migrate_all(self, batch_size: int = 5, pause_between_batches: int = 60):
        """Migre tous les tenants par batches"""
        
        print("=" * 60)
        print(f"Migration de {len(self.tenants)} tenants")
        print(f"Batch size: {batch_size}")
        print("=" * 60)
        
        for i in range(0, len(self.tenants), batch_size):
            batch = self.tenants[i:i+batch_size]
            print(f"\n--- Batch {i//batch_size + 1} ({len(batch)} tenants) ---")
            
            for tenant in batch:
                self.migrate_tenant(tenant)
            
            # Pause entre batches pour observation
            if i + batch_size < len(self.tenants):
                print(f"\nPause de {pause_between_batches}s avant prochain batch...")
                time.sleep(pause_between_batches)
        
        # Rapport final
        print("\n" + "=" * 60)
        print("RAPPORT DE MIGRATION")
        print("=" * 60)
        print(f"Total tenants: {len(self.tenants)}")
        print(f"Migrés avec succès: {len(self.migrated)}")
        print(f"Échecs: {len(self.failed)}")
        
        if self.failed:
            print("\nTenants en échec:")
            for f in self.failed:
                print(f"  - {f['tenant']['id']}: {f['error']}")
    
    def _setup_tenant_replication(self, tenant: Dict):
        """Configure la réplication pour un tenant"""
        # Implémentation spécifique à l'architecture
        pass
    
    def _wait_tenant_sync(self, tenant: Dict):
        """Attend la synchronisation du tenant"""
        pass
    
    def _cutover_tenant(self, tenant: Dict):
        """Cutover du tenant vers MariaDB"""
        pass
    
    def _verify_tenant(self, tenant: Dict) -> bool:
        """Vérifie la migration du tenant"""
        return True
    
    def _rollback_tenant(self, tenant: Dict):
        """Rollback d'un tenant en cas d'erreur"""
        pass


# Utilisation
if __name__ == '__main__':
    tenants = [
        {'id': 'tenant_001', 'database': 'saas_tenant_001', 'tier': 'enterprise'},
        {'id': 'tenant_002', 'database': 'saas_tenant_002', 'tier': 'standard'},
        # ... 200 tenants
    ]
    
    # Trier par importance (enterprise en dernier pour plus de tests)
    tenants.sort(key=lambda t: 0 if t['tier'] == 'standard' else 1)
    
    orchestrator = SaaSMigrationOrchestrator(
        tenants=tenants,
        source_config={'host': 'mysql-primary'},
        target_config={'host': 'mariadb-primary'}
    )
    
    orchestrator.migrate_all(batch_size=10, pause_between_batches=120)
```

---

## Checklist migration zero-downtime

```markdown
## CHECKLIST MIGRATION ZERO-DOWNTIME

### Pré-requis
- [ ] Architecture compatible (proxy, réplication)
- [ ] MariaDB 11.8 installé et configuré sur cible
- [ ] Réplication source → cible fonctionnelle
- [ ] Tests de performance validés sur cible
- [ ] Plan de rollback documenté et testé
- [ ] Équipe formée et disponible
- [ ] Communication planifiée

### J-1
- [ ] Vérification finale de la réplication (lag = 0)
- [ ] Test du script de cutover en staging
- [ ] Backup complet de la source
- [ ] Notification aux équipes
- [ ] Vérification des alertes et monitoring

### Jour J - Pré-cutover
- [ ] Réunion kick-off avec toutes les équipes
- [ ] Vérification santé source et cible
- [ ] Lag de réplication < 1 seconde
- [ ] Connexions ProxySQL nominales
- [ ] War room ouvert

### Cutover
- [ ] Annonce début cutover
- [ ] Drainage des connexions source
- [ ] Activation read_only sur source
- [ ] Attente synchronisation finale
- [ ] STOP SLAVE sur cible
- [ ] Promotion cible via proxy
- [ ] Vérification trafic routé
- [ ] Chrono du downtime effectif

### Post-cutover immédiat
- [ ] Vérification applications connectées
- [ ] Métriques nominales (latence, erreurs)
- [ ] Test transactions critiques
- [ ] Communication "cutover réussi"

### H+1 à H+24
- [ ] Monitoring renforcé
- [ ] Source disponible pour rollback
- [ ] Analyse des logs d'erreur
- [ ] Validation équipes métier

### Post-migration
- [ ] Documentation mise à jour
- [ ] Désactivation source (après période de sécurité)
- [ ] Retour d'expérience (post-mortem)
- [ ] Mise à jour des runbooks
```

---

## ✅ Points clés à retenir

- **Zero-downtime** signifie généralement < 30 secondes d'interruption, pas zéro absolu
- La **réplication** est le fondement de toute migration zero-downtime vers MariaDB
- **ProxySQL** ou équivalent est essentiel pour orchestrer le cutover instantanément
- **pt-online-schema-change** et **gh-ost** permettent les modifications de schéma sans lock
- Le **split-brain** est le risque majeur à prévenir avec des mécanismes de fencing
- MariaDB 11.8 nécessite une attention particulière pour les **System-Versioned Tables** et le **TLS par défaut**
- Toujours avoir un **plan de rollback testé** même pour les migrations zero-downtime
- Le **monitoring continu** pendant et après la migration est critique

---

## 🔗 Ressources et références

- [📖 ProxySQL Documentation](https://proxysql.com/documentation/)
- [📖 pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [📖 gh-ost Documentation](https://github.com/github/gh-ost)
- [📖 MariaDB Replication](https://mariadb.com/kb/en/replication/)
- [📖 MariaDB Galera Cluster](https://mariadb.com/kb/en/galera-cluster/)
- [📖 Blue-Green Deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html)

---

## ➡️ Section suivante

**[19.9 Migration des System-Versioned Tables](./09-migration-system-versioned-tables.md)** : Nous approfondirons la migration des tables temporelles vers MariaDB 11.8, incluant le nouveau format de timestamp, les stratégies de conversion, et la préservation de l'historique.

⏭️ [Migration System-Versioned Tables (changement format timestamp 11.8)](/19-migration-compatibilite/09-migration-system-versioned-tables.md)
