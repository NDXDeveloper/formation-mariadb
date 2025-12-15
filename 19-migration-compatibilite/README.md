🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19. Migration et Compatibilité

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 12-16 heures  
> **Prérequis** : Maîtrise de l'administration MariaDB (chapitres 10-12), connaissance des mécanismes de réplication (chapitre 13), expérience en production avec au moins un SGBD relationnel

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Planifier et exécuter une migration complète depuis MySQL, Oracle, SQL Server ou PostgreSQL vers MariaDB
- Évaluer la compatibilité applicative et anticiper les points de friction lors d'une migration
- Choisir la stratégie de versioning adaptée entre LTS et Rolling Release selon votre contexte
- Maîtriser les différentes méthodes de mise à jour (in-place, logical, blue-green)
- Implémenter des migrations zero-downtime en environnement de production critique
- Gérer les cas particuliers de migration des System-Versioned Tables avec MariaDB 11.8 🆕
- Concevoir des plans de rollback robustes et des procédures de contingence

---

## Introduction

La migration de bases de données représente l'un des projets les plus critiques et complexes qu'une équipe technique puisse entreprendre. Qu'il s'agisse de quitter MySQL suite au rachat par Oracle, de moderniser un legacy Oracle coûteux, ou simplement de mettre à jour vers MariaDB 11.8 LTS, chaque migration comporte des risques significatifs pour la continuité de service et l'intégrité des données.

Ce chapitre adopte une approche pragmatique et orientée production. Nous ne nous contenterons pas d'énumérer les commandes : nous explorerons les **stratégies décisionnelles**, les **pièges réels rencontrés en entreprise**, et les **méthodologies éprouvées** pour réussir vos migrations avec un risque minimal.

MariaDB 11.8 LTS apporte son lot de considérations spécifiques, notamment autour du changement de format des timestamps dans les System-Versioned Tables et de l'adoption d'utf8mb4 par défaut. Ces évolutions, bien que bénéfiques à long terme, nécessitent une attention particulière lors des mises à jour.

---

## Vue d'ensemble du chapitre

Ce chapitre est structuré en neuf sections progressives, de l'évaluation initiale jusqu'aux stratégies avancées de migration sans interruption.

### 📋 Structure des sections

| Section | Titre | Focus principal |
|---------|-------|-----------------|
| 19.1 | Migration depuis MySQL | Compatibilité native, différences subtiles, outils de migration |
| 19.2 | Migration depuis d'autres SGBD | Oracle, SQL Server, PostgreSQL : approches spécifiques |
| 19.3 | Gestion des versions | Stratégie LTS vs Rolling, cycles de support |
| 19.4 | Stratégies de mise à jour | mariadb-upgrade, in-place vs logical |
| 19.5 | Compatibilité des applications | Connecteurs, ORM, requêtes SQL |
| 19.6 | Tests de compatibilité | Méthodologies de validation |
| 19.7 | Rollback et contingence | Plans de repli, procédures d'urgence |
| 19.8 | Zero-downtime migrations | Blue-green, réplication, cutover |
| 19.9 | Migration System-Versioned Tables 🆕 | Changement format timestamp 11.8 |

---

## Concepts fondamentaux

Avant d'entrer dans les détails techniques, il est essentiel de comprendre les concepts qui gouvernent toute stratégie de migration réussie.

### Taxonomie des migrations

Les migrations de bases de données se classifient selon plusieurs axes :

**Par origine et destination :**
- **Homogène** : Migration au sein de la même famille (MySQL → MariaDB, MariaDB 10.6 → 11.8)
- **Hétérogène** : Migration entre familles différentes (Oracle → MariaDB, SQL Server → MariaDB)

**Par méthode technique :**
- **Logical** : Export/import des données via dumps SQL ou outils ETL
- **Physical** : Copie directe des fichiers de données (quand compatible)
- **Streaming** : Réplication en temps réel pendant la migration

**Par impact sur le service :**
- **Offline** : Interruption planifiée pendant la migration
- **Online** : Service maintenu avec dégradation possible
- **Zero-downtime** : Aucune interruption perceptible par les utilisateurs

