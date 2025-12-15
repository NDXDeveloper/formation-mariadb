🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.2 Migration depuis d'autres SGBD

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 4-5 heures  
> **Prérequis** : Expérience avec au moins un SGBD entreprise (Oracle, SQL Server, PostgreSQL), maîtrise des concepts SQL avancés, connaissance de l'architecture MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Évaluer la complexité d'une migration hétérogène selon le SGBD source
- Identifier les incompatibilités majeures entre dialectes SQL et planifier les adaptations
- Choisir les outils appropriés pour chaque type de migration (Oracle, SQL Server, PostgreSQL)
- Concevoir une stratégie de conversion des procédures stockées et du code PL/SQL ou T-SQL
- Anticiper les différences de comportement et de performance post-migration
- Estimer les efforts et délais réalistes pour un projet de migration hétérogène

---

## Introduction

Migrer depuis MySQL vers MariaDB relève de la chirurgie esthétique : même famille, mêmes organes, ajustements cosmétiques. Migrer depuis Oracle, SQL Server ou PostgreSQL vers MariaDB s'apparente davantage à une transplantation : chaque organe doit être adapté, chaque connexion nerveuse recâblée.

Les migrations hétérogènes représentent des projets d'envergure, souvent motivés par des considérations économiques (coût des licences Oracle ou SQL Server), stratégiques (indépendance vis-à-vis d'un éditeur), ou techniques (modernisation d'une stack legacy). Ces projets s'étendent typiquement sur 6 à 24 mois et mobilisent des équipes pluridisciplinaires : DBA, développeurs, architectes, et souvent des consultants spécialisés.

Cette section établit le cadre général des migrations hétérogènes vers MariaDB. Les sous-sections suivantes détailleront les spécificités de chaque SGBD source.

---

## Taxonomie des migrations hétérogènes

### Axes de complexité

Une migration hétérogène se caractérise selon plusieurs dimensions qui déterminent sa complexité globale :

```
                         COMPLEXITÉ CROISSANTE
                              ────────────▶

    ┌─────────────────────────────────────────────────────────┐
    │                      DONNÉES                            │
    │  Types simples ──▶ Types propriétaires ──▶ LOB/Spatial  │
    └─────────────────────────────────────────────────────────┘
    
    ┌─────────────────────────────────────────────────────────┐
    │                      SCHÉMA                             │
    │  Tables basiques ──▶ Contraintes ──▶ Partitionnement    │
    └─────────────────────────────────────────────────────────┘
    
    ┌─────────────────────────────────────────────────────────┐
    │                   CODE PROCÉDURAL                       │
    │  Fonctions simples ──▶ Packages ──▶ Logique métier      │
    └─────────────────────────────────────────────────────────┘
    
    ┌─────────────────────────────────────────────────────────┐
    │                   FONCTIONNALITÉS                       │
    │  SQL standard ──▶ Extensions ──▶ Features propriétaires │
    └─────────────────────────────────────────────────────────┘
```

### Matrice de difficulté par source

| SGBD Source | Difficulté globale | Données | Schéma | Code procédural | SQL |
|-------------|-------------------|---------|--------|-----------------|-----|
| **PostgreSQL** | ⭐⭐⭐ Moyenne | 🟡 | 🟢 | 🟡 | 🟡 |
| **SQL Server** | ⭐⭐⭐ Moyenne | 🟢 | 🟡 | 🔴 | 🟡 |
| **Oracle** | ⭐⭐⭐⭐ Élevée | 🟡 | 🟡 | 🔴 | 🔴 |
| **DB2** | ⭐⭐⭐⭐ Élevée | 🟡 | 🟡 | 🔴 | 🟡 |
| **Sybase** | ⭐⭐⭐ Moyenne | 🟢 | 🟢 | 🟡 | 🟢 |

🟢 Faible effort | 🟡 Effort modéré | 🔴 Effort significatif

---

## Défis communs aux migrations hétérogènes

Quel que soit le SGBD source, certains défis sont universels.

### 1. Mapping des types de données

Chaque SGBD possède ses propres types de données, parfois sans équivalent direct.

| Concept | Oracle | SQL Server | PostgreSQL | MariaDB |
|---------|--------|------------|------------|---------|
| **Entier auto** | SEQUENCE | IDENTITY | SERIAL | AUTO_INCREMENT |
| **GUID** | RAW(16) | UNIQUEIDENTIFIER | UUID | CHAR(36) ou UUID() |
| **Booléen** | NUMBER(1) | BIT | BOOLEAN | TINYINT(1) ou BOOLEAN |
| **Texte long** | CLOB | NVARCHAR(MAX) | TEXT | LONGTEXT |
| **Binaire long** | BLOB | VARBINARY(MAX) | BYTEA | LONGBLOB |
| **Date seule** | DATE (avec heure!) | DATE | DATE | DATE |
| **Timestamp** | TIMESTAMP | DATETIME2 | TIMESTAMP | DATETIME/TIMESTAMP |
| **Intervalle** | INTERVAL | - | INTERVAL | - (calculé) |
| **JSON** | JSON (12c+) | NVARCHAR + JSON | JSONB | JSON |
| **Array** | VARRAY/Nested Table | - | ARRAY[] | JSON (simulation) |
| **Géospatial** | SDO_GEOMETRY | GEOMETRY | PostGIS | GEOMETRY |

⚠️ **Attention aux DATE Oracle** : En Oracle, le type DATE inclut l'heure (jusqu'à la seconde). En MariaDB, DATE est une date pure. Utilisez DATETIME pour une conversion correcte.

