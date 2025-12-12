🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4. Concepts Avancés SQL

> **Niveau** : Avancé
> **Durée estimée** : 12-15 heures
> **Prérequis** : Maîtrise des requêtes SQL intermédiaires (jointures, agrégations, sous-requêtes)

## 🎯 Objectifs d'apprentissage

À l'issue de ce chapitre, vous serez capable de :

- Maîtriser les **Window Functions** pour effectuer des analyses complexes (classements, moyennes mobiles, cumuls)
- Utiliser les **CTE (Common Table Expressions)** pour structurer des requêtes complexes de manière lisible
- Construire des **requêtes récursives** avec `WITH RECURSIVE` pour traiter des données hiérarchiques
- Exploiter pleinement le **type de données JSON** de MariaDB avec les fonctions avancées
- Utiliser les **nouveautés 11.8** : JSON Path Expressions avancées et JSON Schema Validation 🆕
- Indexer efficacement les colonnes virtuelles extraites de JSON
- Appliquer ces concepts à des cas d'usage production réels

---

## 📖 Introduction

Le SQL ne se limite pas aux simples `SELECT`, `JOIN` et `GROUP BY`. MariaDB offre un ensemble riche de fonctionnalités avancées qui permettent de résoudre des problèmes complexes de manière élégante et performante, directement au niveau de la base de données.

### Pourquoi ces concepts sont-ils importants ?

**1. Performance et efficacité**
- Traiter les données là où elles résident (dans la base) est généralement plus rapide que de les ramener dans l'application
- Réduire les allers-retours réseau et la charge applicative
- Exploiter les optimisations du moteur de base de données

**2. Expressivité et maintenabilité**
- Exprimer des logiques métier complexes de manière déclarative
- Produire du code SQL plus lisible et maintenable
- Remplacer des boucles applicatives par des opérations ensemblistes

**3. Analyse de données avancée**
- Effectuer des analyses sophistiquées : classements, tendances, cumuls
- Traiter des structures hiérarchiques (organigrammes, catégories)
- Manipuler des données semi-structurées (JSON)

### Évolution du SQL moderne

Le SQL a considérablement évolué depuis ses débuts. Les fonctionnalités que nous allons explorer ici font partie du **SQL moderne** et sont standardisées (SQL:2003, SQL:2011, SQL:2016). MariaDB implémente ces standards et va même au-delà avec des extensions propriétaires utiles.

### MariaDB 11.8 : JSON encore plus puissant 🆕

Avec la version 11.8 LTS, MariaDB renforce significativement ses capacités JSON :
- **JSON Path Expressions avancées** : Requêtes plus expressives sur les documents JSON
- **JSON Schema Validation** : Validation structurelle des données JSON au niveau base de données
- Performances optimisées pour les opérations JSON courantes

---

## 📚 Contenu du chapitre

Ce chapitre est organisé en 11 sections progressives :

### 🔄 Requêtes récursives
**4.1 Requêtes récursives (WITH RECURSIVE)**
- Parcourir des hiérarchies (arbres, graphes)
- Cas d'usage : organigrammes, catégories de produits, réseaux sociaux

### 📊 Window Functions : Analyses avancées
**4.2 Window Functions**
- Introduction et concepts fondamentaux
- **4.2.1 Fonctions de rang** : `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`
- **4.2.2 Fonctions de valeur** : `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`
- **4.2.3 Frames de fenêtre** : `ROWS`, `RANGE`, `GROUPS`
- **4.2.4 Cas d'usage pratiques** : Top N, moyennes mobiles, cumuls

💡 **Les Window Functions sont essentielles pour** :
- Calculer des classements (top 10, rang dans une catégorie)
- Analyser des tendances (évolution mois par mois)
- Comparer avec les valeurs précédentes/suivantes
- Calculer des agrégations cumulées

### 🔀 Transformations de données
**4.3 Requêtes pivotées et transformations**
- Transformer lignes en colonnes et vice-versa
- Techniques PIVOT/UNPIVOT dans MariaDB

### 🏗️ CTE : Structure et lisibilité
**4.4 Expressions de table communes (CTE)**
- Simplifier les requêtes complexes
- Réutiliser des sous-requêtes nommées
- Améliorer la lisibilité et la maintenance

💡 **Avantage majeur** : Une CTE peut être référencée plusieurs fois dans la même requête, contrairement à une sous-requête.

### 🔗 Requêtes multi-tables avancées
**4.5 Requêtes complexes multi-tables**
- Combiner plusieurs CTE
- Jointures complexes et sous-requêtes imbriquées
- Stratégies d'optimisation

