🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 2 : Requêtes SQL Intermédiaires et Avancées (Intermédiaire)

> **Niveau** : Intermédiaire  
> **Durée estimée** : 2-3 jours  
> **Prérequis** : Maîtrise des bases SQL (SELECT, JOIN, INSERT, UPDATE, DELETE), compréhension des contraintes et des types de données

---

## 🎯 Passez au niveau supérieur en SQL

Après avoir acquis les fondamentaux de SQL dans la Partie 1, vous êtes maintenant prêt à explorer les **techniques avancées de manipulation et d'analyse des données**. Cette deuxième partie vous permettra de résoudre des problèmes complexes avec élégance et efficacité.

L'objectif de cette partie est de vous transformer en **développeur SQL confirmé**, capable d'écrire des requêtes sophistiquées pour répondre à des besoins métier complexes : analyses temporelles, calculs de rangs, agrégations avancées, transformations de données, manipulation JSON, et bien plus encore.

Ces techniques ne sont pas de simples curiosités académiques — elles sont **utilisées quotidiennement en production** dans les applications modernes, les systèmes d'analyse, les pipelines de données et les architectures orientées API. La maîtrise de ces concepts vous distinguera en tant que professionnel capable de résoudre des problèmes que d'autres considèrent comme impossibles en SQL pur.

À l'issue de cette partie, vous aurez non seulement élargi votre palette technique, mais vous aurez également développé une **approche analytique et créative** pour résoudre des problèmes de données complexes directement dans la base de données, sans recourir systématiquement à du code applicatif.

---

## 📚 Les deux modules de cette partie

### Module 3 : Requêtes SQL Intermédiaires (Complément Avancé)
**8 sections | Durée : ~1 jour**

Ce module approfondit les techniques intermédiaires introduites en Partie 1 et les enrichit de patterns avancés :

- **Fonctions d'agrégation avancées** : Au-delà de `COUNT` et `SUM`, exploration des agrégations conditionnelles et des groupements multiples
- **Regroupements complexes** : `GROUP BY` avec `ROLLUP`, `CUBE` (via UNION), groupements hiérarchiques
- **Maîtrise complète des jointures** : Optimisation, jointures multiples, résolution de cas complexes
- **Sous-requêtes optimisées** : Corrélées vs non-corrélées, `EXISTS` vs `IN`, performance
- **Opérateurs ensemblistes avancés** : `UNION ALL`, `INTERSECT`, `EXCEPT` et leurs cas d'usage
- **Manipulation de chaînes avancée** : Parsing, transformation, extraction de données textuelles
- **Calculs temporels complexes** : Intervalles, fuseaux horaires, agrégations temporelles
- **Logique conditionnelle élaborée** : `CASE` imbriqués, expressions conditionnelles complexes

💡 **Point fort** : Des patterns SQL réutilisables qui vous feront gagner des heures de développement applicatif.

---

### Module 4 : Concepts Avancés SQL
**11 sections | Durée : ~2 jours**

Ce module vous introduit aux fonctionnalités SQL les plus puissantes et modernes :

#### 🔄 Requêtes récursives (WITH RECURSIVE)
Parcourir des hiérarchies, générer des séries numériques, résoudre des problèmes de graphes directement en SQL.

#### 📊 Window Functions (Fonctions de fenêtrage)
La fonctionnalité SQL la plus transformatrice des 20 dernières années :
- **Fonctions de rang** : `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()` — résoudre les problèmes de Top-N, pagination avancée
- **Fonctions de valeur** : `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()` — analyses temporelles, comparaisons ligne-à-ligne
- **Frames de fenêtre** : `ROWS`, `RANGE`, `GROUPS` — moyennes mobiles, cumuls, calculs glissants
- **Cas d'usage réels** : Tableaux de bord, reporting, analyses de cohortes

#### 🧩 Expressions de Table Communes (CTE)
Structurer des requêtes complexes de manière lisible et maintenable avec `WITH`.

