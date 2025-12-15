🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.3 Gestion des versions : Stratégie LTS vs Rolling 🔄

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Compréhension de l'écosystème MariaDB, expérience en gestion de parc de bases de données, notions de gestion du cycle de vie logiciel

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre la politique de versioning MariaDB et son évolution récente
- Différencier les versions LTS (Long Term Support) des versions Rolling Release
- Choisir la stratégie de versioning adaptée à votre contexte (production, développement, innovation)
- Planifier les mises à jour sur un horizon de 3 à 5 ans
- Anticiper les cycles de support et les dates de fin de vie
- Aligner votre stratégie de versioning avec les exigences de conformité et de sécurité

---

## Introduction

Le choix d'une version de base de données n'est jamais anodin. Ce qui peut sembler une décision technique mineure impacte en réalité la stabilité de vos systèmes, la charge de maintenance de vos équipes, l'accès aux nouvelles fonctionnalités, et parfois même la conformité réglementaire de votre organisation.

MariaDB a significativement fait évoluer sa politique de versioning depuis 2023. La transition vers un modèle clarifié avec des versions **LTS (Long Term Support)** supportées 5 ans et des versions **Rolling Release** trimestrielles offre désormais un cadre plus prévisible pour planifier vos infrastructures.

Cette section vous guide dans la compréhension de ce modèle et vous aide à définir une stratégie de versioning cohérente avec vos contraintes opérationnelles.

---

## Évolution de la politique de versioning MariaDB

### Historique : le modèle pré-2023

Avant 2023, MariaDB suivait un modèle de versioning moins structuré :

```
2012-2022 : Modèle historique
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

MariaDB 5.5  ─────────────────────────────────────▶ EOL 2020
MariaDB 10.0 ───────────────────────────────▶ EOL 2019
MariaDB 10.1 ─────────────────────────▶ EOL 2020
MariaDB 10.2 ───────────────────────────────────▶ EOL 2022
MariaDB 10.3 ─────────────────────────────────────────▶ EOL 2023
MariaDB 10.4 ───────────────────────────────────────────▶ EOL 2024
MariaDB 10.5 ─────────────────────────────────────────────▶ EOL 2025
MariaDB 10.6 LTS ───────────────────────────────────────────────▶ EOL 2026
MariaDB 10.11 LTS ────────────────────────────────────────────────────▶ EOL 2028
```

Ce modèle présentait plusieurs inconvénients :
- Durées de support variables et parfois imprévisibles
- Confusion sur quelles versions privilégier
- Difficultés de planification à long terme

### Nouveau modèle : LTS + Rolling (depuis 2023)

MariaDB a introduit un nouveau modèle clarifié avec deux tracks distinctes :

```
2023+ : Nouveau modèle LTS + Rolling
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRACK LTS (Long Term Support)
─────────────────────────────
• Support : 5 ans
• Fréquence : ~18 mois entre versions LTS
• Focus : Stabilité, patches sécurité, bug fixes
• Pas de nouvelles fonctionnalités majeures après GA

TRACK ROLLING
─────────────
• Support : Jusqu'à la sortie de la version suivante (~3 mois)
• Fréquence : Trimestrielle
• Focus : Nouvelles fonctionnalités, innovation
• Idéal pour tests et développement
```

---

## Les versions LTS en détail

### Caractéristiques des versions LTS

Les versions LTS constituent le socle recommandé pour les environnements de production.

| Caractéristique | Description |
|-----------------|-------------|
| **Durée de support** | 5 ans à partir de la date GA |
| **Mises à jour de sécurité** | Garanties pendant toute la durée |
| **Bug fixes** | Corrections de bugs critiques et majeurs |
| **Nouvelles fonctionnalités** | Aucune après GA (stabilité prioritaire) |
| **Compatibilité** | API et comportement stables |
| **Recommandation** | Production, environnements critiques |

### Versions LTS actuelles et planifiées

