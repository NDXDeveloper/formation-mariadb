🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12. Sauvegarde et Restauration

> **Niveau** : Avancé  
> **Durée estimée** : 8-10 heures  
> **Prérequis** : Administration MariaDB (Chapitre 11), Réplication (Chapitre 13), Notions de production et disponibilité

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- **Concevoir** une stratégie de sauvegarde adaptée aux besoins métier (RPO/RTO)
- **Maîtriser** les outils de sauvegarde logique (mariadb-dump, mydumper) et physique (Mariabackup)
- **Mettre en œuvre** des sauvegardes incrémentales et le Point-in-Time Recovery (PITR)
- **Automatiser** les sauvegardes dans des environnements cloud et Kubernetes
- **Tester** régulièrement les procédures de restauration et le Plan de Reprise d'Activité (PRA)
- **Sécuriser** et optimiser le stockage des sauvegardes (chiffrement, compression, rétention)

---

## Introduction

La sauvegarde et la restauration constituent **le pilier fondamental de la continuité d'activité** d'une infrastructure de bases de données. Aucune architecture haute disponibilité, aussi sophistiquée soit-elle, ne dispense d'une stratégie de backup rigoureuse et testée régulièrement.

### Pourquoi la sauvegarde est critique

Les données d'une organisation représentent souvent son actif le plus précieux. Leur perte peut résulter de :

- **Erreurs humaines** : DROP TABLE accidentel, UPDATE sans WHERE, DELETE massif
- **Corruptions matérielles** : Panne disque, corruption filesystem, problème RAID
- **Incidents logiciels** : Bugs applicatifs, migrations ratées, conflits de réplication
- **Attaques malveillantes** : Ransomware, sabotage interne, intrusions
- **Catastrophes naturelles** : Incendie, inondation, séisme affectant le datacenter

💡 **Principe fondamental** : *"Une sauvegarde non testée n'est pas une sauvegarde"*. De nombreuses organisations découvrent que leurs backups sont inexploitables au moment critique de la restauration.

### Les métriques de disponibilité

Toute stratégie de sauvegarde doit être définie en fonction de deux indicateurs clés :

**RPO (Recovery Point Objective)** : Perte de données maximale acceptable
- "Combien de données puis-je me permettre de perdre ?"
- Exemple : RPO de 1 heure = accepter de perdre jusqu'à 1h de transactions
- Impact : Détermine la fréquence des sauvegardes

**RTO (Recovery Time Objective)** : Temps de restauration maximal acceptable
- "Combien de temps puis-je rester hors ligne ?"
- Exemple : RTO de 4 heures = restauration complète en moins de 4h
- Impact : Détermine le type de sauvegarde et l'infrastructure de stockage

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Timeline d'incident                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Dernière ──────►  Incident ──────►  Détection ──────►  Restauration │
│  sauvegarde        survient        incident           complète       │
│                                                                      │
│  ◄────RPO────►                                                       │
│               ◄──────────────RTO──────────────►                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Panorama des stratégies de sauvegarde

Ce chapitre couvre trois approches complémentaires :

#### 1. Sauvegardes complètes (Full Backup)
Copie intégrale de toutes les données à un instant T. Simple mais consommatrice en ressources.

#### 2. Sauvegardes incrémentales (Incremental Backup)
Ne sauvegarde que les modifications depuis la dernière sauvegarde (complète ou incrémentale). Économe en espace mais restauration plus complexe.

#### 3. Sauvegardes différentielles (Differential Backup)
Ne sauvegarde que les modifications depuis la dernière sauvegarde complète. Compromis entre full et incrémentale.

---

## Vue d'ensemble des outils MariaDB

### Sauvegardes logiques : Export SQL

Les sauvegardes logiques exportent les données sous forme de requêtes SQL (`CREATE TABLE`, `INSERT`).

**mariadb-dump (mysqldump)**
- ✅ Simplicité d'utilisation
- ✅ Portabilité entre versions/plateformes
- ✅ Filtrage granulaire (bases, tables, lignes)
- ⚠️ Plus lent que les backups physiques
- ⚠️ Verrouille les tables pendant l'export (sans `--single-transaction`)

**mydumper / myloader**
- ✅ Export/import parallélisé (multithreading)
- ✅ Jusqu'à 10x plus rapide que mysqldump
- ✅ Sauvegarde cohérente avec réplication GTID
- ✅ Compression native et chunking intelligent

