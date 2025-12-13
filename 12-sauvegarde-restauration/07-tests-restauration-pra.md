🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7 Tests de restauration et plan de reprise (PRA)

> **Niveau** : Avancé  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : Sections 12.1-12.6, Gestion de crise, Compréhension des RTO/RPO

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Concevoir** une méthodologie de tests de restauration exhaustive
- **Planifier** des exercices de simulation (tabletop et live)
- **Documenter** un Plan de Reprise d'Activité (PRA) complet
- **Définir** des scénarios d'incident et arbres de décision
- **Organiser** la communication de crise entre équipes
- **Mesurer** et améliorer les temps de restauration (RTO)
- **Valider** régulièrement la viabilité des procédures
- **Former** les équipes aux situations d'urgence

---

## Introduction

Le test de restauration est **la seule preuve** qu'un backup est exploitable. Sans tests réguliers, une organisation opère avec une **fausse impression de sécurité**.

### La dure réalité des backups non testés

```
┌──────────────────────────────────────────────────────┐
│         Découverte lors d'un incident réel           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Vendredi 15h : Panne disque serveur production      │
│  Vendredi 15h30: Début restauration depuis backup    │
│  Vendredi 17h : Échec restauration                   │
│                                                      │
│  Causes découvertes :                                │
│  ├─ Binary logs manquants (rotation incorrecte)      │
│  ├─ Backup compressé corrompu (jamais testé)         │
│  ├─ Procédure obsolète (MySQL→MariaDB migration)     │
│  └─ Serveur restauration sous-dimensionné            │
│                                                      │
│  Résultat :                                          │
│  ├─ Downtime : 36 heures                             │
│  ├─ Perte données : 3 jours                          │
│  ├─ Coût : 2M€                                       │
│  └─ Réputation : Gravement affectée                  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

💡 **Principe fondamental** : *"Faire confiance à un backup non testé, c'est jouer à la roulette russe avec ses données"*.

### Statistiques édifiantes

D'après Gartner et Veeam (2023-2024) :

```
Organisations testant leurs restaurations :
├─ Jamais                : 34% ⚠️
├─ Annuellement          : 31%
├─ Trimestriellement     : 23%
└─ Mensuellement ou plus : 12%

Causes d'échec de restauration lors de tests :
├─ Procédure incorrecte/obsolète    : 42%
├─ Fichiers manquants/corrompus     : 28%
├─ Infrastructure inadéquate        : 18%
└─ Erreur humaine (stress)          : 12%

Impact business d'un échec restauration :
├─ < 1h downtime   : 100K€ - 500K€
├─ 1-4h downtime   : 500K€ - 2M€
├─ 4-24h downtime  : 2M€ - 10M€
└─ > 24h downtime  : 10M€+ + fermeture possible
```

---

## Méthodologie de tests de restauration

### Niveaux de tests

Les tests de restauration suivent une approche **pyramidale** :

```
┌─────────────────────────────────────────────────┐
│          Pyramide des tests de restauration     │
├─────────────────────────────────────────────────┤
│                     ╱▲                          │
│                    ╱  ╲                         │
│                   ╱    ╲                        │
│                  ╱      ╲   Niveau 4            │
│                 ╱────────╲  Disaster Recovery   │
│                ╱          ╲ (Annuel)            │
│               ╱────────────╲                    │
│              ╱              ╲ Niveau 3          │
│             ╱   Full DR      ╲ PITR Complet     │
│            ╱──────────────────╲ (Trimestriel)   │
│           ╱                    ╲                │
│          ╱  Niveau 2            ╲               │
│         ╱  Restauration Partielle╲              │
│        ╱──────────────────────────╲ (Mensuel)   │
│       ╱                            ╲            │
│      ╱     Niveau 1                 ╲           │
│     ╱   Validation Intégrité         ╲          │
│    ╱──────────────────────────────────╲         │
│   ╱         (Quotidien/Hebdo)          ╲        │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### Niveau 1 : Validation d'intégrité (Quotidien/Hebdomadaire)

**Objectif** : Vérifier que les backups sont cohérents et non corrompus.