### Le triptyque de la migration

Toute migration réussie repose sur trois piliers interdépendants :

```
                    ┌─────────────────┐
                    │   DONNÉES       │
                    │   Intégrité     │
                    │   Complétude    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
     │   SCHÉMA    │  │APPLICATION  │  │ PERFORMANCE │
     │   DDL       │  │   Requêtes  │  │   SLA       │
     │   Types     │  │   Logique   │  │   Latence   │
     └─────────────┘  └─────────────┘  └─────────────┘
```

Négliger l'un de ces aspects conduit invariablement à des problèmes post-migration. Un schéma parfaitement migré avec des données incomplètes est inutile. Des données intègres avec une application incompatible bloquent la production. Une migration fonctionnelle mais avec des performances dégradées de 50% génère des incidents utilisateurs.

### Évaluation des risques

Avant toute migration, une analyse de risques structurée s'impose :

| Facteur de risque | Impact | Probabilité | Mitigation |
|-------------------|--------|-------------|------------|
| Incompatibilité SQL | Élevé | Variable selon source | Tests exhaustifs, rewrites |
| Perte de données | Critique | Faible si procédures suivies | Checksums, validations |
| Dégradation performance | Moyen | Moyenne | Benchmarks, tuning |
| Downtime prolongé | Élevé | Moyenne | Dry-runs, automation |
| Rollback impossible | Critique | Faible | Tests rollback, backups |

---

## Matrice de compatibilité des sources

Cette matrice synthétise le niveau de difficulté et les principaux défis selon le SGBD source :

| SGBD Source | Difficulté | Compatibilité SQL | Outils natifs | Défis majeurs |
|-------------|------------|-------------------|---------------|---------------|
| **MySQL 5.7/8.0** | ⭐ Faible | 95%+ | mysqldump, réplication | Fonctions JSON, CTE récursives |
| **Percona Server** | ⭐ Faible | 98%+ | mysqldump, xtrabackup | Plugins spécifiques |
| **Oracle** | ⭐⭐⭐⭐ Élevée | 40-60% | SQL Developer, ora2pg | PL/SQL, packages, sequences |
| **SQL Server** | ⭐⭐⭐ Moyenne | 50-70% | SSMA, bcp | T-SQL, CTEs, window functions |
| **PostgreSQL** | ⭐⭐⭐ Moyenne | 60-75% | pgloader, pg_dump | PL/pgSQL, arrays, JSONB |
| **MariaDB < 10.5** | ⭐ Faible | 99%+ | mariadb-upgrade | Deprecated features |

---

## Chronologie type d'un projet de migration

Un projet de migration bien mené suit généralement ces phases :

```
Semaine 1-2      Semaine 3-4        Semaine 5-8     Semaine 9-10     Semaine 11-12
    │                │                  │               │                │
    ▼                ▼                  ▼               ▼                ▼
┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│ ANALYSE │ ──▶  │ DESIGN  │ ──▶  │  BUILD  │ ──▶  │  TEST   │ ──▶  │ DEPLOY  │
│         │      │         │      │         │      │         │      │         │
│• Audit  │      │• Archi  │      │• Scripts│      │• Fonct. │      │• Dry-run│
│• Invent.│      │• Sizing │      │• Mapping│      │• Perf.  │      │• Cutover│
│• Risques│      │• Plan   │      │• Outils │      │• Charge │      │• Valid. │
└─────────┘      └─────────┘      └─────────┘      └─────────┘      └─────────┘
```

💡 **Conseil** : Pour une migration critique en production, prévoyez au minimum **deux dry-runs complets** avant le cutover final. Le premier révèle les problèmes majeurs, le second valide les corrections et affine les estimations de durée.

---

## Spécificités MariaDB 11.8 LTS pour les migrations 🆕

MariaDB 11.8 LTS, disponible depuis juin 2025, introduit plusieurs changements impactant les stratégies de migration :

### Changements majeurs à considérer