### 2. Différences de comportement SQL

Des requêtes syntaxiquement valides peuvent produire des résultats différents selon le SGBD.

**Gestion des NULL dans les comparaisons :**

```sql
-- Oracle : NULL = NULL → NULL (ni vrai ni faux)
-- Mais dans les index UNIQUE, deux NULL sont distincts

-- SQL Server : Similaire, mais configurable via ANSI_NULLS

-- PostgreSQL : NULL = NULL → NULL
-- NULLS DISTINCT/NULLS NOT DISTINCT pour les contraintes UNIQUE (v15+)

-- MariaDB : NULL = NULL → NULL
-- Deux NULL sont distincts dans les contraintes UNIQUE
```

**Division par zéro :**

```sql
-- Oracle : Erreur ORA-01476
SELECT 10/0 FROM dual;

-- SQL Server : Erreur par défaut, NULL si ANSI_WARNINGS OFF
SELECT 10/0;

-- PostgreSQL : Erreur "division by zero"
SELECT 10/0;

-- MariaDB : Retourne NULL (mode SQL par défaut)
-- Avec ERROR_FOR_DIVISION_BY_ZERO : warning ou erreur selon STRICT_*
SELECT 10/0;  -- Retourne NULL
```

**Concaténation de chaînes :**

```sql
-- Oracle : ||
SELECT 'Hello' || ' ' || 'World' FROM dual;

-- SQL Server : +
SELECT 'Hello' + ' ' + 'World';

-- PostgreSQL : || ou CONCAT()
SELECT 'Hello' || ' ' || 'World';

-- MariaDB : CONCAT() ou || (si PIPES_AS_CONCAT activé)
SELECT CONCAT('Hello', ' ', 'World');
-- Ou avec sql_mode incluant PIPES_AS_CONCAT :
SELECT 'Hello' || ' ' || 'World';
```

### 3. Conversion du code procédural

C'est généralement le défi le plus chronophage des migrations hétérogènes.

**Comparaison des langages procéduraux :**

| Aspect | PL/SQL (Oracle) | T-SQL (SQL Server) | PL/pgSQL | MariaDB SQL/PSM |
|--------|-----------------|--------------------|-----------| ----------------|
| **Packages** | ✅ Natif | ❌ Non | ❌ Non | ❌ Non |
| **Curseurs** | Riches | Basiques | Riches | Basiques |
| **Exceptions** | EXCEPTION block | TRY...CATCH | EXCEPTION block | HANDLER |
| **Collections** | Nombreuses | TABLE variables | ARRAY | ❌ Limitées |
| **Bulk operations** | FORALL, BULK COLLECT | Table-valued params | UNNEST | ❌ Boucles |
| **Triggers INSTEAD OF** | ✅ | ✅ | ✅ | ❌ |
| **Autonomous transactions** | ✅ PRAGMA | ✅ | ❌ | ❌ |

💡 **Conseil** : MariaDB 10.3+ supporte un mode de compatibilité PL/SQL (sql_mode='ORACLE') qui facilite la migration du code Oracle simple, mais les packages et fonctionnalités avancées nécessitent une réécriture.

