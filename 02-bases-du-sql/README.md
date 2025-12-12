🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2. Bases du SQL

> **Niveau** : Débutant
> **Durée estimée** : 8-10 heures
> **Prérequis** : Avoir compris les concepts généraux d'un SGBD relationnel (chapitre 1)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Comprendre la syntaxe de base du langage SQL et ses différentes catégories (DDL, DML, DCL)
- Choisir les types de données appropriés pour vos colonnes selon vos besoins
- Créer et gérer des bases de données et des tables avec MariaDB
- Définir des contraintes pour garantir l'intégrité de vos données
- Insérer, consulter, modifier et supprimer des données avec les commandes SQL essentielles
- Utiliser les nouveaux types de données spécifiques à MariaDB 11.8 (JSON, UUID, VECTOR)

---

## 📖 Introduction

Le **SQL (Structured Query Language)** est le langage standard pour interagir avec les bases de données relationnelles. Avec MariaDB 11.8 LTS, vous disposez d'un dialecte SQL riche et moderne qui respecte les standards tout en offrant des extensions puissantes.

### Pourquoi apprendre le SQL avec MariaDB ?

MariaDB propose une implémentation complète du SQL standard, enrichie de fonctionnalités avancées :
- **Compatibilité** : Respecte les standards SQL-92, SQL:1999, SQL:2003, SQL:2016
- **Types modernes** : JSON natif, UUID, INET6, et **VECTOR** 🆕 pour l'IA
- **Performances** : Optimisations spécifiques pour les workloads modernes
- **Flexibilité** : Support de multiples moteurs de stockage avec des caractéristiques différentes

### Structure de ce chapitre

Ce chapitre vous accompagne progressivement dans l'apprentissage du SQL, de la création de votre première base de données jusqu'aux opérations CRUD (Create, Read, Update, Delete) complètes.

Nous couvrirons :

1. **Les fondamentaux du SQL** : Syntaxe, catégories de commandes (DDL, DML, DCL)
2. **Les types de données** : Comment choisir le bon type pour chaque colonne
3. **La gestion des bases** : Création, modification, suppression
4. **La conception de tables** : Structure, contraintes, bonnes pratiques
5. **Les opérations sur les données** : INSERT, SELECT, UPDATE, DELETE

---

## 🆕 Nouveautés MariaDB 11.8 à découvrir

Dans ce chapitre, vous découvrirez plusieurs nouveautés importantes :

- **utf8mb4 par défaut** : MariaDB 11.8 utilise désormais utf8mb4 comme charset par défaut avec la collation UCA 14.0.0, offrant un meilleur support Unicode
- **Type VECTOR** : Nouveau type de données pour le stockage d'embeddings et la recherche vectorielle (IA/ML)
- **Extension TIMESTAMP** : Support jusqu'en 2106 (résolution du problème Y2038)
- **Améliorations JSON** : Fonctions et validations enrichies

💡 **Note** : Même si ce chapitre s'adresse aux débutants, nous mentionnerons ces fonctionnalités avancées pour vous donner une vision complète de MariaDB.

---

## 🗺️ Plan du chapitre

### 2.1 Introduction au langage SQL
Présentation du SQL, ses catégories (DDL, DML, DCL, TCL), et la syntaxe de base.

### 2.2 Types de données MariaDB
Découverte exhaustive des types disponibles :
- **Numériques** : INT, BIGINT, DECIMAL, FLOAT, DOUBLE
- **Texte** : VARCHAR, TEXT, CHAR, ENUM, SET
- **Temporels** : DATE, DATETIME, TIMESTAMP, TIME, YEAR
- **Binaires** : BLOB, BINARY, VARBINARY
- **Spécifiques** : JSON, UUID, INET6, VECTOR 🆕

### 2.3 Création et gestion des bases de données
Commandes CREATE DATABASE, ALTER DATABASE, DROP DATABASE avec gestion des charsets.

### 2.4 Création et modification de tables
Syntaxe CREATE TABLE, ALTER TABLE, DROP TABLE avec exemples pratiques.

### 2.5 Contraintes d'intégrité
PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, DEFAULT, CHECK pour garantir la qualité des données.

### 2.6 Insertion de données
INSERT simple, INSERT multiple, INSERT INTO SELECT, et LOAD DATA pour l'import en masse.

### 2.7 Requêtes de sélection simples
SELECT, WHERE, ORDER BY, LIMIT : les bases de la consultation de données.

### 2.8 Mise à jour et suppression
UPDATE, DELETE, TRUNCATE avec leurs implications et bonnes pratiques.

---

## 🎓 Approche pédagogique

### Comment utiliser ce chapitre

Chaque section suit une progression logique :

1. **Concepts théoriques** : Explication claire et accessible
2. **Syntaxe SQL** : Présentation de la syntaxe officielle
3. **Exemples commentés** : Code SQL réel et exécutable
4. **Cas d'usage pratiques** : Scénarios du monde réel
5. **Points clés** : Résumé des éléments essentiels
6. **Références** : Documentation officielle pour approfondir

