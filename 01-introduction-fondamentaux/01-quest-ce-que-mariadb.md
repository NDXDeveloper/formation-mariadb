🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.1 Qu'est-ce que MariaDB ?

> **Niveau** : Débutant
> **Durée estimée** : 30 minutes
> **Prérequis** : Aucun

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Définir ce qu'est MariaDB et son rôle en tant que SGBDR
- Comprendre les concepts fondamentaux des bases de données relationnelles
- Identifier les caractéristiques principales de MariaDB
- Connaître les avantages de MariaDB par rapport aux alternatives
- Comprendre le modèle open source et la gouvernance de MariaDB

---

## Introduction

Imaginez que vous devez gérer les informations de milliers de clients, leurs commandes, leurs paiements, et tout cela en temps réel, de manière sécurisée et fiable. Comment organiser ces données ? Comment les retrouver rapidement ? Comment s'assurer qu'elles ne soient pas perdues ou corrompues ?

C'est exactement le problème que résout **MariaDB** : un système de gestion de base de données qui vous permet de **stocker, organiser, rechercher et gérer** des données de manière structurée et performante.

---

## Qu'est-ce qu'une base de données ?

Avant de parler de MariaDB, commençons par les bases.

### 📚 Définition simple

Une **base de données** est un ensemble organisé de données stockées électroniquement. Pensez-y comme à une bibliothèque numérique où :
- Les **données** sont les livres
- La **structure** est le système de classement (par genre, auteur, date...)
- Le **système de gestion** est le bibliothécaire qui vous aide à trouver ce que vous cherchez

### 🗂️ Base de données vs fichiers simples

Pourquoi utiliser une base de données plutôt que de simples fichiers Excel ou CSV ?

| Fichiers simples (Excel, CSV) | Base de données (MariaDB) |
|-------------------------------|---------------------------|
| 📄 Limité en taille (~1M lignes) | 🗄️ Capacité quasi illimitée |
| 🐌 Lent pour les recherches | ⚡ Recherches ultra-rapides (index) |
| ❌ Un seul utilisateur à la fois | 👥 Milliers d'utilisateurs simultanés |
| 🔓 Faible sécurité | 🔒 Sécurité avancée (utilisateurs, permissions) |
| ⚠️ Risque de corruption | ✅ Transactions ACID (intégrité garantie) |
| 🔄 Pas de relations complexes | 🔗 Relations entre tables (clés étrangères) |

### 💼 Exemple concret

**Scénario** : Vous gérez une boutique en ligne

**Avec des fichiers Excel** :
- `clients.xlsx` : Liste des clients
- `produits.xlsx` : Catalogue des produits
- `commandes.xlsx` : Historique des commandes

**Problèmes** :
- ❌ Comment lier une commande à un client ?
- ❌ Que se passe-t-il si deux employés modifient `clients.xlsx` en même temps ?
- ❌ Comment rechercher toutes les commandes d'un client en moins d'une seconde ?

**Avec MariaDB** :
```sql
-- Créer une relation entre commandes et clients
SELECT c.nom, c.email, o.date_commande, o.montant_total
FROM clients c
JOIN commandes o ON c.id = o.client_id
WHERE c.id = 12345;
```
✅ Résultat instantané, même avec 10 millions de commandes !

---

## MariaDB : Un système de gestion de base de données relationnelles (SGBDR)

### 🏗️ Définition

**MariaDB** est un **SGBDR** (Système de Gestion de Base de Données Relationnelles), c'est-à-dire un logiciel qui :

1. **Stocke** les données de manière structurée (tables, lignes, colonnes)
2. **Gère** les accès multiples et concurrents
3. **Garantit** l'intégrité et la cohérence des données
4. **Permet** d'interroger et manipuler les données via le langage SQL

### 🔤 L'origine du nom "MariaDB"

Le nom **Maria** vient de **Maria Widenius**, la plus jeune fille de **Michael "Monty" Widenius**, le créateur principal de MariaDB (et co-créateur de MySQL). C'est une tradition familiale : sa première fille s'appelle **My**, d'où le nom **MySQL** !

### 📊 Qu'est-ce que le modèle relationnel ?

Le modèle **relationnel** organise les données en **tables** (aussi appelées "relations") composées de :

- **Lignes** (enregistrements, tuples) : Chaque ligne représente une entité
- **Colonnes** (attributs, champs) : Chaque colonne représente une propriété

