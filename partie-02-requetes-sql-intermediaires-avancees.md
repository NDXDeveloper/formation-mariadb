🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 2 : Requêtes SQL Intermédiaires et Avancées (Intermédiaire)

> **Niveau** : Intermédiaire → Avancé  
> **Durée estimée** : 2-3 jours  
> **Prérequis** : Maîtrise des bases SQL de la Partie 1 (`SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, `INSERT`/`UPDATE`/`DELETE`), des types de données et des contraintes

---

## 🎯 Passez au niveau supérieur en SQL

Après avoir acquis les fondamentaux de SQL dans la Partie 1, vous êtes maintenant prêt à explorer les **techniques de manipulation et d'analyse des données** qui font tout l'intérêt d'une base relationnelle. Cette deuxième partie vous permettra de résoudre des problèmes complexes avec élégance et efficacité.

L'objectif est de vous transformer en **développeur SQL confirmé**, capable d'écrire des requêtes sophistiquées pour répondre à des besoins métier réels : agrégations et regroupements, jointures, sous-requêtes, classements et calculs analytiques, parcours de hiérarchies, manipulation de documents JSON, et bien plus encore.

Ces techniques ne sont pas de simples curiosités académiques — elles sont **utilisées quotidiennement en production** dans les applications modernes, les systèmes d'analyse, les pipelines de données et les architectures orientées API. Les maîtriser vous distinguera comme un professionnel capable de résoudre des problèmes que d'autres croient impossibles en SQL pur.

À l'issue de cette partie, vous aurez non seulement élargi votre palette technique, mais aussi développé une **approche analytique** pour traiter les données au plus près de la base, sans recourir systématiquement à du code applicatif.

---

## 📚 Les deux chapitres de cette partie

### Chapitre 3 : Requêtes SQL Intermédiaires
**8 sections | Durée : ~1 jour**

Ce chapitre marque le passage de la simple consultation à la **véritable interrogation des données** : résumer, regrouper, combiner et transformer.

- **Fonctions d'agrégation** : `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `DISTINCT` et leur comportement face aux valeurs `NULL`
- **Regroupement** : `GROUP BY`, `HAVING`, `WITH ROLLUP`, et la distinction essentielle `WHERE` / `HAVING`
- **Jointures** : `INNER`, `LEFT`/`RIGHT`, `CROSS`, *self-join* — et la syntaxe Oracle `( + )` reconnue en mode de compatibilité 🆕
- **Sous-requêtes** : scalaires, de ligne, de table (`IN`, `EXISTS`, `ANY`/`ALL`), corrélées ou non, tables dérivées
- **Opérateurs ensemblistes** : `UNION` / `UNION ALL`, `INTERSECT`, `EXCEPT`
- **Fonctions de chaînes** : extraction, recherche, transformation (en tenant compte de `utf8mb4`)
- **Fonctions de dates et heures** : calcul, formatage — et les équivalents Oracle `TO_DATE`/`TRUNC`/`TO_NUMBER`/`TO_CHAR` 🆕
- **Expressions conditionnelles** : `CASE`, `IF`, `IFNULL`, `COALESCE`, `NULLIF`

💡 **Point fort** : des patterns SQL réutilisables qui vous feront gagner des heures de développement applicatif.

---

### Chapitre 4 : Concepts Avancés SQL
**11 sections | Durée : ~2 jours**

Ce chapitre rassemble les constructions SQL les plus puissantes : analytiques, hiérarchiques et semi-structurées.

#### 🔄 Requêtes récursives (`WITH RECURSIVE`)
Parcourir des hiérarchies (organigrammes, nomenclatures), générer des séries de valeurs, traiter des graphes — directement en SQL, en une seule requête.

#### 📊 Window Functions (fonctions de fenêtrage)
La famille la plus transformatrice du SQL analytique :
- **Fonctions de rang** : `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` — Top-N par groupe, déduplication
- **Fonctions de valeur** : `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` — analyses temporelles, comparaisons ligne à ligne
- **Frames de fenêtre** : `ROWS` et `RANGE` — moyennes mobiles, cumuls, calculs glissants (l'unité `GROUPS`, l'exclusion de frame et les bornes temporelles `RANGE` ne sont **pas** disponibles en MariaDB)
- **Cas d'usage** : Top-N, moyennes mobiles, cumuls, part dans le total

#### 🧩 CTE et requêtes complexes
- **Expressions de table communes (`WITH`)** : structurer et factoriser des requêtes complexes — et, depuis la 12.3, **lire une CTE depuis un `UPDATE` ou un `DELETE`** 🆕
- **Requêtes multi-tables** : composer jointures, sous-requêtes et CTE, en évitant le piège de la multiplication des lignes (*fan-out*)
- **Logique ternaire des `NULL`** : `TRUE` / `FALSE` / `UNKNOWN` et ses conséquences sur les filtres, jointures et agrégations

#### 📦 JSON dans MariaDB
- **Stockage** : en MariaDB, le type `JSON` est un **alias de `LONGTEXT`** (validation `JSON_VALID` ajoutée automatiquement, stockage *verbatim*)
- **Fonctions** : `JSON_EXTRACT`, `JSON_VALUE`, `JSON_QUERY`, `JSON_SET`, `JSON_OBJECT`, `JSON_TABLE`…
- **Prédicat `IS JSON`** (standard SQL) et **levée de la limite de profondeur** des documents 🆕
- **JSON Path**, **validation par schéma** (`JSON_SCHEMA_VALID`) et **indexation par colonnes virtuelles**

#### 🔍 Expressions régulières
`REGEXP` (alias `RLIKE`), `REGEXP_REPLACE`, `REGEXP_SUBSTR`, `REGEXP_INSTR` — pattern matching avancé via le moteur **PCRE2**.

💡 **Point fort** : ces techniques réduisent drastiquement le code applicatif nécessaire et améliorent les performances en déplaçant la logique au plus près des données.

---

## 🆕 Nouveautés MariaDB 12.3 abordées dans cette partie

La plupart des outils de cette partie (window functions, CTE, JSON, expressions régulières) sont **stables depuis plusieurs versions LTS**. La série **12.x** y apporte surtout des **raffinements alignés sur le standard SQL et sur la compatibilité Oracle/MySQL**, signalés au fil des chapitres par le marqueur 🆕.

### Compatibilité Oracle dans les fonctions (chapitre 3)
- `TO_DATE` (nouveau en **12.3**), `TO_NUMBER` (12.2), `TRUNC` sur une date (12.2) et le modificateur `FM` de `TO_CHAR` (12.0) : des conversions de style Oracle, utiles en migration et activées en mode de compatibilité (§ 3.7.1).
- L'opérateur de jointure externe **`( + )`** d'Oracle, reconnu lorsque `sql_mode = 'ORACLE'` (§ 3.3.5).

### `UPDATE` / `DELETE` lisant une CTE (chapitre 4)
Une clause `WITH` peut désormais précéder une instruction `UPDATE` ou `DELETE` et **être lue** par celle-ci (§ 4.4.1, MDEV-37220) — une extension du standard, alignée sur MySQL. L'usage emblématique est la **suppression de doublons** repérés par une fonction de fenêtrage :

```sql
WITH doublons AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY client_id ORDER BY date_commande DESC) AS rn
    FROM commandes
)
DELETE FROM commandes
WHERE id IN (SELECT id FROM doublons WHERE rn > 1);
```

### JSON : prédicat `IS JSON` et profondeur illimitée (chapitre 4)
- Le **prédicat `IS JSON`** (standard SQL:2016, apparu en **12.3.1**) vérifie la validité, peut contraindre le type et exiger l'unicité des clés (§ 4.7.4) :

```sql
SELECT * FROM journaux WHERE charge_utile IS JSON;

-- En contrainte : rejeter les documents mal formés ou aux clés dupliquées
-- ... CHECK (donnees IS JSON WITH UNIQUE KEYS)
```

- L'ancienne **limite de 32 niveaux d'imbrication** des documents JSON a été **entièrement supprimée** dans la série 12.x, de façon transparente : aucun changement de syntaxe requis.

---

## ✅ Compétences acquises

À la fin de cette deuxième partie, vous serez capable de :

### Analyse de données
- ✅ **Résumer et regrouper** des données avec les agrégats, `GROUP BY` et `HAVING`
- ✅ **Combiner** plusieurs tables avec les différents types de jointures
- ✅ **Construire** des rapports analytiques avec les window functions
- ✅ **Calculer** des moyennes mobiles, cumuls et métriques glissantes
- ✅ **Résoudre** des problèmes de classement (Top-N, quartiles, déduplication)
- ✅ **Comparer** des données ligne à ligne avec `LAG` et `LEAD`

### Manipulation de données complexes
- ✅ **Parcourir** des structures hiérarchiques avec les requêtes récursives
- ✅ **Transformer** des données par pivot et dépivot
- ✅ **Manipuler** des documents JSON nativement (extraction, modification, `JSON_TABLE`)
- ✅ **Valider** la structure de documents JSON par schéma
- ✅ **Extraire** et transformer du texte avec les expressions régulières

### Lisibilité, robustesse et performance
- ✅ **Structurer** des requêtes complexes avec des CTE (`WITH`)
- ✅ **Choisir** entre sous-requêtes et jointures selon la lisibilité et la performance
- ✅ **Maîtriser** la logique ternaire des `NULL` pour éviter les résultats inattendus
- ✅ **Indexer** un champ JSON via une colonne virtuelle générée
- ✅ **Réduire** le code applicatif en déplaçant la logique vers SQL

---

## 🎓 Parcours recommandés

| Parcours | Importance | Justification |
|----------|------------|---------------|
| 🔧 **Développeur** | ⭐⭐⭐ Essentiel | Window functions et JSON sont au cœur du développement moderne — indispensables pour des APIs performantes et des tableaux de bord. |
| 🤖 **IA/ML Engineer** | ⭐⭐⭐ Essentiel | JSON et requêtes analytiques sont cruciaux pour la préparation des données, le *feature engineering* et l'intégration aux pipelines ML. |
| 🔐 **Administrateur/DBA** | ⭐⭐ Recommandé | Comprendre les requêtes complexes aide à diagnostiquer les problèmes de performance et à optimiser les index (le chapitre 4 est particulièrement utile). |
| ⚙️ **DevOps/Cloud** | ⭐⭐ Utile | Les techniques SQL avancées facilitent les tableaux de bord de supervision et l'analyse de logs structurés (JSON). |

💡 **Note importante** : si vous êtes développeur ou travaillez avec des données analytiques, **cette partie est critique**. Les window functions à elles seules peuvent réduire considérablement le code nécessaire pour certaines fonctionnalités.

---

## 🏢 Cas d'usage réels

Voici des exemples concrets, écrits en SQL valide pour MariaDB 12.3, où les techniques de cette partie sont **indispensables**.

### 📊 Tableaux de bord et reporting
```sql
-- Classement des produits par ventes, avec évolution d'un mois sur l'autre
WITH ventes_mensuelles AS (
    SELECT product_id,
           DATE_FORMAT(order_date, '%Y-%m') AS mois,
           SUM(amount) AS total
    FROM commandes
    GROUP BY product_id, mois
)
SELECT product_id,
       mois,
       total,
       LAG(total) OVER (PARTITION BY product_id ORDER BY mois) AS mois_precedent,
       total - LAG(total) OVER (PARTITION BY product_id ORDER BY mois) AS evolution,
       RANK() OVER (PARTITION BY mois ORDER BY total DESC) AS rang_du_mois
FROM ventes_mensuelles;
```

### 🌳 Hiérarchies organisationnelles
```sql
-- Parcourir un organigramme complet avec une requête récursive
WITH RECURSIVE org_chart AS (
    SELECT id, nom, manager_id, 0 AS niveau,
           CAST(nom AS CHAR(200)) AS chemin
    FROM employes
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.nom, e.manager_id, oc.niveau + 1,
           CONCAT(oc.chemin, ' > ', e.nom)
    FROM employes e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT id, nom, niveau, chemin FROM org_chart ORDER BY chemin;
```

### 📱 APIs modernes avec JSON
```sql
-- Stockage de profils utilisateurs, avec champs « chauds » extraits et indexés.
-- MariaDB n'a pas d'opérateur ->/->> : on extrait avec JSON_VALUE().
CREATE TABLE profils_utilisateur (
    user_id INT PRIMARY KEY,
    profil  JSON,
    email   VARCHAR(255) AS (JSON_VALUE(profil, '$.contact.email')) VIRTUAL,
    ville   VARCHAR(100) AS (JSON_VALUE(profil, '$.adresse.ville')) VIRTUAL,
    INDEX idx_email (email),
    INDEX idx_ville (ville)
);

INSERT INTO profils_utilisateur (user_id, profil) VALUES (
    1,
    JSON_OBJECT(
        'nom', 'Alice',
        'contact', JSON_OBJECT('email', 'alice@example.com'),
        'adresse', JSON_OBJECT('ville', 'Paris', 'pays', 'FR')
    )
);

SELECT user_id,
       JSON_VALUE(profil, '$.nom')           AS nom,
       JSON_VALUE(profil, '$.contact.email') AS email
FROM profils_utilisateur
WHERE ville = 'Paris';
```

### 📈 Analyses de cohortes
```sql
-- Rétention par cohorte d'inscription
WITH cohortes AS (
    SELECT user_id, DATE_FORMAT(date_inscription, '%Y-%m') AS mois_cohorte
    FROM utilisateurs
),
activite_cohorte AS (
    SELECT c.mois_cohorte,
           DATE_FORMAT(a.date_activite, '%Y-%m') AS mois_activite,
           COUNT(DISTINCT a.user_id) AS actifs
    FROM cohortes c
    JOIN activite a ON c.user_id = a.user_id
    GROUP BY c.mois_cohorte, mois_activite
)
SELECT mois_cohorte,
       mois_activite,
       actifs,
       FIRST_VALUE(actifs) OVER (PARTITION BY mois_cohorte ORDER BY mois_activite) AS taille_cohorte,
       ROUND(100.0 * actifs
             / FIRST_VALUE(actifs) OVER (PARTITION BY mois_cohorte ORDER BY mois_activite), 1) AS taux_retention
FROM activite_cohorte
ORDER BY mois_cohorte, mois_activite;
```

### 🔍 Analyse de logs applicatifs
```sql
-- Logs JSON : champs extraits dans des colonnes indexées (via JSON_VALUE)
CREATE TABLE logs_application (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    horodatage DATETIME,
    entree    JSON,
    niveau    VARCHAR(20) AS (JSON_VALUE(entree, '$.level'))           VIRTUAL,
    user_id   INT         AS (JSON_VALUE(entree, '$.context.user_id')) VIRTUAL,
    INDEX idx_niveau (niveau),
    INDEX idx_user (user_id),
    INDEX idx_horodatage (horodatage)
);

-- Nombre d'erreurs par utilisateur (agrégat fenêtré sur un champ extrait du JSON)
SELECT user_id,
       horodatage,
       JSON_VALUE(entree, '$.message') AS message_erreur,
       COUNT(*) OVER (PARTITION BY user_id) AS erreurs_utilisateur
FROM logs_application
WHERE niveau = 'ERROR'
ORDER BY erreurs_utilisateur DESC;
```

> ℹ️ MariaDB ne propose pas de borne temporelle `RANGE ... INTERVAL` (par exemple « la dernière heure ») dans les fonctions de fenêtrage ; pour un décompte sur une fenêtre glissante de temps, on passe par une sous-requête corrélée ou par une régularisation préalable de la série (§ 4.2.3 et § 4.2.4).

---

## 💡 Philosophie de cette partie

Les techniques enseignées ici reposent sur un principe : **résoudre les problèmes au plus près des données**.

### Pourquoi c'est important ?

1. **Performance** : traiter un million de lignes en SQL est souvent plus rapide que de les charger en mémoire applicative.
2. **Scalabilité** : le moteur est optimisé pour les opérations sur ensembles.
3. **Maintenabilité** : une requête SQL de 20 lignes remplace parfois 200 lignes de code.
4. **Atomicité** : les opérations complexes restent transactionnelles et cohérentes.
5. **Réutilisabilité** : les vues et les CTE factorisent la logique.

### Quand utiliser ces techniques ?

✅ **OUI** :
- Reporting et analytique
- Transformations de données (ETL)
- Tableaux de bord
- APIs avec logique métier
- Détection d'anomalies

⚠️ **AVEC PRÉCAUTION** :
- Requêtes dépassant plusieurs centaines de lignes (privilégier les vues)
- Traitements nécessitant des boucles applicatives (hors récursion)
- Logique propre à un langage (cryptographie, ML)

---

## 🚀 Conseils pour réussir cette partie

1. **Pratiquez sur des données réelles** : importez un jeu de données conséquent pour observer l'impact des optimisations.
2. **Visualisez les plans d'exécution** : utilisez `EXPLAIN` pour comprendre comment MariaDB traite vos requêtes (chapitre 5).
3. **Commencez simple, puis complexifiez** : démarrez avec un `ROW_NUMBER()` basique avant d'explorer les frames de fenêtre.
4. **Comparez avec votre code habituel** : pour chaque technique, demandez-vous combien de lignes de Python/Java/PHP elle remplacerait.
5. **Documentez vos CTE** : les requêtes décomposées en paliers nommés se relisent et se maintiennent bien mieux qu'un bloc monolithique.
6. **Pensez aux index** : les window functions imposent un tri ; sur de grosses partitions, le coût se mesure avec `EXPLAIN`.

---

## 🎯 Prérequis recommandés

Avant de débuter cette partie, assurez-vous de maîtriser (Partie 1) :

- ✅ Requêtes `SELECT` avec `WHERE`, `ORDER BY`, `LIMIT`
- ✅ Jointures simples (`INNER JOIN`, `LEFT JOIN`)
- ✅ Fonctions d'agrégation de base (`COUNT`, `SUM`, `AVG`)
- ✅ Groupements simples avec `GROUP BY`
- ✅ Types de données MariaDB (notamment `JSON`) et contraintes

Si l'un de ces points n'est pas clair, revoyez la **Partie 1** avant de continuer.

---

## ➡️ Prochaine étape

**Chapitre 3 : Requêtes SQL Intermédiaires** → approfondissez agrégations, regroupements, jointures et sous-requêtes avec des techniques de niveau production, avant d'aborder les concepts avancés du chapitre 4.

Préparez-vous à écrire du SQL qui impressionnera vos collègues ! 💪

---

**MariaDB** : Version 12.3 LTS (GA fin mai 2026, support jusqu'en juin 2029) — LTS précédente : 11.8

⏭️ [Requêtes SQL Intermédiaires](/03-requetes-sql-intermediaires/README.md)