| Fonctionnalité | Impact migration | Action requise |
|----------------|------------------|----------------|
| **utf8mb4 par défaut** | Taille données +33% potentiel | Vérifier espace disque, revoir index |
| **Collation UCA 14.0.0** | Tri différent possible | Tests comparaison chaînes |
| **TIMESTAMP 2038→2106** | Format interne modifié | Rebuild tables temporelles |
| **TLS par défaut** | Connexions legacy KO | Configurer certificats ou désactiver |
| **System-Versioned format** | Incompatibilité binaire | Procédure spécifique section 19.9 |

### utf8mb4 : l'éléphant dans la pièce

Depuis MariaDB 11.8, le charset par défaut passe de `latin1` à `utf8mb4`. Cette évolution, attendue depuis longtemps, a des implications significatives :

```sql
-- Avant MariaDB 11.8 : comportement implicite
CREATE TABLE users (
    name VARCHAR(255)  -- Stocké en latin1, 255 bytes max
);

-- MariaDB 11.8+ : comportement par défaut
CREATE TABLE users (
    name VARCHAR(255)  -- Stocké en utf8mb4, 1020 bytes max
);
```

⚠️ **Attention** : Cette différence impacte directement la taille des index. Un index sur `VARCHAR(255)` en utf8mb4 atteint la limite InnoDB de 3072 bytes avec seulement 768 caractères. Planifiez vos migrations en conséquence.

### Extension TIMESTAMP : résolution du problème Y2038

MariaDB 11.8 étend la plage des TIMESTAMP au-delà de 2038, résolvant définitivement le "bug de l'an 2038". Cependant, cette extension modifie le format de stockage interne :

```sql
-- Vérification de la version de stockage
SELECT @@version, @@system_versioning_asof;

-- Les tables créées avant 11.8 utilisent l'ancien format
-- Les nouvelles tables utilisent le format étendu
```

La section 19.9 détaille la procédure de migration des System-Versioned Tables, particulièrement sensibles à ce changement.

---

## Outils de l'écosystème migration

### Outils natifs MariaDB

| Outil | Usage principal | Avantages | Limitations |
|-------|-----------------|-----------|-------------|
| `mariadb-dump` | Export logique | Simple, portable | Lent sur gros volumes |
| `mariadb-import` | Import bulk | Rapide pour CSV | Format spécifique |
| `mariadb-upgrade` | Mise à jour système | Automatisé | Nécessite arrêt |
| `mariabackup` | Backup physique | Très rapide, PITR | Même version requise |

### Outils tiers recommandés

| Outil | Spécialité | Licence | Cas d'usage |
|-------|------------|---------|-------------|
| **mydumper/myloader** | Dump parallèle | GPL | Bases volumineuses |
| **pt-online-schema-change** | DDL sans lock | GPL | ALTER sur tables actives |
| **gh-ost** | DDL sans triggers | MIT | Environnements répliqués |
| **pgloader** | Migration PostgreSQL | PostgreSQL | ETL PostgreSQL → MariaDB |
| **ora2pg** | Migration Oracle | GPL | Conversion PL/SQL |
| **SQLines** | Multi-source | Commercial | Conversion SQL universelle |

### Outils de validation

```bash
# Comparaison de schémas
mysqldiff --server1=source --server2=target db_name

# Comparaison de données (checksum)
pt-table-checksum --host=source --databases=mydb

# Validation post-migration
pt-table-sync --print --sync-to-master h=replica
```

---

## Métriques de succès d'une migration

Définissez vos critères de succès avant de commencer :

### Critères fonctionnels

- [ ] 100% des tables migrées avec schéma identique ou équivalent
- [ ] 100% des données transférées avec intégrité vérifiée (checksums)
- [ ] 100% des procédures stockées fonctionnelles
- [ ] 100% des triggers actifs et testés
- [ ] 100% des vues accessibles
- [ ] 100% des utilisateurs et permissions recréés

### Critères de performance

