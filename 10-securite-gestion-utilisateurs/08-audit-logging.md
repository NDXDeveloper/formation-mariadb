🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.8 Audit et logging

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures
> **Prérequis** : Sections 10.1-10.7, connaissances en conformité réglementaire

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** l'importance de l'audit pour la sécurité et la conformité
- **Distinguer** les différents types de logs MariaDB (error, general, slow, binary, audit)
- **Choisir** la stratégie d'audit appropriée selon le contexte réglementaire
- **Identifier** les événements critiques à auditer (connexions, privilèges, données)
- **Maîtriser** les plugins d'audit disponibles (Server Audit, MySQL Enterprise Audit)
- **Implémenter** une politique de rétention et rotation des logs
- **Mesurer** l'impact performance de l'audit
- **Intégrer** les logs d'audit avec des SIEM (Splunk, ELK, etc.)
- **Répondre** aux exigences PCI-DSS, RGPD, HIPAA, SOC2

---

## Introduction

L'**audit** et le **logging** sont des composantes essentielles de la sécurité des bases de données. Ils permettent de :

1. **Tracer** toutes les activités (qui a fait quoi, quand, et comment)
2. **Détecter** les comportements anormaux et les tentatives d'intrusion
3. **Enquêter** après un incident de sécurité
4. **Prouver** la conformité réglementaire
5. **Protéger** contre les menaces internes (utilisateurs malveillants)

### Pourquoi l'audit est critique ?

**Scénario sans audit** :

```
Vendredi 18h00: Base de données production
┌─────────────────────────────────────────────┐
│  Table customers (10 000 clients)           │
│  - Données personnelles (RGPD)              │
│  - Informations bancaires (PCI-DSS)         │
└─────────────────────────────────────────────┘

Lundi 9h00: Découverte du problème
┌─────────────────────────────────────────────┐
│  Table customers (0 clients) ❌             │
│  → Toutes les données supprimées!           │
└─────────────────────────────────────────────┘

Questions sans réponse (pas d'audit):
❓ Qui a supprimé les données ?
❓ Quand exactement ?
❓ Depuis quelle IP/machine ?
❓ Avec quel utilisateur ?
❓ Suppression accidentelle ou malveillante ?
❓ D'autres données compromises ?

→ Impossible d'enquêter
→ Impossible de prouver la conformité
→ Perte de confiance des clients
→ Amendes réglementaires potentielles
```

**Avec audit activé** :

```
Audit log révèle:
┌─────────────────────────────────────────────┐
│ 2025-12-13 17:45:23                         │
│ User: contractor_temp@10.0.0.15             │
│ Query: DELETE FROM customers;               │
│ Result: 10000 rows affected                 │
│ Duration: 2.3s                              │
└─────────────────────────────────────────────┘

Actions possibles:
✅ Identifier le coupable (contractor_temp)
✅ Timeline précise (17h45 vendredi)
✅ Origine (IP 10.0.0.15)
✅ Restaurer depuis backup (17h30)
✅ Preuves pour action légale
✅ Corriger les privilèges (contractor ne devrait pas avoir DELETE)
```

### Conformité réglementaire

Les réglementations **exigent** l'audit des bases de données :

| Réglementation | Exigences d'audit | Rétention | MariaDB |
|----------------|-------------------|-----------|---------|
| **PCI-DSS 4.0** | Accès aux données de cartes, modifications privilèges | 1 an (3 mois en ligne) | ✅ Server Audit Plugin |
| **RGPD/GDPR** | Accès aux données personnelles, modifications | Proportionnel au risque | ✅ Audit + chiffrement logs |
| **HIPAA** | Accès aux données de santé (PHI) | 6 ans | ✅ Audit + contrôle d'accès |
| **SOC 2 Type II** | Tous les accès admin, modifications DB | 1 an minimum | ✅ Audit exhaustif |
| **ISO 27001** | Événements de sécurité | Selon politique | ✅ Audit configurable |
| **Sarbanes-Oxley** | Modifications données financières | 7 ans | ✅ Audit + archivage |

