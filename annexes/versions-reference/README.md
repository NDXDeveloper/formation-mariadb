🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G. Versions de Référence 📅

> **Niveau** : Tous niveaux (Référence rapide)  
> **Durée estimée** : 5-10 minutes  
> **Prérequis** : Aucun

## 🎯 Objectif de cette annexe

Fournir une **référence rapide** du calendrier des versions MariaDB pour aider à la planification et aux décisions de migration.

---

## 📊 Versions LTS (Long-Term Support)

### Tableau récapitulatif

| Version | Type | Date GA | Fin de support | Durée support | Status |
|---------|------|---------|----------------|---------------|--------|
| **11.8** | **LTS** | **Juin 2025** | **Juin 2028** | **3 ans** | 🆕 **Actuelle** |
| **11.4** | **LTS** | **Mai 2024** | **Mai 2027** | **3 ans** | ✅ Supportée |
| **10.11** | **LTS** | **Fév 2023** | **Fév 2028** | **5 ans** | ✅ Supportée |
| **10.6** | **LTS** | **Juil 2021** | **Juil 2026** | **5 ans** | ✅ Supportée |
| **10.5** | LTS | Mai 2020 | Juin 2025 | 5 ans | ⚠️ Fin proche |

💡 **Note** : À partir de la version **11.4**, les versions LTS bénéficient d'un support de **3 ans** (au lieu de 5 ans précédemment). Cela permet des cycles de release plus rapides tout en maintenant la stabilité.

---

## 🔄 Versions Rolling Release (Non-LTS)

### Série 11.x

| Version | Type | Date GA | Fin de support | Durée support | Status |
|---------|------|---------|----------------|---------------|--------|
| 11.7 | Rolling | Déc 2024 | Mars 2025 | 3 mois | ⏳ Bientôt EOL |
| 11.6 | Rolling | Sept 2024 | Déc 2024 | 3 mois | ❌ EOL |
| 11.5 | Rolling | Juin 2024 | Sept 2024 | 3 mois | ❌ EOL |
| 11.3 | Rolling | Mars 2024 | Juin 2024 | 3 mois | ❌ EOL |
| 11.2 | Rolling | Déc 2023 | Mars 2024 | 3 mois | ❌ EOL |
| 11.1 | Rolling | Sept 2023 | Déc 2023 | 3 mois | ❌ EOL |
| 11.0 | Rolling | Juin 2023 | Sept 2023 | 3 mois | ❌ EOL |

💡 **Note** : Les versions rolling sont **supportées 3 mois** uniquement et destinées aux environnements de développement ou aux utilisateurs souhaitant tester les dernières fonctionnalités.

---

## 🗓️ Calendrier Visuel

### Timeline des versions LTS

```
2021 ─────┬──────────────────────────────────────────────────────────────
          │
    Jul   ▼ 10.6 LTS (Jul 2021 → Jul 2026)
          ├──────────────────────────────────┤
          │                                  │
2022 ─────┤                                  │
          │                                  │
          │                                  │
2023 ─────┤                                  │
          │                                  │
    Fév   ▼ 10.11 LTS (Fév 2023 → Fév 2028)  │
          ├──────────────────────────────────────────────────┤
          │                                  │               │
2024 ─────┤                                  │               │
          │                                  │               │
    Mai   ▼ 11.4 LTS (Mai 2024 → Mai 2027)   │               │
          ├─────────────────────────┤        │               │
          │                         │        │               │
2025 ─────┤                         │        │               │
          │                         │        │               │
    Jun   ▼ 11.8 LTS (Jun 2025 → Jun 2028)   │               │
          ├─────────────────────────┤        │               │
          │                         │        │               │
2026 ─────┤                         │        ▼               │
          │                         │       EOL              │
          │                         │                        │
2027 ─────┤                         ▼                        │
          │                        EOL                       │
          │                                                  │
2028 ─────┤                                                  ▼
          │                                                 EOL
          │
```

### Fenêtre de support actuelle (Décembre 2025)