```sql
-- Exemple : Export d'une base avec mariadb-dump
mariadb-dump \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --databases myapp > myapp_backup.sql
```

### Sauvegardes physiques : Copie des fichiers

Les sauvegardes physiques copient directement les fichiers de données InnoDB, Aria, etc.

**Mariabackup** 🆕
- ✅ Backup à chaud (hot backup) sans interruption de service
- ✅ Incrémental et différentiel natifs
- ✅ **Support BACKUP STAGE** (MariaDB 11.8) : amélioration performance
- ✅ Compatible avec tous les moteurs transactionnels
- ✅ Restauration plus rapide que logique

```bash
# Backup complet avec Mariabackup
mariabackup --backup \
  --target-dir=/backups/full-2025-12-13 \
  --user=backup_user \
  --password=secure_password
```

🆕 **Nouveauté MariaDB 11.8** : Le support de `BACKUP STAGE` améliore la coordination entre Mariabackup et le serveur, réduisant les verrouillages et augmentant les performances des backups sur des bases volumineuses.

---

## Architecture de sauvegarde en production

### Stratégie hybride recommandée

Une architecture professionnelle combine généralement plusieurs types de sauvegardes :

```
┌────────────────────────────────────────────────────────────┐
│              Stratégie de backup recommandée               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Dimanche      : Full backup (Mariabackup)                 │
│  Lun-Sam       : Incremental backup (Mariabackup)          │
│  En continu    : Binary logs (PITR)                        │
│  Hebdomadaire  : Logical backup (mariadb-dump) - Validation│
│  Mensuel       : Archivage longue durée (S3 Glacier)       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Règle du 3-2-1

La règle d'or pour une protection optimale des données :

- **3** copies des données (production + 2 backups)
- **2** supports de stockage différents (disque local + cloud)
- **1** copie hors site (datacenter distant, cloud, etc.)

```
Production DB ──┬──► Backup local (NAS)
                │
                ├──► Backup cloud (S3/Azure/GCS)
                │
                └──► Backup archivé hors site (Glacier/Cold Storage)
```

💡 **Bonne pratique** : Chiffrer systématiquement les backups, surtout ceux stockés hors site ou dans le cloud.

---

## Point-in-Time Recovery (PITR)

Le PITR permet de restaurer une base de données à n'importe quel instant précis, en combinant :

1. **Une sauvegarde complète** (point de départ)
2. **Les binary logs** (rejeu des transactions jusqu'à l'instant T)

### Cas d'usage typiques

**Scénario 1 : Suppression accidentelle**
```
09h00 : Sauvegarde complète
14h37 : DELETE FROM orders WHERE ... (erreur !)
14h45 : Détection du problème

