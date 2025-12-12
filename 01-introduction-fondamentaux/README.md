🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1. Introduction et Fondamentaux

> **Niveau** : Débutant
> **Durée estimée** : 6-8 heures
> **Prérequis** : Aucun (connaissance basique des bases de données recommandée)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Comprendre ce qu'est MariaDB et son positionnement dans l'écosystème des bases de données
- Identifier les différences clés entre MariaDB et MySQL
- Connaître l'architecture générale d'un système de gestion de base de données relationnel
- Maîtriser la politique de versions LTS et Rolling releases
- Comprendre le cycle de support de 3 ans pour les versions LTS
- Installer et configurer MariaDB sur votre système
- Utiliser les outils d'administration essentiels

---

## 📖 Bienvenue dans l'univers MariaDB

Bienvenue dans cette formation complète sur **MariaDB 11.8 LTS** ! Ce chapitre d'introduction vous permettra de découvrir MariaDB en douceur, que vous soyez développeur, administrateur système, ou simplement curieux d'en apprendre plus sur les bases de données relationnelles modernes.

### Pourquoi MariaDB ?

Dans le monde actuel des applications web, mobiles et cloud, les données sont au cœur de tout. Chaque site e-commerce, réseau social, application de gestion ou système d'information s'appuie sur une base de données pour stocker, organiser et récupérer des informations de manière efficace et sécurisée.

**MariaDB** est l'un des systèmes de gestion de bases de données relationnelles (SGBDR) les plus populaires et performants au monde. Créé par les développeurs originaux de MySQL, MariaDB est :

- 🆓 **Open Source** : Code source libre et gratuit
- ⚡ **Performant** : Optimisé pour la vitesse et l'efficacité
- 🔄 **Compatible** : Drop-in replacement de MySQL
- 🛡️ **Sécurisé** : Chiffrement, authentification avancée, audit
- 🌍 **Largement adopté** : Utilisé par Wikipedia, Google, Booking.com, et des milliers d'entreprises

### Ce que vous allez apprendre

Ce chapitre introductif couvre les fondations essentielles pour démarrer avec MariaDB. Nous explorerons ensemble :

#### 🔍 **Les concepts fondamentaux**
- Qu'est-ce qu'un SGBDR et pourquoi MariaDB ?
- L'histoire de MariaDB et ses liens avec MySQL
- Les différences techniques entre MariaDB et MySQL

#### 🎯 **L'écosystème MariaDB**
- Les cas d'usage : de la startup à l'entreprise
- L'architecture générale d'un SGBDR
- Les composants clés : moteurs de stockage, outils, extensions

#### 📅 **Politique de versions et support**
- **LTS (Long Term Support)** : Stabilité sur 3 ans
- **Rolling releases** : Mises à jour trimestrielles
- **Roadmap 2025-2026** : Vers la série 12.x
- Comment choisir la bonne version pour votre projet

#### 🛠️ **Installation et premiers pas**
- Installation sur Linux, Windows et macOS
- Configuration initiale et sécurisation
- Découverte des outils d'administration

---

## 🗺️ Architecture de ce chapitre

Ce chapitre est organisé en **9 sections progressives** qui vous guideront pas à pas :

### **Section 1.1 - Qu'est-ce que MariaDB ?**
Découverte du système, ses caractéristiques principales et son positionnement.

### **Section 1.2 - Histoire et différences avec MySQL**
Comprendre les origines de MariaDB et pourquoi il se distingue de MySQL.

### **Section 1.3 - Cas d'usage et écosystème**
Qui utilise MariaDB et pour quels types d'applications ?

### **Section 1.4 - Architecture générale d'un SGBD relationnel**
Comment fonctionne MariaDB en interne ? Vue d'ensemble technique.

### **Section 1.5 - Politique de versions : LTS vs Rolling releases** 🔄
Comprendre les deux stratégies de publication et leurs implications.

### **Section 1.6 - Cycle de support : 3 ans LTS** 🆕
Le nouveau modèle de support à partir de la version 11.4.

### **Section 1.7 - Roadmap : série 12.x** 🆕
Quel avenir pour MariaDB ? Planification des prochaines versions.

### **Section 1.8 - Installation et configuration initiale**
Installer MariaDB sur votre machine et le configurer correctement.

### **Section 1.9 - Outils d'administration**
Découvrir les outils CLI et GUI pour gérer vos bases de données.

---

## 🎓 Public cible de ce chapitre

Ce chapitre s'adresse à **tous les débutants**, quel que soit votre profil :

### 👨‍💻 **Développeurs**
Vous allez créer des applications qui utilisent des bases de données ? Ce chapitre vous donnera les fondations pour comprendre comment MariaDB fonctionne et comment l'intégrer à vos projets.

### 🔧 **Administrateurs système**
Vous devez installer et maintenir des serveurs MariaDB ? Vous apprendrez les bases de l'architecture et de la configuration.

