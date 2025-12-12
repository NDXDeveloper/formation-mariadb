🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 Performance des vues : MERGE vs TEMPTABLE

> **Niveau** : Intermédiaire à Avancé
> **Durée estimée** : 2-2.5 heures
> **Prérequis** : Sections 9.1-9.5, Chapitre 5 (Index et performance), compréhension de EXPLAIN

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre en profondeur les algorithmes MERGE et TEMPTABLE
- Analyser l'impact de chaque algorithme sur les performances
- Utiliser EXPLAIN pour diagnostiquer les problèmes de performance des vues
- Déterminer quand MariaDB choisit automatiquement MERGE vs TEMPTABLE
- Forcer l'algorithme approprié selon les cas d'usage
- Optimiser les vues pour des performances maximales
- Identifier et résoudre les problèmes de performance courants
- Choisir entre vues et alternatives selon les contraintes de performance

---

## Introduction

### Les algorithmes de traitement des vues

Lorsqu'une requête interroge une vue, MariaDB doit décider **comment** traiter cette requête. Deux algorithmes principaux sont disponibles :

1. **MERGE (Fusion)** : Le serveur fusionne la définition de la vue avec la requête de l'utilisateur pour créer une seule requête optimisée
2. **TEMPTABLE (Table temporaire)** : Le serveur exécute d'abord la requête de la vue, stocke le résultat dans une table temporaire, puis interroge cette table

Le choix de l'algorithme a un **impact majeur** sur les performances, pouvant varier d'un facteur 10x à 1000x selon les cas.

### Enjeu de performance

```sql
-- Scénario : Vue avec 1 million de lignes

-- Avec MERGE (optimal)
-- Temps d'exécution : 50ms
-- Utilise les index, filtre efficacement

-- Avec TEMPTABLE (sous-optimal)
-- Temps d'exécution : 5000ms (100x plus lent)
-- Crée une table temporaire de 1M lignes, puis la scanne
```

Cette section vous apprendra à **identifier, comprendre et optimiser** l'algorithme utilisé par vos vues.

---

## Algorithme MERGE : Fonctionnement détaillé

### Principe de la fusion

L'algorithme **MERGE** fusionne la requête de la vue avec la requête de l'utilisateur pour créer une **requête unique** qui est ensuite optimisée par l'optimiseur SQL.

### Exemple de fusion

```sql
-- Table source
CREATE TABLE produits (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    categorie VARCHAR(50),
    stock INT,
    INDEX idx_prix (prix),
    INDEX idx_categorie (categorie)
) ENGINE=InnoDB;

-- Vue avec MERGE
CREATE ALGORITHM=MERGE VIEW v_produits_disponibles AS
SELECT
    id,
    nom,
    prix,
    categorie,
    stock
FROM produits
WHERE stock > 0;

-- Requête utilisateur
SELECT nom, prix
FROM v_produits_disponibles
WHERE categorie = 'Électronique'
  AND prix < 100
ORDER BY prix
LIMIT 10;

-- Après fusion (requête réellement exécutée)
SELECT nom, prix
FROM produits
WHERE stock > 0                    -- Condition de la vue
  AND categorie = 'Électronique'   -- Condition utilisateur
  AND prix < 100                   -- Condition utilisateur
ORDER BY prix
LIMIT 10;
-- ✅ L'optimiseur peut utiliser idx_categorie et idx_prix
```

### Avantages de MERGE

1. **Performance optimale** : L'optimiseur voit la requête complète et peut choisir les meilleurs index
2. **Utilisation des index** : Tous les index des tables sous-jacentes sont disponibles
3. **Pas de stockage temporaire** : Pas de création de table temporaire
4. **LIMIT efficace** : Le LIMIT est appliqué directement sur la table source
5. **Peu de mémoire** : Pas de consommation mémoire supplémentaire

### Conditions pour MERGE

MariaDB peut utiliser MERGE uniquement si la vue **ne contient pas** :

```sql
-- ❌ Ces éléments empêchent MERGE

-- Agrégations
SELECT categorie, COUNT(*) FROM produits GROUP BY categorie;

-- DISTINCT
SELECT DISTINCT categorie FROM produits;

-- GROUP BY / HAVING
SELECT categorie, AVG(prix) FROM produits GROUP BY categorie;

-- UNION
SELECT id FROM produits UNION SELECT id FROM archives;

-- Sous-requête dans SELECT
SELECT id, (SELECT COUNT(*) FROM commandes WHERE produit_id = p.id) FROM produits p;

-- Sous-requête dans FROM
SELECT * FROM (SELECT * FROM produits WHERE prix > 100) AS sub;

-- LIMIT (dans la vue)
SELECT * FROM produits LIMIT 100;

-- Fonctions d'agrégation
SELECT MAX(prix), MIN(prix) FROM produits;
```

### Exemple de plan d'exécution avec MERGE

