🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.6 Cycle de support : 3 ans LTS (depuis 11.4), rolling trimestriel 🆕

> **Niveau** : Débutant
> **Durée estimée** : 30 minutes
> **Prérequis** : Section 1.5 (Politique de versions LTS vs Rolling)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre concrètement ce que signifie "3 ans de support"
- Connaître les phases du cycle de vie d'une version LTS
- Comprendre le rythme trimestriel des Rolling releases
- Savoir ce qui est inclus (et exclu) du support
- Planifier vos mises à jour et maintenance
- Anticiper la fin de vie d'une version
- Appliquer les bonnes pratiques de maintenance

---

## Introduction

Dans la section précédente, vous avez découvert la différence entre **LTS** et **Rolling releases**. Maintenant, approfondissons : que signifie **concrètement** "3 ans de support" ? Qu'est-ce qui est garanti ? Qu'est-ce qui ne l'est pas ?

**Pourquoi c'est crucial de comprendre ?**
- 📅 **Planifier** vos projets sur le long terme
- 💰 **Budgétiser** les migrations et mises à jour
- 🛡️ **Garantir** la sécurité de vos systèmes
- ⏰ **Anticiper** les fins de support
- 🎯 **Éviter** les surprises et les urgences

Dans cette section, nous allons décortiquer le **cycle de support LTS de 3 ans** et le **rythme trimestriel des Rolling releases** pour que vous sachiez exactement à quoi vous attendre.

---

## Le cycle de support LTS de 3 ans : Vue d'ensemble

### 📅 Qu'est-ce que "3 ans de support" ?

**Depuis MariaDB 11.4 (Mai 2024)**, chaque nouvelle version LTS bénéficie de **3 ans de support actif**.

```
┌─────────────────────────────────────────────────────────────┐
│              Cycle de vie d'une LTS (3 ans)                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Lancement ──────────────► Support actif ─────────► EOL     │
│   (GA)                       (3 ans)               (Fin)    │
│    │                            │                    │      │
│    ├────────────────────────────┼────────────────────┤      │
│    │         Année 1            │   Année 2   │ An3  │      │
│    │                            │             │      │      │
│    ▼                            ▼             ▼      ▼      │
│  ┌───────┬──────────┬──────────┬──────────┬────────┬─────┐  │
│  │ x.x.0 │  x.x.1   │  x.x.2   │  x.x.3   │ x.x.4  │ EOL │  │
│  │ GA    │  Bug fix │ Security │ Bug fix  │Security│     │  │
│  └───────┴──────────┴──────────┴──────────┴────────┴─────┘  │
│     ▲         ▲          ▲          ▲         ▲        ▲    │
│     │         │          │          │         │        │    │
│   Stable  Patches    Patches    Patches   Patches  Fin      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 🛡️ Ce qui EST inclus dans le support

Pendant **3 ans**, MariaDB Foundation garantit :

#### 1️⃣ **Correctifs de sécurité (Security patches)** 🔒

**Définition** : Corrections de vulnérabilités de sécurité découvertes.

**Délai de publication** :
- 🚨 **Critique** : 24-72 heures
- ⚠️ **Important** : 7 jours
- 📢 **Modéré** : 30 jours

**Exemples de patches sécurité** :
```
CVE-2024-XXXXX : SQL injection possible dans certains cas
→ Patch 11.4.3 publié sous 48h

CVE-2024-YYYYY : Escalade de privilèges avec certains plugins
→ Patch 11.4.4 publié sous 7 jours
```

**Communication** :
- 📧 Mailing list de sécurité
- 🔔 Blog officiel MariaDB
- 📰 Security advisories
- 📱 Notifications via MariaDB SkySQL (si utilisé)

#### 2️⃣ **Corrections de bugs critiques** 🐛

**Définition** : Bugs qui empêchent l'utilisation normale ou causent corruption de données.

**Critères de criticité** :
- 🔴 **Critical** : Perte de données, crash serveur
- 🟠 **Major** : Fonctionnalité clé cassée
- 🟡 **Normal** : Bug gênant mais workaround existe
- 🟢 **Minor** : Bug cosmétique

💡 **Seuls Critical et Major** sont corrigés dans les LTS. Les bugs Normal/Minor attendent la prochaine version majeure.

**Exemple de bug critique corrigé** :
```sql
-- Bug dans 11.4.0 : CRASH lors de certain ALTER TABLE
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ERROR 2013 (HY000): Lost connection to MySQL server during query

