🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.5 Politique de versions : LTS vs Rolling releases 🔄

> **Niveau** : Débutant
> **Durée estimée** : 45 minutes
> **Prérequis** : Sections 1.1 à 1.4

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre ce qu'est une version LTS (Long Term Support)
- Connaître la différence entre LTS et Rolling releases
- Identifier les versions LTS actuelles de MariaDB
- Comprendre l'évolution de la politique de support (passage à 3 ans)
- Choisir la bonne stratégie de versions pour votre projet
- Planifier vos mises à jour et upgrades
- Comprendre le cycle de vie des versions

---

## Introduction

Imaginez que vous construisez une maison. Préférez-vous :
- 🏠 **Option A** : Une maison solide qui nécessite peu d'entretien pendant 3 ans
- 🏗️ **Option B** : Une maison qui reçoit de nouvelles améliorations tous les 3 mois, mais nécessite des ajustements réguliers

C'est exactement le choix entre **LTS** et **Rolling releases** !

**Pourquoi c'est important ?**
- 📅 Planifier vos mises à jour
- 🛡️ Garantir la stabilité en production
- 🚀 Accéder aux nouvelles fonctionnalités
- 💰 Optimiser vos ressources (temps, argent)
- 🔒 Assurer la sécurité et le support

Dans cette section, nous allons démystifier la politique de versions de MariaDB et vous aider à faire le bon choix pour vos projets.

---

## Qu'est-ce qu'une version LTS (Long Term Support) ?

### 📚 Définition

**LTS** = **Long Term Support** (Support à Long Terme)

Une version **LTS** est une version de MariaDB qui bénéficie d'un **support étendu** et d'une **stabilité garantie** sur une longue période.

### 🎯 Caractéristiques d'une version LTS

| Aspect | Version LTS |
|--------|-------------|
| **Durée de support** | 🛡️ **3 ans** (depuis MariaDB 11.4) |
| **Mises à jour** | 🔒 Correctifs de sécurité + bugs critiques uniquement |
| **Nouvelles features** | ❌ Aucune (stabilité maximale) |
| **Fréquence releases** | 📅 Tous les ~18 mois |
| **Stabilité** | ⭐⭐⭐⭐⭐ Maximale |
| **Innovation** | ⚡ Modérée (au moment du lancement) |
| **Cas d'usage** | 🏢 Production, entreprises, applications critiques |

### 🛡️ Que signifie "support 3 ans" ?

**Pendant 3 ans**, MariaDB Foundation s'engage à fournir :

✅ **Correctifs de sécurité (Security patches)**
```
Vulnérabilité découverte → Patch publié sous 72h-7 jours
```

✅ **Corrections de bugs critiques**
```
Bug bloquant en production → Fix prioritaire
```

✅ **Compatibilité garantie**
```
Pas de breaking changes → Migration facile
```

❌ **PAS de nouvelles fonctionnalités**
```
Nouvelles features → Attendez la prochaine LTS ou utilisez Rolling
```

### 📊 Exemple concret : MariaDB 11.4 LTS

**MariaDB 11.4 LTS** (sortie : Mai 2024)

```
Mai 2024              Mai 2025              Mai 2026              Mai 2027
   │                     │                     │                     │
   ├─────────────────────┼─────────────────────┼─────────────────────┤
   │   Année 1           │   Année 2           │   Année 3           │
   │                     │                     │                     │
   │ ✅ Support actif    │ ✅ Support actif    │ ✅ Support actif    │
   │ 🐛 Bug fixes        │ 🐛 Bug fixes        │ 🐛 Bug fixes        │
   │ 🔒 Security patches │ 🔒 Security patches │ 🔒 Security patches │
   │                     │                     │                     │
   └─────────────────────┴─────────────────────┴─────────────────────┘
                                                                 Fin support
                                                                 (Mai 2027)
```

**Releases pendant le cycle** :
- **11.4.0** : Version initiale (Mai 2024)
- **11.4.1** : Bug fixes (Juillet 2024)
- **11.4.2** : Security patches (Septembre 2024)
- **11.4.3** : Bug fixes (Novembre 2024)
- ... *(et ainsi de suite pendant 3 ans)*