### ⚠️ Gestion des NULL
**4.6 Gestion des valeurs NULL : Logique ternaire**
- Comprendre la logique ternaire SQL (TRUE/FALSE/NULL)
- Pièges courants et bonnes pratiques
- Fonctions `IFNULL`, `COALESCE`, `NULLIF`

### 📄 JSON : Stockage flexible
**4.7 JSON dans MariaDB**
- **4.7.1 Stockage et type de données JSON** : Avantages et limitations
- **4.7.2 Fonctions JSON** : `JSON_EXTRACT`, `JSON_SET`, `JSON_ARRAY`, `JSON_OBJECT`, etc.
- **4.7.3 Opérateur raccourci** : `->>` pour extraire facilement des valeurs

### 🆕 Nouveautés JSON 11.8
**4.8 JSON Path Expressions avancées** 🆕
- Syntaxe des chemins JSON complexes
- Filtres et prédicats
- Expressions multiples et wildcards

**4.9 JSON Schema Validation** 🆕
- Définir des schémas JSON
- Contraintes de validation au niveau base
- Cas d'usage : APIs, données semi-structurées

### 🚀 Performance JSON
**4.10 Indexation de colonnes virtuelles extraites du JSON**
- Créer des index sur des fragments JSON
- Optimiser les requêtes JSON fréquentes
- Colonnes générées STORED vs VIRTUAL

### 🔍 Expressions régulières
**4.11 Expressions régulières (REGEXP, REGEXP_REPLACE, REGEXP_SUBSTR)**
- Recherche avancée dans les chaînes
- Extraction et remplacement avec motifs
- Cas d'usage : validation, parsing, nettoyage

---

## 🎓 Progression pédagogique

```
Niveau 1 : Fondations
├─ Requêtes récursives (4.1)
└─ Gestion NULL (4.6)

Niveau 2 : Analyse de données
├─ Window Functions (4.2)
├─ CTE (4.4)
└─ Requêtes multi-tables (4.5)

Niveau 3 : Données semi-structurées
├─ JSON de base (4.7)
├─ JSON Path avancé (4.8) 🆕
├─ JSON Schema (4.9) 🆕
└─ Indexation JSON (4.10)

Niveau 4 : Outils spécialisés
├─ Pivots (4.3)
└─ Regex (4.11)
```

---

## 💡 Quand utiliser ces concepts ?

### Window Functions 👉 Utilisez quand vous devez :
- Calculer un classement ou un rang
- Comparer chaque ligne avec ses voisines (LAG/LEAD)
- Calculer des moyennes mobiles ou cumuls
- Faire des analyses "Top N par groupe"

### CTE 👉 Utilisez quand vous avez :
- Des requêtes complexes difficiles à lire
- Besoin de réutiliser un résultat intermédiaire
- Des logiques métier à isoler pour la clarté

### Requêtes récursives 👉 Utilisez pour :
- Parcourir des hiérarchies (arbre de catégories, organigramme)
- Générer des séries numériques ou de dates
- Résoudre des problèmes de graphes

### JSON 👉 Utilisez pour :
- Stocker des données flexibles (configuration, métadonnées)
- Intégrer avec des APIs REST
- Données semi-structurées variables
- ⚠️ **Mais attention** : N'abusez pas du JSON ! Le relationnel reste optimal pour les données structurées.

---

## ⚠️ Points d'attention importants

### Performance
- **Window Functions** : Peuvent être gourmandes en mémoire pour de gros datasets
- **Requêtes récursives** : Attention aux boucles infinies ! Utilisez toujours une condition de terminaison
- **JSON** : Plus lent que les colonnes natives pour les recherches fréquentes → Utilisez les colonnes virtuelles indexées

### Bonnes pratiques
- **Commencez simple** : Écrivez d'abord la requête de base, puis ajoutez la complexité
- **Nommez clairement vos CTE** : Un bon nom vaut mieux qu'un commentaire
- **Testez avec des petits datasets** : Validez la logique avant de passer à l'échelle
- **Utilisez EXPLAIN** : Vérifiez toujours le plan d'exécution

### Limites MariaDB
- Les Window Functions ne supportent pas toutes les clauses du standard SQL
- Le JSON n'est pas un type binaire comme dans PostgreSQL (stocké comme LONGTEXT)
- Les requêtes récursives ont une limite de profondeur configurable

---

## 📊 Cas d'usage réels par secteur

### 🏪 E-commerce
- **Window Functions** : Top 10 produits par catégorie, analyse de cohortes clients
- **Récursif** : Arbre de catégories de produits
- **JSON** : Attributs variables de produits (tailles, couleurs, specs techniques)