```sql
-- Vue simple avec MERGE
CREATE ALGORITHM=MERGE VIEW v_produits AS
SELECT id, nom, prix, categorie
FROM produits
WHERE prix > 10;

-- Requête
EXPLAIN SELECT * FROM v_produits WHERE categorie = 'Jouets'\G

-- Résultat EXPLAIN
-- *************************** 1. row ***************************
--            id: 1
--   select_type: SIMPLE
--         table: produits
--          type: ref
-- possible_keys: idx_categorie,idx_prix
--           key: idx_categorie
--       key_len: 202
--           ref: const
--          rows: 1000
--         Extra: Using index condition; Using where

-- ✅ Type: ref (utilise l'index)
-- ✅ key: idx_categorie (index utilisé efficacement)
-- ✅ rows: 1000 (estimation précise)
```

---

## Algorithme TEMPTABLE : Fonctionnement détaillé

### Principe de la table temporaire

L'algorithme **TEMPTABLE** exécute la requête de la vue, stocke **tous** les résultats dans une table temporaire, puis exécute la requête utilisateur sur cette table temporaire.

### Processus en étapes

```sql
-- Vue avec agrégation (force TEMPTABLE)
CREATE ALGORITHM=TEMPTABLE VIEW v_stats_categories AS
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix) AS prix_moyen,
    SUM(stock) AS stock_total
FROM produits
GROUP BY categorie;

-- Requête utilisateur
SELECT * FROM v_stats_categories WHERE nb_produits > 100;

-- Processus TEMPTABLE :
-- Étape 1 : Créer une table temporaire
CREATE TEMPORARY TABLE tmp_view_12345 (
    categorie VARCHAR(50),
    nb_produits BIGINT,
    prix_moyen DECIMAL(10,2),
    stock_total BIGINT
);

-- Étape 2 : Remplir la table temporaire (TOUTES les lignes de la vue)
INSERT INTO tmp_view_12345
SELECT
    categorie,
    COUNT(*) AS nb_produits,
    AVG(prix) AS prix_moyen,
    SUM(stock) AS stock_total
FROM produits
GROUP BY categorie;
-- ⚠️ Calcule pour TOUTES les catégories, même si on veut uniquement > 100

-- Étape 3 : Interroger la table temporaire
SELECT * FROM tmp_view_12345 WHERE nb_produits > 100;
-- ❌ Pas d'index disponible sur tmp_view_12345
-- ❌ Full table scan sur la table temporaire

-- Étape 4 : Nettoyer (automatique)
DROP TEMPORARY TABLE tmp_view_12345;
```

### Inconvénients de TEMPTABLE

1. **Création de table temporaire** : Coût I/O pour créer et remplir la table
2. **Pas d'index** : La table temporaire n'a généralement pas d'index
3. **Calcul complet** : La vue est calculée entièrement, même si on veut 1 ligne
4. **Consommation mémoire/disque** : Stockage temporaire des résultats
5. **Pas de push-down** : Les conditions WHERE ne peuvent pas être poussées vers la table source

### Quand TEMPTABLE est obligatoire

MariaDB utilise automatiquement TEMPTABLE si la vue contient :

```sql
-- ✅ Ces éléments forcent TEMPTABLE

-- 1. Agrégations
CREATE VIEW v1 AS
SELECT categorie, COUNT(*) AS total FROM produits GROUP BY categorie;

-- 2. DISTINCT
CREATE VIEW v2 AS
SELECT DISTINCT categorie FROM produits;

-- 3. GROUP BY
CREATE VIEW v3 AS
SELECT categorie, AVG(prix) FROM produits GROUP BY categorie;

-- 4. HAVING
CREATE VIEW v4 AS
SELECT categorie, COUNT(*) AS cnt FROM produits GROUP BY categorie HAVING cnt > 10;

-- 5. UNION
CREATE VIEW v5 AS
SELECT id FROM produits UNION SELECT id FROM archives;

-- 6. Sous-requête dans SELECT
CREATE VIEW v6 AS
SELECT id, (SELECT COUNT(*) FROM commandes WHERE produit_id = p.id) AS nb_cmd FROM produits p;

-- 7. Sous-requête dans FROM
CREATE VIEW v7 AS
SELECT * FROM (SELECT * FROM produits WHERE prix > 100) sub;

-- 8. LIMIT dans la vue
CREATE VIEW v8 AS
SELECT * FROM produits ORDER BY prix DESC LIMIT 100;

-- 9. Fonctions d'agrégation
CREATE VIEW v9 AS
SELECT MAX(prix) AS max_prix, MIN(prix) AS min_prix FROM produits;
```

### Exemple de plan d'exécution avec TEMPTABLE