💡 **Note** : Les mises à jour de patch (11.4.x) sont **sûres et recommandées** - elles ne cassent rien !

---

## Qu'est-ce qu'une Rolling release ?

### 📚 Définition

Une **Rolling release** (version continue) est une version de MariaDB publiée régulièrement (tous les ~3 mois) avec les dernières fonctionnalités et améliorations.

### 🎯 Caractéristiques des Rolling releases

| Aspect | Rolling Release |
|--------|-----------------|
| **Durée de support** | ⏱️ **Jusqu'à la prochaine LTS** (~6-12 mois) |
| **Mises à jour** | 🚀 Nouvelles features + optimisations |
| **Nouvelles features** | ✅ Toutes les dernières innovations |
| **Fréquence releases** | 📅 Tous les ~3 mois |
| **Stabilité** | ⭐⭐⭐⭐ Bonne (mais évolutions fréquentes) |
| **Innovation** | ⚡⚡⚡⚡⚡ Maximale |
| **Cas d'usage** | 🧪 Développement, early adopters, nouveaux projets |

### 🚀 Que signifie "Rolling" ?

Le terme **"Rolling"** (qui roule) signifie que les versions se succèdent **continuellement**, comme une roue qui tourne :

```
12.0 → 12.1 → 12.2 → 12.3 LTS
 (3m)   (3m)   (3m)    └─► Devient LTS
```

Chaque version apporte des nouveautés, et quand suffisamment matures, une version devient **LTS**.

### 📊 Exemple concret : Série 12.x Rolling

**MariaDB 12.0 - 12.2** (Rolling releases)

```
Sep 2024    Déc 2024    Mars 2025    Juin 2025 (prévu)
   │           │            │              │
   ├───────────┼────────────┼──────────────┤
12.0        12.1         12.2          12.3 LTS
   │           │            │              │
   │ 🚀 New    │ 🚀 New     │ 🚀 New       │ 🛡️ LTS
   │ features  │ features   │ features     │ (3 ans)
   │           │            │              │
   │ Support   │ Support    │ Support      │
   │ ~3 mois   │ ~3 mois    │ ~6 mois      │
   └───────────┴────────────┴──────────────┘
                                        Prochaine LTS
```

**Nouveautés apportées** :
- **12.0** : Nouvelles optimisations InnoDB, amélioration Vector
- **12.1** : Support nouveau standard SQL, features JSON avancées
- **12.2** : Performance ColumnStore, nouvelles fonctions
- **12.3 LTS** : Stabilisation → Devient LTS (support 3 ans)

💡 **Important** : Les Rolling releases sont **stables et utilisables**, mais changent plus fréquemment.

---

## Historique : L'évolution de la politique de versions

### 📅 Chronologie complète

#### Avant 2021 : Pas de LTS formalisé

```
2009-2020 : Releases régulières
├─ 5.1, 5.2, 5.3, 5.5 (basées sur MySQL)
├─ 10.0, 10.1, 10.2, 10.3, 10.4, 10.5
└─ Support variable (1-2 ans)
```

#### 2021 : Première version LTS officielle 🎉

**MariaDB 10.6 LTS** (Juillet 2021)
- Premier LTS officiel avec support **5 ans**
- Fin de support : Juillet 2026

#### 2023 : Confirmation du modèle LTS

**MariaDB 10.11 LTS** (Février 2023)
- Support **5 ans**
- Fin de support : Février 2028

#### 2024 : Changement majeur - Support 3 ans 🆕

**MariaDB 11.4 LTS** (Mai 2024)
- 🔄 **Changement de politique** : Support passe à **3 ans** (au lieu de 5)
- Raison : Permettre des innovations plus rapides
- Fin de support : Mai 2027

#### 2025 : Nouvelle LTS avec innovations majeures 🆕

**MariaDB 11.8 LTS** (Juin 2025) ← **Version actuelle de référence**
- Support **3 ans**
- **Nouveautés majeures** : MariaDB Vector, TLS défaut, TIMESTAMP 2106
- Fin de support : Juin 2028