```
Timeline des versions LTS MariaDB
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2021      2022      2023      2024      2025      2026      2027      2028      2029      2030
  │         │         │         │         │         │         │         │         │         │
  │         │         │         │         │         │         │         │         │         │
  ├─────────┴─────────┴─────────┴─────────┴─────────┼─────────┘         │         │         │
  │              MariaDB 10.6 LTS                   │ EOL               │         │         │
  │              (Juil 2021 - Juil 2026)            │                   │         │         │
  │                                                 │                   │         │         │
  │                   ├─────────┴─────────┴─────────┴─────────┴─────────┼─────────┘         │
  │                   │              MariaDB 10.11 LTS                  │ EOL               │
  │                   │              (Fév 2023 - Fév 2028)              │                   │
  │                   │                                                 │                   │
  │                   │         ├─────────┴─────────┴─────────┴─────────┴─────────┴─────────┤
  │                   │         │              MariaDB 11.4 LTS                             │
  │                   │         │              (Mai 2024 - Mai 2029)                        │
  │                   │         │                                                           │
  │                   │         │               ├─────────┴─────────┴─────────┴─────────┴───┤
  │                   │         │               │              MariaDB 11.8 LTS 🆕          │
  │                   │         │               │              (Juin 2025 - Juin 2030)      │
  │                   │         │               │                                           │
```

### Tableau récapitulatif des versions LTS

| Version | Date GA | Date EOL | Durée support | Statut actuel |
|---------|---------|----------|---------------|---------------|
| **10.6 LTS** | Juillet 2021 | Juillet 2026 | 5 ans | ✅ Maintenance |
| **10.11 LTS** | Février 2023 | Février 2028 | 5 ans | ✅ Maintenance |
| **11.4 LTS** | Mai 2024 | Mai 2029 | 5 ans | ✅ Actif |
| **11.8 LTS** 🆕 | Juin 2025 | Juin 2030 | 5 ans | ✅ Actif (Dernier) |
| **12.4 LTS** (prévu) | ~Q2 2027 | ~Q2 2032 | 5 ans | 📅 Planifié |