```sql
-- Vue avec GROUP BY (force TEMPTABLE)
CREATE ALGORITHM=TEMPTABLE VIEW v_stats AS
SELECT categorie, COUNT(*) AS total
FROM produits
GROUP BY categorie;

-- Requête
EXPLAIN SELECT * FROM v_stats WHERE total > 100\G

-- Résultat EXPLAIN
-- *************************** 1. row ***************************
--            id: 1
--   select_type: PRIMARY
--         table: <derived2>
--          type: ALL
-- possible_keys: NULL
--           key: NULL
--       key_len: NULL
--           ref: NULL
--          rows: 50
--         Extra: Using where
-- *************************** 2. row ***************************
--            id: 2
--   select_type: DERIVED
--         table: produits
--          type: ALL
-- possible_keys: NULL
--           key: NULL
--       key_len: NULL
--           ref: NULL
--          rows: 100000
--         Extra: Using temporary; Using filesort

-- ❌ Type: ALL (full table scan sur la table temporaire)
-- ❌ key: NULL (pas d'index utilisable)
-- ❌ Extra: Using temporary (table temporaire créée)
-- ⚠️ rows: 100000 dans la dérivée (scanne toute la table source)
```

---

## Comparaison approfondie : MERGE vs TEMPTABLE

### Tableau comparatif

| Aspect | MERGE | TEMPTABLE |
|--------|-------|-----------|
| **Performance** | ⚡⚡⚡ Excellente | 🐢 Variable (souvent lente) |
| **Utilisation index** | ✅ Oui, tous les index disponibles | ❌ Non, pas d'index sur table temp |
| **Mémoire** | ✅ Minime | ⚠️ Proportionnelle au résultat |
| **I/O disque** | ✅ Minime | ⚠️ Écriture + lecture table temp |
| **LIMIT efficace** | ✅ Oui, appliqué tôt | ❌ Non, après création table temp |
| **WHERE pushdown** | ✅ Oui, conditions fusionnées | ❌ Non, filtrage après |
| **Agrégations** | ❌ Non supporté | ✅ Supporté |
| **DISTINCT** | ❌ Non supporté | ✅ Supporté |
| **GROUP BY** | ❌ Non supporté | ✅ Supporté |
| **UNION** | ❌ Non supporté | ✅ Supporté |
| **Cas d'usage** | Vues de filtrage simples | Vues avec calculs/agrégations |

### Benchmark de performance

Scénario : Table avec 1 million de lignes

```sql
-- Préparer les données de test
CREATE TABLE test_produits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    categorie VARCHAR(50),
    stock INT,
    INDEX idx_categorie (categorie),
    INDEX idx_prix (prix)
) ENGINE=InnoDB;

-- Insérer 1 million de lignes (exemple simplifié)
-- INSERT INTO test_produits ...

-- Vue MERGE
CREATE ALGORITHM=MERGE VIEW v_test_merge AS
SELECT id, nom, prix, categorie
FROM test_produits
WHERE prix > 10;

-- Vue TEMPTABLE (avec DISTINCT pour forcer TEMPTABLE)
CREATE ALGORITHM=TEMPTABLE VIEW v_test_temptable AS
SELECT DISTINCT categorie, prix
FROM test_produits
WHERE prix > 10;

-- Benchmark 1 : Sélection avec filtre
SET profiling = 1;

-- Test MERGE
SELECT * FROM v_test_merge WHERE categorie = 'Électronique' LIMIT 100;
-- Durée : ~50ms
-- Rows examinées : ~10,000 (grâce à l'index)

-- Test TEMPTABLE
SELECT * FROM v_test_temptable WHERE categorie = 'Électronique' LIMIT 100;
-- Durée : ~2000ms (40x plus lent)
-- Rows examinées : 1,000,000 (full scan pour créer la table temp)

SHOW PROFILES;
SET profiling = 0;

-- Résultats typiques :
-- MERGE    : 0.05s  (50ms)
-- TEMPTABLE: 2.00s  (2000ms)
-- Ratio    : 40x plus rapide avec MERGE
```

### Impact de la taille des données

```sql
-- Test avec différentes tailles de table

-- 10,000 lignes
-- MERGE    : 5ms
-- TEMPTABLE: 200ms
-- Ratio    : 40x

-- 100,000 lignes
-- MERGE    : 50ms
-- TEMPTABLE: 1500ms
-- Ratio    : 30x

-- 1,000,000 lignes
-- MERGE    : 100ms
-- TEMPTABLE: 15000ms (15s)
-- Ratio    : 150x

-- 10,000,000 lignes
-- MERGE    : 500ms
-- TEMPTABLE: 180000ms (3min)
-- Ratio    : 360x
```

💡 **Conclusion** : La différence de performance s'aggrave avec la taille des données. TEMPTABLE devient **critique** pour les grandes tables.

---

## ALGORITHM = UNDEFINED : Comportement automatique

### Règle de sélection

Quand `ALGORITHM` n'est pas spécifié (ou vaut `UNDEFINED`), MariaDB choisit automatiquement :