→ Restaurer la base au 14h36 (juste avant l'erreur)
```

**Scénario 2 : Corruption de données**
```
Mardi : Sauvegarde complète
Jeudi 10h : Mise à jour applicative buggée corrompt les données
Jeudi 16h : Détection de l'anomalie

→ Restaurer au Jeudi 09h59 (avant la corruption)
```

### Architecture PITR

```
Full Backup          Binary Logs           Point de restauration
(Dimanche)          (Lun-Ven)                  (Jeudi 14h30)
    │                   │                           │
    ▼                   ▼                           ▼
┌─────────┐       ┌──────────────┐             ┌──────────┐
│ Base    │──────►│ bin.000001   │────────────►│ État     │
│ Complete│       │ bin.000002   │             │ Jeudi    │
│ Dim 00h │       │ bin.000003   │             │ 14h30    │
└─────────┘       │ ...          │             └──────────┘
                  └──────────────┘
```

⚠️ **Attention** : Le PITR n'est possible que si les binary logs sont activés et conservés suffisamment longtemps.

---

## Automatisation des sauvegardes

### Planification avec cron

```bash
# /etc/cron.d/mariadb-backups

# Backup complet quotidien à 2h du matin
0 2 * * * backup_user /scripts/full_backup.sh

# Backup incrémental toutes les 4 heures
0 */4 * * * backup_user /scripts/incremental_backup.sh

# Rotation et nettoyage des anciennes sauvegardes
0 3 * * 0 backup_user /scripts/cleanup_old_backups.sh
```

### Notification et monitoring

Un système de backup professionnel doit inclure :

- ✅ **Notifications** : Email/Slack en cas de succès/échec
- ✅ **Métriques** : Durée, taille, taux de compression
- ✅ **Alertes** : Backup non effectué, espace insuffisant
- ✅ **Dashboards** : Visualisation historique (Grafana)

```bash
# Exemple de script avec notification
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"

if mariabackup --backup --target-dir=$BACKUP_DIR; then
  echo "✅ Backup successful" | mail -s "MariaDB Backup OK" admin@example.com
else
  echo "❌ Backup FAILED" | mail -s "ALERT: MariaDB Backup Failed" admin@example.com
fi
```

---

## Sauvegardes cloud-native

### Stockage objet S3

MariaDB peut sauvegarder directement vers AWS S3, MinIO, ou tout compatible S3 :

```bash
# Backup vers S3 avec compression et chiffrement
mariabackup --backup \
  --stream=xbstream \
  --target-dir=. | \
  gzip | \
  aws s3 cp - s3://my-backups/mariadb/backup-$(date +%Y%m%d).xbstream.gz \
    --storage-class INTELLIGENT_TIERING \
    --server-side-encryption AES256
```

**Avantages S3** :
- ✅ Durabilité 99.999999999% (11 neuf)
- ✅ Stockage illimité avec coûts dégressifs
- ✅ Lifecycle policies automatiques
- ✅ Versioning et Object Lock (protection ransomware)

### Kubernetes VolumeSnapshots

Pour les déploiements Kubernetes, les snapshots de volumes offrent une alternative performante :

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mariadb-snapshot-daily
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: mariadb-pvc
```

**Avantages VolumeSnapshots** :
- ✅ Snapshot instantané (Copy-on-Write)
- ✅ Restauration rapide (quelques secondes)
- ✅ Intégration native avec mariadb-operator
- ✅ Orchestration via CronJob Kubernetes

---

## Chiffrement et sécurité des backups

### Chiffrement à la volée

**Avec mariadb-dump** :
```bash
mariadb-dump --all-databases | \
  openssl enc -aes-256-cbc -pbkdf2 -salt -pass pass:$BACKUP_PASSWORD | \
  aws s3 cp - s3://backups/encrypted-backup.sql.enc
```

**Avec Mariabackup** :
```bash
mariabackup --backup --stream=xbstream --target-dir=. | \
  openssl enc -aes-256-gcm -pbkdf2 -salt -pass file:/etc/backup-key | \
  gzip > /backups/encrypted.xbstream.gz.enc
```

### Gestion des clés de chiffrement

⚠️ **CRITICAL** : Les clés de chiffrement doivent être :
- Stockées séparément des backups
- Sauvegardées en lieu sûr (coffre-fort, HSM, KMS)
- Régulièrement testées lors des exercices de restauration

💡 **Recommandation** : Utiliser un KMS (AWS KMS, Azure Key Vault, HashiCorp Vault) pour la gestion centralisée des clés.

---

## Tests et validation

### Fréquence des tests de restauration

| Environnement | Fréquence minimale | Portée |
|--------------|-------------------|--------|
| Production critique | Mensuelle | Restauration complète + PITR |
| Production standard | Trimestrielle | Restauration complète |
| Développement | Semestrielle | Validation procédures |

### Checklist de validation

Un test de restauration doit vérifier :

- [ ] Intégrité des fichiers de backup (checksums)
- [ ] Restauration complète fonctionnelle
- [ ] PITR jusqu'à un instant précis
- [ ] Temps de restauration conforme au RTO
- [ ] Cohérence des données restaurées
- [ ] Applications fonctionnelles post-restauration
- [ ] Documentation à jour et procédures claires

```bash
# Validation automatique de l'intégrité
md5sum -c backup.md5 && echo "✅ Backup integrity OK"

# Test de restauration en environnement isolé
docker run -d --name restore-test \
  -e MARIADB_ROOT_PASSWORD=test \
  mariadb:11.8

# Restauration et validation
docker exec -i restore-test mariadb < backup.sql
docker exec restore-test mariadb -e "SHOW DATABASES; SELECT COUNT(*) FROM myapp.users;"
```

---

## Plan de Reprise d'Activité (PRA)

### Composantes d'un PRA

Un PRA complet pour MariaDB inclut :

1. **Documentation** :
   - Procédures de restauration étape par étape
   - Contacts d'urgence (équipes, fournisseurs)
   - Inventaire des ressources (serveurs, stockage, réseaux)

2. **Infrastructure de secours** :
   - Serveurs de restauration pré-provisionnés
   - Accès réseau sécurisé aux backups
   - Outils et scripts de restauration testés

3. **Procédures opérationnelles** :
   - Scénarios de disaster recovery documentés
   - Arbre de décision (quel type de restauration utiliser)
   - Communication avec les parties prenantes

4. **Tests réguliers** :
   - Exercices de simulation (tabletop exercises)
   - Restaurations complètes en environnement de test
   - Mesure et amélioration continue des RTO/RPO

### Matrice de décision de restauration

```
┌────────────────────────────────────────────────────────────┐
│ Type d'incident        │ Solution recommandée              │
├────────────────────────────────────────────────────────────┤
│ Corruption limitée     │ PITR jusqu'avant incident         │
│ DROP TABLE accidentel  │ PITR ou restauration table seule  │
│ Panne disque           │ Restauration full backup          │
│ Datacenter détruit     │ Failover vers backup distant      │
│ Ransomware             │ Restauration backup pré-infection │
│ Corruption généralisée │ Restauration full + binary logs   │
└────────────────────────────────────────────────────────────┘
```

---

## Considérations de performance

### Impact sur la production

Les opérations de backup peuvent affecter les performances :

**Sauvegardes logiques (mariadb-dump)** :
- Impact CPU : Moyen (génération SQL)
- Impact I/O : Faible à moyen
- Impact réseau : Négligeable
- Verrouillage : Possible sans `--single-transaction`

**Sauvegardes physiques (Mariabackup)** :
- Impact CPU : Faible (copie fichiers)
- Impact I/O : Moyen à élevé (lecture intensive)
- Impact réseau : Négligeable (sauf copie distante)
- Verrouillage : Minimal (flush logs uniquement)

### Optimisations

```ini
# Configuration pour minimiser l'impact des backups
[mariadb]
# Limiter l'impact I/O de Mariabackup
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Buffer pool suffisant pour éviter les flush excessifs
innodb_buffer_pool_size = 16G

# Compression des backups pour réduire I/O réseau
[mariabackup]
compress
compress-threads = 4
```

💡 **Astuce** : Planifier les backups complets pendant les heures creuses (nuit, week-end) et privilégier les backups incrémentaux en journée.

---

## Conformité réglementaire

### Exigences courantes

Certaines réglementations imposent des contraintes spécifiques :

**RGPD (GDPR)** :
- Droit à l'oubli : Capacité de supprimer définitivement les données personnelles
- Conservation limitée : Durée de rétention documentée et justifiée
- Sécurité : Chiffrement obligatoire des backups

**SOC 2 / ISO 27001** :
- Backups testés régulièrement (preuves documentées)
- Séparation des rôles (backup ≠ restauration)
- Audit trail des opérations de backup/restore

**PCI-DSS** (données cartes bancaires) :
- Chiffrement fort (AES-256 minimum)
- Clés de chiffrement gérées séparément
- Backups stockés dans des zones sécurisées

### Documentation requise

- Politique de sauvegarde formalisée
- Procédures opérationnelles détaillées
- Registre des tests de restauration
- Rapports d'incidents et actions correctives

---

## Architecture de référence

Voici une architecture de sauvegarde complète pour un environnement de production critique :

```
┌─────────────────────────────────────────────────────────────┐
│                   Production MariaDB                        │
│                   (Primary + Replicas)                      │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┬──────────────┬──────────────┐
        │                       │              │              │
        ▼                       ▼              ▼              ▼
┌───────────────┐    ┌──────────────┐  ┌──────────┐  ┌──────────┐
│ Mariabackup   │    │ Binary Logs  │  │ mydumper │  │ Volume   │
│ (Full Daily)  │    │ (Continue)   │  │ (Weekly) │  │ Snapshot │
│               │    │              │  │          │  │ (Hourly) │
└───────┬───────┘    └──────┬───────┘  └────┬─────┘  └────┬─────┘
        │                   │               │             │
        └─────────┬─────────┴───────┬───────┴─────────────┘
                  │                 │
                  ▼                 ▼
          ┌──────────────┐   ┌──────────────┐
          │ Local NAS    │   │ Cloud S3     │
          │ (Hot Backup) │   │ (Warm Backup)│
          └──────┬───────┘   └──────┬───────┘
                 │                  │
                 │                  ▼
                 │          ┌───────────────┐
                 │          │ Glacier       │
                 │          │ (Cold Archive)│
                 │          └───────────────┘
                 │
                 ▼
         ┌──────────────┐
         │ Restore Test │
         │ Environment  │
         │ (Monthly)    │
         └──────────────┘
```

---

## ✅ Points clés à retenir

- **RPO/RTO** définissent la stratégie de sauvegarde ; tout découle de ces métriques métier
- **Règle 3-2-1** : 3 copies, 2 supports, 1 hors site — pas de compromis en production
- **Mariabackup** est l'outil recommandé pour les sauvegardes physiques en production (hot backup, incrémental, support BACKUP STAGE 🆕)
- **PITR** nécessite binary logs activés et conservés — combiner full backup + binlogs
- **Tests réguliers** sont obligatoires — une sauvegarde non testée est une illusion de sécurité
- **Automatisation** est essentielle — cron, notifications, monitoring, alertes
- **Chiffrement** systématique des backups, surtout cloud et hors site
- **Cloud-native** (S3, VolumeSnapshots) simplifie la durabilité et la scalabilité
- **Documentation** : PRA formalisé, procédures à jour, contacts d'urgence identifiés

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Mariabackup - MariaDB Knowledge Base](https://mariadb.com/kb/en/mariabackup/)
- [📖 mariadb-dump - MariaDB Knowledge Base](https://mariadb.com/kb/en/mariadb-dump/)
- [📖 Binary Log - MariaDB Knowledge Base](https://mariadb.com/kb/en/binary-log/)
- [📖 Point-in-Time Recovery - MariaDB Docs](https://mariadb.com/kb/en/point-in-time-recovery/)

### Outils tiers

- [mydumper/myloader - GitHub](https://github.com/mydumper/mydumper)
- [Percona XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) (alternative compatible)

### Articles et guides

- [MariaDB Backup Best Practices - MariaDB Corporation](https://mariadb.com/resources/blog/mariadb-backup-best-practices/)
- [Disaster Recovery Planning for Databases](https://www.percona.com/blog/disaster-recovery-planning/)

### Conformité

- [GDPR Compliance for Databases](https://gdpr.eu/)
- [PCI-DSS Requirements v4.0](https://www.pcisecuritystandards.org/)

---

## ➡️ Sections suivantes

Les sections 12.1 à 12.8 détailleront chacun des aspects abordés dans cette introduction :

**[12.1 - Stratégies de sauvegarde](./01-strategies-sauvegarde.md)** : Comparaison approfondie Full / Incrémentale / Différentielle avec cas d'usage et dimensionnement.

**[12.2 - Sauvegarde logique](./02-sauvegarde-logique.md)** : Guide complet mariadb-dump et mydumper avec options avancées et optimisations.

**[12.3 - Sauvegarde physique (Mariabackup)](./03-sauvegarde-physique-mariabackup.md)** : Full backup, incremental backup, et nouveauté 🆕 Support BACKUP STAGE.

**[12.4 - Sauvegarde incrémentale avec binary logs](./04-sauvegarde-incrementale-binlog.md)** : Configuration, rotation, et stratégies de rétention.

**[12.5 - Restauration](./05-restauration.md)** : Procédures de restauration complète et Point-in-Time Recovery détaillées.

**[12.6 - Automatisation des sauvegardes](./06-automatisation-sauvegardes.md)** : Scripts, planification, monitoring et alerting.

**[12.7 - Tests de restauration et PRA](./07-tests-restauration-pra.md)** : Méthodologie de tests, scénarios d'incident, documentation du PRA.

**[12.8 - Sauvegarde cloud-native](./08-sauvegarde-cloud-native.md)** : S3/Object Storage, Kubernetes VolumeSnapshots, stratégies multi-cloud.

---


⏭️ [Stratégies de sauvegarde : Full, Incrémentale, Différentielle](/12-sauvegarde-restauration/01-strategies-sauvegarde.md)