### 4. Fonctionnalités sans équivalent direct

Certaines fonctionnalités n'existent tout simplement pas dans MariaDB et nécessitent des approches alternatives.

| Fonctionnalité source | Alternative MariaDB |
|-----------------------|---------------------|
| **Oracle : Materialized Views** | Vues + tables + triggers/events |
| **Oracle : Database Links** | CONNECT engine ou FEDERATED |
| **Oracle : Flashback** | System-Versioned Tables |
| **Oracle : Advanced Queuing** | Kafka/RabbitMQ externe |
| **SQL Server : Always Encrypted** | Encryption at rest + applicatif |
| **SQL Server : Change Tracking** | Triggers + table d'audit |
| **SQL Server : In-Memory OLTP** | Memory engine (limité) |
| **PostgreSQL : LISTEN/NOTIFY** | Polling ou message queue externe |
| **PostgreSQL : Extensions** | Plugins MariaDB ou alternatives |
| **PostgreSQL : Row-Level Security** | Vues avec filtrage |

---

## Méthodologie de migration hétérogène

### Phase 1 : Évaluation et découverte (2-4 semaines)

L'évaluation initiale détermine la faisabilité et le coût du projet.

```
┌─────────────────────────────────────────────────────────────┐
│                    PHASE D'ÉVALUATION                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. INVENTAIRE TECHNIQUE                                    │
│     • Schémas, tables, vues, index                          │
│     • Procédures, fonctions, packages, triggers             │
│     • Taille des données et volumétrie                      │
│     • Dépendances inter-schémas                             │
│                                                             │
│  2. ANALYSE DE COMPATIBILITÉ                                │
│     • Types de données à mapper                             │
│     • Fonctions SQL propriétaires utilisées                 │
│     • Features sans équivalent                              │
│     • Code procédural à convertir                           │
│                                                             │
│  3. ESTIMATION DES EFFORTS                                  │
│     • Conversion automatisable vs manuelle                  │
│     • Tests de régression requis                            │
│     • Formation des équipes                                 │
│     • Refactoring applicatif                                │
│                                                             │
│  4. DÉCISION GO/NO-GO                                       │
│     • ROI de la migration                                   │
│     • Risques identifiés                                    │
│     • Planning et ressources                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Script d'évaluation générique :**

```sql
-- Requête adaptable à chaque SGBD source
-- Inventaire des objets à migrer

-- Pour Oracle :
SELECT object_type, COUNT(*) as count
FROM all_objects
WHERE owner = 'MON_SCHEMA'
GROUP BY object_type
ORDER BY count DESC;

-- Pour SQL Server :
SELECT type_desc, COUNT(*) as count
FROM sys.objects
WHERE schema_id = SCHEMA_ID('dbo')
GROUP BY type_desc
ORDER BY count DESC;

-- Pour PostgreSQL :
SELECT 
    CASE 
        WHEN relkind = 'r' THEN 'TABLE'
        WHEN relkind = 'v' THEN 'VIEW'
        WHEN relkind = 'i' THEN 'INDEX'
        WHEN relkind = 'S' THEN 'SEQUENCE'
    END as object_type,
    COUNT(*) as count
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public'
GROUP BY relkind;
```

### Phase 2 : Conception et prototypage (4-8 semaines)

Cette phase définit l'architecture cible et valide les choix techniques.

**Livrables clés :**

1. **Document de mapping des types** : Correspondance exhaustive source → MariaDB
2. **Catalogue des conversions SQL** : Fonctions propriétaires et leurs équivalents
3. **Stratégie de conversion du code** : Automatisée vs manuelle, outils utilisés
4. **Architecture cible** : Schéma MariaDB, partitionnement, indexation
5. **Prototype fonctionnel** : Migration d'un sous-ensemble représentatif

**Exemple de document de mapping (Oracle → MariaDB) :**

```yaml
# mapping_types_oracle_mariadb.yaml

numeric_types:
  NUMBER:
    default: DECIMAL(38,10)
    NUMBER(p): DECIMAL(p,0)
    NUMBER(p,s): DECIMAL(p,s)
    NUMBER(1): TINYINT  # Pour booléens
  BINARY_FLOAT: FLOAT
  BINARY_DOUBLE: DOUBLE