```
┌─────────────────────────────────────────────────────────────┐
│         Versions supportées en Décembre 2025                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  10.6 LTS  ▰▰▰▰▰▰▰▰▰▱▱  (Fin dans 7 mois - Jul 2026)        │
│  10.11 LTS ▰▰▰▰▰▰▰▰▰▰▰▰▰▰▰▰  (Fin dans 26 mois - Fév 2028)  │
│  11.4 LTS  ▰▰▰▰▰▰▰▰▰▰▰  (Fin dans 17 mois - Mai 2027)       │
│  11.8 LTS  ▰▰▰▰▰▰▰▰▰▰▰▰▰▰▰▰▰ (Fin dans 30 mois - Jun 2028)  │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Légende : ▰ Support actif │ ▱ Fin proche (< 12 mois)
```

---

## 🎯 Quelle Version Choisir ?

### Décision rapide

```
┌─────────────────────────────────────────────────────────────┐
│                    Arbre de Décision                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Nouveau projet ?                                           │
│     ├─ OUI → 🟢 MariaDB 11.8 LTS                            │
│     │         (dernières fonctionnalités + support 3 ans)   │
│     │                                                       │
│     └─ NON → Vous avez quelle version ?                     │
│               │                                             │
│               ├─ 11.4 LTS   → 🟡 Rester ou migrer 11.8      │
│               │                  (selon besoins Vector/IA)  │
│               │                                             │
│               ├─ 10.11 LTS  → 🟢 Rester jusqu'en 2027-2028  │
│               │                  (support 5 ans restant)    │
│               │                                             │
│               ├─ 10.6 LTS   → 🟠 Planifier migration        │
│               │                  (EOL Jul 2026, 7 mois)     │
│               │                                             │
│               ├─ 10.5 LTS   → 🔴 Migrer URGENT              │
│               │                  (EOL Juin 2025, 6 mois)    │
│               │                                             │
│               └─ Autres     → 🔴 Migrer IMMÉDIATEMENT       │
│                                  (non supportées)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Recommandations par cas d'usage

| Cas d'usage | Version recommandée | Justification |
|-------------|---------------------|---------------|
| **Nouveau projet** | 11.8 LTS | Dernières features (Vector, TLS, etc.) + support 3 ans |
| **Application IA/ML** | 11.8 LTS | MariaDB Vector essentiel |
| **Production stable** | 10.11 LTS ou 11.4 LTS | Support long-terme, éprouvé |
| **Migration depuis MySQL** | 11.8 LTS | Meilleure compatibilité + innovations |
| **Développement/Test** | 11.7 (latest rolling) | Tester nouvelles features |
| **Legacy (10.5 ou moins)** | Migrer → 10.11 LTS ou 11.8 LTS | Sécurité + support |

---

## 🔄 Politique de Versioning

### Schéma de numérotation

```
MariaDB X.Y.Z
         │ │ │
         │ │ └─ Z : Patch (bugfixes, sécurité)
         │ │       Incrémentation fréquente
         │ │
         │ └─── Y : Minor version
         │         - Pair (0, 2, 4, 6, 8) → Développement/Rolling
         │         - Impair + 1 (5, 9) → Incrémentation vers LTS
         │         - LTS : 4, 8 (série 11.x), 6, 11 (série 10.x)
         │
         └───── X : Major version
                   Changements architecturaux majeurs