```sql
CREATE VIEW v_auto AS
SELECT ...;
-- Équivalent à :
CREATE ALGORITHM=UNDEFINED VIEW v_auto AS
SELECT ...;
```

**Règle de MariaDB** :
1. Essayer **MERGE** d'abord
2. Si MERGE n'est pas possible (agrégation, DISTINCT, etc.) → Utiliser **TEMPTABLE**

### Vérifier l'algorithme utilisé

```sql
-- Méthode 1 : INFORMATION_SCHEMA
SELECT
    TABLE_NAME,
    ALGORITHM
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'v_auto';

-- Résultat :
-- TABLE_NAME | ALGORITHM
-- v_auto     | MERGE  (ou TEMPTABLE)

-- Méthode 2 : SHOW CREATE VIEW
SHOW CREATE VIEW v_auto\G

-- Méthode 3 : EXPLAIN
EXPLAIN SELECT * FROM v_auto\G
-- Si "Using temporary" ou "<derived>" → TEMPTABLE
-- Si accès direct à la table → MERGE
```

### Exemples de sélection automatique

```sql
-- Cas 1 : Vue simple → MERGE
CREATE VIEW v1 AS
SELECT id, nom, prix FROM produits WHERE prix > 10;
-- → MERGE (pas d'agrégation, pas de DISTINCT)

-- Cas 2 : Vue avec DISTINCT → TEMPTABLE
CREATE VIEW v2 AS
SELECT DISTINCT categorie FROM produits;
-- → TEMPTABLE (DISTINCT force TEMPTABLE)

-- Cas 3 : Vue avec GROUP BY → TEMPTABLE
CREATE VIEW v3 AS
SELECT categorie, COUNT(*) FROM produits GROUP BY categorie;
-- → TEMPTABLE (GROUP BY force TEMPTABLE)

-- Cas 4 : Vue avec jointure simple → MERGE
CREATE VIEW v4 AS
SELECT p.nom, c.nom AS categorie_nom
FROM produits p
INNER JOIN categories c ON p.categorie_id = c.id;
-- → MERGE (jointure simple, pas d'agrégation)

-- Cas 5 : Vue avec sous-requête dans SELECT → TEMPTABLE
CREATE VIEW v5 AS
SELECT
    id,
    nom,
    (SELECT COUNT(*) FROM commandes WHERE produit_id = p.id) AS nb_cmd
FROM produits p;
-- → TEMPTABLE (sous-requête dans SELECT)
```

---

## Forcer l'algorithme : Quand et comment

### Syntaxe pour forcer

```sql
-- Forcer MERGE
CREATE ALGORITHM=MERGE VIEW v_force_merge AS
SELECT ...;

-- Forcer TEMPTABLE
CREATE ALGORITHM=TEMPTABLE VIEW v_force_temptable AS
SELECT ...;
```

### Cas 1 : Forcer MERGE pour améliorer les performances

```sql
-- ❌ Problème : Vue lente avec UNDEFINED
CREATE VIEW v_produits_actifs AS
SELECT id, nom, prix, categorie
FROM produits
WHERE actif = 1;

-- Vérification
SELECT ALGORITHM FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_NAME = 'v_produits_actifs';
-- Résultat : UNDEFINED (pourrait être MERGE ou TEMPTABLE)

-- ✅ Solution : Forcer MERGE explicitement
CREATE OR REPLACE ALGORITHM=MERGE VIEW v_produits_actifs AS
SELECT id, nom, prix, categorie
FROM produits
WHERE actif = 1;

-- Performance améliorée si MariaDB utilisait TEMPTABLE par erreur
```

⚠️ **Attention** : Si vous forcez MERGE sur une vue incompatible (avec GROUP BY par exemple), MariaDB va **ignorer** MERGE et utiliser TEMPTABLE quand même.

```sql
-- Essai de forcer MERGE avec GROUP BY
CREATE ALGORITHM=MERGE VIEW v_stats AS
SELECT categorie, COUNT(*) FROM produits GROUP BY categorie;
-- ⚠️ MariaDB ignore MERGE et utilise TEMPTABLE (GROUP BY incompatible)

-- Vérification
SELECT ALGORITHM FROM INFORMATION_SCHEMA.VIEWS WHERE TABLE_NAME = 'v_stats';
-- Résultat : TEMPTABLE (pas MERGE !)
```

### Cas 2 : Forcer TEMPTABLE pour la cohérence

Rarement nécessaire, mais peut être utile pour :

```sql
-- Vue qui doit toujours avoir un snapshot cohérent
CREATE ALGORITHM=TEMPTABLE VIEW v_snapshot AS
SELECT id, nom, prix
FROM produits
WHERE prix > 100;

-- Avec TEMPTABLE, le résultat est calculé une seule fois
-- Utile si la vue est interrogée plusieurs fois dans la même transaction
```