string_types:
  VARCHAR2(n): VARCHAR(n)
  CHAR(n): CHAR(n)
  NVARCHAR2(n): VARCHAR(n) CHARACTER SET utf8mb4
  CLOB: LONGTEXT

date_types:
  DATE: DATETIME  # Attention : DATE Oracle inclut l'heure !
  TIMESTAMP: DATETIME(6)
  TIMESTAMP WITH TIME ZONE: DATETIME(6)  # + gestion TZ applicative
  INTERVAL YEAR TO MONTH: VARCHAR(50)  # Conversion manuelle
  INTERVAL DAY TO SECOND: VARCHAR(50)  # Conversion manuelle

lob_types:
  BLOB: LONGBLOB
  CLOB: LONGTEXT
  NCLOB: LONGTEXT CHARACTER SET utf8mb4
  BFILE: LONGBLOB  # Contenu externalisé à migrer

special_types:
  RAW(16): BINARY(16)  # Pour UUID
  ROWID: Pas d'équivalent direct
  XMLType: LONGTEXT  # Ou JSON avec conversion
  SDO_GEOMETRY: GEOMETRY  # Avec conversion format
```

### Phase 3 : Développement et conversion (8-16 semaines)

La phase de développement transforme le schéma, le code et prépare la migration des données.

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  CONVERSION      │     │  CONVERSION      │     │  CONVERSION      │
│  SCHÉMA          │────▶│  CODE            │────▶│  DONNÉES         │
│                  │     │                  │     │                  │
│ • DDL            │     │ • Procédures     │     │ • ETL pipeline   │
│ • Constraints    │     │ • Fonctions      │     │ • Validation     │
│ • Index          │     │ • Triggers       │     │ • Checksums      │
│ • Partitions     │     │ • Views          │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │   TESTS DE RÉGRESSION    │
                    │                          │
                    │ • Fonctionnels           │
                    │ • Performance            │
                    │ • Intégration            │
                    └──────────────────────────┘
```

### Phase 4 : Tests et validation (4-8 semaines)

Les tests sont critiques pour une migration hétérogène. Le comportement peut différer subtilement même avec une conversion "réussie".

**Niveaux de tests :**

| Niveau | Objectif | Méthode |
|--------|----------|---------|
| **Unitaire** | Chaque objet converti | Jeux de tests par procédure |
| **Intégration** | Interactions entre objets | Scénarios métier |
| **Données** | Intégrité post-migration | Checksums, comptages, échantillons |
| **Performance** | SLA respectés | Benchmarks, profiling |
| **Charge** | Comportement sous stress | Tests de charge réalistes |
| **Régression** | Pas de régression fonctionnelle | Suite de tests applicatifs |

### Phase 5 : Migration et bascule (1-4 semaines)

La migration effective suit généralement ce schéma :

```
J-7     J-3     J-1     J-Day   J+1     J+7     J+30
 │       │       │        │       │       │        │
 ▼       ▼       ▼        ▼       ▼       ▼        ▼
┌───┐   ┌───┐   ┌───┐   ┌────┐  ┌───┐   ┌───┐   ┌────┐
│Dry│   │Dry│   │Gel │  │CUT-│  │Mon│   │Sta│   │Déc-│
│run│   │run│   │Code│  │OVER│  │ito│   │bil│   │omm.│
│ 1 │   │ 2 │   │    │  │    │  │r  │   │   │   │    │
└───┘   └───┘   └───┘   └────┘  └───┘   └───┘   └────┘
```

---

## Outils de migration hétérogène

### Outils commerciaux

| Outil | Éditeur | Sources supportées | Points forts |
|-------|---------|-------------------|--------------|
| **AWS SCT** | Amazon | Oracle, SQL Server, PostgreSQL, DB2 | Gratuit, intégré AWS DMS |
| **SQLines** | SQLines | Oracle, SQL Server, DB2, Sybase | Conversion SQL automatisée |
| **Ispirer MnMTK** | Ispirer | 20+ SGBD | Très complet, conversion code |
| **dbForge Studio** | Devart | Multi-SGBD | IDE complet, comparaison |
| **Full Convert** | Spectral Core | 40+ sources | Simple, rapide |

### Outils open source