**Exemple** : Table `clients`

| id | nom | email | ville | date_inscription |
|----|-----|-------|-------|------------------|
| 1 | Alice Dubois | alice@email.com | Paris | 2024-01-15 |
| 2 | Bob Martin | bob@email.com | Lyon | 2024-02-20 |
| 3 | Claire Petit | claire@email.com | Paris | 2024-03-10 |

**Avantages du modèle relationnel** :
- ✅ **Structure claire** : Données organisées logiquement
- ✅ **Relations** : Les tables peuvent être liées entre elles
- ✅ **Intégrité** : Règles pour éviter les données incohérentes
- ✅ **Langage standard** : SQL (Structured Query Language)

---

## Les caractéristiques principales de MariaDB

### 🆓 1. Open Source et gratuit

**MariaDB est un logiciel libre** sous licence **GPLv2**, ce qui signifie :

- ✅ **Gratuit** : Aucun coût de licence, même en production
- ✅ **Code source ouvert** : Vous pouvez consulter et modifier le code
- ✅ **Communauté active** : Des milliers de contributeurs dans le monde
- ✅ **Pas de vendor lock-in** : Vous n'êtes pas dépendant d'un seul fournisseur

💡 **Note** : MariaDB propose aussi des versions "Enterprise" avec support commercial et fonctionnalités additionnelles, mais la version communautaire est pleinement fonctionnelle et largement utilisée en production.

### ⚡ 2. Performant et optimisé

MariaDB est conçu pour la **performance** :

- 🚀 **Optimisations avancées** : Query cache, index intelligents, optimiseur de requêtes
- 💾 **Gestion mémoire efficace** : Buffer pools, caches adaptatifs
- 📈 **Scalabilité** : Gestion de millions de requêtes par seconde
- 🔄 **Multi-threading** : Traitement parallèle des requêtes

**Benchmark exemple** (matériel standard) :
- 📊 **Lectures** : Jusqu'à 1 million de requêtes/seconde
- ✍️ **Écritures** : Jusqu'à 200 000 transactions/seconde
- 💾 **Bases de données** : Taille illimitée (plusieurs téraoctets)

### 🔒 3. Sécurisé

La sécurité est une priorité dans MariaDB :

- 🔐 **Authentification** : Multiples méthodes (mots de passe, certificats, PAM, LDAP)
- 🔑 **Contrôle d'accès** : Permissions granulaires par utilisateur, base, table, colonne
- 🔒 **Chiffrement** : TLS/SSL pour les connexions, chiffrement des données au repos
- 📝 **Audit** : Traçabilité complète des actions (qui a fait quoi et quand)
- 🆕 **TLS par défaut** : Depuis MariaDB 11.8, TLS est activé automatiquement

### 🔄 4. Compatible avec MySQL

MariaDB est un **"drop-in replacement"** de MySQL, ce qui signifie :

- ✅ **Applications compatibles** : Les apps MySQL fonctionnent avec MariaDB sans modification
- ✅ **Migration facile** : Changez de MySQL à MariaDB en quelques minutes
- ✅ **Même syntaxe SQL** : Les requêtes SQL sont identiques
- ✅ **Protocole client identique** : Mêmes connecteurs et drivers

💡 **Pourquoi cette compatibilité ?** MariaDB a été créé par les développeurs originaux de MySQL et maintient activement cette compatibilité.

### 🎯 5. Multi-moteurs de stockage

MariaDB supporte plusieurs **moteurs de stockage** (storage engines), chacun optimisé pour des cas d'usage spécifiques :

| Moteur | Usage principal | Caractéristiques |
|--------|-----------------|------------------|
| **InnoDB** | Défaut, OLTP | Transactions ACID, clés étrangères |
| **Aria** | Tables système | Crash-safe, successeur de MyISAM |
| **MyISAM** | Legacy | Rapide en lecture, pas de transactions |
| **ColumnStore** | OLAP, analytics | Données en colonnes, compression |
| **Memory** | Cache, temporaire | Données en RAM, ultra-rapide |
| **S3** 🆕 | Archivage | Stockage sur AWS S3 / MinIO |
| **Vector** 🆕 | IA, recherche | Recherche vectorielle pour LLMs |

💡 **Flexibilité** : Vous pouvez utiliser différents moteurs dans la même base de données selon vos besoins !

