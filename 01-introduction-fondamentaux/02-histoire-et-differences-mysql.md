🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.2 Histoire et différences avec MySQL

> **Niveau** : Débutant
> **Durée estimée** : 45 minutes
> **Prérequis** : Section 1.1 (Qu'est-ce que MariaDB ?)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre l'histoire de MySQL et de MariaDB
- Identifier les raisons de la création de MariaDB comme fork de MySQL
- Connaître les différences techniques majeures entre les deux systèmes
- Comprendre le concept de "drop-in replacement"
- Évaluer la compatibilité et les divergences entre MariaDB et MySQL
- Décider quand choisir MariaDB plutôt que MySQL

---

## Introduction

Pourquoi parler de MySQL dans une formation sur MariaDB ? Parce que **l'histoire de MariaDB est indissociable de celle de MySQL**. MariaDB n'est pas né dans le vide : c'est un **fork** (embranchement) de MySQL, créé pour préserver l'esprit open source et l'innovation du projet original.

Comprendre cette histoire vous aidera à :
- 📚 Saisir les valeurs fondatrices de MariaDB
- 🔄 Comprendre pourquoi MariaDB est compatible avec MySQL
- 🚀 Apprécier les innovations apportées par MariaDB
- 🎯 Faire des choix techniques éclairés pour vos projets

---

## L'histoire de MySQL : Les origines (1995-2008)

### 🌱 La naissance de MySQL (1995)

**MySQL** a été créé en **1995** par trois développeurs suédois :
- **Michael "Monty" Widenius** (développeur principal)
- **David Axmark**
- **Allan Larsson**

Ils fondent **MySQL AB** (AB = Aktiebolag, équivalent suédois de "SA") avec une vision révolutionnaire pour l'époque :

💡 **Vision** : Créer un système de base de données **open source, rapide et gratuit**, accessible à tous.

### 📈 L'âge d'or de MySQL (1995-2008)

MySQL connaît une croissance fulgurante :

| Année | Événement clé |
|-------|--------------|
| **1995** | Première version publique |
| **2000** | Passage sous licence GPL |
| **2001** | Version 3.23 - Support des transactions (InnoDB) |
| **2003** | Version 4.0 - Réplication, unions, sous-requêtes |
| **2005** | Version 5.0 - Procédures stockées, triggers, vues |
| **2008** | **10 millions d'installations** dans le monde |

**Adoption massive** :
- 🌐 Sites web : WordPress, Drupal, Joomla
- 🏢 Entreprises : Yahoo!, Google, Facebook, YouTube
- 💻 Stack LAMP : **Linux + Apache + MySQL + PHP**

💡 MySQL devient le **SGBD open source le plus populaire au monde**, symbole du logiciel libre et du web 2.0.

### 💰 Le modèle économique "dual license"

MySQL AB invente un modèle hybride :
- 🆓 **Version Community** : GPL, gratuite
- 💼 **Version Enterprise** : Licence commerciale avec support

Ce modèle permet de financer le développement tout en restant open source.

---

## Le rachat par Sun Microsystems, puis Oracle (2008-2010)

### ☀️ Sun Microsystems rachète MySQL (2008)

En **janvier 2008**, **Sun Microsystems** rachète MySQL AB pour **1 milliard de dollars**.

**Réactions de la communauté** :
- ⚠️ **Inquiétudes** : Sun va-t-il préserver l'esprit open source ?
- ✅ **Optimisme** : Plus de ressources pour le développement
- 🤔 **Attentisme** : Wait and see

**En pratique** : Sun maintient l'open source et continue le développement (MySQL 5.1, 5.5).

### 🏢 Oracle rachète Sun Microsystems (2009-2010)

En **avril 2009**, **Oracle Corporation** annonce le rachat de Sun Microsystems pour **7,4 milliards de dollars**.

Ce rachat inclut MySQL, et **déclenche une onde de choc** dans la communauté open source.

**Pourquoi tant d'inquiétudes ?**

Oracle possède déjà **Oracle Database**, l'un des SGBD commerciaux les plus chers du marché (licences de plusieurs milliers à millions de dollars). MySQL est son **concurrent open source direct**.

**Questions qui se posent** :
- ❓ Oracle va-t-il tuer MySQL pour protéger Oracle DB ?
- ❓ Peut-on faire confiance à Oracle pour maintenir l'open source ?
- ❓ MySQL va-t-il devenir un produit commercial ?
- ❓ L'innovation va-t-elle ralentir ?

### 🚨 Les craintes de Monty Widenius

**Michael "Monty" Widenius**, créateur principal de MySQL, est **fortement opposé** au rachat. Il :

1. 🎤 Mène une campagne publique "Save MySQL"
2. 📝 Écrit des lettres ouvertes à la Commission Européenne
3. ⚠️ Alerte sur les risques pour l'open source
4. 🛡️ Tente de bloquer le rachat (sans succès)

💬 **Citation de Monty** (2009) :
> "Je ne fais pas confiance à Oracle pour maintenir MySQL comme un projet véritablement open source et innovant. Ils ont un conflit d'intérêt évident."

**Résultat** : Le rachat est approuvé par la Commission Européenne en **janvier 2010**, sous certaines conditions.

---

## La naissance de MariaDB (2009)

### 🌟 Le fork historique

Face à l'incertitude, Monty Widenius prend une décision radicale :

📅 **Octobre 2009** : Annonce de **MariaDB**, un **fork** (embranchement) de MySQL 5.1.

**Qu'est-ce qu'un fork ?**

Un **fork** est la création d'un nouveau projet à partir du code source d'un projet existant. C'est possible grâce à la licence open source (GPL).

```
MySQL 5.1 (2008)
    │
    ├─→ Oracle continue MySQL 5.5, 5.6, 5.7, 8.0...
    │
    └─→ Fork → MariaDB 5.1, 5.2, 5.3, 5.5, 10.0... (2009)
```

### 🎯 Les objectifs de MariaDB

Monty définit les principes fondateurs :

1. 🆓 **Rester 100% open source** (GPL)
2. 🔄 **Maintenir la compatibilité** avec MySQL
3. 🚀 **Innover plus rapidement** que MySQL
4. 🌍 **Gouvernance communautaire** (pas de contrôle unique)
5. 🛡️ **Garantir la pérennité** du projet

💡 **Le nom "MariaDB"** : Nommé d'après **Maria Widenius**, la plus jeune fille de Monty (comme MySQL vient de "My", sa fille aînée).

### 🏛️ Création de la MariaDB Foundation (2012)

En **2012**, pour garantir l'indépendance du projet :

**MariaDB Foundation** est créée comme organisation **à but non lucratif** :
- 🎯 Mission : Développer et maintenir MariaDB Server
- 🌍 Gouvernance : Communauté internationale
- 🔓 Garantie : MariaDB restera toujours open source

**Séparation claire** :
- **MariaDB Foundation** : Développement du projet (non-profit)
- **MariaDB Corporation** : Services commerciaux (support, cloud)

Cette séparation protège le projet contre tout rachat hostile.

---

## L'évolution de MariaDB depuis 2009

### 📊 Chronologie des versions majeures

| Année | Version | Nouveautés clés |
|-------|---------|-----------------|
| **2009** | MariaDB 5.1 | Fork initial de MySQL 5.1 |
| **2010** | MariaDB 5.2 | Nouvelle optimisations, Aria storage engine |
| **2011** | MariaDB 5.3 | Sous-requêtes optimisées, NoSQL HandlerSocket |
| **2012** | MariaDB 5.5 | Basé sur MySQL 5.5, Thread Pool |
| **2013** | MariaDB 10.0 | 🚀 **Numérotation indépendante**, Galera Cluster, CONNECT |
| **2014** | MariaDB 10.1 | Encryption at rest, multi-source replication |
| **2017** | MariaDB 10.2 | Window functions, JSON, Common Table Expressions |
| **2018** | MariaDB 10.3 | System-versioned tables, instant ADD COLUMN |
| **2019** | MariaDB 10.4 | User account locking, UUID type |
| **2020** | MariaDB 10.5 | INET6, ColumnStore intégré, S3 storage engine |
| **2021** | MariaDB 10.6 | 🛡️ **Première version LTS** (Long Term Support) |
| **2023** | MariaDB 10.11 | LTS, Atomic DDL |
| **2024** | MariaDB 11.4 | 🆕 **LTS 3 ans**, améliorations réplication |
| **2025** | MariaDB 11.8 | 🆕 **LTS + Vector Search**, TLS défaut, TIMESTAMP 2106 |

### 🎯 Pourquoi la numérotation saute de 5.5 à 10.0 ?

En **2013**, MariaDB passe de la version **5.5** à **10.0** pour :
- 🆔 **Affirmer son identité** propre (pas juste "MySQL amélioré")
- 🔢 **Éviter la confusion** avec MySQL 5.6/5.7
- 🚀 **Signaler les innovations** majeures (Galera, CONNECT...)

💡 **Note** : Depuis cette date, MariaDB et MySQL suivent des numérotations **complètement différentes**.

### 📈 Adoption et croissance

**MariaDB gagne rapidement en popularité** :

| Année | Événement d'adoption |
|-------|---------------------|
| **2012** | Wikipedia migre vers MariaDB |
| **2013** | Google migre vers MariaDB |
| **2013** | Fedora/RedHat remplacent MySQL par MariaDB |
| **2014** | Debian fait de MariaDB le choix par défaut |
| **2015** | SUSE Linux passe à MariaDB |
| **2016** | Ubuntu propose MariaDB par défaut |
| **2018** | Booking.com utilise MariaDB à grande échelle |
| **2020+** | Adoption massive dans le cloud (AWS RDS, Azure, GCP) |

**Chiffres clés (2025)** :
- 🌍 **+10 millions** d'installations
- 📈 **+40%** de croissance depuis 2020
- 🏢 **Fortune 500** : Nombreuses entreprises utilisent MariaDB

---

## MySQL depuis le rachat par Oracle (2010-2025)

### 📉 Les inquiétudes se confirment (partiellement)

Qu'est devenu MySQL sous Oracle ?

**Points positifs** ✅ :
- MySQL continue d'exister et d'être développé
- Versions majeures : 5.6 (2013), 5.7 (2015), 8.0 (2018), 8.4 (2024)
- Améliorations de performance (jusqu'à 3x plus rapide)
- Nouvelles fonctionnalités : JSON natif, window functions, CTEs

**Points négatifs** ⚠️ :
- 🐌 **Innovation ralentie** : Cycles de release plus longs
- 💰 **Commercialisation** accrue : Pression vers MySQL Enterprise
- 🔒 **Fermeture** : Certains composants deviennent propriétaires (MySQL Enterprise Edition)
- 📚 **Documentation** : Parties réservées aux clients payants
- 🛠️ **Outils** : MySQL Workbench, certains plugins deviennent commerciaux

### 🔄 Oracle vs Open Source : Une relation complexe

Oracle marche sur une corde raide :
- 👍 Maintient MySQL open source (version Community)
- 👎 Mais pousse fortement vers les versions payantes
- 👍 Investit dans le développement
- 👎 Mais innove moins vite que MariaDB

💬 **Perception de la communauté** :
> "MySQL existe encore, mais il a perdu son âme open source."

---

## Les différences techniques entre MariaDB et MySQL

Bien que compatibles à la base, MariaDB et MySQL ont **divergé progressivement** depuis 2009.

### 🔄 Compatibilité : Le concept de "Drop-in Replacement"

**MariaDB est un "drop-in replacement" de MySQL**, ce qui signifie :

✅ **Compatible** :
- 🔌 **Protocole client** : Les clients MySQL se connectent à MariaDB sans modification
- 📦 **Connecteurs** : JDBC, PHP mysqli, Python mysql-connector fonctionnent avec MariaDB
- 🗃️ **Format de données** : MariaDB peut lire les fichiers de données MySQL
- 💻 **Syntaxe SQL** : Les requêtes SQL de base sont identiques
- 🛠️ **Commandes** : `mysql` client fonctionne avec MariaDB

📝 **Exemple pratique** :
```bash
# Ces deux commandes sont équivalentes :
mysql -h localhost -u root -p database_name
mariadb -h localhost -u root -p database_name
```

✅ **Migration facile** :
```bash
# Migrer de MySQL vers MariaDB
# 1. Sauvegarde MySQL
mysqldump --all-databases > backup.sql

# 2. Arrêter MySQL, installer MariaDB
systemctl stop mysql
apt install mariadb-server

# 3. Restaurer dans MariaDB
mariadb < backup.sql
```

### 🎯 Différences majeures : Fonctionnalités

Voici les principales divergences techniques :

#### 1️⃣ **Moteurs de stockage**

| Moteur | MariaDB | MySQL |
|--------|---------|-------|
| **InnoDB** | ✅ Oui (fork indépendant) | ✅ Oui (développé par Oracle) |
| **MyISAM** | ✅ Oui | ✅ Oui |
| **Aria** | ✅ Oui (crash-safe MyISAM) | ❌ Non |
| **ColumnStore** | ✅ Oui (analytics/OLAP) | ❌ Non |
| **S3** 🆕 | ✅ Oui (archivage cloud) | ❌ Non |
| **Vector** 🆕 | ✅ Oui (recherche vectorielle IA) | ❌ Non |
| **CONNECT** | ✅ Oui (accès données externes) | ❌ Non |
| **Spider** | ✅ Oui (sharding) | ❌ Non (remplacé par MySQL Cluster/NDB) |

#### 2️⃣ **Réplication et Haute Disponibilité**

| Feature | MariaDB | MySQL |
|---------|---------|-------|
| **Réplication async** | ✅ Oui | ✅ Oui |
| **Réplication semi-sync** | ✅ Oui | ✅ Oui |
| **GTID** | ✅ Oui (format différent) | ✅ Oui |
| **Galera Cluster** | ✅ Intégré (multi-master sync) | ❌ Non (nécessite addon) |
| **MySQL Group Replication** | ❌ Non | ✅ Oui |
| **Multi-source replication** | ✅ Oui | ✅ Oui (depuis 5.7) |

💡 **Galera Cluster** est un **avantage majeur** de MariaDB : réplication **synchrone multi-master** intégrée nativement.

#### 3️⃣ **Fonctionnalités SQL avancées**

| Feature | MariaDB | MySQL | Depuis |
|---------|---------|-------|--------|
| **Window Functions** | ✅ Oui | ✅ Oui | MariaDB 10.2 (2017) / MySQL 8.0 (2018) |
| **CTEs (WITH clause)** | ✅ Oui | ✅ Oui | MariaDB 10.2 (2017) / MySQL 8.0 (2018) |
| **Recursive CTEs** | ✅ Oui | ✅ Oui | MariaDB 10.2 / MySQL 8.0 |
| **JSON Functions** | ✅ Oui (+ avancé) | ✅ Oui | MariaDB 10.2 / MySQL 5.7 |
| **JSON Path expressions** 🆕 | ✅ Oui (amélioré 11.8) | ⚠️ Basique | |
| **JSON Schema validation** 🆕 | ✅ Oui (11.8) | ❌ Non | |
| **Sequences** | ✅ Oui (CREATE SEQUENCE) | ❌ Non (via AUTO_INCREMENT) | MariaDB 10.3 |
| **System-Versioned Tables** | ✅ Oui (historique auto) | ❌ Non | MariaDB 10.3 |
| **Application Time Period** 🆕 | ✅ Oui (11.8) | ❌ Non | |
| **RETURNING clause** | ✅ Oui | ❌ Non | MariaDB 10.5 |
| **INTERSECT / EXCEPT** | ✅ Oui | ❌ Non | MariaDB 10.3 |

#### 4️⃣ **Authentification et sécurité**

| Feature | MariaDB | MySQL |
|---------|---------|-------|
| **mysql_native_password** | ✅ Oui | ✅ Oui (déprécié 8.0) |
| **caching_sha2_password** | ✅ Oui | ✅ Oui (défaut MySQL 8.0) |
| **ed25519** | ✅ Oui (défaut) | ❌ Non |
| **PAM Authentication** | ✅ Oui | ⚠️ Enterprise uniquement |
| **LDAP/Kerberos** | ✅ Oui | ⚠️ Enterprise uniquement |
| **PARSEC plugin** 🆕 | ✅ Oui (11.8) | ❌ Non |
| **TLS par défaut** 🆕 | ✅ Oui (11.8) | ⚠️ Recommandé mais pas défaut |
| **Roles** | ✅ Oui | ✅ Oui |
| **Data at Rest Encryption** | ✅ Oui | ⚠️ Enterprise uniquement |

💡 **MariaDB offre plus de fonctionnalités de sécurité gratuitement** que MySQL Community.

#### 5️⃣ **Performance et optimisation**

| Feature | MariaDB | MySQL |
|---------|---------|-------|
| **Thread Pool** | ✅ Oui (gratuit) | ⚠️ Enterprise uniquement |
| **Optimizer improvements** | ✅ Oui (continu) | ✅ Oui |
| **Cost-based optimizer SSD** 🆕 | ✅ Oui (11.8) | ⚠️ Partiel |
| **innodb_alter_copy_bulk** 🆕 | ✅ Oui (11.8, 2x rapide) | ❌ Non |
| **Optimistic ALTER TABLE** 🆕 | ✅ Oui (11.8, réplication) | ❌ Non |
| **Query Cache** | 🔄 Déprécié | 🔄 Déprécié (MySQL 8.0) |

#### 6️⃣ **Unicode et internationalisation**

| Feature | MariaDB | MySQL |
|---------|---------|-------|
| **utf8mb4 par défaut** 🆕 | ✅ Oui (11.8) | ✅ Oui (MySQL 8.0) |
| **UCA 14.0.0 collations** 🆕 | ✅ Oui (11.8) | ⚠️ Versions antérieures |
| **Support Unicode étendu** | ✅ Excellent | ✅ Bon |

#### 7️⃣ **Innovations uniques à MariaDB** 🆕

| Feature | Description | Depuis |
|---------|-------------|--------|
| **MariaDB Vector** 🤖 | Type VECTOR + Index HNSW pour IA/RAG | 11.8 (2025) |
| **TIMESTAMP 2038→2106** | Extension timestamp (Y2038 résolu) | 11.8 |
| **S3 Storage Engine** | Archivage sur object storage | 10.5 (2020) |
| **MaxScale 25.01** | Workload Capture/Replay, Diff Router | 25.01 (2025) |

### ⚠️ Incompatibilités à connaître

Bien que largement compatible, quelques différences importantes :

#### 🔴 **GTID : Formats différents**
```sql
-- MySQL GTID
SET @@GLOBAL.GTID_MODE = ON;
-- Format : server_uuid:transaction_id

-- MariaDB GTID
SET @@GLOBAL.gtid_strict_mode = ON;
-- Format : domain_id-server_id-sequence_number
```

💡 **Implication** : La réplication entre MySQL et MariaDB nécessite une attention particulière.

#### 🔴 **Authentification par défaut**
- **MySQL 8.0** : `caching_sha2_password` (par défaut)
- **MariaDB** : `mysql_native_password` ou `ed25519`

💡 Peut nécessiter des ajustements lors de migrations.

#### 🔴 **Certaines extensions propriétaires MySQL**
- **MySQL X Protocol** (document store) : Incompatible avec MariaDB
- **MySQL Shell** : Ne fonctionne pas avec MariaDB
- **MySQL Enterprise Audit** : Non compatible

---

## Tableau récapitulatif : MariaDB vs MySQL (2025)

| Critère | MariaDB 11.8 LTS | MySQL 8.4 LTS |
|---------|------------------|---------------|
| **Licence** | GPL v2 (100% libre) | GPL v2 + Commercial |
| **Propriétaire** | MariaDB Foundation (non-profit) | Oracle Corporation |
| **Open Source** | ✅ Totalement | ⚠️ Community vs Enterprise |
| **Compatibilité** | ✅ Drop-in replacement MySQL 5.5-5.7 | - |
| **Moteurs de stockage** | InnoDB, Aria, ColumnStore, S3, Vector | InnoDB, MyISAM |
| **Réplication** | Async, Semi-sync, Galera (sync multi-master) | Async, Semi-sync, Group Replication |
| **Haute dispo** | Galera Cluster intégré | Group Replication, NDB Cluster |
| **Fonctionnalités SQL** | ⭐⭐⭐⭐⭐ (Sequences, RETURNING, etc.) | ⭐⭐⭐⭐ |
| **JSON** | ✅ Avancé (Path, Schema validation) | ✅ Standard |
| **Sécurité gratuite** | ✅ TLS défaut, PAM, LDAP, Encryption | ⚠️ Encryption = Enterprise |
| **Thread Pool** | ✅ Gratuit | ⚠️ Enterprise uniquement |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Innovation** | 🚀 Rapide (Vector, S3, etc.) | 🐢 Plus lente |
| **Support LTS** | ✅ 3 ans (depuis 11.4) | ✅ 5 ans (8.4) |
| **Communauté** | ⭐⭐⭐⭐⭐ Active et transparente | ⭐⭐⭐⭐ |
| **Documentation** | ✅ Entièrement gratuite | ⚠️ Certaines parties Enterprise |
| **IA/ML intégration** 🆕 | ✅ MariaDB Vector natif | ❌ Via plugins externes |
| **Cost** | 🆓 Gratuit | 🆓 Community / 💰 Enterprise |

---

## Quand choisir MariaDB plutôt que MySQL ?

### ✅ Choisissez MariaDB si :

1. 🆓 **Budget limité** : Vous voulez toutes les fonctionnalités gratuitement
2. 🔓 **Open source pur** : Vous valorisez la transparence et la gouvernance communautaire
3. 🚀 **Innovation** : Vous voulez accéder aux dernières fonctionnalités (Vector, S3, etc.)
4. ⚡ **Performance** : Thread Pool gratuit, optimisations avancées
5. 🔄 **Haute dispo** : Galera Cluster multi-master intégré
6. 🤖 **IA/ML** : Vous avez besoin de recherche vectorielle (RAG, embeddings)
7. 📊 **Analytics** : ColumnStore pour OLAP
8. 🔒 **Sécurité avancée** : Encryption, PAM, audit gratuits
9. 🌍 **Indépendance** : Vous ne voulez pas dépendre d'Oracle

### ⚠️ Restez sur MySQL si :

1. 🏢 **Support Oracle** : Vous avez besoin du support commercial d'Oracle
2. 🔧 **Outils spécifiques** : Vous dépendez de MySQL Shell, X Protocol
3. 📚 **Écosystème** : Certains outils tiers ne supportent que MySQL
4. 🔄 **Réplication vers MySQL** : Vous avez une infra MySQL existante complexe
5. 💼 **Contraintes d'entreprise** : Politique de standardisation sur Oracle

### 🎯 Migration MySQL → MariaDB

**La migration est généralement simple** :

```bash
# Scénario typique de migration

# 1. Sauvegarde MySQL
mysqldump --all-databases --routines --triggers > full_backup.sql

# 2. Installer MariaDB
apt remove mysql-server mysql-client
apt install mariadb-server mariadb-client

# 3. Restaurer
mariadb < full_backup.sql

# 4. Upgrade (si nécessaire)
mariadb-upgrade -u root -p

# 5. Vérifier
mariadb -u root -p -e "SELECT VERSION();"
```

💡 **Temps d'arrêt** : Quelques minutes à quelques heures selon la taille de la base.

**Stratégie zero-downtime** (réplication) :
```bash
# 1. Configurer réplication MySQL (master) → MariaDB (replica)
# 2. Attendre synchronisation complète
# 3. Basculer le trafic vers MariaDB
# 4. Désactiver l'ancien MySQL
```

---

## ✅ Points clés à retenir

- 📚 **MariaDB est un fork de MySQL** créé en 2009 par Monty Widenius, créateur original de MySQL
- 🏢 **Rachat par Oracle** (2010) a déclenché la création de MariaDB pour préserver l'open source
- 🏛️ **MariaDB Foundation** (non-profit) garantit que MariaDB reste 100% libre
- 🔄 **Drop-in replacement** : MariaDB est compatible avec MySQL (protocole, données, SQL)
- 🚀 **Innovation** : MariaDB innove plus rapidement (Vector, S3, Galera, Sequences, etc.)
- 🆓 **Gratuit** : Toutes les fonctionnalités MariaDB sont gratuites (vs MySQL Enterprise)
- ⚡ **Performance** : Thread Pool, optimisations avancées gratuites
- 🔒 **Sécurité** : Plus de fonctionnalités de sécurité gratuites que MySQL
- 🤖 **IA/ML** 🆕 : MariaDB Vector (11.8) pour recherche vectorielle native
- 🌍 **Adoption** : Wikipedia, Google, Booking.com utilisent MariaDB
- 📊 **Différences** : Moteurs (Aria, ColumnStore, Vector), Galera, Sequences, RETURNING
- ⚠️ **Divergence** : GTID différents, certaines extensions MySQL incompatibles
- 🎯 **Migration facile** : De MySQL vers MariaDB en quelques étapes

---

## 🔗 Ressources et références

### 📖 Documentation officielle
- [MariaDB History](https://mariadb.org/about/)
- [MariaDB vs MySQL Feature Comparison](https://mariadb.com/kb/en/mariadb-vs-mysql-features/)
- [Incompatibilities and Feature Differences](https://mariadb.com/kb/en/mariadb-vs-mysql-compatibility/)

### 📰 Articles historiques
- [Why MariaDB was created](https://mariadb.org/history/)
- [Monty's Blog : Save MySQL from Oracle](https://blog.mariadb.org/tag/mysql/)
- [Oracle Buys Sun - The Story Behind](https://www.computerworld.com/article/2525314/oracle-buys-sun.html)

### 🎥 Vidéos
- [Michael "Monty" Widenius : Why MariaDB](https://www.youtube.com/watch?v=...)
- [MariaDB vs MySQL: What's the Difference?](https://www.youtube.com/results?search_query=mariadb+vs+mysql)

### 🔧 Migration guides
- [Migrating from MySQL to MariaDB](https://mariadb.com/kb/en/migrating-from-mysql-to-mariadb/)
- [MySQL Compatibility Guide](https://mariadb.com/kb/en/mysql-compatibility/)

---

## ➡️ Section suivante

**[1.3 - Cas d'usage et écosystème](./03-cas-usage-et-ecosysteme.md)**

Maintenant que vous connaissez l'histoire de MariaDB et ses différences avec MySQL, explorons dans la section suivante les **cas d'usage concrets** où MariaDB excelle, et découvrons son **riche écosystème** d'outils et d'intégrations.

---

*Document rédigé pour MariaDB 11.8 LTS (Juin 2025)*
*Formation "De Débutant à Expert" - Section 1.2*
*Licence : CC BY-NC-SA 4.0*

⏭️ [Cas d'usage et écosystème](/01-introduction-fondamentaux/03-cas-usage-et-ecosysteme.md)