💡 **PCI-DSS 4.0** (Requirement 10) exige l'audit de :
- Tous les accès aux données de cartes (PAN)
- Toutes les modifications de privilèges
- Toutes les tentatives d'accès refusées
- Tous les arrêts/démarrages du système

---

## Types de logs MariaDB

MariaDB propose **5 types de logs** différents, chacun avec un objectif spécifique.

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                    Logs MariaDB                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. ERROR LOG (erreurs serveur)                              │
│     → Démarrages, arrêts, erreurs critiques                  │
│     → /var/log/mysql/error.log                               │
│     → TOUJOURS activé                                        │
│                                                              │
│  2. GENERAL QUERY LOG (toutes les requêtes)                  │
│     → Toutes les connexions et requêtes                      │
│     → /var/log/mysql/general.log                             │
│     → DÉSACTIVÉ par défaut (impact performance)              │
│                                                              │
│  3. SLOW QUERY LOG (requêtes lentes)                         │
│     → Requêtes > long_query_time                             │
│     → /var/log/mysql/slow.log                                │
│     → Activé pour optimisation performance                   │
│                                                              │
│  4. BINARY LOG (réplication et PITR)                         │
│     → Toutes les modifications de données                    │
│     → /var/log/mysql/binlog.*                                │
│     → Activé pour HA et backup                               │
│                                                              │
│  5. AUDIT LOG (sécurité et conformité) 🎯                    │
│     → Connexions, requêtes, événements sécurité              │
│     → /var/log/mysql/audit.log                               │
│     → Activé via plugin (server_audit)                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 1. Error Log

**Objectif** : Diagnostiquer les problèmes serveur.

**Contenu** :
- Démarrages et arrêts du serveur
- Erreurs critiques (crash, corruption)
- Warnings (mémoire, connexions)
- Messages plugins

**Exemple** :

```
2025-12-13 10:30:15 0 [Note] /usr/sbin/mariadbd: ready for connections.
Version: '11.8.0-MariaDB'  socket: '/run/mysqld/mysqld.sock'  port: 3306

2025-12-13 14:22:43 123 [Warning] Aborted connection 123 to db: 'production'
user: 'app_user' host: '10.0.0.5' (Got an error reading communication packets)

2025-12-13 17:45:01 0 [ERROR] InnoDB: Corruption detected in tablespace
'production/orders' page 12345
```

**Configuration** :

```ini
# /etc/my.cnf.d/server.cnf
[mysqld]
log_error = /var/log/mysql/error.log
log_warnings = 2  # 0=aucun, 1=warnings, 2=notes+warnings
```

**Cas d'usage** :
- ✅ Troubleshooting serveur
- ✅ Détection de crashes
- ❌ Pas pour l'audit sécurité (pas assez détaillé)

### 2. General Query Log

**Objectif** : Enregistrer **toutes** les requêtes (debug).

**Contenu** :
- Toutes les connexions (succès + échecs)
- Toutes les requêtes SQL
- Timestamp précis

**Exemple** :

```
2025-12-13 10:15:23   45 Connect   app_user@10.0.0.5 on production
2025-12-13 10:15:23   45 Query     SELECT * FROM users WHERE id = 123
2025-12-13 10:15:24   45 Query     UPDATE users SET last_login = NOW() WHERE id = 123
2025-12-13 10:15:25   45 Quit
```

**Configuration** :

```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/general.log
```

⚠️ **Attention** :
- **Impact performance MAJEUR** (peut réduire de 50% le throughput)
- Fichier log **très volumineux** (plusieurs Go/heure)
- **À utiliser uniquement pour debug temporaire**

**Cas d'usage** :
- ✅ Debug application (voir toutes les requêtes)
- ✅ Troubleshooting temporaire
- ❌ **JAMAIS en production continue** (trop lourd)
- ❌ Pas pour conformité (pas assez structuré)

### 3. Slow Query Log

**Objectif** : Identifier les requêtes lentes pour optimisation.

**Contenu** :
- Requêtes dépassant `long_query_time`
- Durée d'exécution
- Nombre de lignes examinées/retournées
- Requêtes sans index (`log_queries_not_using_indexes`)

**Exemple** :