### 🌍 6. Multi-plateforme

MariaDB fonctionne sur tous les systèmes d'exploitation majeurs :

- 🐧 **Linux** : Toutes les distributions (Ubuntu, Debian, CentOS, RHEL, etc.)
- 🪟 **Windows** : Windows Server et Desktop
- 🍎 **macOS** : Support natif via Homebrew
- ☁️ **Cloud** : AWS, Azure, GCP, DigitalOcean, etc.
- 🐳 **Containers** : Docker, Kubernetes, OpenShift

### 🔧 7. Riche en fonctionnalités

MariaDB offre des fonctionnalités avancées :

- 📝 **SQL complet** : Standard SQL:2016 avec extensions
- 🔗 **Réplication** : Master-Slave, Multi-Master (Galera Cluster)
- 🎯 **Haute disponibilité** : Failover automatique, load balancing
- 📊 **Partitionnement** : Distribution horizontale des données
- 🗄️ **JSON natif** : Support complet du format JSON
- 🤖 **MariaDB Vector** 🆕 : Recherche vectorielle pour IA/RAG
- ⏰ **Tables temporelles** : Historique automatique des modifications

---

## MariaDB dans l'écosystème des bases de données

### 📊 Positionnement

MariaDB se positionne comme une alternative open source aux bases de données relationnelles commerciales :

**Catégorie** : SGBDR (Relationnel SQL)

**Alternatives principales** :

| Base de données | Type | Open Source | Avantages | Inconvénients |
|----------------|------|-------------|-----------|---------------|
| **MariaDB** | SGBDR | ✅ Oui | Gratuit, performant, compatible MySQL | - |
| **MySQL** | SGBDR | ⚠️ Partiel | Très populaire | Propriété d'Oracle |
| **PostgreSQL** | SGBDR | ✅ Oui | Très avancé | Moins compatible MySQL |
| **Oracle DB** | SGBDR | ❌ Non | Enterprise features | Très cher |
| **SQL Server** | SGBDR | ⚠️ Partiel | Intégration Microsoft | Licence payante |
| **MongoDB** | NoSQL | ⚠️ Partiel | Flexible (JSON) | Pas de relations |
| **Redis** | NoSQL | ✅ Oui | Ultra-rapide (RAM) | Pas de requêtes SQL |

💡 **Choix de MariaDB** : Équilibre optimal entre **performance**, **coût** (gratuit), **fonctionnalités** et **support communautaire**.

### 🌟 Parts de marché

MariaDB est **largement adopté** dans le monde :

- 📈 **Croissance** : +40% d'adoption depuis 2020
- 🌍 **Utilisateurs** : Plus de 10 millions de bases de données déployées
- 🏢 **Entreprises** : Fortune 500, startups, administrations publiques

**Qui utilise MariaDB ?**
- 🌐 **Wikipedia** : Toutes les données des projets Wikimedia
- 🔍 **Google** : Infrastructure interne (migration depuis MySQL)
- 🏨 **Booking.com** : Système de réservation mondial
- 💾 **RedHat / CentOS** : Base de données par défaut
- 🛒 **Shopify** : E-commerce à grande échelle

---

## Architecture et composants de MariaDB

### 🏗️ Architecture simplifiée

```
┌─────────────────────────────────────────────┐
│           Applications Clientes             │
│  (Web, Mobile, Desktop, API, Scripts...)    │
└──────────────────┬──────────────────────────┘
                   │ Requêtes SQL
                   ↓
┌─────────────────────────────────────────────┐
│          MariaDB Server (mysqld)            │
│ ┌─────────────────────────────────────────┐ │
│ │    Parser SQL & Optimiseur              │ │
│ │  (Analyse et optimisation des requêtes) │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                           │
│ ┌─────────────────────────────────────────┐ │
│ │    Query Cache & Buffer Pool            │ │
│ │       (Mise en cache mémoire)           │ │
│ └───────────────┬─────────────────────────┘ │
│                 ↓                           │
│ ┌─────────────────────────────────────────┐ │
│ │      Storage Engines (Moteurs)          │ │
│ │  InnoDB | Aria | ColumnStore | Vector   │ │
│ └───────────────┬─────────────────────────┘ │
└─────────────────┼───────────────────────────┘
                  ↓
        ┌──────────────────┐
        │  Fichiers Disque │
        │  (.ibd, .frm)    │
        └──────────────────┘
```