#### 📦 Manipulation JSON avancée
MariaDB 11.8 offre des capacités JSON de niveau entreprise :
- **Stockage et indexation** : Type `JSON` optimisé, colonnes virtuelles indexées
- **Fonctions complètes** : `JSON_EXTRACT()`, `JSON_SET()`, `JSON_ARRAY()`, `JSON_OBJECT()`
- 🆕 **JSON Path Expressions avancées** : Requêtes complexes dans structures JSON imbriquées
- 🆕 **JSON Schema Validation** : Validation automatique de la structure des documents JSON
- **Opérateur raccourci `->>>`** : Syntaxe simplifiée pour l'extraction

#### 🔍 Expressions régulières
`REGEXP`, `REGEXP_REPLACE()`, `REGEXP_SUBSTR()` pour du pattern matching avancé.

💡 **Point fort** : Ces techniques réduisent drastiquement le code applicatif nécessaire et améliorent les performances en déplaçant la logique au plus près des données.

---

## 🆕 Nouveautés MariaDB 11.8 pour le SQL avancé

MariaDB 11.8 LTS introduit des améliorations majeures pour les développeurs SQL avancés :

### 📋 JSON Path Expressions avancées
```sql
-- Navigation profonde dans des structures JSON complexes
SELECT JSON_EXTRACT(data, '$.users[*].addresses[?(@.type=="primary")].city') 
FROM user_profiles;
```
Support complet des expressions de chemin JSON avec filtres conditionnels, permettant des requêtes complexes sur des documents JSON imbriqués.

### ✅ JSON Schema Validation
```sql
-- Validation automatique à l'insertion
CREATE TABLE api_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    payload JSON CHECK (JSON_SCHEMA_VALID('{
        "type": "object",
        "required": ["user_id", "action"],
        "properties": {
            "user_id": {"type": "number"},
            "action": {"type": "string"}
        }
    }', payload))
);
```
Garantissez l'intégrité structurelle de vos données JSON directement au niveau de la base de données.

### 🚀 Performance JSON améliorée
- Extraction JSON plus rapide de 30-40% grâce aux optimisations du parseur
- Indexation automatique des colonnes virtuelles JSON
- Support complet de l'opérateur `->>` pour une syntaxe plus concise

### 🔧 Améliorations des Window Functions
- Support complet des clauses `ROWS`, `RANGE`, et `GROUPS`
- Optimisation des plans d'exécution pour les partitions volumineuses
- Compatibilité étendue avec le standard SQL:2023

Ces nouveautés positionnent MariaDB 11.8 comme une base de données **moderne et adaptée aux architectures API-first et aux applications orientées documents**, tout en conservant la robustesse du modèle relationnel.

---

## ✅ Compétences acquises

À la fin de cette deuxième partie, vous serez capable de :

### Analyse de données avancée
- ✅ **Construire** des rapports analytiques complexes avec Window Functions
- ✅ **Calculer** des moyennes mobiles, cumuls, et métriques glissantes
- ✅ **Résoudre** des problèmes de classement (Top-N, pagination, quartiles)
- ✅ **Analyser** des tendances temporelles et des séries chronologiques
- ✅ **Comparer** des données ligne-à-ligne avec `LAG()` et `LEAD()`

### Manipulation de données complexes
- ✅ **Parcourir** des structures hiérarchiques avec requêtes récursives
- ✅ **Transformer** des données avec pivots et dépivots
- ✅ **Manipuler** des documents JSON de manière native
- ✅ **Valider** la structure de données JSON avec schemas
- ✅ **Extraire** des informations avec expressions régulières

### Optimisation et architecture
- ✅ **Structurer** des requêtes complexes avec CTE pour la lisibilité
- ✅ **Choisir** entre sous-requêtes et jointures pour la performance
- ✅ **Indexer** efficacement des colonnes JSON virtuelles
- ✅ **Réduire** le code applicatif en déplaçant la logique vers SQL

### Résolution de problèmes métier
- ✅ **Implémenter** des tableaux de bord analytiques performants
- ✅ **Gérer** des données semi-structurées (logs, APIs, événements)
- ✅ **Créer** des vues matérialisées virtuelles avec CTE
- ✅ **Répondre** à des questions métier complexes avec élégance

---

## 🎓 Parcours recommandés