### 📊 Tableau récapitulatif des versions LTS

| Version | Date GA | Type | Durée support | Fin support | Statut |
|---------|---------|------|---------------|-------------|--------|
| **10.6** | Juil 2021 | LTS | 5 ans | Juil 2026 | ✅ Supportée |
| **10.11** | Fév 2023 | LTS | 5 ans | Fév 2028 | ✅ Supportée |
| **11.4** | Mai 2024 | LTS | 3 ans 🆕 | Mai 2027 | ✅ Supportée |
| **11.8** | Juin 2025 | LTS | 3 ans | Juin 2028 | ✅ Supportée |
| **12.3** | Q2 2026 | LTS (prévu) | 3 ans | Q2 2029 | 🔮 Futur |

---

## Pourquoi le passage de 5 à 3 ans ? 🔄

### 🤔 Les raisons du changement

**Annoncé en 2024**, le passage à **3 ans de support LTS** répond à plusieurs objectifs :

#### 1️⃣ **Innovation plus rapide**

**Avant (5 ans)** :
```
10.6 LTS ────────────────────► (2021-2026 = 5 ans)
              ↓
         Trop long pour
       nouvelles features
```

**Maintenant (3 ans)** :
```
11.4 LTS ──────────► (2024-2027 = 3 ans)
              ↓
    Cycle plus court
  = Nouvelles LTS plus
     fréquentes avec
    nouvelles features
```

#### 2️⃣ **Alignement avec l'industrie**

| Base de données | Cycle LTS |
|-----------------|-----------|
| **PostgreSQL** | ~5 ans |
| **MySQL** | 5-8 ans (dépend version) |
| **MariaDB (nouveau)** | **3 ans** |
| **Ubuntu LTS** | 5 ans (serveur) |

💡 **3 ans** est un bon compromis : assez long pour la stabilité, assez court pour l'innovation.

#### 3️⃣ **Ressources de maintenance**

Maintenir plusieurs LTS sur 5 ans chacune = beaucoup de versions actives simultanément.

**Exemple 2025** :
```
Versions actives :
├─ 10.6 LTS (2021-2026) : 5 ans
├─ 10.11 LTS (2023-2028) : 5 ans
├─ 11.4 LTS (2024-2027) : 3 ans
└─ 11.8 LTS (2025-2028) : 3 ans

= 4 versions LTS à maintenir en parallèle !
```

Avec 3 ans → Moins de versions actives → Ressources concentrées sur les versions récentes.

#### 4️⃣ **Adoption des nouvelles fonctionnalités**

**Problème avec 5 ans** :
- Utilisateurs restent sur anciennes versions trop longtemps
- Nouvelles features (Vector, S3, etc.) adoptées lentement

**Avec 3 ans** :
- Migration plus fréquente → Adoption plus rapide des innovations
- Écosystème plus dynamique

### ✅ Avantages du cycle 3 ans

| Pour les utilisateurs | Pour MariaDB Foundation |
|----------------------|-------------------------|
| ✅ Nouvelles LTS plus fréquentes | ✅ Moins de versions en parallèle |
| ✅ Accès plus rapide aux innovations | ✅ Focus sur versions récentes |
| ✅ Moins de gap entre LTS | ✅ Meilleure qualité du support |
| ✅ Migration moins complexe | ✅ Innovation plus rapide |

### ⚠️ Implications pour les utilisateurs

**Si vous utilisez une ancienne LTS 5 ans (10.6, 10.11)** :
- ✅ Vous gardez les 5 ans promis (contrat respecté)
- ⏰ Mais les **nouvelles LTS** auront 3 ans seulement
- 📅 Planifiez vos migrations en conséquence

**Si vous démarrez un nouveau projet** :
- 🎯 Comptez sur **3 ans de support** pour les nouvelles LTS
- 📅 Planifiez une migration tous les ~2,5 ans pour rester à jour
- 🔄 C'est un rythme sain pour bénéficier des innovations

---

## LTS vs Rolling : Comparaison détaillée

### 📊 Tableau comparatif complet

