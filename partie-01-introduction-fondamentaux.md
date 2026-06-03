🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 1 : Introduction et Fondamentaux (Débutant)

> **Niveau** : Débutant  
> **Durée estimée** : ~2 jours  
> **Prérequis** : Aucun — Cette partie est votre porte d'entrée dans le monde de MariaDB

---

## 🎯 Bienvenue dans votre parcours MariaDB !

Cette première partie constitue les **fondations essentielles** de votre apprentissage de **MariaDB 12.3 LTS**. Que vous soyez développeur, administrateur système en devenir, ou simplement curieux de découvrir les bases de données relationnelles, vous êtes au bon endroit !

L'objectif de cette partie est de vous donner une **compréhension solide et progressive** de MariaDB : ce qu'il est, comment il fonctionne, et comment commencer à l'utiliser efficacement. Nous aborderons les concepts fondamentaux sans supposer aucune connaissance préalable, tout en vous préparant aux chapitres plus avancés de la formation.

À l'issue de cette partie, vous serez capable de **créer vos premières bases de données, écrire des requêtes SQL simples** (sélectionner, insérer, modifier et supprimer des données) et **comprendre l'écosystème MariaDB** dans son ensemble. Vous aurez acquis les bases nécessaires pour aborder sereinement les requêtes intermédiaires et avancées de la **Partie 2**.

---

## 📚 Les deux chapitres de cette partie

### Chapitre 1 : Introduction et Fondamentaux
**9 sections | Durée : ~1 jour**

Ce premier chapitre vous introduit à MariaDB de manière progressive et accessible. Vous découvrirez :
- **Qu'est-ce que MariaDB ?** Une présentation claire du système de gestion de bases de données
- **L'histoire et les différences avec MySQL** : comprendre les origines et l'évolution de MariaDB
- **Les cas d'usage réels** : où et pourquoi utilise-t-on MariaDB en production
- **L'architecture générale** : comment fonctionne un SGBD relationnel
- **La politique de versions** : le modèle LTS (*Long Term Support*) et les *rolling releases*
- **Le cycle de support** : 3 ans pour les LTS récentes (11.8, 12.3)
- **La roadmap** : de la 12.3 LTS vers la série 13.x
- **Installation et configuration** : mettre en place votre premier serveur MariaDB
- **Les outils d'administration** : CLI, HeidiSQL, DBeaver, phpMyAdmin

💡 **Point fort** : une approche pratique qui vous permet d'installer et de commencer à utiliser MariaDB dès les premières heures.

---

### Chapitre 2 : Bases du SQL
**8 sections | Durée : ~1 jour**

Le langage SQL est le cœur de votre interaction avec MariaDB. Ce chapitre vous en enseigne les fondamentaux :
- **Introduction au SQL** : comprendre ce langage universel et ses sous-langages (DDL, DML, DQL, DCL, TCL)
- **Types de données** : numériques, texte, temporels, binaires, types spécifiques MariaDB (`JSON`, `UUID`, `INET6`, `VECTOR`) et le type `XMLTYPE` 🆕
- **Création et gestion des bases de données** : vos premiers `CREATE DATABASE`
- **Création et modification de tables** : `CREATE`, `ALTER`, `DROP`
- **Contraintes** : clés primaires, clés étrangères, `UNIQUE`, `NOT NULL`, `DEFAULT`, `CHECK`
- **Insertion de données** : `INSERT`, `INSERT ... SELECT`, `LOAD DATA`
- **Requêtes de sélection simples** : `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`
- **Mise à jour et suppression** : `UPDATE`, `DELETE`, `TRUNCATE`

💡 **Point fort** : des exemples SQL commentés et testables pour chaque concept, vous permettant d'expérimenter immédiatement.

> ℹ️ Les **requêtes intermédiaires** (agrégations, regroupements, jointures, sous-requêtes…) ne relèvent pas de cette première partie : elles ouvrent la **Partie 2 — Requêtes SQL Intermédiaires et Avancées** (chapitre 3).

---

## ✅ Compétences acquises

À la fin de cette première partie, vous serez capable de :

### Connaissances théoriques
- ✅ **Comprendre** ce qu'est MariaDB et son positionnement dans l'écosystème des bases de données
- ✅ **Expliquer** les différences entre MariaDB et MySQL
- ✅ **Identifier** les cas d'usage appropriés pour MariaDB
- ✅ **Comprendre** l'architecture d'un SGBD relationnel
- ✅ **Choisir** entre versions LTS et rolling selon vos besoins

### Compétences pratiques
- ✅ **Installer et configurer** MariaDB sur votre système
- ✅ **Utiliser** les outils d'administration (CLI, clients graphiques)
- ✅ **Créer et gérer** des bases de données et des tables
- ✅ **Maîtriser** les types de données MariaDB et choisir le plus approprié
- ✅ **Définir des contraintes** pour garantir l'intégrité des données
- ✅ **Écrire** des requêtes `SELECT` simples (filtrer, trier, paginer)
- ✅ **Insérer, modifier et supprimer** des données efficacement et en toute sécurité