```bash
#!/bin/bash
# level1_integrity_check.sh

BACKUP_DIR="/backups/mariadb/full/latest"
LOG_FILE="/var/log/backup_validation.log"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Test 1: Fichiers présents
log "Test 1: Checking files presence..."
required_files=(
  "xtrabackup_checkpoints"
  "xtrabackup_info"
  "xtrabackup_binlog_info"
)

for file in "${required_files[@]}"; do
  if [[ ! -f "$BACKUP_DIR/$file" ]]; then
    log "ERROR: Missing required file: $file"
    exit 1
  fi
done
log "✅ All required files present"

# Test 2: Checksum validation
log "Test 2: Validating checksums..."
if [[ -f "$BACKUP_DIR.sha256" ]]; then
  cd "$(dirname "$BACKUP_DIR")" || exit 1
  if sha256sum -c "$(basename "$BACKUP_DIR").sha256"; then
    log "✅ Checksums valid"
  else
    log "ERROR: Checksum validation failed"
    exit 1
  fi
else
  log "WARNING: No checksum file found"
fi

# Test 3: Compressed files integrity
log "Test 3: Testing compressed files..."
find "$BACKUP_DIR" -name "*.gz" -exec gzip -t {} \; || {
  log "ERROR: Corrupted compressed file found"
  exit 1
}
log "✅ All compressed files valid"

# Test 4: Metadata validation
log "Test 4: Checking metadata..."
if grep -q "backup_type = full-backuped" "$BACKUP_DIR/xtrabackup_checkpoints"; then
  log "✅ Metadata valid"
else
  log "ERROR: Invalid backup metadata"
  exit 1
fi

log "========================================="
log "Level 1 validation PASSED"
log "========================================="
```

#### Niveau 2 : Restauration partielle (Mensuelle)

**Objectif** : Restaurer une seule base ou table pour valider le processus.

```bash
#!/bin/bash
# level2_partial_restore.sh

BACKUP_FILE="/backups/mariadb/logical/latest.sql.gz"
TEST_DB="restore_test_$(date +%s)"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a /var/log/restore_test.log
}

log "========================================="
log "Level 2: Partial Restore Test"
log "========================================="

# Docker-based isolated test
log "Creating isolated test environment..."
docker run -d --name restore_test \
  -e MARIADB_ROOT_PASSWORD=test \
  mariadb:11.8

sleep 15

# Extract and restore single database
log "Extracting test database..."
zcat "$BACKUP_FILE" | \
  sed -n "/^-- Current Database: \`myapp\`/,/^-- Current Database:/p" | \
  docker exec -i restore_test mariadb -ptest

# Validation queries
log "Running validation queries..."
USERS_COUNT=$(docker exec restore_test mariadb -ptest myapp -Nse "SELECT COUNT(*) FROM users;" 2>/dev/null || echo "0")
ORDERS_COUNT=$(docker exec restore_test mariadb -ptest myapp -Nse "SELECT COUNT(*) FROM orders;" 2>/dev/null || echo "0")

log "Users count: $USERS_COUNT"
log "Orders count: $ORDERS_COUNT"

# Cleanup
docker rm -f restore_test

if [[ $USERS_COUNT -gt 0 && $ORDERS_COUNT -gt 0 ]]; then
  log "✅ Level 2 test PASSED"
  exit 0
else
  log "❌ Level 2 test FAILED"
  exit 1
fi
```

#### Niveau 3 : PITR complet (Trimestrielle)

**Objectif** : Tester le Point-in-Time Recovery complet.