```
# Time: 2025-12-13T14:30:45.123456Z
# User@Host: app_user[app_user] @ app-server [10.0.0.5]
# Query_time: 12.345678  Lock_time: 0.000234  Rows_sent: 1  Rows_examined: 1000000
SET timestamp=1702477845;
SELECT * FROM orders WHERE customer_id = 123 ORDER BY created_at DESC;
```

**Configuration** :

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2  # Secondes (2s par défaut)
log_queries_not_using_indexes = 1
```

**Cas d'usage** :
- ✅ Optimisation performance
- ✅ Détection requêtes problématiques
- ❌ Pas pour audit sécurité (focus performance)

### 4. Binary Log (binlog)

**Objectif** : Réplication et Point-in-Time Recovery (PITR).

**Contenu** :
- Toutes les modifications de données (INSERT, UPDATE, DELETE, DDL)
- Format binaire (efficace)
- Séquence chronologique

**Configuration** :

```ini
[mysqld]
log_bin = /var/log/mysql/binlog
binlog_format = ROW  # ROW, STATEMENT, ou MIXED
expire_logs_days = 7
max_binlog_size = 100M
```

**Cas d'usage** :
- ✅ Réplication master-slave
- ✅ Point-in-Time Recovery
- ✅ Audit de modifications (indirect)
- ❌ Pas conçu pour audit sécurité (format binaire, pas de connexions)

### 5. Audit Log 🎯 (via plugin)

**Objectif** : Audit de sécurité et conformité réglementaire.

**Contenu configurable** :
- Connexions (succès + échecs)
- Requêtes SQL (SELECT, INSERT, UPDATE, DELETE, DDL, DCL)
- Modifications de privilèges
- Utilisateur, timestamp, IP source
- Résultat (succès/échec)

**Exemple (format JSON)** :

```json
{
  "timestamp": "2025-12-13T17:45:23.123Z",
  "event": "QUERY",
  "user": "contractor_temp@10.0.0.15",
  "database": "production",
  "query": "DELETE FROM customers",
  "rows_affected": 10000,
  "status": "SUCCESS"
}
```

**Plugins disponibles** :

1. **MariaDB Server Audit Plugin** (open source, inclus) ⭐
2. MySQL Enterprise Audit (propriétaire, MySQL commercial)
3. Percona Audit Log Plugin

**Cas d'usage** :
- ✅ **Conformité PCI-DSS, RGPD, HIPAA** ⭐
- ✅ Détection intrusions
- ✅ Enquêtes forensiques
- ✅ Audit interne

---

## Logs standards vs Audit

### Tableau comparatif

| Aspect | Error Log | General Log | Slow Log | Binary Log | **Audit Log** 🎯 |
|--------|-----------|-------------|----------|------------|------------------|
| **Objectif** | Erreurs serveur | Debug | Performance | Réplication | **Sécurité** |
| **Impact perf** | Négligeable | **Très élevé** | Faible | Faible | **Moyen** |
| **Taille fichier** | Petite | **Énorme** | Moyenne | Moyenne | Moyenne |
| **Format** | Texte | Texte | Texte | Binaire | **JSON/XML** |
| **Connexions** | Non | Oui | Non | Non | **Oui** |
| **Échecs auth** | Non | Oui | Non | Non | **Oui** |
| **Requêtes** | Non | Toutes | Lentes | Modifications | **Configurables** |
| **Filtrage** | Non | Non | Non | Non | **Oui** ✓ |
| **Conformité** | ❌ | ❌ | ❌ | ❌ | **✅** |
| **Production** | ✅ | ❌ | ✅ | ✅ | **✅** |

### Pourquoi les logs standards ne suffisent pas ?

**Problème 1 : General log trop lourd**

```
Production: 1000 requêtes/seconde
General log: 86 400 000 requêtes/jour
Taille: ~50 Go/jour (texte)

→ Impact performance: -50% throughput
→ Coût stockage: 1.5 To/mois
→ Impossible à analyser manuellement
```

**Problème 2 : Logs non structurés**

```
Error log:
2025-12-13 14:22:43 123 [Warning] Aborted connection...

General log:
2025-12-13 10:15:23   45 Query     SELECT * FROM users...