### 🎯 **Chefs de projet / Product Owners**
Vous devez choisir une technologie de base de données ? Ce chapitre vous aidera à comprendre les avantages de MariaDB et à prendre des décisions éclairées.

### 🔰 **Étudiants et curieux**
Vous voulez apprendre les bases de données relationnelles ? MariaDB est un excellent point de départ, et ce chapitre ne nécessite aucune connaissance préalable.

---

## 📊 Politique de versions : LTS vs Rolling (Aperçu)

L'un des points clés à comprendre avant de commencer avec MariaDB est son **modèle de versions**. Contrairement à certaines bases de données qui publient des versions majeures espacées de plusieurs années, MariaDB propose deux stratégies complémentaires :

### 🛡️ **LTS (Long Term Support)**
- **Support de 3 ans** depuis la version 11.4 (mai 2024)
- **Versions LTS récentes** : 10.6, 10.11, 11.4, **11.8** (juin 2025)
- **Usage recommandé** : Production, applications critiques, entreprises
- **Mises à jour** : Correctifs de sécurité et bugs uniquement

### 🚀 **Rolling Releases (Trimestrielles)**
- **Nouvelles versions tous les 3 mois** (ex: 12.0, 12.1, 12.2...)
- **Fonctionnalités récentes** et améliorations continues
- **Usage recommandé** : Développement, early adopters, cas d'usage nécessitant les dernières features
- **Support** : Jusqu'à la prochaine version LTS

### 🔄 **Exemple de chronologie (2024-2026)**

```
2024
├── Mai : 11.4 LTS (support jusqu'en 2027)
├── Septembre : 12.0 Rolling
└── Décembre : 12.1 Rolling

2025
├── Mars : 12.2 Rolling
├── Juin : 11.8 LTS (support jusqu'en 2028) ← Nous sommes ici !
└── Septembre : 12.3 Rolling

2026
└── Q2 : 12.3 LTS (anticipé)
```

💡 **Notre recommandation** : Pour cette formation, nous utilisons **MariaDB 11.8 LTS** (juin 2025), qui combine stabilité LTS et nouvelles fonctionnalités majeures comme **MariaDB Vector** pour l'IA.

> 📖 Les sections 1.5, 1.6 et 1.7 détailleront cette politique de versions en profondeur.

---

## 🆕 Nouveautés MariaDB 11.8 LTS (Aperçu)

Bien que ce soit une version débutant, il est important de connaître les innovations majeures de MariaDB 11.8, que vous découvrirez tout au long de la formation :

### 🤖 **MariaDB Vector** - LA fonctionnalité phare
- Type de données `VECTOR` pour l'intelligence artificielle
- Index **HNSW** pour la recherche vectorielle ultra-rapide
- Intégration avec les LLMs (OpenAI, Claude, LLaMA)
- Cas d'usage : RAG (Retrieval-Augmented Generation), recherche sémantique

### 🔒 **Sécurité renforcée**
- **TLS activé par défaut** pour toutes les connexions
- Plugin d'authentification **PARSEC**
- Privilèges granulaires améliorés

### ⚡ **Performance**
- `innodb_alter_copy_bulk` : Construction d'index 2x plus rapide
- Optimiseur de coûts amélioré pour les SSD
- Optimisations SIMD (AVX2, AVX512, ARM)

### 🌍 **Unicode et internationalisation**
- **utf8mb4 par défaut** avec collations UCA 14.0.0
- Meilleur support des langues non latines

### 📅 **Extension TIMESTAMP**
- Support jusqu'en **2106** (problème Y2038 résolu)

### 🔧 **Outils et intégration**
- **MaxScale 25.01** : Workload Capture/Replay, Diff Router
- **Application Time Period Tables**
- Réplication optimisée avec Optimistic ALTER TABLE

> 🎯 Ne vous inquiétez pas si ces termes sont nouveaux ! Vous les découvrirez progressivement dans les chapitres appropriés.

---

## 🛣️ Votre parcours d'apprentissage

Ce chapitre est la **fondation** de votre apprentissage MariaDB. Voici comment il s'intègre dans la formation complète :

```
[1] Introduction et Fondamentaux (VOUS ÊTES ICI)
    ↓
[2] Bases du SQL → Créer vos premières requêtes
    ↓
[3-4] SQL Intermédiaire et Avancé → Maîtriser le langage
    ↓
[5-6] Index, Transactions, Performance → Optimiser
    ↓
[7-9] Moteurs, Programmation, Vues → Fonctionnalités avancées
    ↓
[10-12] Sécurité, Administration, Sauvegardes → Gérer en production
    ↓
[13-14] Réplication, Haute Disponibilité → Architectures distribuées
    ↓
[15] Performance Tuning → Devenir expert
    ↓
[16] DevOps et Cloud → Automatisation moderne
    ↓
[17-18] Intégration et Fonctionnalités Avancées → Aller plus loin
    ↓
[19-20] Migration et Architectures → Cas d'usage réels
```