```bash
#!/bin/bash
# level3_pitr_test.sh

set -euo pipefail

FULL_BACKUP="/backups/mariadb/full/20251213"
BINLOG_DIR="/backups/binlogs"
TARGET_TIME="2025-12-13 14:30:00"
TEST_DATADIR="/var/lib/mysql_pitr_test"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a /var/log/pitr_test.log
}

log "========================================="
log "Level 3: PITR Test"
log "Target time: $TARGET_TIME"
log "========================================="

# Preparation
log "Preparing test environment..."
systemctl stop mariadb-test || true
rm -rf "$TEST_DATADIR"
mkdir -p "$TEST_DATADIR"

# Restore full backup
log "Restoring full backup..."
mariabackup --prepare --target-dir="$FULL_BACKUP"
mariabackup --copy-back --target-dir="$FULL_BACKUP" --datadir="$TEST_DATADIR"
chown -R mysql:mysql "$TEST_DATADIR"

# Start test instance on different port
log "Starting test MariaDB instance..."
mysqld_safe --datadir="$TEST_DATADIR" --port=3307 --socket=/tmp/mysql_test.sock &
MYSQLD_PID=$!
sleep 10

# Apply binary logs
log "Applying binary logs until $TARGET_TIME..."
START_POS=$(cat "$FULL_BACKUP/xtrabackup_binlog_info" | awk '{print $2}')
BINLOGS=$(ls "$BINLOG_DIR"/mariadb-bin.* | sort)

mysqlbinlog \
  --start-position="$START_POS" \
  --stop-datetime="$TARGET_TIME" \
  $BINLOGS | \
  mariadb --socket=/tmp/mysql_test.sock -u root

# Validation
log "Validating restored data..."
LAST_TIMESTAMP=$(mariadb --socket=/tmp/mysql_test.sock -u root -Nse \
  "SELECT MAX(created_at) FROM myapp.orders;")

log "Last order timestamp: $LAST_TIMESTAMP"
log "Target timestamp: $TARGET_TIME"

# Cleanup
kill $MYSQLD_PID
rm -rf "$TEST_DATADIR"

if [[ "$LAST_TIMESTAMP" < "$TARGET_TIME" ]]; then
  log "✅ Level 3 PITR test PASSED"
  exit 0
else
  log "❌ Level 3 PITR test FAILED"
  exit 1
fi
```

#### Niveau 4 : Disaster Recovery complet (Annuelle)

**Objectif** : Simuler une perte totale du datacenter.

Couvert dans la section suivante sur les exercices de simulation.

---

## Plan de Reprise d'Activité (PRA)

### Structure du PRA

Un PRA complet pour MariaDB comprend les sections suivantes :

```
┌──────────────────────────────────────────────────┐
│          Plan de Reprise d'Activité              │
├──────────────────────────────────────────────────┤
│                                                  │
│  1. INFORMATIONS GÉNÉRALES                       │
│     ├─ Périmètre et objectifs                    │
│     ├─ RTO/RPO cibles                            │
│     └─ Contacts d'urgence                        │
│                                                  │
│  2. INVENTAIRE DES RESSOURCES                    │
│     ├─ Serveurs et infrastructure                │
│     ├─ Backups et emplacements                   │
│     └─ Documentation technique                   │
│                                                  │
│  3. SCÉNARIOS D'INCIDENT                         │
│     ├─ Matrice de décision                       │
│     ├─ Arbres de décision                        │
│     └─ Procédures par scénario                   │
│                                                  │
│  4. PROCÉDURES DE RESTAURATION                   │
│     ├─ Restauration complète                     │
│     ├─ PITR                                      │
│     └─ Basculement DR                            │
│                                                  │
│  5. COMMUNICATION DE CRISE                       │
│     ├─ Arbre d'escalade                          │
│     ├─ Templates de communication                │
│     └─ Stakeholders                              │
│                                                  │
│  6. VALIDATION POST-INCIDENT                     │
│     ├─ Checklist de validation                   │
│     ├─ Tests fonctionnels                        │
│     └─ Retour en production                      │
│                                                  │
│  7. TESTS ET EXERCICES                           │
│     ├─ Planning annuel                           │
│     ├─ Scénarios de simulation                   │
│     └─ Amélioration continue                     │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Template PRA

```markdown
# Plan de Reprise d'Activité - MariaDB
**Version** : 2.0  
**Dernière mise à jour** : 2025-12-13  
**Prochain test** : 2026-01-15  
**Responsable** : John Doe (DBA Lead)

---

## 1. INFORMATIONS GÉNÉRALES

### 1.1 Périmètre
- **Base de données** : Production MariaDB 11.8 (myapp)
- **Taille** : 2 To
- **Transactions/jour** : ~50 millions
- **Utilisateurs** : 500 000 actifs/jour

### 1.2 Objectifs RTO/RPO

| Criticité | RTO | RPO | Justification |
|-----------|-----|-----|---------------|
| CRITIQUE (transactions) | 2h | 5 min | Perte revenus 50K€/h |
| HAUTE (analytics) | 4h | 1h | Impact business modéré |
| MOYENNE (archivage) | 24h | 24h | Pas d'impact immédiat |

### 1.3 Contacts d'urgence