→ Formats différents
→ Pas de JSON/XML
→ Intégration SIEM difficile
```

**Problème 3 : Pas de filtrage**

```
Besoin: Auditer uniquement les accès à la table 'credit_cards'

General log: TOUT enregistré (clients, produits, logs, etc.)
→ Impossible de filtrer par table
→ Noyé dans les données inutiles
```

**Solution : Audit Log dédié** 🎯

```
Server Audit Plugin:
✓ Filtrage par utilisateur, base, table, requête
✓ Format structuré (JSON, XML)
✓ Performance optimisée
✓ Impact mesuré (~5-15% selon config)
✓ Conforme PCI-DSS, RGPD, HIPAA
```

---

## Architecture du système d'audit

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                   MariaDB Server                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Événements serveur                                 │  │
│  │     - Connexions (succès/échecs)                       │  │
│  │     - Requêtes SQL                                     │  │
│  │     - Modifications privilèges                         │  │
│  │     - Arrêts/démarrages                                │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  2. Plugin d'audit (server_audit.so)                   │  │
│  │     - Interception événements                          │  │
│  │     - Filtrage (utilisateurs, bases, tables)           │  │
│  │     - Formatage (JSON, XML, CSV)                       │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  3. Écriture fichier log                               │  │
│  │     - Buffer en mémoire                                │  │
│  │     - Flush asynchrone sur disque                      │  │
│  │     - Rotation automatique                             │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  4. Fichiers d'audit                                         │
│     /var/log/mysql/audit.log                                 │
│     /var/log/mysql/audit.log.1                               │
│     /var/log/mysql/audit.log.2                               │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  5. Analyse et monitoring                                    │
│     - SIEM (Splunk, ELK, Datadog)                            │
│     - Alertes temps réel                                     │
│     - Dashboards sécurité                                    │
│     - Archivage long terme                                   │
└──────────────────────────────────────────────────────────────┘
```

### Flux d'un événement audité

```
Étape 1: Événement se produit
┌──────────────────────────────────────────┐
│ Client exécute:                          │
│ DELETE FROM customers WHERE id = 123;    │
└──────────────────────────────────────────┘
                ↓
Étape 2: Plugin intercepte
┌──────────────────────────────────────────┐
│ Server Audit Plugin détecte:             │
│ - Type: QUERY                            │
│ - User: contractor_temp@10.0.0.15        │
│ - Database: production                   │
│ - Query: DELETE FROM customers...        │
└──────────────────────────────────────────┘
                ↓
Étape 3: Filtrage
┌──────────────────────────────────────────┐
│ Vérification règles:                     │
│ ✓ Utilisateur dans liste audit ?         │
│ ✓ Base de données concernée ?            │
│ ✓ Type requête (DELETE) audité ?         │
│ → OUI, enregistrer                       │
└──────────────────────────────────────────┘
                ↓
Étape 4: Formatage
┌──────────────────────────────────────────┐
│ Conversion JSON:                         │
│ {                                        │
│   "timestamp": "2025-12-13T17:45:23Z",   │
│   "event": "QUERY",                      │
│   "user": "contractor_temp@10.0.0.15",   │
│   "query": "DELETE FROM customers...",   │
│   "status": "SUCCESS"                    │
│ }                                        │
└──────────────────────────────────────────┘
                ↓
Étape 5: Écriture log
┌──────────────────────────────────────────┐
│ Buffer mémoire (async) → Flush disque    │
│ /var/log/mysql/audit.log                 │
└──────────────────────────────────────────┘
```

---

## Plugins d'audit disponibles

### 1. MariaDB Server Audit Plugin ⭐ (Recommandé)

**Description** : Plugin officiel MariaDB (open source).

**Avantages** :
- ✅ **Gratuit** et open source
- ✅ Inclus dans MariaDB 10.0+
- ✅ Performance optimisée
- ✅ Formats multiples (JSON, XML, CSV)
- ✅ Filtrage granulaire
- ✅ **Conforme PCI-DSS**

**Inconvénients** :
- ❌ Moins de fonctionnalités que MySQL Enterprise Audit
- ❌ Pas de filtrage par requête SQL complexe