### Cas 3 : Ne PAS forcer TEMPTABLE par erreur

```sql
-- ❌ Erreur courante : Forcer TEMPTABLE sans raison
CREATE ALGORITHM=TEMPTABLE VIEW v_simple AS
SELECT id, nom FROM produits WHERE actif = 1;

-- Cette vue pourrait utiliser MERGE (beaucoup plus rapide)
-- Forcer TEMPTABLE dégrade les performances sans bénéfice

-- ✅ Correction : Laisser UNDEFINED ou forcer MERGE
CREATE OR REPLACE ALGORITHM=MERGE VIEW v_simple AS
SELECT id, nom FROM produits WHERE actif = 1;
```

---

## Optimisation des vues pour la performance

### Stratégie 1 : Favoriser MERGE quand possible

```sql
-- ❌ Vue qui force TEMPTABLE inutilement
CREATE VIEW v_produits_details AS
SELECT
    p.*,
    (SELECT c.nom FROM categories c WHERE c.id = p.categorie_id) AS categorie_nom
FROM produits p;
-- Sous-requête dans SELECT → TEMPTABLE

-- ✅ Alternative avec MERGE (jointure)
CREATE VIEW v_produits_details AS
SELECT
    p.id,
    p.nom,
    p.prix,
    c.nom AS categorie_nom
FROM produits p
LEFT JOIN categories c ON p.categorie_id = c.id;
-- Jointure simple → MERGE possible
```

### Stratégie 2 : Créer des index sur les tables sous-jacentes

```sql
-- Vue qui filtre par categorie
CREATE ALGORITHM=MERGE VIEW v_electronique AS
SELECT id, nom, prix
FROM produits
WHERE categorie = 'Électronique';

-- ❌ Sans index : Full table scan
SELECT * FROM v_electronique WHERE prix < 100;
-- Scanne toute la table produits

-- ✅ Avec index : Accès rapide
CREATE INDEX idx_categorie ON produits(categorie);
CREATE INDEX idx_prix ON produits(prix);

-- Maintenant très rapide
SELECT * FROM v_electronique WHERE prix < 100;
-- Utilise idx_categorie ET idx_prix
```

### Stratégie 3 : Simplifier les vues complexes

```sql
-- ❌ Vue trop complexe
CREATE VIEW v_complexe AS
SELECT
    p.id,
    p.nom,
    (SELECT COUNT(*) FROM commandes WHERE produit_id = p.id) AS nb_cmd,
    (SELECT AVG(note) FROM avis WHERE produit_id = p.id) AS note_moyenne,
    (SELECT COUNT(*) FROM favoris WHERE produit_id = p.id) AS nb_favoris
FROM produits p;
-- 3 sous-requêtes → TEMPTABLE forcé

-- ✅ Alternative : Plusieurs vues simples
CREATE VIEW v_produits_base AS
SELECT id, nom FROM produits;

CREATE VIEW v_stats_commandes AS
SELECT produit_id, COUNT(*) AS nb_cmd FROM commandes GROUP BY produit_id;

CREATE VIEW v_stats_avis AS
SELECT produit_id, AVG(note) AS note_moyenne FROM avis GROUP BY produit_id;

-- L'application peut joindre ces vues selon les besoins
-- Ou créer une vue matérialisée (table de cache)
```

### Stratégie 4 : Utiliser des colonnes générées pour les calculs

```sql
-- ❌ Vue avec calculs répétés
CREATE VIEW v_produits_prix_ttc AS
SELECT
    id,
    nom,
    prix_ht,
    (prix_ht * 1.20) AS prix_ttc
FROM produits;

-- ✅ Alternative : Colonne générée dans la table
ALTER TABLE produits
ADD COLUMN prix_ttc DECIMAL(10,2) AS (prix_ht * 1.20) STORED;

-- Vue simplifiée
CREATE VIEW v_produits_prix AS
SELECT id, nom, prix_ht, prix_ttc
FROM produits;
-- Plus rapide car prix_ttc est déjà calculé et indexable
```

### Stratégie 5 : Remplacer les vues TEMPTABLE par des tables de cache

```sql
-- ❌ Vue lente avec agrégation (TEMPTABLE)
CREATE VIEW v_stats_ventes AS
SELECT
    produit_id,
    COUNT(*) AS nb_ventes,
    SUM(quantite) AS quantite_totale,
    SUM(montant) AS ca_total
FROM ventes
GROUP BY produit_id;
-- TEMPTABLE, recalculé à chaque interrogation

-- ✅ Alternative : Table de cache mise à jour périodiquement
CREATE TABLE cache_stats_ventes (
    produit_id INT PRIMARY KEY,
    nb_ventes INT,
    quantite_totale INT,
    ca_total DECIMAL(15,2),
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Rafraîchir avec un EVENT ou trigger
INSERT INTO cache_stats_ventes (produit_id, nb_ventes, quantite_totale, ca_total)
SELECT
    produit_id,
    COUNT(*),
    SUM(quantite),
    SUM(montant)
FROM ventes
GROUP BY produit_id
ON DUPLICATE KEY UPDATE
    nb_ventes = VALUES(nb_ventes),
    quantite_totale = VALUES(quantite_totale),
    ca_total = VALUES(ca_total);

-- Interrogation ultra-rapide (table réelle avec index)
SELECT * FROM cache_stats_ventes WHERE produit_id = 100;
```