| Parcours | Importance | Justification |
|----------|------------|---------------|
| 🔧 **Développeur** | ⭐⭐⭐ Essentiel | Les Window Functions et JSON sont au cœur du développement moderne. Indispensable pour créer des APIs performantes et des tableaux de bord. |
| 🤖 **IA/ML Engineer** | ⭐⭐⭐ Essentiel | JSON et requêtes analytiques sont cruciaux pour la préparation des données, les features engineering, et l'intégration avec les pipelines ML. |
| 🔐 **Administrateur/DBA** | ⭐⭐ Recommandé | Comprendre les requêtes complexes aide à diagnostiquer les problèmes de performance et à optimiser les index. Module 4 particulièrement utile. |
| ⚙️ **DevOps/Cloud** | ⭐⭐ Utile | Les techniques SQL avancées facilitent la création de dashboards de monitoring et l'analyse de logs structurés (JSON). |

💡 **Note importante** : Si vous êtes développeur ou travaillez avec des données analytiques, **cette partie est critique**. Les Window Functions à elles seules peuvent réduire de 80% le code nécessaire pour certaines fonctionnalités.

---

## 🏢 Cas d'usage réels

Voici des exemples concrets où les techniques de cette partie sont **indispensables** :

### 📊 Tableaux de bord et reporting
```sql
-- Classement des produits par ventes avec évolution mois-sur-mois
WITH monthly_sales AS (
    SELECT 
        product_id,
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        SUM(amount) AS total_sales
    FROM orders
    GROUP BY product_id, month
)
SELECT 
    product_id,
    month,
    total_sales,
    LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month) AS previous_month,
    total_sales - LAG(total_sales) OVER (PARTITION BY product_id ORDER BY month) AS growth,
    RANK() OVER (PARTITION BY month ORDER BY total_sales DESC) AS rank_in_month
FROM monthly_sales;
```

### 🌳 Gestion de hiérarchies organisationnelles
```sql
-- Parcourir un organigramme complet avec requête récursive
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS level, CAST(name AS CHAR(200)) AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    SELECT e.id, e.name, e.manager_id, oc.level + 1, CONCAT(oc.path, ' > ', e.name)
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY path;
```

### 📱 APIs modernes avec JSON
```sql
-- Stockage et requêtage de profils utilisateurs complexes
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    profile JSON,
    -- Colonnes virtuelles pour indexation
    email VARCHAR(255) AS (profile->>'$.contact.email') STORED,
    city VARCHAR(100) AS (profile->>'$.address.city') STORED,
    INDEX idx_email (email),
    INDEX idx_city (city)
);

-- Requête avec validation et extraction
INSERT INTO user_profiles (user_id, profile) VALUES (
    1, 
    JSON_OBJECT(
        'name', 'Alice',
        'contact', JSON_OBJECT('email', 'alice@example.com'),
        'address', JSON_OBJECT('city', 'Paris', 'country', 'FR')
    )
);

SELECT 
    user_id,
    profile->>'$.name' AS name,
    profile->>'$.contact.email' AS email
FROM user_profiles
WHERE city = 'Paris';
```

### 📈 Analyses de cohortes
```sql
-- Analyse de rétention utilisateur par cohorte d'inscription
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_FORMAT(signup_date, '%Y-%m') AS cohort_month
    FROM users
),
cohort_activity AS (
    SELECT 
        uc.cohort_month,
        DATE_FORMAT(a.activity_date, '%Y-%m') AS activity_month,
        COUNT(DISTINCT a.user_id) AS active_users
    FROM user_cohorts uc
    JOIN user_activity a ON uc.user_id = a.user_id
    GROUP BY uc.cohort_month, activity_month
)
SELECT 
    cohort_month,
    activity_month,
    active_users,
    FIRST_VALUE(active_users) OVER (PARTITION BY cohort_month ORDER BY activity_month) AS cohort_size,
    100.0 * active_users / FIRST_VALUE(active_users) OVER (PARTITION BY cohort_month ORDER BY activity_month) AS retention_rate
FROM cohort_activity
ORDER BY cohort_month, activity_month;
```