| Rôle | Nom | Mobile | Email | Escalade |
|------|-----|--------|-------|----------|
| DBA Lead | John Doe | +33 6 XX XX XX XX | john@example.com | - |
| DBA On-call | Jane Smith | +33 6 YY YY YY YY | jane@example.com | 15 min |
| SRE Lead | Bob Wilson | +33 6 ZZ ZZ ZZ ZZ | bob@example.com | 30 min |
| CTO | Alice Brown | +33 6 AA AA AA AA | alice@example.com | 1h |
| Vendor Support | MariaDB Corp | +1-XXX-XXX-XXXX | support@mariadb.com | Critical |

---

## 2. INVENTAIRE DES RESSOURCES

### 2.1 Infrastructure Production

| Composant | Hostname | IP | Rôle | Specs |
|-----------|----------|-----|------|-------|
| mariadb-prod-01 | db-prod-01.example.com | 10.0.1.10 | Primary | 64 vCPU, 256GB RAM, 4TB NVMe |
| mariadb-prod-02 | db-prod-02.example.com | 10.0.1.11 | Replica | 64 vCPU, 256GB RAM, 4TB NVMe |
| mariadb-backup | db-backup.example.com | 10.0.1.20 | Backup | 16 vCPU, 64GB RAM, 10TB HDD |

### 2.2 Emplacements des backups

| Type | Emplacement | Rétention | Accès |
|------|-------------|-----------|-------|
| Full | /backups/mariadb/full | 30 jours | NAS local + S3 |
| Incremental | /backups/mariadb/inc | 7 jours | NAS local + S3 |
| Binary logs | /backups/binlogs | 7 jours | NAS local + S3 |
| Archivage | s3://backups/archive | 365 jours | S3 Glacier |

### 2.3 Infrastructure DR (Site secondaire)

| Composant | Hostname | IP | État |
|-----------|----------|----|------|
| mariadb-dr-01 | db-dr-01.aws.example.com | 172.16.1.10 | Standby cold |
| mariadb-dr-02 | db-dr-02.aws.example.com | 172.16.1.11 | Standby cold |

---

## 3. SCÉNARIOS D'INCIDENT

### 3.1 Matrice de décision

| Scénario | Probabilité | Impact | Procédure | RTO cible |
|----------|-------------|--------|-----------|-----------|
| Corruption table unique | Élevée | Faible | PITR partiel | 1h |
| DELETE sans WHERE | Moyenne | Moyen | PITR complet | 2h |
| Panne disque primary | Moyenne | Élevé | Failover replica | 15 min |
| Corruption base complète | Faible | Critique | Full restore + binlogs | 4h |
| Datacenter détruit | Très faible | Catastrophique | DR site secondaire | 8h |
| Ransomware | Faible | Critique | Restore backup pré-infection | 6h |

### 3.2 Arbre de décision

```
Incident détecté
    │
    ├─ Base accessible ? ────[OUI]──► Corruption partielle
    │                                      │
    │                                      ├─ Une table ?
    │                                      │   └─► PITR partiel
    │                                      │
    │                                      └─ Plusieurs tables ?
    │                                          └─► PITR complet
    │
    └─ Base inaccessible ? ──[OUI]──► Panne serveur
                                           │
                                           ├─ Replica disponible ?
                                           │   └─[OUI]─► Failover
                                           │
                                           └─[NON]──► Restauration DR
                                                        │
                                                        ├─ Backup local OK ?
                                                        │   └─► Restore local
                                                        │
                                                        └─ Backup distant ?
                                                            └─► Restore S3/DR
```

---

## 4. PROCÉDURES DE RESTAURATION

### 4.1 Procédure : Restauration complète

**Durée estimée** : 4-6 heures (base 2 To)

**Étapes** :

1. **Activation du PRA** (T+0)
   ```bash
   # Déclarer l'incident
   pagerduty trigger --service mariadb --severity critical \
     --description "Full database restore initiated"
   ```

2. **Provisionnement serveur restauration** (T+30min)
   ```bash
   # Démarrer instance AWS préconfigurée
   aws ec2 start-instances --instance-ids i-xxxxx
   ```

3. **Téléchargement backup depuis S3** (T+1h)
   ```bash
   aws s3 sync s3://backups/mariadb/full/latest /restore/
   ```

4. **Restauration Mariabackup** (T+3h)
   ```bash
   # Voir section 12.5.1 pour procédure détaillée
   mariabackup --prepare --target-dir=/restore/full
   mariabackup --copy-back --target-dir=/restore/full
   ```

