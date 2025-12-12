🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 3 : Requêtes SQL Intermédiaires

> **Niveau** : Intermédiaire
> **Durée estimée** : 8-12 heures
> **Prérequis** : Chapitre 2 (Bases du SQL), maîtrise des requêtes SELECT simples, WHERE, ORDER BY, LIMIT

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :
- Utiliser les fonctions d'agrégation pour calculer des statistiques sur vos données
- Regrouper et filtrer des résultats avec GROUP BY et HAVING
- Maîtriser tous les types de jointures (INNER, LEFT, RIGHT, CROSS, Self-Join)
- Écrire des sous-requêtes et requêtes imbriquées complexes
- Combiner des résultats avec les opérateurs ensemblistes (UNION, INTERSECT, EXCEPT)
- Manipuler efficacement les chaînes de caractères et les dates
- Utiliser les expressions conditionnelles pour enrichir vos requêtes

---

## Introduction

Les **requêtes SQL intermédiaires** constituent une étape cruciale dans votre progression vers la maîtrise de MariaDB. Alors que les requêtes de base permettent de sélectionner, filtrer et trier des données simples, ce chapitre vous ouvre les portes de l'**analyse de données** et des **relations complexes** entre tables.

### Pourquoi ce niveau est important ?

En environnement de production, la majorité des requêtes métier nécessitent :
- **L'agrégation de données** : Calculer des totaux, moyennes, compter des occurrences
- **Le regroupement** : Analyser les données par catégories, périodes, segments
- **Les jointures** : Croiser les informations de plusieurs tables pour obtenir une vue cohérente
- **Les transformations** : Formater, nettoyer et enrichir les données à la volée

Ces compétences sont indispensables pour tout développeur travaillant avec des bases de données relationnelles, et constituent la base des requêtes d'analyse métier.

### Ce que vous allez apprendre

Ce chapitre est structuré en **8 sections progressives** qui s'appuient les unes sur les autres :

#### 📊 **Section 3.1 : Fonctions d'agrégation**
Les cinq fonctions essentielles (COUNT, SUM, AVG, MIN, MAX) pour transformer des ensembles de lignes en valeurs synthétiques. Vous apprendrez à calculer des statistiques simples et à comprendre comment MariaDB traite les valeurs NULL dans les agrégations.

#### 📦 **Section 3.2 : Regroupement de données**
GROUP BY et HAVING vous permettent de segmenter vos analyses. Vous découvrirez la différence fondamentale entre WHERE (filtre avant regroupement) et HAVING (filtre après regroupement), ainsi que les règles de composition des clauses GROUP BY.

#### 🔗 **Section 3.3 : Jointures**
Le cœur du modèle relationnel ! Cette section détaillée couvre :
- **INNER JOIN** : L'intersection, pour ne garder que les correspondances
- **LEFT/RIGHT JOIN** : Les jointures externes pour préserver toutes les lignes d'un côté
- **CROSS JOIN** : Le produit cartésien pour toutes les combinaisons possibles
- **Self-Join** : La technique avancée pour joindre une table à elle-même

Chaque type est illustré avec des schémas conceptuels et des cas d'usage réels.

#### 🎯 **Section 3.4 : Sous-requêtes**
Les requêtes imbriquées permettent de résoudre des problèmes complexes en plusieurs étapes logiques. Vous maîtriserez les sous-requêtes scalaires, les sous-requêtes dans les clauses WHERE et FROM, ainsi que les opérateurs IN, EXISTS, ANY et ALL.

#### ⚡ **Section 3.5 : Opérateurs ensemblistes**
UNION, INTERSECT et EXCEPT (MariaDB 10.3+) permettent de combiner les résultats de plusieurs requêtes. Vous comprendrez quand utiliser UNION vs UNION ALL, et comment ces opérateurs diffèrent des jointures.

#### 🔤 **Section 3.6 : Fonctions de chaînes**
Manipulation de texte avec CONCAT, SUBSTRING, REPLACE, UPPER, LOWER, TRIM, LENGTH, et bien d'autres. Ces fonctions sont essentielles pour le nettoyage et la transformation de données textuelles.

#### 📅 **Section 3.7 : Fonctions de dates**
Travailler avec les types temporels : extraction de parties de dates (YEAR, MONTH, DAY), calculs d'intervalles (DATE_ADD, DATE_SUB, DATEDIFF), formatage (DATE_FORMAT), et conversions. Indispensable pour les analyses temporelles.