---

## 💡 Conseils pour tirer le meilleur parti de ce chapitre

### ✅ **À FAIRE**
- 📖 Lisez les sections dans l'ordre (progression logique)
- 💻 Installez MariaDB sur votre machine (section 1.8)
- 🧪 Testez les commandes et outils présentés
- 📝 Prenez des notes sur les concepts clés
- ❓ N'hésitez pas à consulter la documentation officielle

### ⚠️ **À ÉVITER**
- ❌ Ne sautez pas l'installation pratique (section 1.8)
- ❌ N'essayez pas de tout mémoriser dès la première lecture
- ❌ Ne vous découragez pas si certains concepts semblent complexes
- ❌ N'oubliez pas de consulter les annexes en cas de doute

### 🎯 **Objectif final du chapitre**

À la fin de ce chapitre, vous devriez :
1. ✅ Comprendre ce qu'est MariaDB et pourquoi l'utiliser
2. ✅ Avoir MariaDB installé et fonctionnel sur votre machine
3. ✅ Savoir choisir entre LTS et Rolling releases
4. ✅ Connaître l'architecture générale d'un SGBDR
5. ✅ Être capable d'utiliser les outils d'administration de base

---

## 🔗 Ressources essentielles

Avant de commencer, marquez ces ressources officielles :

### 📚 **Documentation officielle**
- [Documentation MariaDB 11.8](https://mariadb.com/kb/en/documentation/)
- [MariaDB.org](https://mariadb.org/)
- [MariaDB Foundation](https://mariadb.org/about/)

### 💬 **Communauté**
- [Forum MariaDB](https://mariadb.org/forums/)
- [Zulip Chat](https://mariadb.zulipchat.com/)
- [Stack Overflow (tag: mariadb)](https://stackoverflow.com/questions/tagged/mariadb)

### 🐛 **Support et bugs**
- [JIRA MariaDB](https://jira.mariadb.org/)
- [GitHub MariaDB Server](https://github.com/MariaDB/server)

### 📰 **Actualités**
- [Blog officiel MariaDB](https://mariadb.com/blog/)
- [Planet MariaDB](https://planet.mariadb.org/)

---

## 📋 Structure détaillée du chapitre

Voici un aperçu complet des 9 sections à venir :

| Section | Titre | Durée | Niveau |
|---------|-------|-------|--------|
| 1.1 | Qu'est-ce que MariaDB ? | 30 min | Débutant |
| 1.2 | Histoire et différences avec MySQL | 45 min | Débutant |
| 1.3 | Cas d'usage et écosystème | 45 min | Débutant |
| 1.4 | Architecture générale d'un SGBD | 1h | Débutant |
| 1.5 | Politique de versions : LTS vs Rolling | 45 min | Débutant |
| 1.6 | Cycle de support : 3 ans LTS | 30 min | Débutant |
| 1.7 | Roadmap : série 12.x | 30 min | Débutant |
| 1.8 | Installation et configuration initiale | 2h | Débutant |
| 1.9 | Outils d'administration | 1h | Débutant |

**Durée totale estimée** : 6h30 - 8h (lecture + pratique)

---

## ✅ Points clés à retenir de cette introduction

Avant de plonger dans les sections détaillées, retenez ces points essentiels :

- 🗄️ **MariaDB est un SGBDR open source**, performant et largement adopté
- 🔄 **Compatible avec MySQL**, c'est un "drop-in replacement"
- 📅 **Deux stratégies de versions** : LTS (3 ans de support) et Rolling (trimestriel)
- 🆕 **Version 11.8 LTS** (juin 2025) apporte des innovations majeures comme MariaDB Vector
- 🎯 **Cette formation couvre** du niveau débutant à expert en 20 chapitres
- 🛠️ **L'installation pratique** (section 1.8) est essentielle pour suivre la formation
- 📖 **La documentation officielle** est votre alliée tout au long de l'apprentissage

---

## ➡️ Section suivante

**[1.1 - Qu'est-ce que MariaDB ?](./01-quest-ce-que-mariadb.md)**

Dans la prochaine section, nous découvrirons en détail ce qu'est MariaDB, ses caractéristiques principales, et pourquoi il est devenu l'un des SGBDR les plus populaires au monde.

---

## 🆘 Besoin d'aide ?

Si vous avez des questions ou rencontrez des difficultés :

1. 📖 Consultez d'abord la documentation officielle
2. 🔍 Recherchez dans les forums et Stack Overflow
3. 💬 Rejoignez la communauté MariaDB (Zulip, forums)
4. 📧 Contactez le support si vous utilisez MariaDB Enterprise

---

**Bonne formation et bienvenue dans l'écosystème MariaDB ! 🚀**

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Chapitre 1*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Qu'est-ce que MariaDB ?](/01-introduction-fondamentaux/01-quest-ce-que-mariadb.md)