| Critère | LTS (ex: 11.8) | Rolling (ex: 12.1) |
|---------|----------------|-------------------|
| **Support** | 🛡️ 3 ans | ⏱️ Jusqu'à prochaine LTS (~6-12 mois) |
| **Stabilité** | ⭐⭐⭐⭐⭐ Maximale | ⭐⭐⭐⭐ Bonne |
| **Mises à jour** | 🔒 Bug/Security fixes uniquement | 🚀 Nouvelles features régulières |
| **Breaking changes** | ❌ Aucun | ⚠️ Possibles (rares) |
| **Nouveautés** | ❌ Figées à la release | ✅ Continues |
| **Fréquence patches** | 📅 ~Mensuel (si nécessaire) | 📅 ~Trimestriel (nouvelle version) |
| **Documentation** | ✅ Stable et complète | 🔄 Évolutive |
| **Communauté** | 👥👥👥👥👥 Très large | 👥👥👥 Active mais plus petite |
| **Tests en production** | ✅ Éprouvée par millions d'users | 🧪 Testée mais moins répandue |
| **Risque** | 🟢 Très faible | 🟡 Faible à modéré |

### 🎯 Cas d'usage recommandés

#### Choisissez **LTS** si :

✅ **Production critique**
```
Site e-commerce, banking, healthcare
→ Stabilité > Innovation
```

✅ **Équipe IT limitée**
```
Pas de temps pour mises à jour fréquentes
→ "Set and forget" pendant 3 ans
```

✅ **Conformité réglementaire**
```
Certifications, audits de sécurité
→ Version stable et documentée
```

✅ **Applications enterprise**
```
ERP, CRM, systèmes critiques
→ Support garanti
```

✅ **Projet long terme**
```
Déploiement prévu pour 2-3+ ans
→ Éviter migrations fréquentes
```

#### Choisissez **Rolling** si :

✅ **Nouveau projet / Développement**
```
Application en cours de création
→ Accès aux dernières features
```

✅ **Besoin de nouvelles fonctionnalités**
```
MariaDB Vector pour IA, S3 storage, etc.
→ Innovation > Stabilité
```

✅ **Early adopters**
```
Équipe technique expérimentée
→ Suivre l'innovation
```

✅ **Environnement de test**
```
Dev, staging, CI/CD
→ Tester les futures LTS
```

✅ **Projet court terme**
```
POC, MVP, prototype
→ Pas besoin de support long
```

### 💡 Stratégie hybride

Beaucoup d'organisations utilisent **les deux** :

```
┌─────────────────────────────────────────────┐
│              Production                     │
│          MariaDB 11.8 LTS                   │
│        (Stabilité maximale)                 │
└─────────────────────────────────────────────┘
                    ▲
                    │ Validation
                    │
┌─────────────────────────────────────────────┐
│            Staging / PreProd                │
│          MariaDB 12.2 Rolling               │
│      (Test nouvelles versions)              │
└─────────────────────────────────────────────┘
                    ▲
                    │ Développement
                    │
┌─────────────────────────────────────────────┐
│          Dev / Local                        │
│     MariaDB 12.2 Rolling ou 11.8 LTS        │
│   (Développement avec dernières features)   │
└─────────────────────────────────────────────┘
```

**Avantages** :
- ✅ Production stable sur LTS
- ✅ Test des futures versions sur Rolling
- ✅ Migration anticipée et préparée
- ✅ Équilibre stabilité/innovation

---

## Planning de releases (2024-2028)

### 📅 Calendrier des versions

```
2024
├── Mai      : 11.4 LTS (support 3 ans → 2027)
├── Sep      : 12.0 Rolling
└── Déc      : 12.1 Rolling

2025
├── Mars     : 12.2 Rolling
├── Juin     : 11.8 LTS (support 3 ans → 2028) ← Nous sommes ici
└── Sep      : 12.3 Rolling

2026
├── Q2       : 12.3 ou 12.4 LTS (anticipé, 3 ans → 2029)
└── ...      : Rolling continues (13.x ?)

2027
├── Mai      : Fin support 11.4 LTS
└── ...      : Nouvelle LTS potentielle

2028
├── Fév      : Fin support 10.11 LTS (5 ans)
├── Juin     : Fin support 11.8 LTS (3 ans)
└── ...      : ...
```

