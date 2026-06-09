# 🐬 Formation MariaDB 12.3 LTS

![License](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-blue.svg)  
![MariaDB Version](https://img.shields.io/badge/MariaDB-12.3%20LTS-orange.svg)  
![Modules](https://img.shields.io/badge/Modules-20-green.svg)  
![Language](https://img.shields.io/badge/Langue-Français-blue.svg)  
![Status](https://img.shields.io/badge/Status-En%20cours-yellow.svg)

**Un guide progressif pour découvrir et approfondir MariaDB — Du débutant à l'expert**

<p align="center">
  <img src="https://mariadb.com/wp-content/uploads/2019/11/mariadb-logo-vert_blue-transparent.png" alt="MariaDB Logo" width="300"/>
</p>

---

## 📖 Table des matières

- [À propos](#-%C3%A0-propos)
- [Points forts](#-points-forts)
- [Pour qui ?](#-pour-qui-)
- [Contenu](#-contenu-de-la-formation)
- [Démarrage](#-d%C3%A9marrage-rapide)
- [Parcours](#-comment-utiliser-cette-formation)
- [Structure](#-structure-du-projet)
- [FAQ](#-faq)
- [Licence](#-licence)
- [Contact](#%E2%80%8D-contact)

---

## 📋 À propos

Formation complète sur MariaDB couvrant tous les aspects du SGBD, des bases SQL aux architectures cloud-native, en passant par la haute disponibilité et l'intégration IA avec MariaDB Vector.

Cette formation tente de rassembler en un seul endroit tout ce qu'il faut savoir pour travailler avec MariaDB, que vous débutiez ou que vous cherchiez à approfondir. Elle n'est pas parfaite, mais l'objectif est de vous faire gagner du temps dans votre apprentissage.

**✨ Ce que vous y trouverez :**

- 📚 **20 modules progressifs** — Du SQL de base aux architectures distribuées
- 🆕 **MariaDB 12.3 LTS** — Version de référence (mai 2026), supportée jusqu'en juin 2029
- 🤖 **MariaDB Vector** — Recherche vectorielle, IA, RAG, intégration LLMs
- 🏗️ **Production-ready** — HA, monitoring, backup, Kubernetes, DevOps
- 🔧 **Pratique** — 200+ exemples SQL, configurations, checklists
- 📖 **9 annexes** — Glossaire, commandes, configurations, références
- 🇫🇷 **En français** — Parce que c'est plus accessible

**Durée estimée :** 35-50 heures • **Niveau :** Tous niveaux

---

## ✨ Points forts

- ✅ **Progressive** — Parcours structuré du débutant à l'expert
- ✅ **Complète** — 20 chapitres, 200+ sous-sections
- ✅ **À jour** — Intégration complète de MariaDB 12.3 LTS et ses nouveautés
- ✅ **Professionnelle** — Production, monitoring, HA, architectures modernes
- ✅ **Moderne** — Cloud, Kubernetes, IA, architectures distribuées
- ✅ **Pratique** — Annexes et références techniques détaillées

---

## 🎯 Pour qui ?

Cette formation s'adresse à différents profils. Choisissez votre parcours :

| 👤 Profil | 📚 Modules recommandés | ⏱️ Durée estimée |
|-----------|------------------------|------------------|
| **Développeur** | 1-6, 8-9, 17-18 | 5-7 jours |
| **Administrateur/DBA** | 1, 7, 10-15, 19 | 7-10 jours |
| **DevOps/Cloud** | 1, 11-12, 14, 16, 20 | 4-6 jours |
| **IA/ML Engineer** | 1, 4 (JSON/Vector), 17-18, 20 | 3-4 jours |
| **Parcours complet** | 1-20 | 15-20 jours |

---

## 📚 Contenu de la formation

### Les 10 Parties

| # | Partie | Intro | Modules | Niveau | Sujets clés |
|---|--------|-------|---------|--------|-------------|
| **1** | Introduction et Fondamentaux | [📄](/partie-01-introduction-fondamentaux.md) | 1-2 | 🌱 Débutant | Installation, SQL de base, types de données, CRUD |
| **2** | Requêtes SQL Intermédiaires et Avancées | [📄](/partie-02-requetes-sql-intermediaires-avancees.md) | 3-4 | 🌱 Débutant | Agrégations, jointures, window functions, JSON |
| **3** | Index, Transactions et Performance | [📄](/partie-03-index-transactions-performance.md) | 5-6 | 🌿 Intermédiaire | B-Tree, ACID, isolation, MVCC, EXPLAIN |
| **4** | Moteurs de Stockage et Programmation Serveur | [📄](/partie-04-moteurs-stockage-programmation.md) | 7-9 | 🌿 Intermédiaire | InnoDB, ColumnStore, procédures, triggers, vues |
| **5** | Sécurité et Administration | [📄](/partie-05-securite-administration.md) | 10-12 | 🌳 Avancé | Utilisateurs, SSL/TLS, audit, backup, restauration |
| **6** | Réplication et Haute Disponibilité | [📄](/partie-06-replication-haute-disponibilite.md) | 13-14 | 🌳 Avancé | Master-Slave, GTID, Galera, MaxScale |
| **7** | Performance et Tuning | [📄](/partie-07-performance-tuning.md) | 15 | 🌳 Avancé | Buffer pool, slow queries, partitionnement |
| **8** | DevOps, Cloud et Automatisation | [📄](/partie-08-devops-cloud-automatisation.md) | 16 | 🌳 Avancé | Docker, Kubernetes, Ansible, CI/CD, monitoring |
| **9** | Intégration et Fonctionnalités Avancées | [📄](/partie-09-integration-fonctionnalites-avancees.md) | 17-18 | 🌳 Avancé | APIs, ORM, encryption, **MariaDB Vector** |
| **10** | Migration, Compatibilité et Architectures | [📄](/partie-10-migration-architectures.md) | 19-20 | 🌳 Avancé | Migration MySQL, microservices, RAG |

> 💡 **Note** : Chaque partie dispose d'une **page d'introduction** (📄) présentant les objectifs, prérequis, durée estimée, et compétences acquises. Consultez-les avant de plonger dans les modules détaillés !

### 🆕 Nouveautés MariaDB 12.3 LTS

Cette formation est centrée sur **MariaDB 12.3 LTS** (GA 12.3.2, mai 2026, support jusqu'en juin 2029), qui consolide la série *rolling* 12.0 → 12.2. Les apports majeurs :

- ⚡ **Binlog intégré à InnoDB** — journal binaire réécrit (*opt-in*), suppression de la synchronisation redondante binlog↔redo
- 🎛️ **Optimizer Hints** — contrôle fin du plan d'exécution via les commentaires `/*+ … */`
- 🟠 **Compatibilité Oracle** — `TO_DATE`, `TO_NUMBER`, `TRUNC`, jointures `( + )`, tableaux associatifs, `SYS_REFCURSOR`, type `XML`, `SET PATH`
- 🔵 **Compatibilité MySQL 8** — `caching_sha2_password`, nouvelles fonctions GIS
- 📐 **Standard SQL** — prédicat `IS JSON`, `UPDATE`/`DELETE` depuis une CTE
- 🔐 **Sécurité** — `SET SESSION AUTHORIZATION`, clés SSL avec *passphrase*, audit bufferisé
- 🟢 **Galera** — *packaging* séparé (`mariadb-server-galera`), réplication parallèle inter-clusters

> Les fonctionnalités de la **11.8 LTS** (MariaDB Vector, PARSEC, `utf8mb4` par défaut/UCA 14.0.0, extension `TIMESTAMP` 2106, MaxScale 25.01…) sont désormais traitées comme **contenu standard** de la formation.

### 📎 Les 9 Annexes

- **A.** Glossaire des termes techniques (ACID, MVCC, GTID...)
- **B.** Commandes mariadb CLI essentielles
- **C.** Requêtes SQL de référence (admin, monitoring, analyse)
- **D.** Configurations par cas d'usage (OLTP, OLAP, dev...)
- **E.** Checklist de performance (audit config, index, requêtes)
- **F.** Nouveautés 12.3 en un coup d'œil
- **G.** Versions de référence (12.3, 11.8, 11.4, 10.11, 10.6)
- **H.** Ressources et documentation
- **I.** Changelog de la formation

📖 **Sommaire détaillé** → [SOMMAIRE.md](/SOMMAIRE.md)

---

## 🚀 Démarrage rapide

### Installation MariaDB

```bash
# Vérifier si MariaDB est installé
mariadb --version

# Installer MariaDB 12.3 LTS
# Documentation : https://mariadb.com/downloads/
# Windows : https://mariadb.com/downloads/community/
# macOS   : brew install mariadb
# Ubuntu  : sudo apt install mariadb-server
```

### Configuration minimale

```sql
-- Se connecter
mariadb -u root -p

-- Créer un utilisateur
CREATE USER 'dev'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'dev'@'localhost';

-- Créer une base de test
CREATE DATABASE formation_mariadb CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
USE formation_mariadb;

-- Vérifier la version
SELECT VERSION();
-- ➜ 12.3.x-MariaDB

-- ✅ Vous êtes prêt !
```

### Cloner cette formation

```bash
git clone https://github.com/NDXDeveloper/formation-mariadb.git
cd formation-mariadb
```

---

## 📁 Structure du projet

```
formation-mariadb/
│
├── 📄 README.md                    # Ce fichier
├── 📄 SOMMAIRE.md                  # Table des matières complète
├── 📄 LICENSE                      # Licence CC BY-NC-SA 4.0
│
├── 📄 partie-01-introduction-fondamentaux.md          # 🆕 Intro Partie 1
├── 📄 partie-02-requetes-sql-intermediaires-avancees.md
├── 📄 partie-03-index-transactions-performance.md
├── 📄 partie-04-moteurs-stockage-programmation.md
├── 📄 partie-05-securite-administration.md
├── 📄 partie-06-replication-haute-disponibilite.md
├── 📄 partie-07-performance-tuning.md
├── 📄 partie-08-devops-cloud-automatisation.md
├── 📄 partie-09-integration-fonctionnalites-avancees.md
├── 📄 partie-10-migration-architectures.md             # 🆕 Intro Partie 10
│
├── 📂 01-introduction-fondamentaux/        # Contenu détaillé Partie 1
├── 📂 02-bases-du-sql/
├── 📂 03-requetes-sql-intermediaires/
├── 📂 04-concepts-avances-sql/
├── 📂 05-index-et-performance/
├── 📂 06-transactions-et-concurrence/
├── 📂 07-moteurs-de-stockage/
├── 📂 08-programmation-cote-serveur/
├── 📂 09-vues-et-donnees-virtuelles/
├── 📂 10-securite-gestion-utilisateurs/
├── 📂 11-administration-configuration/
├── 📂 12-sauvegarde-restauration/
├── 📂 13-replication/
├── 📂 14-haute-disponibilite/
├── 📂 15-performance-tuning/
├── 📂 16-devops-automatisation/
├── 📂 17-integration-developpement/
├── 📂 18-fonctionnalites-avancees/
├── 📂 19-migration-compatibilite/
├── 📂 20-cas-usage-architectures/
│
├── 📂 annexes/
│   ├── a-glossaire/
│   ├── b-commandes-cli/
│   ├── c-requetes-sql-reference/
│   ├── d-configuration-reference/
│   ├── e-checklist-performance/
│   ├── f-nouveautes-12-3/
│   ├── g-versions-reference/
│   ├── h-ressources-documentation/
│   └── i-changelog/
│
└── 📂 assets/
```

---

## 🎯 Comment utiliser cette formation

### Débutant complet
👉 Commencez par la [Partie 1 : Introduction et Fondamentaux](/partie-01-introduction-fondamentaux.md) et suivez l'ordre

### Développeur
👉 Modules 1-6, 8-9, 17-18 — Focus SQL, intégration apps, MariaDB Vector

### DBA / Administrateur
👉 Modules 1, 7, 10-15, 19 — Administration, sécurité, backup, HA

### DevOps / Cloud
👉 Modules 1, 11-12, 14, 16, 20 — Kubernetes, Docker, monitoring, architectures

### IA / ML Engineer
👉 Modules 1, 4 (JSON/Vector), 17-18, 20 — MariaDB Vector, RAG, semantic search

### Besoin d'une référence rapide
👉 Consultez les [Annexes](/annexes/) (glossaire, commandes, configurations)

**💡 Conseil :** Créez une base de test pour pratiquer :
```sql
CREATE DATABASE test_mariadb;
USE test_mariadb;
```

---

## 🗓️ Parcours d'apprentissage suggéré

```
🌱 DÉBUTANT (1-2 semaines)
│
├─ Partie 1 : Fondamentaux
├─ Partie 2 : SQL Intermédiaire
└─ Partie 3 : Index et Transactions
   │
   ▼
🌿 INTERMÉDIAIRE (2-3 semaines)
│
├─ Partie 4 : Moteurs de Stockage
├─ Partie 5 : Sécurité et Admin
└─ Partie 9 : Intégration
   │
   ▼
🌳 AVANCÉ (3-4 semaines)
│
├─ Partie 6 : Haute Disponibilité
├─ Partie 7 : Performance Tuning
├─ Partie 8 : DevOps & Cloud
└─ Partie 10 : Architectures

🎓 Total : 6-8 semaines (30min-1h par jour)
```

**🎯 Parcours Express (3-4 jours)** pour les pressés :
- Module 1 : Fondamentaux
- Module 5 : Index et Performance
- Module 14 : Haute Disponibilité
- Module 18.10 : MariaDB Vector
- Module 20 : Architectures IA/RAG

---

## 🔗 Ressources officielles

| Ressource | Lien |
|-----------|------|
| 📖 Documentation MariaDB | [mariadb.com/docs](https://mariadb.com/docs/) |
| 🏠 MariaDB Foundation | [mariadb.org](https://mariadb.org) |
| 🤖 MariaDB Vector | [mariadb.org/projects/mariadb-vector](https://mariadb.org/projects/mariadb-vector) |
| ⚙️ MaxScale | [mariadb.com/products/maxscale](https://mariadb.com/products/maxscale) |
| 🐳 mariadb-operator (K8s) | [github.com/mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) |
| 📥 Télécharger MariaDB | [mariadb.com/downloads](https://mariadb.com/downloads/) |
| 💬 Forum communautaire | [mariadb.com/kb/en/community](https://mariadb.com/kb/en/community/) |

---

## ❓ FAQ

**Q : Dois-je suivre les modules dans l'ordre ?**
> Oui si vous débutez. Sinon, choisissez votre parcours selon votre profil dans la section [Pour qui ?](#-pour-qui-).

**Q : Quelle version de MariaDB utiliser ?**
> MariaDB 12.3 LTS (recommandée, supportée jusqu'en juin 2029). La 11.8 LTS, très proche, convient aussi.

**Q : Cette formation remplace-t-elle la documentation officielle ?**
> Non, elle la complète. C'est un guide d'apprentissage progressif, pas une référence exhaustive.

**Q : Combien de temps faut-il pour tout faire ?**
> 35-50 heures sur 6-8 semaines (en apprenant 30min-1h par jour). Mais vous pouvez adapter selon vos besoins.

**Q : Y a-t-il des exercices pratiques ?**
> Chaque module contient des exemples SQL. Créez une base de test pour pratiquer en parallèle.

**Q : Puis-je utiliser ce contenu pour enseigner ?**
> Oui, sous licence CC BY-NC-SA 4.0 — Attribution requise, usage non commercial, partage identique.

**Q : La formation couvre-t-elle MariaDB Vector ?**
> Oui ! Module 18.10 complet + intégrations IA dans la partie 20 (RAG, semantic search, LLMs).

**Q : Est-ce adapté pour production ?**
> Oui, les parties 5-8 couvrent tout ce qu'il faut : sécurité, backup, HA, monitoring, Kubernetes.

---

## 📝 Licence

Ce projet est sous licence **Creative Commons Attribution - Pas d'Utilisation Commerciale - Partage dans les Mêmes Conditions 4.0 International (CC BY-NC-SA 4.0)**.

✅ **Vous pouvez :**
- Partager — Copier et redistribuer le matériel
- Adapter — Remixer, transformer et créer à partir du matériel

⚠️ **Selon les conditions suivantes :**
- **Attribution** — Vous devez créditer l'œuvre originale
- **Pas d'Utilisation Commerciale** — Vous ne pouvez pas utiliser le matériel à des fins commerciales
- **Partage dans les Mêmes Conditions** — Toute redistribution doit utiliser la même licence

📄 Voir le fichier [LICENSE](/LICENSE) pour les détails complets.

**Attribution suggérée :**
```
Formation MariaDB par Nicolas DEOUX
https://github.com/NDXDeveloper/formation-mariadb
Licence CC BY-NC-SA 4.0
```

---

## 👨‍💻 Contact

**Nicolas DEOUX**
- 📧 [NDXDev@gmail.com](mailto:NDXDev@gmail.com)
- 💼 [LinkedIn](https://www.linkedin.com/in/nicolas-deoux-ab295980/)
- 🐙 [GitHub](https://github.com/NDXDeveloper)

---

## 🙏 Remerciements

Merci à :
- La **MariaDB Foundation** et toute la communauté MariaDB
- Les contributeurs **open source** qui rendent ces outils accessibles
- **Vous** qui prenez le temps d'apprendre avec cette formation

**Ressources :**
[MariaDB Knowledge Base](https://mariadb.com/kb) • [MariaDB Server](https://github.com/MariaDB/server) • [Percona Blog](https://www.percona.com/blog)

---

<div align="center">

## 🐬 Bon apprentissage avec MariaDB ! 🐬

*Cette formation est un travail en cours. Elle n'est pas parfaite, mais j'espère sincèrement qu'elle vous sera utile dans votre parcours d'apprentissage.*

**[📖 Consulter le sommaire complet →](/SOMMAIRE.md)**

[![Star on GitHub](https://img.shields.io/github/stars/NDXDeveloper/formation-mariadb?style=social)](https://github.com/NDXDeveloper/formation-mariadb)
[![Follow](https://img.shields.io/github/followers/NDXDeveloper?style=social)](https://github.com/NDXDeveloper)

**[⬆ Retour en haut](#-formation-mariadb-123-lts)**

![Made with](https://img.shields.io/badge/Made%20with-☕%20and%20❤️-blue)

*Dernière mise à jour : Juin 2026*

</div>