---

## Diagnostic des problèmes de performance

### Utiliser EXPLAIN pour analyser les vues

```sql
-- Vue à analyser
CREATE VIEW v_produits_populaires AS
SELECT
    p.id,
    p.nom,
    p.prix,
    COUNT(c.id) AS nb_commandes
FROM produits p
LEFT JOIN commandes c ON p.id = c.produit_id
GROUP BY p.id, p.nom, p.prix
HAVING nb_commandes > 100;

-- Analyser avec EXPLAIN
EXPLAIN SELECT * FROM v_produits_populaires WHERE prix < 50\G

-- Indicateurs de problème :
-- 1. type: ALL → Full table scan
-- 2. key: NULL → Pas d'index utilisé
-- 3. rows: 1000000 → Beaucoup de lignes examinées
-- 4. Extra: Using temporary → Table temporaire
-- 5. Extra: Using filesort → Tri coûteux
-- 6. <derived> → Table dérivée (TEMPTABLE)
```

### EXPLAIN détaillé

```sql
-- Vue simple MERGE
EXPLAIN FORMAT=JSON SELECT * FROM v_produits WHERE categorie = 'Jouets'\G

{
  "query_block": {
    "select_id": 1,
    "table": {
      "table_name": "produits",
      "access_type": "ref",
      "possible_keys": ["idx_categorie"],
      "key": "idx_categorie",
      "key_length": "202",
      "ref": ["const"],
      "rows": 1000,
      "filtered": 100,
      "attached_condition": "(produits.categorie = 'Jouets')"
    }
  }
}
-- ✅ Accès direct à 'produits' avec index → MERGE

-- Vue avec GROUP BY (TEMPTABLE)
EXPLAIN FORMAT=JSON SELECT * FROM v_stats_categories WHERE nb_produits > 10\G

{
  "query_block": {
    "select_id": 1,
    "table": {
      "table_name": "<derived2>",
      "access_type": "ALL",
      "rows": 50,
      "filtered": 100,
      "attached_condition": "(v_stats_categories.nb_produits > 10)",
      "materialized": {
        "query_block": {
          "select_id": 2,
          "table": {
            "table_name": "produits",
            "access_type": "ALL",
            "rows": 100000
          }
        }
      }
    }
  }
}
-- ❌ <derived2> indique TEMPTABLE
-- ⚠️ access_type: ALL sur la dérivée (full scan)
```

### Profiling pour mesurer le temps

```sql
SET profiling = 1;

-- Test 1 : Vue MERGE
SELECT * FROM v_produits_merge WHERE categorie = 'Test';

-- Test 2 : Vue TEMPTABLE
SELECT * FROM v_produits_temptable WHERE categorie = 'Test';

-- Voir les résultats
SHOW PROFILES;

-- Résultat exemple :
-- Query_ID | Duration  | Query
-- 1        | 0.005234  | SELECT * FROM v_produits_merge ...
-- 2        | 1.234567  | SELECT * FROM v_produits_temptable ...
-- → TEMPTABLE est 235x plus lent !

-- Détails d'un profil
SHOW PROFILE FOR QUERY 2;

-- Résultat exemple pour TEMPTABLE :
-- Status                   | Duration
-- starting                 | 0.000123
-- executing                | 0.834567  ← Création table temp
-- Sending data             | 0.234567
-- Creating tmp table       | 0.123456  ← Coût supplémentaire
-- end                      | 0.000234

SET profiling = 0;
```

### Monitoring en production

```sql
-- Vue pour identifier les vues lentes
CREATE VIEW v_slow_views AS
SELECT
    TABLE_NAME,
    ALGORITHM,
    VIEW_DEFINITION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'production_db'
  AND (
    ALGORITHM = 'TEMPTABLE'
    OR VIEW_DEFINITION LIKE '%GROUP BY%'
    OR VIEW_DEFINITION LIKE '%DISTINCT%'
    OR VIEW_DEFINITION LIKE '%UNION%'
  );

-- Lister les vues potentiellement problématiques
SELECT * FROM v_slow_views;
```

---

## Cas d'usage et recommandations

### Quand utiliser MERGE

✅ **Utilisez MERGE (ou laissez UNDEFINED) pour** :