### 💡 Conseils pour bien apprendre

- **Testez les exemples** : Installez MariaDB et exécutez chaque exemple
- **Progressez étape par étape** : Ne sautez pas de section, chacune construit sur les précédentes
- **Expérimentez** : Modifiez les exemples pour comprendre leur fonctionnement
- **Consultez la documentation** : Les liens vers la doc officielle sont précieux
- **Prenez des notes** : Créez votre propre référence SQL

### ⚠️ Pièges courants à éviter

Nous vous alerterons régulièrement sur les erreurs fréquentes :
- Confusion entre types de données similaires (VARCHAR vs TEXT)
- Oubli de contraintes d'intégrité
- Mauvais choix de types numériques
- Problèmes de charset et collation
- Utilisation inappropriée de NULL

---

## 🛠️ Environnement de travail

### Prérequis techniques

Pour suivre ce chapitre, vous devez avoir :
- ✅ MariaDB 11.8 LTS installé (voir chapitre 1.8)
- ✅ Accès au client `mariadb` CLI ou à un outil graphique (HeidiSQL, DBeaver)
- ✅ Droits suffisants pour créer des bases et des tables

### Base de données exemple

Nous utiliserons une base de données d'exemple tout au long du chapitre :

```sql
-- Base de données pour les exemples du chapitre
CREATE DATABASE IF NOT EXISTS formation_mariadb
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE formation_mariadb;
```

Cette base servira de bac à sable pour tous nos exemples.

---

## 📊 Vue d'ensemble : Structure d'une base de données

Avant de plonger dans les détails, voici un aperçu de la hiérarchie des objets dans MariaDB :

```
Serveur MariaDB
    ├── Base de données 1 (Database/Schema)
    │   ├── Table 1
    │   │   ├── Colonnes (avec types de données)
    │   │   ├── Contraintes (PK, FK, UNIQUE, etc.)
    │   │   └── Index
    │   ├── Table 2
    │   ├── Vue 1
    │   ├── Procédure stockée 1
    │   └── Trigger 1
    ├── Base de données 2
    └── Utilisateurs et privilèges
```

Dans ce chapitre, nous nous concentrerons principalement sur :
- Les **bases de données**
- Les **tables** avec leurs colonnes et contraintes
- Les **types de données**
- Les **opérations CRUD** de base

---

## 🎯 Exemple fil rouge

Tout au long du chapitre, nous construirons progressivement un système simple de gestion de bibliothèque avec :

- Une table **`livres`** : catalogue des ouvrages
- Une table **`auteurs`** : informations sur les auteurs
- Une table **`emprunts`** : suivi des prêts

Cet exemple concret vous permettra de voir comment les concepts s'appliquent dans un contexte réel.

**Schéma relationnel simplifié :**

```
auteurs (id, nom, prenom, date_naissance, nationalite)
    └── 1:N
livres (id, titre, isbn, auteur_id, date_publication, prix)
    └── 1:N
emprunts (id, livre_id, nom_emprunteur, date_emprunt, date_retour)
```

---

## 📚 Ressources complémentaires

### Documentation officielle MariaDB
- [SQL Statements](https://mariadb.com/kb/en/sql-statements/)
- [Data Types](https://mariadb.com/kb/en/data-types/)
- [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-11-8-0-release-notes/)

### Standards SQL
- [SQL:2016 Standard](https://www.iso.org/standard/63555.html)
- [SQL Tutorial W3Schools](https://www.w3schools.com/sql/)

---

## ✅ Checklist avant de commencer

Avant de passer à la section 2.1, assurez-vous de :

- [ ] Avoir MariaDB 11.8 installé et fonctionnel
- [ ] Pouvoir vous connecter au serveur MariaDB
- [ ] Avoir créé la base `formation_mariadb` pour les exemples
- [ ] Avoir accès à la documentation officielle
- [ ] Être prêt à tester chaque exemple SQL

---

## ➡️ Section suivante

**[2.1 Introduction au langage SQL](/02-bases-du-sql/01-introduction-langage-sql.md)**

Découvrez les fondamentaux du SQL, sa syntaxe, et ses différentes catégories de commandes. Vous apprendrez à structurer vos premières requêtes et à comprendre la logique du langage SQL avec MariaDB.

---

**💬 Besoin d'aide ?**

- 📖 Consultez la [documentation officielle MariaDB](https://mariadb.com/kb/)
- 💬 Rejoignez le [forum communautaire](https://mariadb.com/kb/en/community/)
- 🐛 Signalez les bugs sur [JIRA](https://jira.mariadb.org/)

---


⏭️ [Introduction au langage SQL](/02-bases-du-sql/01-introduction-langage-sql.md)