5. **Application binary logs** (T+4h)
   ```bash
   # Voir section 12.5.2 pour PITR
   mysqlbinlog ... | mariadb
   ```

6. **Validation** (T+5h)
   - Exécuter checklist validation (section 12.5)
   - Tests applicatifs
   - Smoke tests

7. **Basculement production** (T+6h)
   - Update DNS/VIP
   - Redémarrage applications
   - Monitoring intensif

### 4.2 Procédure : Failover vers replica

**Durée estimée** : 15-30 minutes

**Étapes** :

1. **Vérification état replica** (T+0)
   ```sql
   SHOW SLAVE STATUS\G
   # Vérifier Seconds_Behind_Master < 60
   ```

2. **Promotion replica** (T+5min)
   ```sql
   STOP SLAVE;
   RESET SLAVE ALL;
   SET GLOBAL read_only = 0;
   ```

3. **Basculement applicatif** (T+10min)
   ```bash
   # Update DNS ou VIP
   # Redémarrage applications avec nouvelle config
   ```

4. **Validation** (T+15min)
   - Tests connexion
   - Vérification transactions

---

## 5. COMMUNICATION DE CRISE

### 5.1 Arbre d'escalade

```
Incident détecté (DBA On-call)
    │
    ├─ T+0 : Notification équipe technique (Slack #incidents)
    │        └─ DBA Lead, SRE Lead
    │
    ├─ T+15min : Escalade management si non résolu
    │            └─ CTO, VP Engineering
    │
    ├─ T+30min : Notification business stakeholders
    │            └─ CEO, VP Product, Customer Success
    │
    └─ T+1h : Communication externe si nécessaire
                └─ Status page, Email clients
```

### 5.2 Template de communication

**Email aux stakeholders** :

```
Objet: [INCIDENT] Base de données MariaDB - Impact production

Statut: EN COURS
Sévérité: CRITIQUE
Début: 2025-12-13 14:37 UTC
Impact: Indisponibilité complète application

Actions en cours:
- Restauration backup en cours (ETA: 16:00 UTC)
- Équipe DBA mobilisée
- Communication client préparée

Prochaine mise à jour: 15:30 UTC

Contact: John Doe (DBA Lead) - john@example.com
```

---

## 6. VALIDATION POST-INCIDENT

### 6.1 Checklist de validation technique

- [ ] Base de données démarrée sans erreur
- [ ] Toutes les bases présentes (SHOW DATABASES)
- [ ] Toutes les tables accessibles
- [ ] Pas d'erreur dans error log
- [ ] Cohérence référentielle validée (FK checks)
- [ ] Timestamp des données cohérent avec objectif RPO
- [ ] Procédures stockées, triggers, events présents
- [ ] Utilisateurs et privilèges restaurés
- [ ] Réplication configurée (si applicable)

### 6.2 Checklist de validation fonctionnelle

- [ ] Connexion application réussie
- [ ] Login utilisateur fonctionnel
- [ ] Création de commande test OK
- [ ] Consultation données existantes OK
- [ ] Performance acceptable (temps réponse < 100ms)
- [ ] Batch nocturnes validés
- [ ] Exports/rapports fonctionnels

### 6.3 Critères de retour en production

| Critère | Seuil | Validation |
|---------|-------|------------|
| Uptime | > 30 min | □ |
| Erreurs logs | 0 ERROR | □ |
| Latency p95 | < 50ms | □ |
| Tests fonctionnels | 100% pass | □ |
| Approbation DBA Lead | Signature | □ |
| Approbation CTO | Signature | □ |

---

## 7. TESTS ET EXERCICES

### 7.1 Planning annuel

| Mois | Type de test | Périmètre | Responsable |
|------|--------------|-----------|-------------|
| Janvier | Level 1 (intégrité) | Tous backups | Automatisé |
| Février | Level 2 (partiel) | 1 base | DBA On-call |
| Mars | Level 3 (PITR) | Complet | DBA Lead |
| Avril | Level 1 | Tous backups | Automatisé |
| Mai | Tabletop exercise | Corruption | Équipe complète |
| Juin | Level 2 | 1 base | DBA On-call |
| Juillet | Level 1 | Tous backups | Automatisé |
| Août | Level 3 (PITR) | Complet | DBA Lead |
| Septembre | Exercice live | Failover | Équipe + Management |
| Octobre | Level 2 | 1 base | DBA On-call |
| Novembre | Level 1 | Tous backups | Automatisé |
| Décembre | Level 4 (DR complet) | Site secondaire | Toute organisation |