```sql
-- 1. Vues de filtrage simple
CREATE VIEW v_clients_actifs AS
SELECT * FROM clients WHERE statut = 'ACTIF';

-- 2. Vues de sécurité (masquage colonnes)
CREATE VIEW v_employes_publics AS
SELECT id, nom, email FROM employes;

-- 3. Vues avec jointures simples
CREATE VIEW v_commandes_details AS
SELECT c.*, cl.nom AS client_nom
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id;

-- 4. Vues pour simplifier les requêtes
CREATE VIEW v_produits_avec_categorie AS
SELECT p.*, cat.nom AS categorie_nom
FROM produits p
LEFT JOIN categories cat ON p.categorie_id = cat.id;

-- 5. Vues avec calculs simples
CREATE VIEW v_prix_ttc AS
SELECT id, nom, prix_ht, (prix_ht * 1.20) AS prix_ttc
FROM produits;
```

### Quand TEMPTABLE est acceptable

⚠️ **TEMPTABLE est acceptable (inévitable) pour** :

```sql
-- 1. Statistiques et agrégations
CREATE VIEW v_stats_mensuelles AS
SELECT
    DATE_FORMAT(date_vente, '%Y-%m') AS mois,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca
FROM ventes
GROUP BY DATE_FORMAT(date_vente, '%Y-%m');

-- 2. Top N avec LIMIT
CREATE VIEW v_top_produits AS
SELECT id, nom, nb_ventes
FROM produits
ORDER BY nb_ventes DESC
LIMIT 100;

-- 3. DISTINCT pour dédoublonner
CREATE VIEW v_villes_uniques AS
SELECT DISTINCT ville FROM clients;

-- 4. Unions
CREATE VIEW v_contacts_tous AS
SELECT id, nom, email FROM clients
UNION
SELECT id, nom, email FROM prospects;
```

**Mais considérez des alternatives** :
- Tables de cache rafraîchies périodiquement
- Colonnes générées pour les calculs
- Procédures stockées pour les rapports complexes

### Quand remplacer une vue par une table

🔄 **Remplacez une vue TEMPTABLE par une table si** :

1. **Haute fréquence d'interrogation** : Vue appelée > 100 fois/minute
2. **Données volumineuses** : Vue retourne > 100 000 lignes
3. **Calculs coûteux** : Agrégations sur millions de lignes
4. **Latence critique** : Réponse requise en < 100ms
5. **Fraîcheur acceptable** : Données peuvent avoir 5-15min de décalage

```sql
-- ❌ Vue lente interrogée fréquemment
CREATE VIEW v_dashboard AS
SELECT
    DATE(date_vente) AS jour,
    COUNT(*) AS nb_ventes,
    SUM(montant) AS ca,
    AVG(montant) AS panier_moyen
FROM ventes
WHERE date_vente >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(date_vente);
-- Interrogée 1000 fois/minute → Surcharge serveur

-- ✅ Table de cache rafraîchie toutes les 5 minutes
CREATE TABLE cache_dashboard (
    jour DATE PRIMARY KEY,
    nb_ventes INT,
    ca DECIMAL(15,2),
    panier_moyen DECIMAL(10,2),
    derniere_maj TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- EVENT pour rafraîchir
CREATE EVENT evt_refresh_dashboard
ON SCHEDULE EVERY 5 MINUTE
DO
    INSERT INTO cache_dashboard (jour, nb_ventes, ca, panier_moyen)
    SELECT
        DATE(date_vente),
        COUNT(*),
        SUM(montant),
        AVG(montant)
    FROM ventes
    WHERE date_vente >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    GROUP BY DATE(date_vente)
    ON DUPLICATE KEY UPDATE
        nb_ventes = VALUES(nb_ventes),
        ca = VALUES(ca),
        panier_moyen = VALUES(panier_moyen);
```

---

## Problèmes de performance courants

### Problème 1 : Vue TEMPTABLE sur grande table

```sql
-- ❌ Problème
CREATE VIEW v_stats AS
SELECT categorie, COUNT(*) AS total
FROM produits  -- 10 millions de lignes
GROUP BY categorie;

SELECT * FROM v_stats WHERE total > 1000;
-- Temps : 30 secondes (inacceptable)

-- ✅ Solution : Table de cache
CREATE TABLE cache_stats_categories AS
SELECT categorie, COUNT(*) AS total
FROM produits
GROUP BY categorie;

CREATE INDEX idx_total ON cache_stats_categories(total);

-- Rafraîchir périodiquement
-- Temps de lecture : 5ms (6000x plus rapide)
```

### Problème 2 : Vues imbriquées avec TEMPTABLE