### 💰 Finance
- **Window Functions** : Moyennes mobiles de cours boursiers, cumuls de transactions
- **CTE** : Calculs d'agrégations multiples (soldes, intérêts, commissions)
- **JSON** : Métadonnées de transactions, configurations de produits financiers

### 👥 RH / Organigramme
- **Récursif** : Hiérarchie managériale complète
- **Window Functions** : Analyse de salaires par département avec percentiles
- **CTE** : Rapports RH complexes multi-niveaux

### 📱 SaaS / Applications web
- **JSON** : Préférences utilisateurs, paramètres d'applications
- **Window Functions** : Analytics, dashboards, métriques d'usage
- **CTE** : Rapports personnalisés pour chaque tenant

---

## ✅ Points clés à retenir

- 🎯 **Window Functions** = Analyses avancées sans GROUP BY, accès aux lignes adjacentes
- 🏗️ **CTE** = Clarté et réutilisabilité, "vues temporaires" dans une requête
- 🔄 **WITH RECURSIVE** = Parcours de hiérarchies et graphes
- 📄 **JSON** = Flexibilité pour données semi-structurées, mais avec des compromis de performance
- 🆕 **11.8** = JSON Path avancé + Schema Validation pour plus de puissance
- 📊 **Indexation JSON** = Clé pour de bonnes performances sur les requêtes JSON fréquentes
- 🔍 **Regex** = Manipulation textuelle avancée directement en SQL

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 Window Functions](https://mariadb.com/kb/en/window-functions/)
- [📖 Common Table Expressions](https://mariadb.com/kb/en/common-table-expressions/)
- [📖 Recursive CTE](https://mariadb.com/kb/en/recursive-common-table-expressions-overview/)
- [📖 JSON Functions](https://mariadb.com/kb/en/json-functions/)
- [📖 Generated Columns](https://mariadb.com/kb/en/generated-columns/)

### Nouveautés 11.8 🆕
- [📖 JSON Path Expressions](https://mariadb.com/kb/en/json-path-expressions/) - Documentation mise à jour 11.8
- [📖 JSON Schema Validation](https://mariadb.com/kb/en/json_schema/) - Nouvelle fonctionnalité 11.8

### Standards SQL
- [SQL:2003](https://en.wikipedia.org/wiki/SQL:2003) - Introduction des Window Functions
- [SQL:2011](https://en.wikipedia.org/wiki/SQL:2011) - Extensions temporelles
- [SQL:2016](https://en.wikipedia.org/wiki/SQL:2016) - JSON support

### Articles recommandés
- [Modern SQL - Window Functions](https://modern-sql.com/feature/over) - Excellente ressource pédagogique
- [Use The Index, Luke - Ranking](https://use-the-index-luke.com/sql/partial-results/window-functions) - Perspective performance

---

## 🎯 Approche d'apprentissage recommandée

### Pour les développeurs
**Parcours suggéré** : 4.2 → 4.4 → 4.7 → 4.10 → 4.1 → 4.8 🆕
- Concentrez-vous d'abord sur Window Functions et CTE (très utilisés)
- Puis JSON pour les applications modernes
- Les requêtes récursives et regex selon vos besoins spécifiques

### Pour les data analysts
**Parcours suggéré** : 4.2 → 4.4 → 4.5 → 4.3
- Window Functions = votre outil principal pour l'analyse
- CTE pour structurer des rapports complexes
- Pivots pour présentation des données

### Pour les architectes
**Parcours complet recommandé** : 4.1 → 4.11
- Vision complète des capacités SQL de MariaDB
- Comprendre les compromis de chaque approche
- Évaluer les meilleures solutions pour vos cas d'usage

---

## ⏭️ Structure du chapitre

Chaque section suivante contiendra :
- 📖 **Explications théoriques** progressives
- 💻 **Exemples SQL** commentés et testables
- 🏢 **Cas d'usage réels** en production
- ⚡ **Considérations de performance**
- 💡 **Best practices** et pièges à éviter
- 🆕 **Nouveautés 11.8** quand applicable

---

## ➡️ Section suivante

**[4.1 Requêtes récursives (WITH RECURSIVE)](./01-requetes-recursives.md)** : Découvrez comment parcourir des structures hiérarchiques avec élégance, résoudre des problèmes de graphes, et maîtriser l'une des fonctionnalités les plus puissantes du SQL moderne.

---

**💡 Conseil avant de commencer** : Ayez un environnement MariaDB 11.8 (ou 11.4+) prêt pour tester les exemples. Créez-vous quelques tables de test avec des données représentatives de vos cas d'usage. L'apprentissage par la pratique est la clé pour maîtriser ces concepts avancés !

---


⏭️ [Requêtes récursives (WITH RECURSIVE)](/04-concepts-avances-sql/01-requetes-recursives.md)