### 7.2 Post-mortem template

Après chaque incident (réel ou simulé), documenter :

```markdown
# Post-Mortem Incident - [DATE]

## Résumé
- **Incident** : [Description]
- **Durée** : [X heures]
- **Impact** : [Utilisateurs affectés, revenus]

## Timeline
- T+0 : [Événement déclencheur]
- T+15min : [Action 1]
- ...

## Cause racine
[Analyse détaillée]

## Ce qui a bien fonctionné
- [Point positif 1]
- [Point positif 2]

## Ce qui doit être amélioré
- [Point d'amélioration 1]
- [Point d'amélioration 2]

## Actions correctives
| Action | Responsable | Échéance | Statut |
|--------|-------------|----------|--------|
| [Action 1] | [Nom] | [Date] | [ ] |
```

---

## Exercices de simulation

### Tabletop Exercise (Discussion guidée)

**Durée** : 2-3 heures  
**Participants** : DBA, SRE, Devs, Management  
**Fréquence** : Trimestrielle

**Déroulement** :

1. **Briefing** (15 min)
   - Rappel objectifs
   - Présentation scénario

2. **Scénario** (90 min)
   ```
   "Lundi 14h37 : Alerte monitoring - Primary database inaccessible.
    Cause : Corruption filesystem suite mise à jour kernel.
    Replica en lag de 2h (problème réseau).
    Dernier backup complet : Dimanche 2h.
    Binary logs disponibles jusqu'à 14h30.
   
   QUESTION : Quelle procédure appliquez-vous ?"
   ```

3. **Discussion** (60 min)
   - Chaque participant expose son raisonnement
   - Identification des gaps
   - Validation décisions

4. **Debriefing** (30 min)
   - Synthèse des apprentissages
   - Actions correctives
   - Mise à jour PRA

### Live Exercise (Exercice réel)

**Durée** : 1 journée  
**Participants** : Toute l'équipe tech + Management  
**Fréquence** : Annuelle

**Préparation** (J-30) :
- [ ] Planification date (éviter périodes critiques)
- [ ] Communication stakeholders
- [ ] Provisionnement infrastructure test
- [ ] Préparation scénario détaillé
- [ ] Briefing participants

**Exécution** (Jour J) :

```
08:00 - Briefing général
08:30 - Injection scénario (ex: destruction volontaire base test)
08:31 - Chronomètre démarre → Équipe applique PRA
        ├─ Communication
        ├─ Diagnostic
        ├─ Restauration
        └─ Validation
??:?? - Retour en production déclaré
        └─ Chronomètre arrêté (mesure RTO réel)
```

**Debriefing** (J+1) :
- Analyse timeline détaillée
- Écarts RTO prévu vs réel
- Difficultés rencontrées
- Mise à jour procédures
- Actions correctives planifiées

---

## Métriques et KPI

### Indicateurs de performance du PRA

| KPI | Cible | Mesure | Fréquence |
|-----|-------|--------|-----------|
| Taux de succès tests | 100% | Tests réussis / Tests totaux | Mensuelle |
| RTO moyen | < 4h | Temps restauration mesuré | Par test |
| RPO réel | < 15 min | Écart dernier binlog archivé | Continue |
| Taux de disponibilité | 99.95% | Uptime / Temps total | Mensuelle |
| MTTR (Mean Time To Repair) | < 2h | Moyenne temps réparation | Trimestrielle |
| Taux conformité tests | 100% | Tests effectués / Tests planifiés | Annuelle |

### Dashboard de suivi