```sql
-- ❌ Problème : Vue sur vue avec TEMPTABLE
CREATE VIEW v_base AS
SELECT categorie, AVG(prix) AS prix_moyen
FROM produits
GROUP BY categorie;  -- TEMPTABLE

CREATE VIEW v_niveau2 AS
SELECT * FROM v_base WHERE prix_moyen > 50;  -- Interroge TEMPTABLE

CREATE VIEW v_niveau3 AS
SELECT * FROM v_niveau2 WHERE categorie != 'Divers';  -- Interroge TEMPTABLE de TEMPTABLE

-- Chaque niveau crée une table temporaire !
-- v_niveau3 = 3 tables temporaires imbriquées → Très lent

-- ✅ Solution : Une seule vue ou table de cache
CREATE VIEW v_direct AS
SELECT categorie, AVG(prix) AS prix_moyen
FROM produits
WHERE categorie != 'Divers'
GROUP BY categorie
HAVING prix_moyen > 50;
```

### Problème 3 : LIMIT non efficace avec TEMPTABLE

```sql
-- ❌ Problème : LIMIT n'optimise pas avec TEMPTABLE
CREATE VIEW v_tous_produits AS
SELECT * FROM produits ORDER BY date_creation DESC;
-- Pas de LIMIT dans la vue → Pourrait être MERGE

SELECT * FROM v_tous_produits LIMIT 10;
-- Avec MERGE : Trie et retourne 10 lignes (rapide)

-- Mais si TEMPTABLE était forcé :
CREATE ALGORITHM=TEMPTABLE VIEW v_tous_produits_temp AS
SELECT * FROM produits ORDER BY date_creation DESC;

SELECT * FROM v_tous_produits_temp LIMIT 10;
-- ⚠️ TEMPTABLE traite TOUTES les lignes, crée la table temp complète
-- puis extrait 10 lignes → Très inefficace

-- ✅ Solution : Ne pas forcer TEMPTABLE, laisser MERGE
```

### Problème 4 : Pas d'index sur colonnes de la vue

```sql
-- Vue MERGE
CREATE ALGORITHM=MERGE VIEW v_produits AS
SELECT id, nom, categorie, prix
FROM produits
WHERE actif = 1;

-- ❌ Requête lente : pas d'index sur 'prix'
SELECT * FROM v_produits WHERE prix < 50;
-- Full table scan si pas d'index sur 'prix'

-- ✅ Solution : Créer l'index sur la table source
CREATE INDEX idx_prix ON produits(prix);

-- Maintenant rapide
SELECT * FROM v_produits WHERE prix < 50;
-- Utilise idx_prix efficacement
```

---

## ✅ Points clés à retenir

- **MERGE** fusionne la requête de la vue avec la requête utilisateur → **Performance optimale**
- **TEMPTABLE** crée une table temporaire avec tout le résultat → **Performance variable, souvent lente**
- MERGE utilise les **index** des tables sous-jacentes, TEMPTABLE **non**
- **ALGORITHM=UNDEFINED** (défaut) : MariaDB choisit MERGE si possible, sinon TEMPTABLE
- TEMPTABLE est **obligatoire** pour : GROUP BY, DISTINCT, agrégations, UNION, sous-requêtes SELECT
- La différence de performance peut être de **10x à 1000x** selon la taille des données
- Pour vérifier l'algorithme : **INFORMATION_SCHEMA.VIEWS** ou **EXPLAIN**
- **Préférez MERGE** : Simplifiez les vues pour éviter TEMPTABLE
- Pour les vues TEMPTABLE fréquentes : Remplacez par des **tables de cache**
- Utilisez **EXPLAIN** et **profiling** pour diagnostiquer les problèmes
- Créez des **index appropriés** sur les tables sous-jacentes pour optimiser les vues MERGE

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 View Algorithms](https://mariadb.com/kb/en/view-algorithms/) - MERGE vs TEMPTABLE
- [📖 CREATE VIEW - ALGORITHM](https://mariadb.com/kb/en/create-view/#algorithm) - Syntaxe et options
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/) - Analyse des plans d'exécution
- [📖 Optimizing Queries](https://mariadb.com/kb/en/optimization-and-indexes/) - Optimisation générale

### Articles techniques
- **"Understanding MySQL View Algorithms"** - Percona Blog
- **"Performance Impact of MERGE vs TEMPTABLE"** - MariaDB Blog
- **"Optimizing View Performance"** - High Performance MySQL

### Outils de diagnostic
- **pt-query-digest** (Percona Toolkit) - Analyse des requêtes lentes
- **mysqltuner** - Analyse de configuration et performance
- **EXPLAIN Analyzer** - Outils graphiques pour visualiser les plans

---

## ➡️ Section suivante

**[9.7 Vues système](./07-vues-systeme.md)** : Exploration approfondie des vues système essentielles (INFORMATION_SCHEMA, PERFORMANCE_SCHEMA, mysql system tables), leur utilité pour l'administration, le monitoring et l'optimisation, avec des requêtes pratiques pour exploiter ces métadonnées.

---


⏭️ [Vues système](/09-vues-et-donnees-virtuelles/07-vues-systeme.md)