### 🧩 Composants principaux

#### 1️⃣ **MariaDB Server (mysqld)**
Le cœur du système, processus principal qui :
- Écoute les connexions clients (port 3306 par défaut)
- Authentifie les utilisateurs
- Parse et exécute les requêtes SQL
- Gère les transactions

#### 2️⃣ **Client MariaDB (mariadb / mysql)**
Outil en ligne de commande pour se connecter au serveur :
```bash
mariadb -u root -p
```

#### 3️⃣ **Storage Engines**
Modules interchangeables qui gèrent le stockage physique des données (InnoDB, Aria, etc.)

#### 4️⃣ **Outils d'administration**
- `mariadb-admin` : Administration serveur
- `mariadb-dump` : Sauvegarde bases de données
- `mariadb-upgrade` : Mise à jour après upgrade

---

## MariaDB Foundation et gouvernance

### 🏛️ La MariaDB Foundation

**MariaDB Foundation** est une organisation à but non lucratif qui :

- 🎯 **Développe** le projet MariaDB Server
- 🌍 **Garantit** que MariaDB reste libre et open source
- 👥 **Coordonne** la communauté mondiale de contributeurs
- 📚 **Maintient** la documentation et les ressources

**Site officiel** : [mariadb.org](https://mariadb.org/)

### 🏢 MariaDB Corporation vs Foundation

Il existe deux entités distinctes :

| MariaDB Foundation | MariaDB Corporation |
|-------------------|---------------------|
| 🆓 Organisation à but non lucratif | 🏢 Société commerciale |
| 🔓 Développe MariaDB Server (GPL) | 💼 Propose MariaDB Enterprise + Support |
| 🌍 Communauté et open source | 💰 Services payants (SkySQL, MaxScale Ent.) |
| 👥 Gouvernance collective | 🤝 Partenaire mais indépendant |

💡 **Important** : La Foundation garantit que MariaDB Server restera **toujours gratuit et open source**, même si la Corporation devait disparaître.

### 🤝 Modèle de développement

MariaDB utilise un modèle **ouvert et collaboratif** :

- ✅ **Public** : Développement sur GitHub, issues publiques
- ✅ **Contributeurs** : Toute personne peut contribuer (patches, bugs, features)
- ✅ **Transparence** : Roadmap publique, décisions communautaires
- ✅ **Releases régulières** : LTS tous les ~18 mois, Rolling tous les 3 mois

---

## Pourquoi choisir MariaDB ?

### ✅ Avantages principaux

#### 1. **Coût : 0€**
- Aucun frais de licence
- Idéal pour startups et projets à budget limité
- Support communautaire gratuit

#### 2. **Performance éprouvée**
- Utilisé par Wikipedia (milliards de requêtes/jour)
- Benchmarks supérieurs à MySQL dans de nombreux cas
- Optimisations continues

#### 3. **Liberté et indépendance**
- Pas de dépendance à Oracle ou Microsoft
- Possibilité de modifier le code source
- Communauté mondiale active

#### 4. **Innovation constante** 🆕
- Nouvelles fonctionnalités régulières (MariaDB Vector, JSON avancé, etc.)
- Adoption rapide des standards SQL
- Intégration des retours utilisateurs

#### 5. **Compatibilité maximale**
- Migration depuis MySQL sans effort
- Large écosystème d'outils compatibles
- Support de tous les frameworks modernes

#### 6. **Sécurité renforcée**
- Correctifs de sécurité rapides
- Fonctionnalités avancées (audit, encryption, PAM)
- Communauté vigilante

### ⚠️ Considérations

Bien que MariaDB soit excellent, quelques points à considérer :

- 📚 **Écosystème MySQL** : Certains outils spécifiques à MySQL peuvent nécessiter des adaptations
- 🔧 **Fonctionnalités spécifiques** : Certaines features MariaDB ne sont pas rétro-compatibles MySQL
- 📖 **Documentation** : Moins de tutoriels que MySQL (mais en croissance rapide)

💡 **Verdict** : Pour la grande majorité des cas d'usage, MariaDB est un **excellent choix**, offrant plus de fonctionnalités que MySQL tout en restant gratuit et performant.

---

## Cas d'usage typiques

MariaDB s'adapte à de nombreux scénarios :

### 🌐 1. Applications Web
- Sites e-commerce (Magento, PrestaShop)
- CMS (WordPress, Drupal, Joomla)
- Réseaux sociaux et plateformes collaboratives

### 📱 2. Applications mobiles
- Backend API pour apps iOS/Android
- Synchronisation de données
- Systèmes de notifications

### 📊 3. Business Intelligence et Analytics
- Data warehousing avec ColumnStore
- Tableaux de bord et reporting
- Analyse de données volumineuses

### 🏭 4. Systèmes d'information d'entreprise
- ERP (Enterprise Resource Planning)
- CRM (Customer Relationship Management)
- Gestion de stocks et supply chain

### 🎮 5. Gaming
- Gestion des profils joueurs
- Classements et leaderboards
- Économie in-game

### 🤖 6. Intelligence Artificielle 🆕
- Recherche vectorielle avec MariaDB Vector
- RAG (Retrieval-Augmented Generation)
- Bases de connaissances pour LLMs

---

## Comparaison rapide : MariaDB vs MySQL

Bien que compatibles, il existe des différences importantes :

| Aspect | MariaDB | MySQL |
|--------|---------|-------|
| **Licence** | GPL v2 (totalement libre) | GPL v2 + Commercial |
| **Propriétaire** | MariaDB Foundation (non-profit) | Oracle Corporation |
| **Moteurs de stockage** | InnoDB, Aria, ColumnStore, S3, Vector | InnoDB, MyISAM |
| **Réplication** | Galera Cluster intégré | Requires NDB Cluster |
| **Performance** | Optimisations supplémentaires | Standard |
| **Fonctionnalités** | JSON avancé, Vector, Sequences | Standard SQL |
| **Support Oracle** | ❌ Non | ✅ Oui (payant) |
| **Support communautaire** | ✅ Excellent | ✅ Très bon |
| **Innovation** | 🚀 Rapide | 🐢 Plus lente |

💡 **En résumé** : MariaDB offre **plus de fonctionnalités** et **plus de liberté** que MySQL, tout en restant **compatible**.

---

## ✅ Points clés à retenir

- 🗄️ **MariaDB est un SGBDR** : Système de gestion de base de données relationnelles performant et gratuit
- 🆓 **Open Source** : Licence GPLv2, code source ouvert, communauté active
- ⚡ **Performant** : Optimisé pour la vitesse, capable de gérer des millions de requêtes
- 🔒 **Sécurisé** : Authentification avancée, chiffrement, audit, TLS par défaut (11.8)
- 🔄 **Compatible MySQL** : Drop-in replacement, migration facile
- 🎯 **Multi-moteurs** : InnoDB, Aria, ColumnStore, Vector (nouveauté 11.8)
- 🌍 **Multi-plateforme** : Linux, Windows, macOS, Cloud, Containers
- 🏛️ **Gouvernance saine** : MariaDB Foundation garantit la pérennité open source
- 🚀 **Innovation continue** : Nouvelles fonctionnalités comme MariaDB Vector pour l'IA
- 💼 **Large adoption** : Wikipedia, Google, Booking.com, et des millions d'utilisateurs

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB Knowledge Base](https://mariadb.com/kb/en/documentation/)
- [À propos de MariaDB](https://mariadb.com/kb/en/about-mariadb/)
- [MariaDB Foundation](https://mariadb.org/about/)

### 🎥 Vidéos et tutoriels
- [MariaDB YouTube Channel](https://www.youtube.com/user/mariadbserver)
- [Introduction to MariaDB (Official)](https://mariadb.com/resources/webinars/)

### 📰 Articles recommandés
- [Why MariaDB over MySQL](https://mariadb.org/mariadb-vs-mysql/)
- [MariaDB Success Stories](https://mariadb.com/customers/)

---

## ➡️ Section suivante

**[1.2 - Histoire et différences avec MySQL](./02-histoire-et-differences-mysql.md)**

Maintenant que vous savez ce qu'est MariaDB, découvrons dans la prochaine section son histoire fascinante et les raisons de sa création. Nous explorerons également en détail les différences techniques entre MariaDB et MySQL pour vous aider à mieux comprendre les forces de MariaDB.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.1*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Histoire et différences avec MySQL](/01-introduction-fondamentaux/02-histoire-et-differences-mysql.md)