```
┌─────────────────────────────────────────────────┐
│           PRA Dashboard (Grafana)               │
├─────────────────────────────────────────────────┤
│                                                 │
│  📊 Tests de restauration (12 derniers mois)    │
│  ├─ Succès : 47 / 48 (98%)                      │
│  └─ Échecs : 1 (Binlog manquant - corrigé)      │
│                                                 │
│  ⏱️ RTO moyen : 3h 22min                        │
│  📈 Trend : -15% (amélioration)                 │
│                                                 │
│  🎯 RPO moyen : 8 minutes                       │
│  📈 Trend : Stable                              │
│                                                 │
│  ✅ Conformité tests : 100%                     │
│  (48/48 tests planifiés effectués)              │
│                                                 │
│  🚨 Incidents réels (12 mois) : 2               │
│  ├─ Corruption partielle (PITR 1h10)            │
│  └─ Failover replica (12 min)                   │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Amélioration continue

### Cycle PDCA (Plan-Do-Check-Act)

```
┌────────────────────────────────────────┐
│         Cycle d'amélioration           │
├────────────────────────────────────────┤
│                                        │
│  1. PLAN (Planifier)                   │
│     ├─ Analyser incidents passés       │
│     ├─ Identifier améliorations        │
│     └─ Planifier tests                 │
│          │                             │
│          ▼                             │
│  2. DO (Faire)                         │
│     ├─ Exécuter tests planifiés        │
│     ├─ Documenter procédures           │
│     └─ Former les équipes              │
│          │                             │
│          ▼                             │
│  3. CHECK (Vérifier)                   │
│     ├─ Analyser résultats tests        │
│     ├─ Mesurer RTO/RPO                 │
│     └─ Identifier gaps                 │
│          │                             │
│          ▼                             │
│  4. ACT (Agir)                         │
│     ├─ Implémenter corrections         │
│     ├─ Mettre à jour PRA               │
│     └─ Partager apprentissages         │
│          │                             │
│          └──────────────► Retour PLAN  │
│                                        │
└────────────────────────────────────────┘
```

### Revue annuelle du PRA

**Agenda type** (réunion de 4h) :

1. **Bilan année écoulée** (60 min)
   - Incidents réels : timeline, impact, résolution
   - Tests effectués : taux de réussite, RTO/RPO
   - Évolutions infrastructure

2. **Analyse des écarts** (60 min)
   - RTO/RPO cibles vs réels
   - Procédures obsolètes identifiées
   - Points de blocage récurrents

3. **Mise à jour du PRA** (90 min)
   - Nouvelles procédures
   - Contacts à jour
   - Infrastructure DR actualisée
   - Scénarios ajoutés/modifiés

4. **Planning année suivante** (30 min)
   - Calendrier tests
   - Exercices live
   - Formations équipe

---

## ✅ Points clés à retenir

- **Tests obligatoires** : Un backup non testé = Pas de backup (34% n'ont jamais testé !)
- **Méthodologie pyramidale** : 4 niveaux de tests (intégrité → DR complet)
- **PRA documenté** : Document vivant, mis à jour après chaque incident/test
- **Exercices réguliers** : Tabletop trimestriel + Live annuel minimum
- **Communication claire** : Arbre d'escalade, templates, stakeholders identifiés
- **Validation rigoureuse** : Checklist technique + fonctionnelle avant retour prod
- **Métriques suivies** : RTO/RPO mesurés, taux succès tests, conformité planning
- **Amélioration continue** : Post-mortem systématique, cycle PDCA, revue annuelle
- **Formation équipes** : Tous doivent connaître leur rôle en cas d'incident
- **Infrastructure DR** : Site secondaire, cold standby minimum, procédures testées

---

## 🔗 Ressources et références

### Documentation et standards

- [📖 ITIL Disaster Recovery - AXELOS](https://www.axelos.com/certifications/itil-service-management)
- [📖 ISO 22301 - Business Continuity Management](https://www.iso.org/iso-22301-business-continuity.html)
- [📖 NIST Contingency Planning Guide - SP 800-34](https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final)

### Articles techniques

- [Disaster Recovery Best Practices - MariaDB](https://mariadb.com/resources/blog/disaster-recovery-best-practices/)
- [Testing Your Backups - Percona](https://www.percona.com/blog/testing-mysql-backups/)
- [DR Testing Methodologies - AWS](https://aws.amazon.com/disaster-recovery/)

---

## ➡️ Section suivante

**[12.8 - Sauvegarde cloud-native](./08-sauvegarde-cloud-native.md)** : S3, Azure Blob Storage, Google Cloud Storage, Kubernetes VolumeSnapshots, architectures multi-cloud.

---


⏭️ [Sauvegarde cloud-native](/12-sauvegarde-restauration/08-sauvegarde-cloud-native.md)