**Installation** :

```sql
INSTALL SONAME 'server_audit';
```

**Cas d'usage** :
- ✅ PME et startups
- ✅ Conformité PCI-DSS, RGPD
- ✅ Audit général de sécurité

### 2. MySQL Enterprise Audit (propriétaire)

**Description** : Plugin commercial MySQL (licence payante).

**Avantages** :
- ✅ Filtrage très avancé (règles SQL complexes)
- ✅ Compression logs intégrée
- ✅ Chiffrement logs intégré
- ✅ Support Oracle

**Inconvénients** :
- ❌ **Payant** (licence MySQL Enterprise)
- ❌ Propriétaire (pas open source)
- ❌ Moins performant que Server Audit sur MariaDB

**Cas d'usage** :
- MySQL Enterprise Edition
- Grandes entreprises avec budget
- Exigences filtrage très avancé

### 3. Percona Audit Log Plugin

**Description** : Plugin Percona Server (fork MySQL).

**Avantages** :
- ✅ Gratuit (open source)
- ✅ Compatible MySQL et Percona Server
- ✅ Format JSON

**Inconvénients** :
- ❌ Moins maintenu que Server Audit
- ❌ Compatibilité MariaDB variable

**Cas d'usage** :
- Percona Server
- Migration MySQL → Percona

### Comparaison

| Fonctionnalité | Server Audit ⭐ | MySQL Enterprise | Percona Audit |
|----------------|----------------|------------------|---------------|
| **Licence** | Open source (GPL) | **Propriétaire** | Open source |
| **Coût** | **Gratuit** | Payant | Gratuit |
| **Performance** | **Excellente** | Bonne | Bonne |
| **MariaDB** | **Natif** | Compatible | Compatible |
| **MySQL** | Compatible | **Natif** | Compatible |
| **Formats** | JSON, XML, CSV | JSON, XML | JSON |
| **Filtrage** | Utilisateur, DB, table | **SQL avancé** | Utilisateur, DB |
| **PCI-DSS** | ✅ | ✅ | ✅ |

**Recommandation** : **MariaDB Server Audit Plugin** pour la majorité des cas.

---

## Que faut-il auditer ?

### Matrice d'audit par conformité

| Événement | PCI-DSS | RGPD | HIPAA | SOC2 | Recommandation |
|-----------|---------|------|-------|------|----------------|
| **Connexions réussies** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **Connexions échouées** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **Requêtes SELECT** | ✅* | ✅* | ✅* | ❌ | **Tables sensibles** |
| **Requêtes INSERT/UPDATE/DELETE** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **DDL (CREATE/ALTER/DROP)** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **DCL (GRANT/REVOKE)** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **Modifications privilèges** | ✅ | ✅ | ✅ | ✅ | **Toujours** |
| **Démarrages/arrêts serveur** | ✅ | ❌ | ✅ | ✅ | **Toujours** |

*Uniquement pour tables contenant des données sensibles (PAN, PII, PHI)

### Configuration par niveau de sécurité

#### Niveau 1 : Audit minimal (dev/test)

```
Auditer:
✓ Connexions (succès + échecs)
✓ Modifications privilèges (GRANT/REVOKE)

Ne pas auditer:
✗ Requêtes SELECT (trop verbeux)
✗ Requêtes DML courantes

Impact performance: ~2-5%
Taille logs: ~100 Mo/jour
```

#### Niveau 2 : Audit standard (production)

```
Auditer:
✓ Connexions (succès + échecs)
✓ Toutes les requêtes DML (INSERT/UPDATE/DELETE)
✓ Toutes les requêtes DDL (CREATE/ALTER/DROP)
✓ Toutes les requêtes DCL (GRANT/REVOKE)

Ne pas auditer:
✗ SELECT (sauf tables sensibles)

Impact performance: ~5-10%
Taille logs: ~500 Mo/jour
```

#### Niveau 3 : Audit complet (PCI-DSS, bancaire)

```
Auditer:
✓ TOUT (connexions, toutes requêtes, démarrages)
✓ Y compris SELECT sur toutes les tables

Impact performance: ~10-15%
Taille logs: ~2-5 Go/jour
Compression recommandée
```