#### 🎭 **Section 3.8 : Expressions conditionnelles**
CASE, IF, IFNULL, COALESCE et NULLIF permettent d'ajouter de la logique conditionnelle directement dans vos requêtes SQL, pour créer des colonnes calculées complexes et gérer élégamment les valeurs NULL.

---

## 🎓 Approche pédagogique de ce chapitre

### Progression graduelle
Chaque section introduit des concepts en partant du plus simple pour aller vers le plus complexe. Les exemples s'appuient sur un **schéma de base de données cohérent** utilisé tout au long du chapitre.

### Schéma de référence : E-commerce

Nous utiliserons un schéma simplifié de commerce en ligne pour tous les exemples :

```sql
-- Clients
CREATE TABLE clients (
    id_client INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE,
    ville VARCHAR(100),
    pays VARCHAR(50),
    date_inscription DATE
);

-- Produits
CREATE TABLE produits (
    id_produit INT PRIMARY KEY AUTO_INCREMENT,
    nom_produit VARCHAR(200) NOT NULL,
    categorie VARCHAR(50),
    prix_unitaire DECIMAL(10, 2),
    stock INT DEFAULT 0
);

-- Commandes
CREATE TABLE commandes (
    id_commande INT PRIMARY KEY AUTO_INCREMENT,
    id_client INT NOT NULL,
    date_commande DATETIME DEFAULT CURRENT_TIMESTAMP,
    statut ENUM('en_attente', 'confirmée', 'expédiée', 'livrée', 'annulée'),
    montant_total DECIMAL(10, 2),
    FOREIGN KEY (id_client) REFERENCES clients(id_client)
);

-- Détails commandes (table de liaison)
CREATE TABLE details_commande (
    id_detail INT PRIMARY KEY AUTO_INCREMENT,
    id_commande INT NOT NULL,
    id_produit INT NOT NULL,
    quantite INT NOT NULL,
    prix_unitaire DECIMAL(10, 2),
    FOREIGN KEY (id_commande) REFERENCES commandes(id_commande),
    FOREIGN KEY (id_produit) REFERENCES produits(id_produit)
);
```

Ce schéma représente une situation réelle et permet d'illustrer naturellement les jointures et les agrégations.

### Exemples commentés
Chaque requête est accompagnée de :
- **Commentaires SQL** expliquant chaque partie
- **Résultat attendu** sous forme de tableau
- **Explication du "pourquoi"** : quand utiliser cette approche
- **Points d'attention** : pièges courants et optimisations

### Cas d'usage métier
Les exemples sont tirés de situations concrètes :
- Calculer le chiffre d'affaires par catégorie
- Trouver les clients sans commande
- Analyser les ventes par période
- Identifier les produits en rupture de stock
- Détecter les anomalies dans les données

---

## 💡 Conseils pour tirer le meilleur parti de ce chapitre

### 1. Suivez l'ordre des sections
Les concepts s'appuient les uns sur les autres. GROUP BY utilise les fonctions d'agrégation, les jointures sont essentielles pour les sous-requêtes complexes, etc.

### 2. Testez les exemples
Bien que ce cours ne contienne pas d'exercices formels, nous vous encourageons vivement à :
- Créer les tables d'exemple dans votre environnement local
- Exécuter chaque requête présentée
- Modifier légèrement les exemples pour observer le comportement

💡 **Astuce** : Utilisez une base de données de test dédiée :
```sql
CREATE DATABASE formation_mariadb_intermediaire;
USE formation_mariadb_intermediaire;
```

### 3. Utilisez EXPLAIN
Pour chaque requête complexe (surtout les jointures), prenez l'habitude d'analyser le plan d'exécution :
```sql
EXPLAIN SELECT ... FROM ... JOIN ...;
```
Cela vous aidera à comprendre comment MariaDB traite votre requête.

### 4. Expérimentez avec vos propres données
Une fois les concepts maîtrisés avec nos exemples, appliquez-les à vos propres tables et cas d'usage. C'est ainsi que vous ancrerez durablement les connaissances.

### 5. Consultez la documentation officielle
Chaque section référence la documentation MariaDB. N'hésitez pas à l'explorer pour approfondir un point particulier.

---

## ⚠️ Prérequis techniques

Avant de commencer ce chapitre, assurez-vous de :

- ✅ Avoir accès à une instance MariaDB 11.4+ (idéalement 11.8 LTS)
- ✅ Être capable de vous connecter avec le client `mariadb` ou un outil graphique (HeidiSQL, DBeaver)
- ✅ Maîtriser les commandes de base :
    - `CREATE DATABASE`, `USE`
    - `CREATE TABLE`, `INSERT INTO`
    - `SELECT ... FROM ... WHERE ... ORDER BY ... LIMIT`