### Autonomie
- ✅ **Naviguer** dans la documentation officielle MariaDB
- ✅ **Résoudre** des problèmes simples de manière autonome
- ✅ **Progresser** vers les parties plus avancées de la formation avec confiance

---

## 🎓 Parcours recommandés

Cette première partie est **essentielle et commune à tous les parcours** de formation :

| Parcours | Importance | Notes |
|----------|------------|-------|
| 🔧 **Développeur** | ⭐⭐⭐ Obligatoire | Fondation absolue pour l'intégration applicative |
| 🔐 **Administrateur/DBA** | ⭐⭐⭐ Obligatoire | Base nécessaire avant l'administration avancée |
| ⚙️ **DevOps/Cloud** | ⭐⭐⭐ Obligatoire | Prérequis pour l'automatisation et l'orchestration |
| 🤖 **IA/ML Engineer** | ⭐⭐⭐ Obligatoire | Fondamentaux avant MariaDB Vector et RAG |

💡 **Conseil** : même si vous êtes pressé d'arriver aux fonctionnalités avancées (Vector, Galera, Kubernetes…), **ne sautez pas cette partie**. Les concepts abordés ici sont la clé pour comprendre et maîtriser efficacement les chapitres suivants.

---

## Ce que MariaDB 12.3 LTS vous apporte

Dès vos débuts, vous profitez des atouts de la version de référence de cette formation, **MariaDB 12.3 LTS** (GA fin mai 2026, supportée jusqu'en **juin 2029**).

### 🎉 Du confort dès l'installation (acquis hérités de la 11.8)

- **Jeu de caractères `utf8mb4` par défaut** : plus besoin de configurer manuellement l'encodage — le support complet d'Unicode (collations UCA 14.0.0), émojis compris, est actif dès l'installation.
- **TLS/SSL en « zéro-configuration »** : la prise en charge du chiffrement des connexions est intégrée et prête à l'emploi, sans paramétrage manuel.
- **`TIMESTAMP` au-delà de 2038** : le problème de l'an 2038 (Y2038) est résolu, la plage s'étendant désormais jusqu'en **2106**.

### 🔧 Stabilité et support

- **LTS supportée 3 ans** : la 12.3 est maintenue jusqu'en **juin 2029** (la **11.8**, LTS précédente, l'est jusqu'en 2028).
- **Installation simplifiée** sur toutes les plateformes (Linux, Windows, macOS, Docker).
- **Documentation riche** et exemples accessibles aux débutants.

### 🚀 Des nouveautés 12.x que vous croiserez plus loin

Bien que cette partie se concentre sur les fondamentaux, MariaDB 12.3 apporte des fonctionnalités puissantes que vous découvrirez plus tard :
- **Binlog réécrit, intégré à InnoDB** : jusqu'à ~4× plus rapide en écriture (Partie 5).
- **Optimizer Hints** : contrôle fin du plan d'exécution des requêtes (Partie 7).
- **Compatibilité Oracle/MySQL renforcée** : `caching_sha2_password`, fonctions `TO_DATE`/`TO_CHAR`, type `XMLTYPE`… (Partie 10).
- **MariaDB Vector** : recherche vectorielle pour l'IA et le RAG (Partie 9).

💡 **Bonne nouvelle** : les exemples SQL de cette formation sont pensés et validés pour **MariaDB 12.3 LTS** — vous apprenez donc sur la dernière version stable et durablement supportée.

---

## 🚀 Prêt à commencer ?

Cette première partie est conçue pour être **progressive, encourageante et pratique**. N'hésitez pas à :

- ⏸️ **Prendre votre temps** : mieux vaut bien comprendre chaque concept que de se précipiter.
- 🔬 **Expérimenter** : testez tous les exemples SQL fournis, modifiez-les, cassez-les, réparez-les.
- 📖 **Consulter la documentation** : les liens vers la documentation officielle sont là pour approfondir.
- ❓ **Poser des questions** : la communauté MariaDB est accueillante et réactive.

Chaque section inclut :
- 🎯 des objectifs d'apprentissage clairs ;
- 📝 des explications progressives ;
- 💻 des exemples SQL commentés et testables ;
- 💡 des conseils pratiques ;
- ⚠️ des pièges courants à éviter ;
- ✅ un résumé des points clés.

**Bienvenue dans l'univers de MariaDB — votre aventure commence maintenant !** 🎉

---

## ➡️ Prochaine étape

**Chapitre 1 : Introduction et Fondamentaux** → commencez par la section 1.1 « Qu'est-ce que MariaDB ? » pour découvrir les bases de ce puissant système de gestion de bases de données.

---

**MariaDB** : Version 12.3 LTS (GA fin mai 2026, support jusqu'en juin 2029) — LTS précédente : 11.8

⏭️ [Introduction et Fondamentaux](/01-introduction-fondamentaux/README.md)