### Tables sensibles à auditer prioritairement

```sql
-- Données de cartes bancaires (PCI-DSS)
credit_cards, payments, transactions

-- Données personnelles (RGPD)
users, customers, employees, addresses

-- Données de santé (HIPAA)
patients, medical_records, prescriptions

-- Données financières (SOX)
accounts, invoices, financial_statements

-- Métadonnées sécurité
mysql.user, mysql.db, mysql.tables_priv
```

---

## Rétention et rotation des logs

### Exigences réglementaires

| Réglementation | Rétention minimum | Rétention online | Archivage |
|----------------|-------------------|------------------|-----------|
| **PCI-DSS 4.0** | 1 an | 3 mois | 9 mois (hors ligne) |
| **RGPD** | Proportionnel | Variable | Selon politique |
| **HIPAA** | 6 ans | 1 an | 5 ans |
| **SOC 2** | 1 an | 1 an | - |
| **SOX** | 7 ans | 1 an | 6 ans |

### Stratégie de rotation

**Rotation par taille** :

```ini
[mysqld]
server_audit_file_rotate_size = 100M
server_audit_file_rotations = 10

# Résultat: 10 fichiers × 100 Mo = 1 Go max
# audit.log (actif)
# audit.log.1
# audit.log.2
# ...
# audit.log.10 (plus ancien, supprimé lors de la rotation)
```

**Rotation par temps** (via logrotate) :

```bash
# /etc/logrotate.d/mariadb-audit
/var/log/mysql/audit.log {
    daily                # Rotation quotidienne
    rotate 90            # Garder 90 jours (3 mois)
    compress             # Compresser avec gzip
    delaycompress        # Compresser J-1 (pas le jour même)
    notifempty           # Ne pas tourner si vide
    missingok            # Pas d'erreur si fichier absent
    postrotate
        # Signal MariaDB de rouvrir le fichier
        systemctl reload mariadb
    endscript
}
```

### Archivage long terme

**Stratégie 3-2-1** :

```
3 copies des logs
2 supports différents (disque + cloud)
1 copie hors site

Exemple:
┌─────────────────────────────────────────────┐
│ 1. Logs actifs (3 mois)                     │
│    → /var/log/mysql/audit.log.*             │
│    → Disque local SSD                       │
└─────────────────────────────────────────────┘
                ↓ (rotation quotidienne)
┌─────────────────────────────────────────────┐
│ 2. Archive court terme (9 mois)             │
│    → /backup/audit/2025-12-*.log.gz         │
│    → NAS local                              │
└─────────────────────────────────────────────┘
                ↓ (archivage mensuel)
┌─────────────────────────────────────────────┐
│ 3. Archive long terme (1-7 ans)             │
│    → s3://audit-logs/2025/12/*.log.gz       │
│    → AWS S3 Glacier / Azure Archive         │
│    → Immuable (WORM - Write Once Read Many) │
└─────────────────────────────────────────────┘
```

**Script d'archivage** :

```bash
#!/bin/bash
# archive_audit_logs.sh - Archivage mensuel vers S3

AUDIT_DIR="/var/log/mysql"
ARCHIVE_DIR="/backup/audit"
S3_BUCKET="s3://company-audit-logs"
RETENTION_DAYS=90  # Garder 90 jours local

# Compresser et archiver logs > 30 jours
find $AUDIT_DIR -name "audit.log.*" -mtime +30 -exec gzip {} \;
find $AUDIT_DIR -name "audit.log.*.gz" -mtime +30 -exec mv {} $ARCHIVE_DIR/ \;

# Upload vers S3 (logs > 90 jours)
aws s3 sync $ARCHIVE_DIR/ $S3_BUCKET/ \
  --storage-class GLACIER \
  --exclude "*" \
  --include "*.log.gz"

# Supprimer local après upload réussi
find $ARCHIVE_DIR -name "*.log.gz" -mtime +$RETENTION_DAYS -delete

# Notification
echo "Audit logs archived: $(date)" | mail -s "Audit Archive" admin@example.com
```

---

## Impact performance