- ✅ Comprendre les concepts de clés primaires et étrangères
- ✅ Connaître les types de données de base (INT, VARCHAR, DATE, DECIMAL)

---

## 🗺️ Plan du chapitre

| Section | Titre | Concepts clés |
|---------|-------|---------------|
| **3.1** | Fonctions d'agrégation | COUNT, SUM, AVG, MIN, MAX, gestion des NULL |
| **3.2** | Regroupement de données | GROUP BY, HAVING, WHERE vs HAVING |
| **3.3** | Jointures | INNER, LEFT, RIGHT, CROSS, Self-Join |
| **3.4** | Sous-requêtes | Scalaires, IN, EXISTS, ANY, ALL, FROM |
| **3.5** | Opérateurs ensemblistes | UNION, UNION ALL, INTERSECT, EXCEPT |
| **3.6** | Fonctions de chaînes | CONCAT, SUBSTRING, REPLACE, TRIM, etc. |
| **3.7** | Fonctions de dates | DATE_ADD, DATEDIFF, DATE_FORMAT, extraction |
| **3.8** | Expressions conditionnelles | CASE, IF, IFNULL, COALESCE, NULLIF |

---

## 🎯 Compétences visées en fin de chapitre

Après avoir complété ce chapitre, vous serez en mesure de :

### Analyse de données
- ✅ Calculer des statistiques agrégées (totaux, moyennes, comptages)
- ✅ Segmenter les analyses par catégories avec GROUP BY
- ✅ Filtrer les résultats agrégés avec HAVING
- ✅ Réaliser des analyses de cohortes et de tendances

### Jointures de tables
- ✅ Choisir le type de jointure approprié selon le besoin métier
- ✅ Comprendre l'impact des jointures sur les performances
- ✅ Joindre efficacement 3, 4 tables ou plus
- ✅ Utiliser les self-joins pour des hiérarchies ou comparaisons

### Requêtes complexes
- ✅ Écrire des sous-requêtes dans WHERE, FROM et SELECT
- ✅ Combiner des résultats avec UNION et ses variantes
- ✅ Transformer et nettoyer des données avec les fonctions SQL
- ✅ Ajouter de la logique conditionnelle dans vos SELECT

### Bonnes pratiques
- ✅ Écrire des requêtes lisibles et maintenables
- ✅ Anticiper les problèmes de performance
- ✅ Gérer correctement les valeurs NULL
- ✅ Documenter les requêtes complexes

---

## 📊 Différences avec le niveau débutant

| Aspect | Niveau Débutant (Chapitre 2) | Niveau Intermédiaire (Chapitre 3) |
|--------|------------------------------|-----------------------------------|
| **Requêtes** | Une seule table | Plusieurs tables jointes |
| **Filtrage** | WHERE simple | WHERE + HAVING, sous-requêtes |
| **Résultats** | Lignes individuelles | Données agrégées et groupées |
| **Logique** | Conditions simples | Expressions conditionnelles complexes |
| **Transformations** | Aucune ou minimales | Fonctions de chaînes, dates, calculs |
| **Complexité** | Requêtes < 5 lignes | Requêtes 10-30 lignes courantes |

---

## 🔗 Liens avec les autres chapitres

### En amont
- **Chapitre 1** : Architecture MariaDB (pour comprendre le traitement des requêtes)
- **Chapitre 2** : Bases du SQL (fondations indispensables)

### En aval
- **Chapitre 4** : Concepts Avancés SQL (Window Functions, CTE, JSON)
- **Chapitre 5** : Index et Performance (optimisation des requêtes complexes)
- **Chapitre 6** : Transactions (cohérence lors de requêtes multi-tables)

---

## 💼 Applications pratiques

Les compétences de ce chapitre sont utilisées quotidiennement pour :

### Développement d'applications
- Endpoints API retournant des données agrégées
- Dashboards avec statistiques en temps réel
- Rapports utilisateurs personnalisés
- Systèmes de recommandation basés sur l'historique

### Analyse métier
- Reporting mensuel/trimestriel
- Calcul de KPI (taux de conversion, panier moyen, etc.)
- Segmentation client (RFM, cohortes)
- Analyse de tendances et prévisions

### Administration système
- Monitoring de la base de données
- Détection d'anomalies dans les données
- Audit et conformité
- Nettoyage et migration de données

---

## ⚡ Performance : Un premier aperçu

Bien que le **Chapitre 5** soit dédié à la performance, voici quelques principes à garder en tête dès maintenant :