-- Corrigé dans 11.4.1 (release suivante)
```

#### 3️⃣ **Mises à jour mineures (Point releases)** 📦

**Format de version** : `MAJOR.MINOR.PATCH`

Exemple : **11.4.5**
- **11** = Série majeure
- **4** = Version LTS
- **5** = 5ème patch release

**Fréquence** :
- 🗓️ **Planifiées** : ~Tous les 2-3 mois
- 🚨 **Urgentes** : Dès que nécessaire (sécurité critique)

**Exemple de cycle de patches** :
```
11.4.0 ─► 11.4.1 ─► 11.4.2 ─► 11.4.3 ─► ... ─► 11.4.N
(GA)      (Bug)     (Sec)     (Bug)              (Final)
Mai 24    Jul 24    Sep 24    Nov 24             Avr 27
```

#### 4️⃣ **Rétrocompatibilité garantie** ✅

**Promesse** : Aucun breaking change pendant les 3 ans.

**Cela signifie** :
```sql
-- Une application qui fonctionne sur 11.4.0
-- DOIT fonctionner sur 11.4.N sans modification
-- du code applicatif

-- ✅ Garanti compatible
11.4.0 → 11.4.1 → 11.4.2 → ... → 11.4.N

-- ❌ PAS garanti compatible (version majeure différente)
11.4.N → 11.5.0 (cette version n'existe pas, juste un exemple)
```

**Exceptions très rares** :
- Correction d'un comportement documenté comme bug
- Changement nécessaire pour la sécurité
- Toujours documenté et annoncé à l'avance

#### 5️⃣ **Documentation et support communautaire** 📚

**Documentation** :
- ✅ Knowledge Base maintenue à jour
- ✅ Release notes détaillées
- ✅ Changelog complet
- ✅ Known issues listés

**Support communautaire** :
- 💬 Forums actifs
- 🗨️ Zulip chat
- 📧 Mailing lists
- 🐛 Bug tracker (JIRA)

### ❌ Ce qui N'EST PAS inclus dans le support

**Pendant les 3 ans LTS, il n'y a PAS** :

#### 1️⃣ **Nouvelles fonctionnalités** 🚫

```
❌ Pas de :
- Nouvelles syntaxes SQL
- Nouveaux storage engines
- Nouvelles fonctions
- Nouvelles optimisations (sauf bugs)

✅ Seulement :
- Corrections de bugs
- Patches de sécurité
```

**Exemple** :
```sql
-- MariaDB 11.4.0 : Pas de fonction hypothétique NEW_FUNCTION()
SELECT NEW_FUNCTION(column) FROM table;
ERROR 1305 (42000): FUNCTION NEW_FUNCTION does not exist

-- MariaDB 11.4.5 : Toujours pas présente
-- (il faudra attendre 11.8 ou 12.x)
```

#### 2️⃣ **Optimisations de performance** 🚫

Sauf si l'optimisation corrige un bug de performance critique.

```
❌ Pas de :
- Amélioration générale des performances
- Optimisations du query optimizer
- Nouveaux algorithmes

✅ Seulement :
- Fix de régression de performance (bug)
- Correction de requêtes anormalement lentes (bug)
```

#### 3️⃣ **Dépréciations / Suppressions** 🚫

Aucune fonctionnalité existante ne sera dépréciée ou supprimée.

```
✅ Si une feature existe en 11.4.0
→ Elle existera toujours en 11.4.N

Exemple :
- mysql_native_password existe en 11.4.0
- Il existera toujours en 11.4.N
```

#### 4️⃣ **Support de nouvelles plateformes** 🚫

Les plateformes supportées à la sortie le restent, mais pas d'ajout.

```
❌ Pas de :
- Support d'une nouvelle version d'OS
- Support d'une nouvelle architecture CPU
- Support de nouveaux compilateurs

✅ Maintenu :
- Toutes les plateformes de la release initiale
```

---

## Les trois phases du cycle de vie LTS

Une version LTS traverse **3 phases** distinctes pendant ses 3 ans de vie.

### Phase 1 : Stabilisation (Mois 1-6) 🏗️

**Objectif** : Corriger les bugs découverts après le lancement.

```
┌─────────────────────────────────────────────┐
│        Phase 1 : STABILISATION              │
│         (6 premiers mois)                   │
├─────────────────────────────────────────────┤
│                                             │
│  GA ──► x.x.1 ──► x.x.2 ──► x.x.3           │
│        (1 mois) (1 mois)  (2 mois)          │
│                                             │
│  ✅ Corrections bugs fréquentes             │
│  ✅ Optimisations rapides                   │
│  ⚠️ Risque : Bugs non détectés en beta      │
│                                             │
└─────────────────────────────────────────────┘
```

**Caractéristiques** :
- 🐛 **Fréquence élevée** de patches (toutes les 2-4 semaines)
- 🧪 **Tests intensifs** par la communauté
- 🔍 **Découverte** de bugs edge cases
- ⚡ **Corrections rapides**

**Recommandation** :
```
🏢 Production critique : Attendre 3-6 mois (x.x.3+)
🚀 Production standard : x.x.1+ acceptable
🧪 Dev/Staging : x.x.0 OK
```

**Exemple : MariaDB 11.4**
```
Mai 2024  : 11.4.0 GA
Juin 2024 : 11.4.1 (bugs mineurs)
Juil 2024 : 11.4.2 (corrections)
Sep 2024  : 11.4.3 (stabilisation)
Nov 2024  : 11.4.4 (mature)
```

### Phase 2 : Maturité (Mois 7-30) ⭐

**Objectif** : Maintenance stable avec peu de changements.

```
┌─────────────────────────────────────────────┐
│           Phase 2 : MATURITÉ                │
│            (2 ans environ)                  │
├─────────────────────────────────────────────┤
│                                             │
│  x.x.4 ──────► x.x.N ──────► x.x.N+M        │
│       (3-6 mois) (3-6 mois)                 │
│                                             │
│  ✅ Très stable                             │
│  ✅ Patches espacés (sécurité surtout)      │
│  ✅ Production-ready                        │
│  ⚠️ Risque : Très faible                    │
│                                             │
└─────────────────────────────────────────────┘
```

**Caractéristiques** :
- 🛡️ **Stabilité maximale**
- 🔒 **Sécurité** : Seule raison principale de release
- 📅 **Fréquence réduite** : Tous les 3-6 mois
- ⭐ **Recommandée pour production**

**Recommandation** :
```
🏢 Production critique : ✅ Parfait
🚀 Production standard : ✅ Idéal
🧪 Dev/Staging : ✅ OK (ou plus récent)
```

**Exemple** :
```
Année 1 (mois 7-12) : 11.4.4, 11.4.5
Année 2 (mois 13-24) : 11.4.6, 11.4.7, 11.4.8
Année 3 (mois 25-30) : 11.4.9, 11.4.10
```

### Phase 3 : Fin de vie (Mois 31-36) ⏰

**Objectif** : Support minimal, préparation migration.

```
┌─────────────────────────────────────────────┐
│          Phase 3 : FIN DE VIE               │
│           (6 derniers mois)                 │
├─────────────────────────────────────────────┤
│                                             │
│  x.x.N ──────────────────────► EOL          │
│                        (6 mois)             │
│                                             │
│  ⚠️ Support réduit (sécurité uniquement)    │
│  📢 Annonces de migration                   │
│  🔄 Préparation upgrade                     │
│  ❗ Risque : Pas de nouveaux patches après  │
│                                             │
└─────────────────────────────────────────────┘
```

**Caractéristiques** :
- ⚠️ **Sécurité uniquement** (bugs critiques possibles)
- 📢 **Annonces** répétées de fin de support
- 🔄 **Incitation** à migrer vers nouvelle LTS
- ❌ **Pas de nouveau patch** après EOL

**Recommandation** :
```
🏢 Production critique : ⚠️ Planifier migration MAINTENANT
🚀 Production standard : ⚠️ Tester nouvelle LTS en staging
🧪 Dev/Staging : 🔄 Migrer vers version récente
```

**Timeline de fin de vie** :
```
Mois 31 : 📢 Première annonce "6 mois avant EOL"
Mois 33 : 📢 Deuxième annonce "4 mois avant EOL"
Mois 35 : 📢 Dernière annonce "1 mois avant EOL"
Mois 36 : 🛑 EOL - Fin du support
```

---

## Le cycle Rolling : Releases trimestrielles

### 📅 Qu'est-ce que "Rolling trimestriel" ?

**Rolling releases** sont publiées tous les **~3 mois** (trimestriel) avec les dernières fonctionnalités.

```
┌─────────────────────────────────────────────────────────────┐
│            Cycle Rolling : ~3 mois par version              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  12.0 ──► 12.1 ──► 12.2 ──► 12.3 LTS                        │
│  (3m)     (3m)     (3m)     (devient LTS)                   │
│                                                             │
│  Sep 24   Déc 24   Mars 25  Juin 25                         │
│                                                             │
│  ↓         ↓        ↓         ↓                             │
│  Beta →   Beta →   Beta →    Stable                         │
│  Dev      Dev      Dev       Production OK                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 🔄 Phases d'une Rolling release

Chaque Rolling passe par **3 phases** sur ~3 mois :

#### Phase 1 : Beta / RC (4-6 semaines) 🧪

```
Semaine 1-2 : Alpha (interne)
Semaine 3-4 : Beta (publique)
Semaine 5-6 : Release Candidate
```

**Qui teste ?** :
- 🔬 Early adopters
- 🧪 Environnements de dev
- 💻 Contributeurs MariaDB

#### Phase 2 : GA (General Availability) (1-2 semaines) 🚀

```
Semaine 7 : Release GA
- Annonce officielle
- Documentation finalisée
- Images Docker publiées
```

**Utilisateurs** :
- ✅ Projets non critiques
- ✅ Nouveaux projets
- ✅ Staging/Test environments

#### Phase 3 : Stabilisation (Reste du trimestre) ⭐

```
Semaine 8-12 : Patches mineurs
- x.x.1 : Bugs découverts rapidement
- x.x.2 : Corrections finales
```

**Production ?** :
- ⚠️ Possible si équipe technique expérimentée
- ⚠️ Acceptable si besoin de nouvelles features
- 🛡️ LTS recommandée pour production critique

### 📊 Exemple concret : Série 12.x

```
┌───────────────────────────────────────────────────────────┐
│              MariaDB 12.x Rolling Timeline                │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Sep 2024          Déc 2024          Mars 2025            │
│     ▼                 ▼                 ▼                 │
│  ┌──────┐         ┌──────┐         ┌──────┐               │
│  │ 12.0 │────────►│ 12.1 │────────►│ 12.2 │────────►      │
│  └──────┘  3 mois └──────┘  3 mois └──────┘  3 mois       │
│     │                │                │                   │
│  Features:        Features:        Features:              │
│  - Optim A        - Feature B      - Feature C            │
│  - Fix X          - Optim Y        - Fix Z                │
│                                                           │
│  Support:         Support:         Support:               │
│  Jusqu'à 12.1     Jusqu'à 12.2     Jusqu'à 12.3 LTS       │
│  (3 mois)         (3 mois)         (6 mois)               │
│                                                           │
│  Juin 2025                                                │
│     ▼                                                     │
│  ┌──────────┐                                             │
│  │ 12.3 LTS │ ◄──── Devient LTS (support 3 ans)           │
│  └──────────┘                                             │
│     │                                                     │
│  Support: 3 ans (Juin 2025 → Juin 2028)                   │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### ⚠️ Support limité des Rolling

**Important** : Une Rolling release est supportée **seulement jusqu'à la suivante** !

```
12.0 sortie : Sep 2024
  │
  ├─ Support actif (Sep-Nov 2024)
  │
12.1 sortie : Déc 2024
  │
  └─ 12.0 N'EST PLUS SUPPORTÉE !
     (Upgrade vers 12.1 ou restez sur LTS)
```

💡 **Conseil** : Si vous utilisez Rolling, prévoyez de migrer **tous les 3-6 mois**.

---

## Planification des mises à jour

### 🗓️ Stratégie de mise à jour pour LTS

#### Stratégie conservatrice (entreprises) 🛡️

```
┌──────────────────────────────────────────────────┐
│     Stratégie : ATTENDRE LA STABILISATION        │
├──────────────────────────────────────────────────┤
│                                                  │
│  x.x.0 GA ──► Attendre ──► x.x.3+ ──► Upgrade    │
│   (Mois 0)    (6 mois)     (Mois 6)              │
│                                                  │
│  ✅ Avantages :                                  │
│  - Version très stable                           │
│  - Bugs majeurs déjà corrigés                    │
│  - Documentation mature                          │
│                                                  │
│  ❌ Inconvénients :                              │
│  - Pas d'accès immédiat aux nouvelles features   │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Timeline recommandée** :
```
Mois 0  : Nouvelle LTS sort (ex: 11.8.0) - Ne rien faire
Mois 1-3: Suivre les releases (11.8.1, 11.8.2)
Mois 4-6: Tester en staging (11.8.3+)
Mois 6  : Upgrade production si tests OK
```

#### Stratégie progressive (standard) ⚡

```
┌──────────────────────────────────────────────────┐
│       Stratégie : ADOPTER RAPIDEMENT             │
├──────────────────────────────────────────────────┤
│                                                  │
│  x.x.0 GA ──► x.x.1 ──► Upgrade                  │
│   (Mois 0)    (Mois 1)   (Mois 2)                │
│                                                  │
│  ✅ Avantages :                                  │
│  - Accès rapide aux nouvelles features           │
│  - Toujours à jour                               │
│                                                  │
│  ❌ Inconvénients :                              │
│  - Risque légèrement plus élevé                  │
│  - Nécessite tests approfondis                   │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Timeline recommandée** :
```
Mois 0 : Nouvelle LTS sort (ex: 11.8.0)
Mois 1 : Tester en dev/staging
Mois 2 : Upgrade production si tests OK
```

#### Stratégie patch only (maintenance) 🔄

```
┌──────────────────────────────────────────────────┐
│     Stratégie : PATCHES UNIQUEMENT               │
├──────────────────────────────────────────────────┤
│                                                  │
│  11.4.N ──► 11.4.N+1 ──► 11.4.N+2                │
│   (Actuel)   (Patch)      (Patch)                │
│                                                  │
│  ✅ Mise à jour patches de sécurité              │
│  ❌ PAS de migration vers 11.8                   │
│                                                  │
│  Quand ?                                         │
│  - Production très stable                        │
│  - Encore 1+ an avant EOL                        │
│  - Pas besoin nouvelles features                 │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 🗓️ Stratégie de mise à jour pour Rolling

#### Si vous utilisez Rolling 🚀

```
┌──────────────────────────────────────────────────┐
│       OBLIGATION : Migrer tous les 3 mois        │
├──────────────────────────────────────────────────┤
│                                                  │
│  12.0 ──► 12.1 ──► 12.2 ──► 12.3 LTS             │
│  (3m)     (3m)     (3m)     (Stabiliser)         │
│                                                  │
│  Chaque trimestre :                              │
│  1. Tester nouvelle version (2-4 semaines)       │
│  2. Valider (1 semaine)                          │
│  3. Upgrade production (1 jour)                  │
│                                                  │
│  Ou : Migrer vers LTS quand elle sort            │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Cycle trimestriel** :
```
T1 (Jan-Mar) : 12.1 → Préparer migration vers 12.2
T2 (Apr-Jun) : 12.2 → Préparer migration vers 12.3
T3 (Jul-Sep) : 12.3 LTS → RESTER sur 12.3 (devient LTS)
T4 (Oct-Déc) : Profiter de la stabilité LTS
```

---

## Calendrier de maintenance recommandé

### 📅 Planning annuel type (LTS)

```
┌────────────────────────────────────────────────────────┐
│         Calendrier de maintenance LTS                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Janvier   : 🔍 Audit sécurité + Patch si disponible   │
│  Février   : 📊 Monitoring et performance review       │
│  Mars      : 🔄 Patch trimestriel (si disponible)      │
│  Avril     : 🧪 Tests de charge                        │
│  Mai       : 🔧 Maintenance base de données            │
│  Juin      : 🔄 Patch trimestriel (si disponible)      │
│  Juillet   : 📚 Formation équipe / doc update          │
│  Août      : 🏖️  Période calme / monitoring seul       │
│  Septembre : 🔄 Patch trimestriel (si disponible)      │
│  Octobre   : 🔍 Audit EOL / Planification migrations   │
│  Novembre  : 🧪 Tests nouvelle LTS en staging          │
│  Décembre  : 🔄 Patch annuel final (si disponible)     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### ⏰ Alertes à configurer

**6 mois avant EOL** :
```
🔔 Alerte : "Version X.X EOL dans 6 mois"
Action :
- Identifier version cible
- Budgétiser la migration
- Planifier les tests
```

**3 mois avant EOL** :
```
🔔 Alerte : "Version X.X EOL dans 3 mois"
Action :
- Démarrer tests en staging
- Former équipe sur nouveautés
- Valider compatibilité applications
```

**1 mois avant EOL** :
```
🔔 Alerte : "Version X.X EOL dans 1 mois !"
Action :
- Finaliser migration
- Date de cutover confirmée
- Backup complet avant migration
```

**Jour EOL** :
```
🛑 Alerte : "Version X.X N'EST PLUS SUPPORTÉE"
Action :
- Si pas encore migré : URGENCE
- Planifier migration d'urgence
- Risque sécurité élevé
```

---

## Bonnes pratiques de maintenance

### ✅ Do's (À FAIRE)

1. **Appliquer les patches de sécurité rapidement**
   ```
   🔒 Patch sécurité disponible
   → Test en staging (24-48h)
   → Déploiement production (sous 1 semaine)
   ```

2. **Rester dans la même série LTS (patches)**
   ```
   ✅ 11.4.0 → 11.4.5 : SAFE
   ❌ 11.4.5 → 11.8.0 : Planifier migration complète
   ```

3. **Suivre les release notes**
   ```
   📧 S'inscrire : announce@mariadb.org
   📰 Lire : https://mariadb.com/kb/en/release-notes/
   ```

4. **Tester AVANT production**
   ```
   Dev → Staging → PreProd → Production
    ✅     ✅       ✅         ✅
   ```

5. **Avoir un plan de rollback**
   ```
   Backup complet AVANT upgrade
   + Procédure de rollback documentée
   + Testée au moins une fois
   ```

6. **Monitorer après mise à jour**
   ```
   J+0 : Monitoring continu 24h
   J+1 : Vérifications manuelles
   J+7 : Review complète
   ```

### ❌ Don'ts (À ÉVITER)

1. **Ne PAS ignorer les patches de sécurité**
   ```
   ❌ "On appliquera le patch plus tard"
   → Fenêtre de vulnérabilité = Risque !
   ```

2. **Ne PAS sauter de versions mineures**
   ```
   ❌ 11.4.0 → 11.4.5 directement (si patches importants entre)
   ✅ 11.4.0 → 11.4.1 → 11.4.2 → ... → 11.4.5

   OU

   ✅ 11.4.0 → 11.4.5 (si review de TOUS les changelogs)
   ```

3. **Ne PAS rester sur version EOL**
   ```
   ❌ Version EOL = Plus de patches sécurité
   → Vulnérabilités non corrigées
   → Risque légal (conformité)
   ```

4. **Ne PAS upgrader en production sans tests**
   ```
   ❌ Upgrade direct production
   → Risque de casse
   → Downtime non planifié
   ```

5. **Ne PAS mélanger versions**
   ```
   ❌ Serveur 11.4.3 + Client 11.8.0
   → Problèmes de compatibilité possibles
   ```

---

## Outils de suivi

### 🔔 Notifications et alertes

**1. MariaDB.org notifications**
```bash
# S'inscrire à la mailing list
https://lists.mariadb.org/
→ announce@mariadb.org (releases)
→ security@mariadb.org (sécurité)
```

**2. RSS Feeds**
```
Blog MariaDB : https://mariadb.com/blog/feed/
Release notes : https://mariadb.com/kb/en/release-notes/
```

**3. GitHub Watch**
```
https://github.com/MariaDB/server
→ Watch → Custom → Releases
```

**4. Site EOL tracking**
```
https://endoflife.date/mariadb
→ API disponible pour automatisation
```

### 📊 Monitoring version en production

**1. Vérifier version installée**
```sql
-- Connexion MariaDB
SELECT VERSION();
-- Résultat : 11.8.1-MariaDB

-- Détails complets
SHOW VARIABLES LIKE 'version%';
```

```bash
# En shell
mariadb --version
# mariadb Ver 15.1 Distrib 11.8.1-MariaDB
```

**2. Vérifier EOL distance**
```bash
# Script bash exemple
#!/bin/bash
VERSION=$(mariadb -e "SELECT VERSION()" -sN | cut -d'-' -f1)
echo "Version actuelle : $VERSION"

# Comparer avec dates EOL connues
# 11.4 → Mai 2027
# 11.8 → Juin 2028
```

**3. Monitoring automatisé**
```yaml
# Exemple Prometheus alert
- alert: MariaDBVersionOld
  expr: mariadb_version_info{version!~"11\\.8.*"}
  for: 7d
  labels:
    severity: warning
  annotations:
    summary: "MariaDB version obsolète"
```

---

## ✅ Points clés à retenir

- 🛡️ **Support 3 ans** depuis MariaDB 11.4 (Mai 2024)
- 🔒 **Inclus** : Sécurité, bugs critiques, patches, compatibilité
- ❌ **Exclu** : Nouvelles features, optimisations, nouvelles plateformes
- 📅 **3 phases LTS** : Stabilisation (6m) → Maturité (24m) → Fin de vie (6m)
- 🚀 **Rolling trimestriel** : Nouvelle version tous les ~3 mois
- ⏰ **Support Rolling** : Seulement jusqu'à la version suivante
- 🔄 **Stratégie conservatrice** : Attendre x.x.3+ avant upgrade (6 mois)
- ⚡ **Stratégie progressive** : Upgrade dès x.x.1 (2 mois)
- 📊 **Cycle annuel** : Patches trimestriels + audit semestriel
- 🔔 **Alertes EOL** : 6 mois, 3 mois, 1 mois avant
- ✅ **Patches sécurité** : À appliquer sous 1 semaine (critical)
- 🧪 **Toujours tester** en staging avant production
- 🛑 **Ne JAMAIS rester** sur version EOL
- 📈 **Calendrier 2025** : 11.4 (EOL 2027), 11.8 (EOL 2028)

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB Maintenance Policy](https://mariadb.org/about/#maintenance-policy)
- [Release Notes](https://mariadb.com/kb/en/release-notes/)
- [Security Advisories](https://mariadb.com/kb/en/security/)

### 📅 Calendriers et planning
- [Release Calendar](https://mariadb.com/kb/en/release-calendar/)
- [End of Life Dates](https://endoflife.date/mariadb)
- [Supported Versions](https://mariadb.com/kb/en/mariadb-server-versions/)

### 🔔 Notifications
- [Mailing Lists](https://lists.mariadb.org/)
- [Security Mailing List](https://mariadb.org/about/security/)
- [Blog MariaDB](https://mariadb.com/blog/)

### 🔧 Outils de migration
- [Upgrading MariaDB](https://mariadb.com/kb/en/upgrading/)
- [mariadb-upgrade tool](https://mariadb.com/kb/en/mysql_upgrade/)

---

## ➡️ Section suivante

**[1.7 - Roadmap : série 12.x (12.0→12.2 rolling, 12.3 LTS prévu Q2 2026)](./07-roadmap-serie-12.md)** 🆕

Maintenant que vous comprenez le cycle de support, découvrons dans la section suivante la **roadmap de MariaDB** : quelles sont les versions à venir ? Quelles nouvelles fonctionnalités sont prévues ? Comment se préparer pour la série 12.x et au-delà ? Nous explorerons l'avenir de MariaDB et comment anticiper les évolutions futures.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.6*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Roadmap : série 12.x (12.0→12.2 rolling, 12.3 LTS prévu Q2 2026)](/01-introduction-fondamentaux/07-roadmap-serie-12.md)