### Mesures d'impact typiques

| Configuration | Impact CPU | Impact I/O | Impact Latence | Taille logs |
|---------------|------------|------------|----------------|-------------|
| **Audit désactivé** | 0% | 0% | 0 ms | 0 |
| **Connexions seulement** | +1-2% | +5% | +0.1 ms | ~10 Mo/jour |
| **DML (INSERT/UPDATE/DELETE)** | +3-5% | +10% | +0.3 ms | ~100 Mo/jour |
| **DDL + DCL** | +2% | +5% | +0.2 ms | ~20 Mo/jour |
| **Tout sauf SELECT** | +5-8% | +15% | +0.5 ms | ~500 Mo/jour |
| **TOUT (SELECT inclus)** | +10-15% | +25% | +1 ms | ~5 Go/jour |

### Benchmark réel

**Scénario** : Serveur production 1000 req/s

```
Configuration 1: Audit désactivé
  → Throughput: 1000 req/s
  → Latence P95: 10 ms
  → CPU: 40%

Configuration 2: Audit connexions + DML/DDL
  → Throughput: 950 req/s (-5%)
  → Latence P95: 10.5 ms (+5%)
  → CPU: 43% (+3%)
  → Logs: 450 Mo/jour

Configuration 3: Audit COMPLET (SELECT inclus)
  → Throughput: 850 req/s (-15%)
  → Latence P95: 12 ms (+20%)
  → CPU: 48% (+8%)
  → Logs: 4.2 Go/jour
```

💡 **Recommandation** : Auditer connexions + DML/DDL (impact acceptable ~5%).

### Optimisations

**1. Buffer en mémoire** :

```ini
[mysqld]
server_audit_file_rotate_size = 100M
# Buffer 1 Mo en RAM avant flush disque
```

**2. Disque dédié pour logs** :

```bash
# Logs audit sur SSD séparé (pas sur même disque que data)
/dev/sdb1 → /var/log/mysql/  (SSD)
/dev/sda1 → /var/lib/mysql/  (SSD data)
```

**3. Filtrage intelligent** :

```ini
# N'auditer QUE les utilisateurs sensibles
server_audit_incl_users = 'admin,root,contractor_%'

# N'auditer QUE les bases sensibles
server_audit_excl_databases = 'test,dev,temp'
```

**4. Compression asynchrone** :

```bash
# Compresser logs en arrière-plan (cron)
*/15 * * * * gzip /var/log/mysql/audit.log.* 2>/dev/null
```

---

## Intégration SIEM

### Architecture SIEM

```
┌──────────────────────────────────────────────────────────────┐
│              MariaDB Server                                  │
│  /var/log/mysql/audit.log (JSON)                             │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  Agent de collecte (Filebeat, Fluentd, Logstash)             │
│  - Lecture fichier log                                       │
│  - Parsing JSON                                              │
│  - Enrichissement (hostname, environnement)                  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│  SIEM (Splunk, ELK, Datadog, Azure Sentinel)                 │
│  - Indexation                                                │
│  - Corrélation événements                                    │
│  - Alertes temps réel                                        │
│  - Dashboards sécurité                                       │
└──────────────────────────────────────────────────────────────┘
```

### Filebeat (ELK Stack)