| Outil | Spécialité | Licence |
|-------|------------|---------|
| **ora2pg** | Oracle → PostgreSQL/MariaDB | GPL |
| **pgloader** | Multi-source → PostgreSQL/MariaDB | PostgreSQL |
| **SQLFairy** | Conversion DDL multi-SGBD | GPL |
| **Apache DolphinScheduler** | ETL/orchestration | Apache 2.0 |
| **Pentaho Data Integration** | ETL visuel | Apache 2.0 |

### AWS Database Migration Service (DMS)

AWS DMS est particulièrement efficace pour les migrations vers MariaDB, avec support natif de nombreuses sources.

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   Source DB     │         │   AWS DMS       │         │   MariaDB       │
│                 │         │                 │         │                 │
│ • Oracle        │────────▶│ • Full Load     │────────▶│ • RDS MariaDB   │
│ • SQL Server    │         │ • CDC (ongoing) │         │ • EC2 MariaDB   │
│ • PostgreSQL    │         │ • Validation    │         │ • On-premise    │
│ • MySQL         │         │                 │         │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

**Avantages AWS DMS :**
- Migration continue avec Change Data Capture (CDC)
- Validation automatique des données
- Support des transformations basiques
- Intégration avec AWS Schema Conversion Tool (SCT)

**Limitations :**
- Conversion du code procédural limitée (utiliser SCT séparément)
- Certains types de données nécessitent des transformations manuelles
- Coût potentiellement élevé pour gros volumes

---

## Considérations spécifiques MariaDB 11.8 🆕

MariaDB 11.8 LTS apporte des fonctionnalités facilitant certaines migrations hétérogènes.

### MariaDB Vector pour les migrations de bases analytiques

Si votre base source contient des données vectorielles ou si vous prévoyez d'ajouter des capacités de recherche sémantique, MariaDB 11.8 offre désormais le type VECTOR natif.

```sql
-- Création d'une table avec colonne vectorielle
CREATE TABLE documents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    embedding VECTOR(1536) NOT NULL,  -- Dimension OpenAI ada-002
    VECTOR INDEX idx_embedding (embedding) 
        DISTANCE_FUNCTION=COSINE
);

-- Recherche par similarité
SELECT title, VEC_DISTANCE_COSINE(embedding, @query_vector) AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

### utf8mb4 par défaut : simplification des migrations

Avec utf8mb4 comme charset par défaut, les migrations depuis des sources Unicode (PostgreSQL, Oracle AL32UTF8, SQL Server avec NVARCHAR) sont simplifiées.

```sql
-- MariaDB 11.8 : Plus besoin de spécifier explicitement
CREATE TABLE imported_data (
    name VARCHAR(255),  -- Automatiquement utf8mb4
    description TEXT    -- Automatiquement utf8mb4
);

-- Vérification
SHOW CREATE TABLE imported_data\G
```

### Collations UCA 14.0.0 : meilleure compatibilité Unicode

Les nouvelles collations `utf8mb4_uca1400_*` offrent un tri Unicode plus conforme aux standards actuels, facilitant la compatibilité avec les comportements de tri d'Oracle et PostgreSQL.

```sql
-- Utilisation des nouvelles collations
CREATE TABLE contacts (
    name VARCHAR(100) COLLATE utf8mb4_uca1400_ai_ci
);

-- Comparaison des tris
SELECT name FROM contacts ORDER BY name COLLATE utf8mb4_uca1400_as_cs;
```

---

## Estimation des efforts

### Facteurs de complexité

L'effort de migration dépend de nombreux facteurs. Voici une grille d'estimation :

| Facteur | Impact faible | Impact moyen | Impact élevé |
|---------|---------------|--------------|--------------|
| **Volume de données** | < 100 GB | 100 GB - 1 TB | > 1 TB |
| **Nombre de tables** | < 100 | 100 - 500 | > 500 |
| **Procédures stockées** | < 20 | 20 - 100 | > 100 |
| **Complexité SQL** | SQL standard | Extensions modérées | Heavily proprietary |
| **Intégrations** | Standalone | Quelques interfaces | Écosystème complexe |
| **Criticité** | Dev/Test | Production non critique | Mission critical |

### Formule d'estimation (ordre de grandeur)

```
Effort (jours-homme) = 
    Base × Facteur_Source × Facteur_Volume × Facteur_Code × Facteur_Risque