- [ ] Temps de réponse P95 ≤ 110% du baseline source
- [ ] Throughput ≥ 95% du baseline source
- [ ] Aucune requête > 10x plus lente qu'en source
- [ ] Buffer pool hit ratio ≥ 99% après warm-up

### Critères opérationnels

- [ ] Downtime réel ≤ downtime planifié
- [ ] Rollback testé et documenté
- [ ] Monitoring opérationnel
- [ ] Alerting configuré
- [ ] Documentation à jour

---

## ✅ Points clés à retenir

- La **migration homogène** (MySQL → MariaDB) est nettement plus simple que l'hétérogène, avec une compatibilité SQL supérieure à 95%
- Toute migration repose sur le **triptyque données-schéma-application** : négliger un aspect compromet l'ensemble
- MariaDB 11.8 LTS introduit des **changements structurels** (utf8mb4, TIMESTAMP étendu) nécessitant une attention particulière
- Les **dry-runs multiples** sont indispensables pour les migrations critiques
- Un **plan de rollback testé** n'est pas optionnel : c'est une assurance-vie
- La stratégie **LTS vs Rolling** doit être alignée avec votre capacité de mise à jour
- Les **outils tiers** (mydumper, pt-osc, gh-ost) complètent efficacement les outils natifs

---

## 🔗 Ressources et références

- [📖 MariaDB Knowledge Base - Migration](https://mariadb.com/kb/en/migration/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-release-notes/)
- [📖 Upgrading MariaDB](https://mariadb.com/kb/en/upgrading/)
- [📖 MySQL to MariaDB Migration](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)
- [🔧 Percona Toolkit Documentation](https://docs.percona.com/percona-toolkit/)
- [🔧 gh-ost Documentation](https://github.com/github/gh-ost)
- [🔧 pgloader Documentation](https://pgloader.io/)

---

## 📚 Sommaire détaillé des sections

### 19.1 Migration depuis MySQL
Compatibilité MySQL/MariaDB, points d'attention critiques, différences de comportement SQL, outils de migration automatisée, scénarios de coexistence.

### 19.2 Migration depuis d'autres SGBD  
Stratégies spécifiques Oracle, SQL Server et PostgreSQL. Mapping des types de données, conversion des procédures stockées, gestion des fonctionnalités propriétaires.

### 19.3 Gestion des versions : Stratégie LTS vs Rolling 🔄
Comprendre le cycle de releases MariaDB, choisir entre stabilité LTS et fonctionnalités Rolling, planifier les mises à jour sur le long terme.

### 19.4 Stratégies de mise à jour et upgrade paths
Utilisation de mariadb-upgrade, comparaison upgrade in-place vs logical, gestion des versions intermédiaires, automatisation des mises à jour.

### 19.5 Compatibilité des applications
Validation des connecteurs, comportement des ORM, différences de parsing SQL, tests de régression applicative.

### 19.6 Tests de compatibilité
Méthodologies de test, environnements de validation, automatisation des tests de non-régression, benchmarking comparatif.

### 19.7 Rollback et contingence
Conception de plans de rollback, procédures d'urgence, gestion des données créées post-migration, communication de crise.

### 19.8 Zero-downtime migrations
Architecture blue-green, utilisation de la réplication, orchestration du cutover, gestion du split-brain, validation en temps réel.

### 19.9 Migration System-Versioned Tables 🆕
Changement de format timestamp MariaDB 11.8, procédure de migration des tables temporelles, validation de l'historique, cas particuliers.

---

## ➡️ Section suivante

**[19.1 Migration depuis MySQL](./01-migration-depuis-mysql.md)** : Nous commencerons par le cas le plus fréquent — la migration depuis MySQL. Vous découvrirez pourquoi cette migration est généralement fluide, mais aussi les pièges subtils qui peuvent transformer une migration "simple" en cauchemar opérationnel.

---

*Ce chapitre s'adresse aux architectes de données et DBA expérimentés. Les concepts présentés supposent une maîtrise préalable de l'administration MariaDB et une expérience significative en environnement de production.*

⏭️ [Migration depuis MySQL](/19-migration-compatibilite/01-migration-depuis-mysql.md)