### Jointures
- Les jointures avec index sur les colonnes de jointure sont **très rapides**
- Sans index, les jointures deviennent **coûteuses** (scan complet de tables)
- Ordre des tables dans la requête peut avoir un impact

### GROUP BY
- L'utilisation d'index sur les colonnes GROUP BY **améliore drastiquement** les performances
- Les GROUP BY sur colonnes non indexées nécessitent un tri temporaire (filesort)

### Sous-requêtes
- Les sous-requêtes corrélées (exécutées pour chaque ligne) peuvent être **lentes**
- Souvent, une jointure est plus performante qu'une sous-requête

💡 **Bonne pratique** : Utilisez systématiquement `EXPLAIN` pour comprendre l'exécution de vos requêtes complexes.

---

## 🆕 Nouveautés MariaDB 11.8 LTS pertinentes

Bien que ce chapitre couvre des concepts SQL standards, MariaDB 11.8 apporte quelques améliorations :

- **Optimiseur amélioré** : Meilleure gestion des jointures complexes
- **JSON Path Expressions** : Manipulation JSON dans les requêtes (Section 4.7+)
- **Fonctions de fenêtre optimisées** : Préparation pour le Chapitre 4
- **Charset UTF8MB4 par défaut** : Gestion native des emojis et caractères internationaux
- **Cost-based optimizer SSD** : Meilleures estimations de coût pour les requêtes

Ces améliorations rendent vos requêtes intermédiaires **plus rapides et plus fiables** qu'avec les versions antérieures.

---

## ✅ Points clés à retenir

Avant de plonger dans les sections détaillées, retenez ces principes fondamentaux :

1. **Les agrégations transforment des ensembles de lignes en valeurs uniques** – COUNT, SUM, AVG sont vos outils d'analyse essentiels

2. **GROUP BY segmente vos données avant agrégation** – Pensez "catégories", "périodes", "segments"

3. **HAVING filtre après agrégation, WHERE filtre avant** – Distinction cruciale pour obtenir les résultats attendus

4. **Les jointures sont au cœur du modèle relationnel** – Maîtrisez-les pour exploiter pleinement votre base de données

5. **INNER JOIN ne garde que les correspondances** – Pour les correspondances complètes entre tables

6. **LEFT/RIGHT JOIN préservent toutes les lignes d'un côté** – Pour détecter les absences et les orphelins

7. **Les sous-requêtes permettent de résoudre des problèmes en étapes** – Diviser pour mieux régner

8. **NULL nécessite une attention particulière** – Les fonctions et opérateurs se comportent différemment avec NULL

9. **La lisibilité est aussi importante que la performance** – Commentez vos requêtes complexes, utilisez des alias explicites

10. **EXPLAIN est votre ami** – Prenez l'habitude d'analyser vos requêtes avant de les mettre en production

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Aggregate Functions](https://mariadb.com/kb/en/aggregate-functions/)
- [📖 JOIN Syntax](https://mariadb.com/kb/en/join-syntax/)
- [📖 Subqueries](https://mariadb.com/kb/en/subqueries/)
- [📖 String Functions](https://mariadb.com/kb/en/string-functions/)
- [📖 Date and Time Functions](https://mariadb.com/kb/en/date-time-functions/)

### Outils pratiques
- [SQLFiddle](http://sqlfiddle.com/) : Tester vos requêtes en ligne
- [DB Fiddle](https://www.db-fiddle.com/) : Alternative moderne
- [EXPLAIN Analyzer](https://mariadb.org/explain-analyzer/) : Visualiser les plans d'exécution

### Lectures complémentaires
- SQL Performance Explained (Markus Winand) – Chapitres sur les jointures
- High Performance MySQL (Baron Schwartz) – S'applique aussi à MariaDB

---

## ➡️ Section suivante

**[3.1 Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)](./01-fonctions-agregation.md)**

Dans la première section, nous démarrons avec les **cinq fonctions d'agrégation fondamentales** qui vous permettront de transformer des milliers de lignes en statistiques exploitables. Vous apprendrez comment MariaDB calcule les totaux, moyennes, minimums et maximums, et surtout comment gérer correctement les valeurs NULL dans vos calculs.

Ces fonctions sont la base de toute analyse de données – maîtrisez-les et vous débloquerez 80% des besoins de reporting !

---

**Bonne formation ! 🚀**

N'hésitez pas à prendre des notes, à tester les exemples dans votre environnement, et à consulter la documentation officielle pour approfondir les points qui vous intéressent particulièrement.

⏭️ [Fonctions d'agrégation (COUNT, SUM, AVG, MIN, MAX)](/03-requetes-sql-intermediaires/01-fonctions-agregation.md)