### 🔮 Prévisions long terme

**Modèle établi** :
- 📅 **Nouvelle LTS** : Tous les ~18 mois
- 📅 **Rolling releases** : Tous les ~3 mois
- ⏰ **Support LTS** : 3 ans systématiquement

**Prochaines LTS anticipées** :
```
11.4 LTS (Mai 2024) ──────► 11.8 LTS (Juin 2025) ──────► ~12.3 LTS (Q2 2026)
        └─ 13 mois ─────┘              └─ ~12 mois ─────┘
```

---

## Comment choisir la bonne version ?

### 🎯 Arbre de décision

```
┌─────────────────────────────────────┐
│   Nouveau projet ou migration ?     │
└──────────┬──────────────────────────┘
           │
           ▼
    ┌──────────────┐
    │ Quel usage ? │
    └──────┬───────┘
           │
    ┌──────┴──────────────────────────┐
    │                                 │
    ▼                                 ▼
Production                        Dev/Test
    │                                 │
    ▼                                 ▼
┌────────────────┐            ┌────────────────┐
│ Besoin feature │            │ Tester futures │
│ spécifique ?   │            │ versions ?     │
└────┬─────┬─────┘            └────┬─────┬─────┘
     │     │                       │     │
   Non   Oui                     Non   Oui
     │     │                       │     │
     ▼     ▼                       ▼     ▼
   LTS  Rolling                  LTS  Rolling
 (11.8)  (12.x)                (11.8)  (12.x)
```

### 📝 Questionnaire de décision

**Posez-vous ces questions** :

1. **Stabilité ou Innovation ?**
   - Stabilité → **LTS**
   - Innovation → **Rolling**

2. **Durée du projet ?**
   - 2-3+ ans → **LTS**
   - < 1 an ou POC → **Rolling**

3. **Ressources IT ?**
   - Équipe petite/moyenne → **LTS**
   - Équipe tech expérimentée → **Rolling** possible

4. **Tolérance au changement ?**
   - Faible → **LTS**
   - Élevée → **Rolling**

5. **Besoin de features récentes ?**
   - Non → **LTS**
   - Oui (Vector, S3, etc.) → **Rolling** ou **dernière LTS**

6. **Conformité / Audits ?**
   - Oui → **LTS**
   - Non → **Rolling** OK

### 🎯 Recommandations par profil

#### 🏢 Entreprise / Production
```
Recommandation : LTS (11.8 ou 11.4)
Raison : Stabilité, support 3 ans, conformité
```

#### 🚀 Startup / Agile
```
Recommandation : LTS récente (11.8) ou Rolling (12.x)
Raison : Innovation + flexibilité
```

#### 🎓 Développeur / Apprenant
```
Recommandation : LTS récente (11.8)
Raison : Documentation stable, communauté large
```

#### 🔬 Chercheur / Early adopter
```
Recommandation : Rolling (12.x)
Raison : Accès aux dernières fonctionnalités
```

#### 🏛️ Administration publique
```
Recommandation : LTS (11.4 ou 10.11 si migration complexe)
Raison : Processus de validation long, stabilité requise
```

---

## Migration entre versions

### 🔄 Stratégies de migration

#### Scénario 1 : LTS → LTS (recommandé)

**Exemple** : 10.11 LTS → 11.8 LTS

```
2023                2025                2028
 │                   │                   │
10.11 LTS ───────────┼─────────────────────► Fin support 2028
 │                   │
 │                   ├──────────────────────► 11.8 LTS (Juin 2028)
 │                   │ Migration
 └───────────────────┘ (période flexible)
```

**Avantages** :
- ✅ Migration planifiée calmement
- ✅ Documentation complète des changements
- ✅ Pas de précipitation (overlap de support)
- ✅ Communauté expérimentée

**Timeline recommandée** :
```
Phase 1 (6 mois avant) : Tests en staging
Phase 2 (3 mois avant) : Validation complète
Phase 3 (Jour J)       : Migration production
Phase 4 (1 mois après) : Monitoring renforcé
```

#### Scénario 2 : Rolling → LTS