```

### Exemples

- **11.8.2** → Version LTS, 2ème patch
- **11.7.1** → Version rolling, 1er patch
- **10.11.8** → Version LTS (ancienne série), 8ème patch

---

## 📅 Roadmap Future

### Série 12.x (Prévisions)

| Version | Type | Date GA prévu | Fin support prévu | Status |
|---------|------|---------------|-------------------|--------|
| 12.0 | Rolling | Q1 2026 | Q2 2026 | 🔮 Prévu |
| 12.1 | Rolling | Q2 2026 | Q3 2026 | 🔮 Prévu |
| 12.2 | Rolling | Q3 2026 | Q4 2026 | 🔮 Prévu |
| **12.3** | **LTS** | **Q4 2026** | **Q4 2029** | 🔮 **Prévu** |

💡 **Note** : La série 12.x suivra la même politique que 11.x : releases rolling trimestrielles, puis LTS avec support 3 ans.

### Features anticipées 12.x (non confirmé)

- Amélioration continue MariaDB Vector
- Support PostgreSQL Wire Protocol (potentiel)
- Enhanced JSON capabilities
- Performance optimizations

⚠️ **Disclaimer** : La roadmap 12.x est **indicative** et peut évoluer. Consulter la [documentation officielle](https://mariadb.org/roadmap/) pour les informations à jour.

---

## 🔗 Changements de Politique de Support

### Avant 11.4 (série 10.x)

- **Support LTS** : 5 ans
- **Fréquence LTS** : ~2 ans entre versions
- **Exemple** : 10.6 (2021-2026), 10.11 (2023-2028)

### Depuis 11.4 (série 11.x et ultérieure)

- **Support LTS** : 3 ans
- **Fréquence LTS** : ~1 an entre versions
- **Cycle rolling** : 3 mois de support
- **Exemple** : 11.4 (2024-2027), 11.8 (2025-2028)

### Justification du changement

| Aspect | Avant (5 ans) | Après (3 ans) |
|--------|---------------|---------------|
| **Innovation** | Lente | Rapide |
| **Fréquence updates** | ~2 ans | ~1 an |
| **Backports** | Complexes | Simplifiés |
| **Modernité** | Vieillissement | Toujours récent |
| **Maintenance** | Charge élevée | Charge réduite |

💡 **Avantage** : Les utilisateurs bénéficient de nouvelles fonctionnalités plus rapidement tout en conservant la stabilité LTS.

---

## 📋 Checklist Migration de Version

### Urgence de migration

| Version actuelle | Urgence | Action requise | Timeline |
|------------------|---------|----------------|----------|
| 11.7 (rolling) | 🟡 Moyenne | Migrer vers 11.8 LTS | Q1 2026 |
| 11.4 LTS | 🟢 Faible | Évaluer 11.8 (Vector/IA) | 2026-2027 |
| 10.11 LTS | 🟢 Faible | Stable jusqu'en 2028 | 2027-2028 |
| 10.6 LTS | 🟠 Élevée | Planifier migration | Q1-Q2 2026 |
| 10.5 LTS | 🔴 Critique | Migrer URGENT | Immédiat |
| < 10.5 | 🔴 Critique | Migrer IMMÉDIATEMENT | Immédiat |

---

## 🔍 Vérifier Votre Version Actuelle

### Commandes SQL

```sql
-- Afficher la version
SELECT VERSION();

-- Résultat exemple :
-- 11.8.0-MariaDB

-- Informations détaillées
SHOW VARIABLES LIKE 'version%';

-- Version commentée
SELECT @@version_comment;
```

### Commandes Shell

```bash
# Via client mariadb
mariadb --version
# mariadb  Ver 15.1 Distrib 11.8.0-MariaDB

# Via mysqladmin
mysqladmin version

# Via package manager
# Debian/Ubuntu
dpkg -l | grep mariadb-server