Où :
- Base = 20 jours (minimum incompressible)
- Facteur_Source : Oracle=2.5, SQL Server=1.8, PostgreSQL=1.3
- Facteur_Volume : <100GB=1, 100GB-1TB=1.5, >1TB=2.5
- Facteur_Code : <20 procédures=1, 20-100=2, >100=3
- Facteur_Risque : Dev=1, Prod non critique=1.5, Mission critical=2
```

**Exemple : Migration Oracle 500GB, 80 procédures, production critique**

```
Effort = 20 × 2.5 × 1.5 × 2 × 2 = 300 jours-homme
Soit environ 15 mois avec une équipe de 2 personnes
```

---

## Risques et mitigations

### Risques techniques

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| Incompatibilité SQL non détectée | Moyenne | Élevé | Tests exhaustifs, sandbox |
| Dégradation performance | Élevée | Moyen | Benchmarks précoces, tuning |
| Perte de données migration | Faible | Critique | Checksums, validation |
| Code procédural incorrect | Moyenne | Élevé | Tests unitaires, revue code |
| Intégrations cassées | Moyenne | Élevé | Tests d'intégration E2E |

### Risques projet

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| Dépassement délais | Élevée | Moyen | Buffer 30%, jalons intermédiaires |
| Dépassement budget | Moyenne | Moyen | Estimation conservatrice, suivi |
| Résistance au changement | Moyenne | Moyen | Communication, formation |
| Perte de compétences | Faible | Élevé | Documentation, knowledge transfer |
| Rollback impossible | Faible | Critique | Tests rollback, période parallèle |

---

## Patterns d'architecture post-migration

### Pattern 1 : Migration complète (Rip and Replace)

```
AVANT                           APRÈS
┌─────────────────┐            ┌─────────────────┐
│   Oracle/       │            │    MariaDB      │
│   SQL Server    │  ────────▶ │                 │
│                 │            │                 │
└─────────────────┘            └─────────────────┘
        │                              │
        ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│  Applications   │            │  Applications   │
│  (modifiées)    │            │  (adaptées)     │
└─────────────────┘            └─────────────────┘
```

**Avantages** : Architecture simple, pas de dette technique  
**Inconvénients** : Risque élevé, big bang

### Pattern 2 : Strangler Fig (Migration progressive)

```
PHASE 1                    PHASE 2                    PHASE 3
┌─────────┐               ┌─────────┐               ┌─────────┐
│ Source  │               │ Source  │               │ MariaDB │
│  100%   │               │  40%    │               │  100%   │
└─────────┘               └────┬────┘               └─────────┘
                               │                         
                          ┌────┴────┐                    
                          │ MariaDB │                    
                          │  60%    │                    
                          └─────────┘                    

Nouvelles fonctionnalités sur MariaDB
Migration progressive des existantes
```

**Avantages** : Risque dilué, rollback partiel possible  
**Inconvénients** : Complexité opérationnelle, double maintenance

### Pattern 3 : Coexistence avec synchronisation

```
┌─────────────────┐         ┌─────────────────┐
│   Source DB     │◀───────▶│    MariaDB      │
│   (Legacy)      │  Sync   │   (Moderne)     │
└────────┬────────┘ bidirec └────────┬────────┘
         │          tionnel          │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Apps Legacy    │         │  Apps Modernes  │