**Exemple** : 12.2 Rolling → 12.3 LTS

```
2025
 │
 ├─ 12.2 Rolling (Mars)
 │    │
 │    └─ 12.3 LTS (Sep) ← Migration simple !
```

**Avantages** :
- ✅ Migration très simple (même série)
- ✅ Changements mineurs
- ✅ Stabilisation d'une version connue

#### Scénario 3 : Ancienne version → LTS récente

**Exemple** : 10.5 (non-LTS) → 11.8 LTS

```
2020        2025
 │           │
10.5 ────────┼───► Non supportée
             │
             └────► 11.8 LTS
                (Migration majeure)
```

**Attention** :
- ⚠️ Gap important de versions
- ⚠️ Nombreux changements à valider
- ⚠️ Peut nécessiter migration intermédiaire

**Recommandation** : Tester TOUS les aspects de l'application !

### 🛡️ Bonnes pratiques de migration

1. **Toujours tester en staging** avant production
2. **Lire les release notes** et changelog complets
3. **Vérifier les breaking changes**
4. **Sauvegarder** (backup complet avant migration)
5. **Planifier un rollback** (plan B)
6. **Monitorer** intensivement après migration
7. **Migrer progressivement** (canary deployment si possible)

---

## ✅ Points clés à retenir

- 📚 **LTS** = Long Term Support (3 ans de support depuis 11.4)
- 🚀 **Rolling** = Nouvelles versions tous les ~3 mois
- 🔄 **Changement majeur 2024** : Support passe de 5 à 3 ans pour nouvelles LTS
- 📅 **LTS actuelles** : 10.6 (5 ans), 10.11 (5 ans), 11.4 (3 ans), **11.8 (3 ans)**
- 🎯 **LTS pour production** : Stabilité, support garanti, updates sécurité uniquement
- 🧪 **Rolling pour dev/test** : Dernières features, innovation rapide
- ⏱️ **Nouvelle LTS** : Tous les ~18 mois
- 🔄 **Rolling releases** : Tous les ~3 mois
- 🛡️ **Version recommandée 2025** : MariaDB 11.8 LTS
- 📊 **Cycle de vie** : LTS figée sauf bugs/sécurité, Rolling évolutive
- 🏢 **Entreprise** → LTS | 🚀 **Startup/Dev** → LTS récente ou Rolling
- 🔄 **Migration LTS → LTS** : Planifiée calmement avec overlap de support
- ⚠️ **3 ans** = Rythme de migration tous les ~2,5 ans recommandé

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB Release Policy](https://mariadb.org/about/#maintenance-policy)
- [MariaDB Releases](https://mariadb.com/kb/en/mariadb-server-release-dates/)
- [Supported Versions](https://mariadb.com/kb/en/mariadb-server-versions/)

### 📅 Planning et roadmap
- [MariaDB Roadmap](https://mariadb.org/roadmap/)
- [Release Calendar](https://mariadb.com/kb/en/release-calendar/)
- [EOL Dates](https://endoflife.date/mariadb)

### 📰 Annonces officielles
- [3-Year LTS Announcement (2024)](https://mariadb.org/blog/)
- [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)

### 🔧 Migration guides
- [Upgrading MariaDB](https://mariadb.com/kb/en/upgrading/)
- [Upgrade Guide 10.x to 11.x](https://mariadb.com/kb/en/upgrading-from-mariadb-10-to-mariadb-11/)

---

## ➡️ Section suivante

**[1.6 - Cycle de support : 3 ans LTS (depuis 11.4)](./06-cycle-support-lts.md)** 🆕

Dans la section suivante, nous approfondirons le **cycle de support de 3 ans** : qu'est-ce que cela signifie concrètement ? Quelles sont les phases du cycle de vie d'une version LTS ? Comment planifier vos mises à jour ? Nous détaillerons également les **bonnes pratiques** pour maintenir votre MariaDB à jour tout en garantissant la stabilité de vos applications.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.5*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Cycle de support : 3 ans LTS (depuis 11.4), rolling trimestriel](/01-introduction-fondamentaux/06-cycle-support-lts.md)