### 🔍 Analyse de logs applicatifs
```sql
-- Parsing et analyse de logs JSON
CREATE TABLE application_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp DATETIME,
    log_entry JSON,
    level VARCHAR(20) AS (log_entry->>'$.level') STORED,
    user_id INT AS (log_entry->>'$.context.user_id') STORED,
    INDEX idx_level (level),
    INDEX idx_user (user_id),
    INDEX idx_timestamp (timestamp)
);

-- Analyse des erreurs par utilisateur avec fenêtrage
SELECT 
    user_id,
    timestamp,
    log_entry->>'$.message' AS error_message,
    COUNT(*) OVER (
        PARTITION BY user_id 
        ORDER BY timestamp 
        RANGE INTERVAL 1 HOUR PRECEDING
    ) AS errors_last_hour
FROM application_logs
WHERE level = 'ERROR'
ORDER BY errors_last_hour DESC;
```

---

## 💡 Philosophie de cette partie

Les techniques enseignées dans cette partie reposent sur un principe fondamental : **résoudre les problèmes au plus près des données**.

### Pourquoi c'est important ?

1. **Performance** : Traiter 1 million de lignes en SQL est plus rapide que de les charger en mémoire applicative
2. **Scalabilité** : La base de données est optimisée pour les opérations sur ensembles
3. **Maintenabilité** : Une requête SQL de 20 lignes remplace souvent 200 lignes de code
4. **Atomicité** : Les opérations complexes restent transactionnelles et cohérentes
5. **Réutilisabilité** : Les vues et CTE peuvent être partagées entre applications

### Quand utiliser ces techniques ?

✅ **OUI** : 
- Reporting et analytics
- Transformations de données ETL
- Tableaux de bord temps réel
- APIs avec logique métier complexe
- Détection d'anomalies

⚠️ **AVEC PRÉCAUTION** :
- Requêtes dépassant plusieurs centaines de lignes (privilégier les vues)
- Opérations nécessitant des boucles côté applicatif (hors récursion)
- Logique métier spécifique à un langage (crypto, ML)

---

## 🚀 Conseils pour réussir cette partie

### Pour tirer le meilleur parti de cette formation :

1. **Pratiquez sur des données réelles** : Importez un dataset conséquent (100k+ lignes) pour voir l'impact des optimisations

2. **Visualisez les plans d'exécution** : Utilisez `EXPLAIN` systématiquement pour comprendre comment MariaDB traite vos requêtes complexes

3. **Commencez simple, puis complexifiez** : Les Window Functions peuvent sembler intimidantes — démarrez avec un `ROW_NUMBER()` basique avant d'explorer les frames

4. **Comparez avec votre code habituel** : Pour chaque technique, demandez-vous "combien de lignes de Python/Java/PHP cela remplacerait-il ?"

5. **Documentez vos CTE** : Les requêtes avec CTE sont puissantes mais peuvent devenir difficiles à lire — commentez abondamment

6. **Testez les performances** : Les Window Functions sont efficaces, mais sur des partitions de plusieurs millions de lignes, les index deviennent cruciaux

---

## 🎯 Prérequis recommandés

Avant de débuter cette partie, assurez-vous de maîtriser :

- ✅ Requêtes `SELECT` avec `WHERE`, `ORDER BY`, `LIMIT`
- ✅ Jointures simples (`INNER JOIN`, `LEFT JOIN`)
- ✅ Fonctions d'agrégation de base (`COUNT`, `SUM`, `AVG`)
- ✅ Groupements simples avec `GROUP BY`
- ✅ Sous-requêtes scalaires basiques
- ✅ Types de données MariaDB (notamment `JSON`)

Si l'un de ces points n'est pas clair, revoyez la **Partie 1** avant de continuer.

---

## ➡️ Prochaine étape

**Module 3 : Requêtes SQL Intermédiaires (Complément Avancé)** → Approfondissez vos connaissances des jointures, agrégations, et sous-requêtes avec des techniques de niveau production.

Préparez-vous à écrire du SQL qui impressionnera vos collègues ! 💪

---

**MariaDB** : Version 11.8 LTS

⏭️ [Requêtes SQL Intermédiaires](/03-requetes-sql-intermediaires/README.md)