└─────────────────┘         └─────────────────┘
```

**Avantages** : Transition douce, rollback facile  
**Inconvénients** : Synchronisation complexe, coût double infra

---

## Scénarios de migration réels

### Scénario 1 : Éditeur SaaS — Oracle → MariaDB

**Contexte :**
- Application ERP SaaS multi-tenant
- Oracle 19c, 2 TB de données, 450 tables
- 280 procédures PL/SQL, 50 packages
- Motivation : réduction coûts licences (500k€/an → 0)

**Approche :**
1. Audit avec ora2pg : 4 semaines
2. Conversion schéma : 3 semaines
3. Conversion code PL/SQL : 16 semaines (mode ORACLE MariaDB + réécriture)
4. Migration données avec AWS DMS : 2 semaines
5. Tests : 8 semaines
6. Bascule progressive par tenant : 4 semaines

**Résultat :** 
- Durée totale : 14 mois
- Effort : 420 jours-homme
- ROI atteint en 18 mois

### Scénario 2 : Banque régionale — SQL Server → MariaDB Galera

**Contexte :**
- Application core banking
- SQL Server 2019, 800 GB, 320 tables
- 150 procédures T-SQL, 40 triggers
- Exigence : HA multi-datacenter

**Approche :**
1. Audit et POC : 6 semaines
2. Architecture Galera 3 nœuds : 2 semaines
3. Conversion avec SQLines : 8 semaines
4. Adaptation T-SQL manuelle : 10 semaines
5. Migration avec réplication SQL Server → MariaDB : 3 semaines
6. Validation réglementaire : 4 semaines
7. Bascule avec fallback : 1 semaine

**Résultat :**
- Durée totale : 10 mois
- Disponibilité améliorée (99.99% vs 99.9%)
- Coût TCO réduit de 40%

### Scénario 3 : Startup data — PostgreSQL → MariaDB + Vector 🆕

**Contexte :**
- Application de recommandation e-commerce
- PostgreSQL 15 + pgvector
- 50 GB données, 5M embeddings vectoriels
- Motivation : unification stack sur MariaDB existant

**Approche :**
1. Évaluation compatibilité : 2 semaines
2. Migration schéma avec pgloader : 1 semaine
3. Adaptation requêtes pgvector → MariaDB Vector : 3 semaines
4. Migration données : 1 semaine
5. Tuning index HNSW : 2 semaines
6. Tests performance : 2 semaines

**Résultat :**
- Durée totale : 3 mois
- Performance recherche vectorielle : comparable
- Simplification opérationnelle (1 SGBD au lieu de 2)

---

## ✅ Points clés à retenir

- Les migrations hétérogènes sont des **projets d'envergure** nécessitant 6-24 mois selon la complexité
- Le **mapping des types de données** et la **conversion du code procédural** représentent les défis majeurs
- Chaque SGBD source a ses spécificités : **Oracle** (PL/SQL, packages), **SQL Server** (T-SQL), **PostgreSQL** (extensions, PL/pgSQL)
- L'**évaluation préalable** est critique : inventaire exhaustif, estimation réaliste, décision go/no-go éclairée
- Les **outils automatisés** (ora2pg, AWS SCT/DMS, pgloader) accélèrent significativement les migrations
- MariaDB 11.8 LTS facilite certaines migrations avec **utf8mb4 par défaut**, **UCA 14.0.0**, et **MariaDB Vector** 🆕
- Prévoyez toujours une **période de coexistence** et un **plan de rollback testé**
- Le **ROI** des migrations hétérogènes se calcule sur 3-5 ans (économies licences, maintenance)

---

## 🔗 Ressources et références

- [📖 MariaDB KB : Migrating from Other Database Systems](https://mariadb.com/kb/en/migrating-to-mariadb/)
- [📖 ora2pg Documentation](https://ora2pg.darold.net/documentation.html)
- [📖 pgloader Documentation](https://pgloader.io/)
- [📖 AWS Database Migration Service Guide](https://docs.aws.amazon.com/dms/)
- [📖 AWS Schema Conversion Tool](https://docs.aws.amazon.com/SchemaConversionTool/)
- [🔧 SQLines Tools](https://www.sqlines.com/)
- [📖 MariaDB Oracle Compatibility Mode](https://mariadb.com/kb/en/sql_modeoracle/)

---

## 📚 Sections suivantes

Ce chapitre se poursuit avec des guides détaillés pour chaque SGBD source :

### 19.2.1 Depuis Oracle
Conversion PL/SQL, mapping des types Oracle, gestion des packages, ora2pg en détail, mode de compatibilité Oracle MariaDB.

### 19.2.2 Depuis SQL Server
Conversion T-SQL, types SQL Server spécifiques, Always On vs Galera, intégration écosystème Microsoft.

### 19.2.3 Depuis PostgreSQL
Différences PostgreSQL/MariaDB, conversion PL/pgSQL, pgloader, migration des extensions, PostGIS vers MariaDB Spatial.

---

## ➡️ Section suivante

**[19.2.1 Depuis Oracle](./02.1-depuis-oracle.md)** : Nous détaillerons la migration depuis Oracle, le cas le plus complexe mais aussi le plus fréquent des migrations hétérogènes. Vous découvrirez comment convertir du PL/SQL, gérer les packages, et exploiter le mode de compatibilité Oracle de MariaDB.

⏭️ [Depuis Oracle](/19-migration-compatibilite/02.1-depuis-oracle.md)