💡 **Conseil** : Pour les nouveaux déploiements en production, **MariaDB 11.8 LTS** est le choix recommandé en décembre 2025. Il offre le support le plus long (jusqu'en 2030) et les fonctionnalités les plus récentes stabilisées.

---

## Les versions Rolling Release en détail

### Caractéristiques des versions Rolling

Les versions Rolling Release sont destinées aux utilisateurs souhaitant accéder rapidement aux nouvelles fonctionnalités.

| Caractéristique | Description |
|-----------------|-------------|
| **Durée de support** | ~3 mois (jusqu'à la version suivante) |
| **Fréquence** | Trimestrielle |
| **Nouvelles fonctionnalités** | Oui, en continu |
| **Stabilité** | Moins garantie qu'une LTS |
| **Recommandation** | Développement, tests, early adopters |

### Cycle de vie Rolling Release

```
Cycle Rolling Release (exemple)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Q1 2025          Q2 2025          Q3 2025          Q4 2025          Q1 2026
    │                │                │                │                │
    ▼                ▼                ▼                ▼                ▼
┌───────┐        ┌───────┐        ┌───────┐        ┌───────┐        ┌───────┐
│ 11.6  │───────▶│ 11.7  │───────▶│ 11.8  │───────▶│ 12.0  │───────▶│ 12.1  │
│Rolling│  EOL   │Rolling│  EOL   │→ LTS! │        │Rolling│  EOL   │Rolling│
└───────┘        └───────┘        └───────┘        └───────┘        └───────┘
    │                │                                  │                │
    │                │                                  │                │
    └────────────────┴──────────────────────────────────┴────────────────┘
                   Support court (3 mois chacune)
                   Sauf 11.8 qui devient LTS
```

### Roadmap série 12.x 🆕

La prochaine génération MariaDB 12.x suivra le même modèle :

| Version | Type | Date estimée | Nouvelles fonctionnalités attendues |
|---------|------|--------------|-------------------------------------|
| **12.0** | Rolling | Q4 2025 | Améliorations optimizer, Vector v2 |
| **12.1** | Rolling | Q1 2026 | À définir |
| **12.2** | Rolling | Q2 2026 | À définir |
| **12.3** | Rolling | Q3 2026 | Candidat LTS |
| **12.4 LTS** | LTS | Q2 2027 | Stabilisation des 12.x |

⚠️ **Attention** : Les dates et fonctionnalités de la série 12.x sont prévisionnelles et sujettes à modification. Consultez la roadmap officielle pour les informations à jour.

---

## Comparaison LTS vs Rolling

### Tableau comparatif détaillé

| Critère | LTS | Rolling |
|---------|-----|---------|
| **Durée de support** | 5 ans | ~3 mois |
| **Fréquence de sortie** | ~18 mois | Trimestrielle |
| **Nouvelles fonctionnalités** | Non (après GA) | Oui |
| **Patches sécurité** | Oui (5 ans) | Oui (3 mois) |
| **Stabilité** | Maximale | Variable |
| **Migration requise** | Tous les 3-4 ans | Tous les 3 mois |
| **Tests de régression** | Moins fréquents | Très fréquents |
| **Idéal pour** | Production | Développement, labs |
| **Risque** | Faible | Modéré |
| **Accès aux nouveautés** | Retardé | Immédiat |

### Arbre de décision

```
                    ┌─────────────────────────────┐
                    │  Quel est votre contexte ?  │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────┴───────────────┐
                    │                             │
                    ▼                             ▼
        ┌───────────────────┐         ┌───────────────────┐
        │    PRODUCTION     │         │   DÉVELOPPEMENT   │
        │   Mission critique│         │   Tests / Labs    │
        └─────────┬─────────┘         └─────────┬─────────┘
                  │                             │
                  ▼                             ▼
        ┌───────────────────┐         ┌───────────────────┐
        │  Exigences de     │         │  Besoin des       │
        │  conformité ?     │         │  dernières        │
        │  (PCI, SOC2...)   │         │  fonctionnalités? │
        └─────────┬─────────┘         └─────────┬─────────┘
                  │                             │
         ┌────────┴────────┐           ┌────────┴────────┐
         │                 │           │                 │
         ▼                 ▼           ▼                 ▼
    ┌─────────┐      ┌─────────┐ ┌─────────┐      ┌─────────┐
    │   OUI   │      │   NON   │ │   OUI   │      │   NON   │
    └────┬────┘      └────┬────┘ └────┬────┘      └────┬────┘
         │                │           │                │
         ▼                ▼           ▼                ▼
    ╔═════════╗      ╔═════════╗ ╔═════════╗      ╔══════════╗
    ║   LTS   ║      ║   LTS   ║ ║ ROLLING ║      ║   LTS    ║
    ║ (oblig.)║      ║(recomm.)║ ║         ║      ║(plus sûr)║
    ╚═════════╝      ╚═════════╝ ╚═════════╝      ╚══════════╝
```

---

## Stratégies de versioning par profil

### Profil 1 : Production critique (Banque, Santé, E-commerce)

**Recommandation : LTS uniquement**

```yaml
# Stratégie versioning - Production critique
strategy: LTS_ONLY

current_version: "11.8 LTS"
target_version: "11.8 LTS"
next_planned_upgrade: "12.4 LTS (Q2 2027)"

upgrade_policy:
  frequency: "Tous les 2-3 ans"
  timing: "6-12 mois après GA d'une nouvelle LTS"
  validation: "3-6 mois de tests en staging"
  
security_patches:
  policy: "Appliquer sous 30 jours"
  process: "Test staging → Prod après validation"
  
exceptions:
  - "Patch critique zero-day : 48-72h"
  - "CVE critique : 7 jours maximum"
```

**Avantages :**
- Stabilité maximale
- Support garanti 5 ans
- Moins de migrations
- Conformité facilitée

**Inconvénients :**
- Pas d'accès aux dernières fonctionnalités
- Peut accumuler de la dette technique

### Profil 2 : Production standard (PME, Applications internes)

**Recommandation : LTS avec mise à jour régulière**

```yaml
# Stratégie versioning - Production standard
strategy: LTS_REGULAR_UPDATE

current_version: "11.4 LTS"
target_version: "11.8 LTS"
upgrade_deadline: "Q2 2026"

upgrade_policy:
  frequency: "Chaque nouvelle LTS (18 mois)"
  timing: "3-6 mois après GA"
  validation: "1-2 mois de tests"
  
benefits:
  - "Accès aux nouvelles fonctionnalités LTS"
  - "Support étendu"
  - "Correctifs de performance"
```

### Profil 3 : Développement et staging

**Recommandation : Rolling ou dernière LTS**

```yaml
# Stratégie versioning - Développement
strategy: ROLLING_OR_LATEST_LTS

environments:
  dev:
    version: "Rolling (latest)"
    rationale: "Tester les nouvelles fonctionnalités"
  
  staging:
    version: "Same as production LTS"
    rationale: "Reproduire le comportement production"
  
  sandbox:
    version: "Rolling (latest)"
    rationale: "Expérimentation, POC"
```

### Profil 4 : Early adopter / Innovation

**Recommandation : Rolling Release**

```yaml
# Stratégie versioning - Innovation
strategy: ROLLING_CONTINUOUS

policy:
  upgrade_frequency: "Chaque nouvelle rolling (~3 mois)"
  testing: "Tests automatisés CI/CD"
  rollback: "Blue-green deployment"
  
use_cases:
  - "Startup tech"
  - "Labs R&D"
  - "POC clients"
  - "Évaluation nouvelles fonctionnalités"
  
risks_accepted:
  - "Breaking changes possibles"
  - "Bugs non découverts"
  - "Support limité"
```

---

## Planification des mises à jour

### Calendrier de maintenance recommandé

Pour une infrastructure production utilisant des versions LTS :

```
Calendrier type sur 5 ans (2025-2030)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2025 │ Q1: Préparation 11.8 LTS
     │ Q2: Déploiement 11.8 LTS (staging)
     │ Q3: Migration production vers 11.8 LTS
     │ Q4: Stabilisation, monitoring
     │
2026 │ Q1-Q4: Patches sécurité 11.8.x
     │        Surveillance roadmap 12.x
     │
2027 │ Q1: Évaluation 12.4 LTS (GA prévu Q2)
     │ Q2: POC 12.4 LTS
     │ Q3: Tests staging 12.4 LTS
     │ Q4: Préparation migration
     │
2028 │ Q1: Migration production vers 12.4 LTS
     │ Q2: Stabilisation
     │ Q3-Q4: Maintenance 12.4.x
     │
2029 │ Maintenance 12.4 LTS
     │ Surveillance prochaine LTS
     │
2030 │ Juin: EOL 11.8 LTS (doit être migré)
     │ Préparation migration vers 13.x LTS
```

### Fenêtre de chevauchement LTS

Les versions LTS se chevauchent intentionnellement pour permettre des migrations planifiées :

```
Chevauchement des versions LTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

         2024        2025        2026        2027        2028        2029        2030
           │           │           │           │           │           │           │
10.6 LTS   ├───────────┼───────────┼───────────┤ EOL       │           │           │
           │           │           │           │           │           │           │
10.11 LTS  ├───────────┼───────────┼───────────┼───────────┼───────────┤ EOL       │
           │           │           │           │           │           │           │
11.4 LTS   ├───────────┼───────────┼───────────┼───────────┼───────────┼───────────┤
           │           │           │           │           │           │           │
11.8 LTS   │           ├───────────┼───────────┼───────────┼───────────┼───────────┤
           │           │           │           │           │           │           │
           │           │           │           │           │           │           │
           │◀─────────▶│           │           │           │           │           │
           │ Fenêtre   │           │           │           │           │           │
           │ migration │           │           │           │           │           │
           │ 10.6→11.x │           │           │           │           │           │
```

💡 **Conseil** : Planifiez vos migrations au moins **12 mois avant l'EOL** de votre version actuelle. Cela laisse du temps pour les tests, les imprévus, et évite les migrations dans l'urgence.

---

## Gestion des patches de sécurité

### Types de releases

MariaDB utilise une numérotation sémantique pour ses releases :

```
MariaDB 11.8.3
        │  │ │
        │  │ └── Patch release (bug fixes, sécurité)
        │  └──── Point release (corrections mineures)
        └─────── Version majeure
```

| Type | Exemple | Contenu | Fréquence | Action requise |
|------|---------|---------|-----------|----------------|
| **Majeure** | 11.4 → 11.8 | Nouvelles fonctionnalités | ~18 mois | Migration planifiée |
| **Point** | 11.8.0 → 11.8.1 | Bug fixes | 1-3 mois | Mise à jour recommandée |
| **Patch** | 11.8.1 → 11.8.2 | Sécurité critique | Variable | Mise à jour urgente |

### Politique de patches recommandée

```sql
-- Vérification de la version actuelle
SELECT VERSION();

-- Vérification des variables de sécurité
SHOW VARIABLES LIKE '%ssl%';
SHOW VARIABLES LIKE '%tls%';
```

**Matrice de décision pour les patches :**

| Criticité CVE | Délai d'application | Processus |
|---------------|---------------------|-----------|
| **Critique (9.0-10.0)** | < 7 jours | Fast-track, tests minimaux |
| **Haute (7.0-8.9)** | < 30 jours | Tests staging, déploiement rapide |
| **Moyenne (4.0-6.9)** | < 90 jours | Cycle normal de maintenance |
| **Basse (0.1-3.9)** | Prochain cycle | Batch avec autres updates |

---

## Spécificités MariaDB 11.8 LTS 🆕

### Nouveautés impactant le versioning

MariaDB 11.8 LTS introduit des changements qui peuvent influencer votre stratégie de versioning :

| Changement | Impact | Considération versioning |
|------------|--------|--------------------------|
| **utf8mb4 par défaut** | Taille données, index | Migration peut nécessiter plus d'espace |
| **TLS par défaut** | Connexions legacy | Clients anciens à mettre à jour |
| **TIMESTAMP 2106** | Format stockage | Incompatibilité binaire tables temporelles |
| **Collations UCA 14.0** | Tri chaînes | Comportement différent possible |
| **MariaDB Vector** | Nouvelle fonctionnalité | Non disponible sur versions antérieures |

### Migration depuis versions antérieures

```
Chemins de migration vers 11.8 LTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

10.6 LTS ──────────────────────────────▶ 11.8 LTS
           Direct (recommandé)           ✅ Supporté

10.11 LTS ─────────────────────────────▶ 11.8 LTS
           Direct (recommandé)           ✅ Supporté

11.4 LTS ──────────────────────────────▶ 11.8 LTS
           Direct (recommandé)           ✅ Supporté

10.5 ─────▶ 10.6 LTS ──────────────────▶ 11.8 LTS
            Via 10.6                     ✅ Recommandé

10.4 ─────▶ 10.6 LTS ──────────────────▶ 11.8 LTS
            Via 10.6                     ✅ Recommandé

< 10.4 ───▶ 10.6 LTS ──▶ 11.4 LTS ────▶ 11.8 LTS
            Migration par étapes         ⚠️ Progressive
```

⚠️ **Attention** : Pour les versions très anciennes (< 10.4), une migration par étapes est fortement recommandée pour éviter les incompatibilités cumulées.

---

## Outils de gestion des versions

### Vérification de version et compatibilité

```bash
#!/bin/bash
# Script de vérification de version MariaDB

# Version actuelle
echo "=== Version MariaDB actuelle ==="
mariadb -e "SELECT VERSION() as version, @@version_comment as edition;"

# Vérification des EOL
echo ""
echo "=== Statut du support ==="
VERSION=$(mariadb -N -e "SELECT SUBSTRING_INDEX(VERSION(), '-', 1);")
echo "Version détectée: $VERSION"

# Informations de build
echo ""
echo "=== Informations de build ==="
mariadb -e "SHOW VARIABLES LIKE 'version%';"

# Variables importantes pour compatibilité
echo ""
echo "=== Variables de compatibilité ==="
mariadb -e "SHOW VARIABLES WHERE Variable_name IN 
    ('sql_mode', 'character_set_server', 'collation_server', 
     'default_storage_engine', 'innodb_file_per_table');"
```

### Surveillance des annonces de versions

```bash
#!/bin/bash
# Script de surveillance des nouvelles versions MariaDB

# Sources RSS/Atom à surveiller
FEEDS=(
    "https://mariadb.com/feed/"
    "https://mariadb.org/feed/"
)

# Vérification via API (si disponible)
echo "=== Dernières versions MariaDB ==="
curl -s "https://downloads.mariadb.org/rest-api/mariadb/" | \
    jq -r '.major_releases[] | "\(.release_id): \(.release_status)"' | \
    head -10

# Alternative : page de téléchargement
echo ""
echo "Consultez: https://mariadb.org/download/"
echo "Release notes: https://mariadb.com/kb/en/release-notes/"
```

### Automatisation des mises à jour (avec précautions)

```yaml
# Exemple Ansible pour gestion des versions MariaDB
# ansible/roles/mariadb-version-check/tasks/main.yml

---
- name: Get current MariaDB version
  shell: mariadb -N -e "SELECT VERSION();"
  register: current_version
  changed_when: false

- name: Check if upgrade is needed
  set_fact:
    upgrade_needed: "{{ current_version.stdout is version(target_mariadb_version, '<') }}"

- name: Display version status
  debug:
    msg: |
      Current version: {{ current_version.stdout }}
      Target version: {{ target_mariadb_version }}
      Upgrade needed: {{ upgrade_needed }}

- name: Warn if EOL approaching
  debug:
    msg: "WARNING: MariaDB {{ current_version.stdout }} EOL approaching!"
  when: 
    - current_version.stdout is match('^10\.6\.')
    - ansible_date_time.year | int >= 2026
```

---

## Considérations pour environnements multi-versions

### Architecture avec versions hétérogènes

Dans les grandes organisations, plusieurs versions coexistent souvent :

```
Architecture multi-versions typique
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────┐
│                     ENVIRONNEMENTS                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Production  │  │  Staging    │  │   Dev/Test  │          │
│  │ 11.8 LTS    │  │ 11.8 LTS    │  │ 12.x Rolling│          │
│  │             │  │ (même que   │  │ (dernière)  │          │
│  │             │  │  prod)      │  │             │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│        │                │                │                  │
│        └────────────────┼────────────────┘                  │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │   Outils communs    │                        │
│              │   • Monitoring      │                        │
│              │   • Backup          │                        │
│              │   • CI/CD           │                        │
│              └─────────────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Bonnes pratiques multi-versions

| Pratique | Objectif | Mise en œuvre |
|----------|----------|---------------|
| **Staging = Production** | Éviter les surprises | Même version exacte |
| **Dev ≥ Production** | Compatibilité descendante | Rolling ou même LTS |
| **Documentation versions** | Traçabilité | CMDB, wiki, tags |
| **Tests de compatibilité** | Détection précoce | CI/CD avec matrice |
| **Alerting EOL** | Anticipation | Monitoring, calendar |

---

## ✅ Points clés à retenir

- MariaDB propose deux tracks : **LTS** (5 ans de support) et **Rolling** (trimestriel, ~3 mois de support)
- **MariaDB 11.8 LTS** (juin 2025) est la version LTS recommandée pour les nouveaux déploiements production
- Les versions LTS se **chevauchent** intentionnellement pour permettre des migrations planifiées
- Choisissez **LTS pour la production**, **Rolling pour le développement/innovation**
- Planifiez les migrations **12 mois avant l'EOL** de votre version actuelle
- Appliquez les **patches de sécurité** selon leur criticité (critique < 7 jours, haute < 30 jours)
- La série **12.x débutera fin 2025** avec 12.4 LTS prévue pour Q2 2027
- Maintenez le **staging aligné sur la production** pour éviter les surprises

---

## 🔗 Ressources et références

- [📖 MariaDB Server Releases](https://mariadb.com/kb/en/mariadb-server-release-dates/)
- [📖 MariaDB Maintenance Policy](https://mariadb.org/about/#maintenance-policy)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-release-notes/)
- [📖 MariaDB Roadmap](https://mariadb.org/about/roadmap/)
- [📖 MariaDB Security Vulnerabilities](https://mariadb.com/kb/en/security/)
- [🔧 MariaDB Download Page](https://mariadb.org/download/)

---

## ➡️ Section suivante

**[19.4 Stratégies de mise à jour et upgrade paths](./04-strategies-mise-a-jour.md)** : Nous détaillerons les méthodes concrètes de mise à jour : utilisation de `mariadb-upgrade`, comparaison entre upgrade in-place et migration logique, gestion des versions intermédiaires, et automatisation des processus d'upgrade.

⏭️ [Stratégies de mise à jour et upgrade paths](/19-migration-compatibilite/04-strategies-mise-a-jour.md)