**Configuration** :

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/mysql/audit.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      source: mariadb_audit
      environment: production
      datacenter: eu-west-1

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "mariadb-audit-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"
```

### Splunk

**Configuration** :

```ini
# /opt/splunk/etc/apps/mariadb_audit/local/inputs.conf
[monitor:///var/log/mysql/audit.log]
disabled = false
index = mariadb_audit
sourcetype = mariadb:audit:json
```

**Alertes Splunk** :

```
# Alerte: Tentatives connexion échouées > 10 en 5 min
search = index=mariadb_audit event=FAILED_CONNECT
| stats count by user, host
| where count > 10

# Alerte: DELETE massif (> 1000 lignes)
search = index=mariadb_audit event=QUERY query="DELETE FROM*"
| where rows_affected > 1000
```

### Datadog

**Agent Datadog** :

```yaml
# /etc/datadog-agent/conf.d/mariadb_audit.d/conf.yaml
logs:
  - type: file
    path: /var/log/mysql/audit.log
    service: mariadb
    source: mariadb-audit
    log_processing_rules:
      - type: multi_line
        name: log_start_with_timestamp
        pattern: \d{4}-\d{2}-\d{2}
```

---

## Alertes et détection d'anomalies

### Scénarios d'alertes critiques

**1. Tentatives de connexion brutales** :

```sql
-- Détection brute force
-- > 50 échecs de connexion en 1 minute depuis une IP

Alerte:
  Titre: "Brute Force Attack Detected"
  Sévérité: CRITICAL
  Action: Bloquer IP, notifier SOC
```

**2. Accès hors horaires** :

```sql
-- Connexion admin en dehors 9h-18h

Alerte:
  Titre: "After-hours Admin Access"
  Sévérité: HIGH
  Action: Vérifier légitimité, notifier responsable
```

**3. Modifications massives** :

```sql
-- DELETE/UPDATE > 1000 lignes

Alerte:
  Titre: "Mass Data Modification"
  Sévérité: HIGH
  Action: Investiguer, backup immédiat si malveillant
```

**4. Escalade de privilèges** :

```sql
-- GRANT ALL PRIVILEGES détecté

Alerte:
  Titre: "Privilege Escalation Detected"
  Sévérité: CRITICAL
  Action: Révoquer si non autorisé, notifier CISO
```

**5. Accès tables sensibles** :

```sql
-- SELECT sur table credit_cards par utilisateur non autorisé

Alerte:
  Titre: "Unauthorized Access to Sensitive Data"
  Sévérité: CRITICAL
  Action: Bloquer utilisateur, notification RGPD si requis
```

---

## ✅ Points clés à retenir

- **L'audit est obligatoire pour la conformité** PCI-DSS, RGPD, HIPAA, SOC2
- **5 types de logs MariaDB** : Error, General, Slow, Binary, **Audit** (sécurité)
- **Logs standards ≠ Audit** : General log trop lourd, pas de filtrage, pas conforme
- **Server Audit Plugin est recommandé** : gratuit, performant, conforme PCI-DSS
- **Auditer au minimum** : connexions + DML/DDL/DCL (impact ~5-10%)
- **Rétention réglementaire** : 1 an (PCI-DSS), 6 ans (HIPAA), 7 ans (SOX)
- **Rotation obligatoire** : par taille (100 Mo) et temps (quotidien/mensuel)
- **Archivage 3-2-1** : 3 copies, 2 supports, 1 hors site
- **Impact performance mesuré** : 5-15% selon configuration
- **Intégration SIEM essentielle** : Splunk, ELK, Datadog pour alertes temps réel

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Server Audit Plugin](https://mariadb.com/kb/en/mariadb-audit-plugin/)
- [📖 Log Files Overview](https://mariadb.com/kb/en/log-files/)
- [📖 General Query Log](https://mariadb.com/kb/en/general-query-log/)
- [📖 Binary Log](https://mariadb.com/kb/en/binary-log/)

### Conformité

- [PCI-DSS v4.0 Requirement 10](https://www.pcisecuritystandards.org/)
- [GDPR Article 30 - Records of Processing](https://gdpr-info.eu/)
- [HIPAA Security Rule - Audit Controls](https://www.hhs.gov/hipaa/)

### Outils SIEM

- [ELK Stack](https://www.elastic.co/elk-stack)
- [Splunk](https://www.splunk.com/)
- [Datadog](https://www.datadoghq.com/)

---

## ➡️ Sections suivantes

Les sous-sections détailleront la configuration pratique de l'audit :

- **10.8.1** : Server Audit Plugin (configuration complète)
- **10.8.2** : Filtrage et règles d'audit
- **10.8.3** : Analyse et exploitation des logs
- **10.8.4** : Conformité PCI-DSS/RGPD avec l'audit

**La section suivante (10.8.1)** entrera dans le détail de la **configuration du Server Audit Plugin** avec tous les paramètres et exemples concrets.

---


⏭️ [Server Audit Plugin](/10-securite-gestion-utilisateurs/08.1-server-audit-plugin.md)