# RHEL/CentOS
rpm -qa | grep mariadb-server
```

---

## 📊 Comparaison des Versions LTS Actuelles

### Features par version

| Feature | 10.6 | 10.11 | 11.4 | 11.8 |
|---------|------|-------|------|------|
| **MariaDB Vector** | ❌ | ❌ | ❌ | ✅ |
| **HNSW Index** | ❌ | ❌ | ❌ | ✅ |
| **utf8mb4 défaut** | ❌ | ❌ | ❌ | ✅ |
| **UCA 14.0.0** | ❌ | ❌ | ❌ | ✅ |
| **TIMESTAMP 2106** | ❌ | ❌ | ❌ | ✅ |
| **TLS défaut** | ❌ | ❌ | ❌ | ✅ |
| **System Versioned** | ✅ | ✅ | ✅ | ✅ |
| **JSON Functions** | ✅ | ✅ | ✅ | ✅ Enhanced |
| **Window Functions** | ✅ | ✅ | ✅ | ✅ |
| **Sequences** | ✅ | ✅ | ✅ | ✅ |
| **InnoDB Default** | ✅ | ✅ | ✅ | ✅ |

### Recommandation selon priorités

| Priorité | Version recommandée |
|----------|---------------------|
| **IA/ML (Vector)** | 11.8 LTS uniquement |
| **Sécurité maximale** | 11.8 LTS |
| **Stabilité long-terme** | 10.11 LTS (jusqu'en 2028) |
| **Production éprouvée** | 10.11 LTS ou 11.4 LTS |
| **Nouvelles features** | 11.8 LTS |

---

## 🔗 Ressources Officielles

### Documentation

- 📖 [MariaDB Version Policy](https://mariadb.org/about/#maintenance-policy)
- 📖 [Release Notes 11.8](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- 📖 [Release Notes 11.4](https://mariadb.com/kb/en/mariadb-1140-release-notes/)
- 📖 [Release Notes 10.11](https://mariadb.com/kb/en/mariadb-10110-release-notes/)
- 📖 [MariaDB Downloads](https://mariadb.org/download/)

### Calendrier de release

- 📅 [MariaDB Roadmap](https://mariadb.org/roadmap/)
- 📅 [MariaDB Release Calendar](https://mariadb.com/kb/en/mariadb-releases/)

### Migration

- 🔄 [Upgrading Guide](https://mariadb.com/kb/en/upgrading/)
- 🔄 [Version-specific Upgrade Notes](https://mariadb.com/kb/en/upgrading-between-major-mariadb-versions/)

---

## ✅ Points Clés à Retenir

- **11.8 LTS** est la version actuelle (Juin 2025), supportée jusqu'en 2028
- **Support 3 ans** pour LTS depuis 11.4 (au lieu de 5 ans)
- **10.11 LTS** bénéficie encore d'un support de 5 ans (jusqu'en 2028)
- **10.6 LTS** arrive en fin de support (Juillet 2026) → Migration à planifier
- **Versions rolling** ont un support de 3 mois seulement
- **MariaDB Vector** disponible uniquement depuis 11.8
- **Série 12.x** attendue en 2026 avec 12.3 LTS en Q4 2026
- **Nouveau projet** → Toujours choisir la dernière LTS (11.8)
- **Production stable** → 10.11, 11.4 ou 11.8 selon besoins
- **Migration urgente** requise si version < 10.5

---

## 📌 Tableau de Référence Rapide

| Version | Type | GA | EOL | Support | Recommandation |
|---------|------|----|----|---------|----------------|
| **11.8** | **LTS** | **Jun 2025** | **Jun 2028** | **3 ans** | ✅ **Nouveau projet** |
| **11.4** | **LTS** | **Mai 2024** | **Mai 2027** | **3 ans** | ✅ Stable |
| 11.7 | Rolling | Déc 2024 | Mar 2025 | 3 mois | ⚠️ Dev uniquement |
| **10.11** | **LTS** | **Fév 2023** | **Fév 2028** | **5 ans** | ✅ Production |
| **10.6** | **LTS** | **Jul 2021** | **Jul 2026** | **5 ans** | ⚠️ Migrer bientôt |
| 10.5 | LTS | Mai 2020 | Jun 2025 | 5 ans | 🔴 Migrer URGENT |

---

**Source** : [MariaDB Foundation](https://mariadb.org/)  

---

## ➡️ Sections Connexes

- **Section 1.5-1.7** - Politique de versions détaillée
- **Section 19** - Migration et compatibilité
- **Annexe F** - Nouveautés MariaDB 11.8 LTS

---

💡 **Conseil** : Marquer cette page en favoris pour référence rapide lors de vos décisions de versioning !

⏭️ [Ressources et Documentation](/annexes/ressources-documentation/README.md)
