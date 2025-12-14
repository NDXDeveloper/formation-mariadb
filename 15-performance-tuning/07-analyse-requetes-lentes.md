🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7 Analyse des requêtes lentes

> **Niveau** : Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : 
> - Sections 15.1-15.6 (Méthodologie, Mémoire, I/O, InnoDB, alter_copy_bulk)
> - Maîtrise du SQL et de l'optimisation de requêtes
> - Compréhension de EXPLAIN et des plans d'exécution
> - Expérience en diagnostic de performance

---

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Identifier les requêtes problématiques** dans un workload de production
- **Comprendre la règle des 80/20** appliquée aux performances SQL
- **Choisir les outils appropriés** pour chaque type d'analyse
- **Mettre en place une méthodologie** systématique d'optimisation
- **Utiliser les métriques clés** pour prioriser les optimisations
- **Appliquer le workflow d'analyse** de bout en bout
- **Éviter les pièges courants** de l'optimisation de requêtes
- **Exploiter les nouveautés MariaDB 11.8** pour l'analyse

---

## Introduction

L'optimisation des requêtes est souvent **le levier de performance le plus impactant** après la configuration système. Une seule requête mal optimisée peut :

- 🔴 Consommer 80% des ressources CPU
- 🔴 Saturer les I/O disque
- 🔴 Bloquer des centaines de transactions
- 🔴 Causer des timeouts applicatifs
- 🔴 Dégrader l'expérience utilisateur globale

### La règle des 80/20 en SQL

```
┌────────────────────────────────────────────────────┐
│         RÈGLE DE PARETO APPLIQUÉE AU SQL           │
├────────────────────────────────────────────────────┤
│                                                    │
│  Dans un système typique :                         │
│                                                    │
│  📊 20% des requêtes uniques                       │
│     consomment 80% des ressources                  │
│                                                    │
│  📊 1-5% des requêtes (top queries)                │
│     sont responsables de 50%+ du temps total       │
│                                                    │
│  Implication :                                     │
│  ────────────                                      │
│  Optimiser les 10-20 requêtes les plus lentes      │
│  = Gain de 50-80% sur les performances globales    │
│                                                    │
│  ⚠️ Ne PAS optimiser toutes les requêtes !         │
│  Focus sur celles qui ont le plus d'impact         │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Exemple concret

```sql
-- Application e-commerce avec 500 types de requêtes différentes

-- Top 5 requêtes par temps cumulé (50% du temps total)
-- 1. Dashboard admin : Agrégation sur 100M lignes    → 35% temps
-- 2. Recherche produits : Full text sans index      → 8% temps
-- 3. Calcul recommandations : 10 subqueries         → 4% temps
-- 4. Export CSV : Scan complet sans WHERE           → 2% temps
-- 5. Rapport ventes : GROUP BY sans index           → 1% temps

-- Optimiser ces 5 requêtes = Gain de 50% sur tout le système
-- vs optimiser les 495 autres = Gain <5%

-- Conclusion : Identifier et prioriser est CRUCIAL
```

---

## Philosophie de l'analyse de requêtes

### Les 4 piliers de l'optimisation SQL

```
┌────────────────────────────────────────────────────┐
│  1. MESURER (Don't guess, measure)                 │
├────────────────────────────────────────────────────┤
│  • Collecter des données objectives                │
│  • Métriques quantifiables                         │
│  • Baseline avant optimisation                     │
│  • Comparaison avant/après                         │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  2. PRIORISER (Impact vs Effort)                   │
├────────────────────────────────────────────────────┤
│  • Temps cumulé (fréquence × durée)                │
│  • Impact utilisateur                              │
│  • Facilité d'optimisation                         │
│  • Risque de régression                            │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  3. COMPRENDRE (Root cause)                        │
├────────────────────────────────────────────────────┤
│  • Analyser le plan d'exécution                    │
│  • Identifier le bottleneck réel                   │
│  • Ne pas traiter les symptômes                    │
│  • Comprendre "pourquoi" pas juste "quoi"          │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  4. VALIDER (Verify improvement)                   │
├────────────────────────────────────────────────────┤
│  • Tester en staging d'abord                       │
│  • Mesurer le gain réel                            │
│  • Surveiller les effets de bord                   │
│  • Rollback plan prêt                              │
└────────────────────────────────────────────────────┘
```

### Méthodologie en 7 étapes

```sql
-- ÉTAPE 1 : IDENTIFIER les requêtes problématiques
-- Outils : Slow query log, Performance Schema, monitoring

-- ÉTAPE 2 : COLLECTER les métriques
-- • Temps d'exécution
-- • Fréquence d'exécution
-- • Lignes examinées vs retournées
-- • Ressources consommées (CPU, I/O, locks)

-- ÉTAPE 3 : PRIORISER par impact
-- Formule : Impact = Fréquence × Temps_moyen × Criticité_business

-- ÉTAPE 4 : ANALYSER le plan d'exécution
-- EXPLAIN, EXPLAIN ANALYZE, SHOW WARNINGS

-- ÉTAPE 5 : OPTIMISER
-- Index, réécriture requête, dénormalisation, cache

-- ÉTAPE 6 : TESTER
-- Staging, load testing, comparaison métriques

-- ÉTAPE 7 : DÉPLOYER et MONITORER
-- Production progressive, surveillance continue
```

---

## Vue d'ensemble des outils disponibles

### Carte des outils d'analyse

```
┌───────────────────────────────────────────────────────────┐
│               OUTILS D'ANALYSE MARIADB                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  NIVEAU 1 : IDENTIFICATION GLOBALE                        │
│  ───────────────────────────────────                      │
│  • Slow Query Log                                         │
│    → Enregistre requêtes > seuil                          │
│    → Simple, natif, production-ready                      │
│    → Analyse post-mortem                                  │
│                                                           │
│  • pt-query-digest (Percona)                              │
│    → Analyse et agrégation slow log                       │
│    → Rapport détaillé par pattern                         │
│    → Outil externe mais essentiel                         │
│                                                           │
│  NIVEAU 2 : MONITORING TEMPS RÉEL                         │
│  ──────────────────────────────                           │
│  • Performance Schema                                     │
│    → Instrumentation fine-grained                         │
│    → Temps réel, bas overhead                             │
│    → Tableaux système complets                            │
│                                                           │
│  • sys Schema                                             │
│    → Vues simplifiées sur Performance Schema              │
│    → Rapports pré-construits                              │
│    → Facilite l'analyse                                   │
│                                                           │
│  NIVEAU 3 : ANALYSE DÉTAILLÉE                             │
│  ─────────────────────────                                │
│  • EXPLAIN / EXPLAIN ANALYZE                              │
│    → Plan d'exécution détaillé                            │
│    → Coûts estimés et réels                               │
│    → Par requête                                          │
│                                                           │
│  • SHOW PROFILE                                           │
│    → Profiling CPU/IO par étape                           │
│    → Déprécié, utiliser Performance Schema                │
│                                                           │
│  NIVEAU 4 : MONITORING EXTERNE                            │
│  ───────────────────────────                              │
│  • PMM (Percona Monitoring)                               │
│  • Datadog, New Relic, etc.                               │
│  • Grafana + Prometheus                                   │
│    → Dashboards visuels                                   │
│    → Alerting automatique                                 │
│    → Historique long terme                                │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Matrice de sélection d'outil

```
┌──────────────────────────────────────────────────────────────┐
│  BESOIN                    OUTIL RECOMMANDÉ                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  "Quelles sont mes requêtes    → Slow Query Log              │
│   les plus lentes ?"           → pt-query-digest             │
│                                                              │
│  "Cette requête spécifique     → EXPLAIN ANALYZE             │
│   est lente, pourquoi ?"       → Performance Schema          │
│                                                              │
│  "Quelles tables sont          → sys.schema_table_statistics │
│   les plus sollicitées ?"      → sys.io_global_by_file       │
│                                                              │
│  "Quels index ne servent       → sys.schema_unused_indexes   │
│   jamais ?"                    → Performance Schema          │
│                                                              │
│  "Y a-t-il des locks           → SHOW ENGINE INNODB STATUS   │
│   qui bloquent ?"              → Performance Schema locks    │
│                                                              │
│  "Monitoring production        → PMM (Percona)               │
│   en continu ?"                → Grafana + Prometheus        │
│                                                              │
│  "Audit historique des         → Slow log rotaté             │
│   performances ?"              → Stockage long terme         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Métriques clés pour l'analyse

### Les 8 métriques essentielles

```sql
-- Vue d'ensemble des métriques critiques

-- 1. TEMPS D'EXÉCUTION (Query Time)
--    La métrique la plus évidente
--    Mais attention : pas toujours la plus importante !

-- 2. FRÉQUENCE (Execution Count)
--    Une requête 0.1s exécutée 10,000x/min = 1000s/min !
--    vs requête 10s exécutée 1x/min = 10s/min

-- 3. TEMPS CUMULÉ (Total Time)
--    = Temps moyen × Fréquence
--    Meilleure métrique pour prioriser

-- 4. ROWS EXAMINED vs ROWS SENT
--    Ratio élevé = inefficacité
--    Exemple : Examine 1M lignes, retourne 10 → Problème !

-- 5. LOCK TIME
--    Temps passé à attendre des locks
--    Indique contention

-- 6. ROWS SORTED
--    Tris en mémoire vs sur disque
--    Tri sur disque = très lent

-- 7. TEMP TABLES
--    Tables temporaires créées
--    Disk temp tables = problème

-- 8. INDEX USAGE
--    Full table scan vs index scan
--    Key indicator d'optimisation possible
```

### Formule de priorisation

```
┌────────────────────────────────────────────────────┐
│  SCORE DE PRIORISATION                             │
├────────────────────────────────────────────────────┤
│                                                    │
│  Score = (Temps_cumulé × Criticité) / Effort       │
│                                                    │
│  Temps_cumulé = Temps_moyen × Fréquence            │
│                                                    │
│  Criticité :                                       │
│    5 = Requête critique business                   │
│    3 = Requête importante                          │
│    1 = Requête secondaire                          │
│                                                    │
│  Effort :                                          │
│    1 = Simple (ajouter index)                      │
│    3 = Moyen (réécriture requête)                  │
│    5 = Complexe (refonte architecture)             │
│                                                    │
│  Prioriser : Score élevé d'abord                   │
│                                                    │
└────────────────────────────────────────────────────┘

Exemple :

Requête A :
- Temps moyen : 2s
- Fréquence : 100/min → Temps cumulé = 200s/min
- Criticité : 5 (checkout e-commerce)
- Effort : 1 (index manquant)
→ Score = (200 × 5) / 1 = 1000 ← TOP PRIORITÉ

Requête B :
- Temps moyen : 30s
- Fréquence : 1/heure → Temps cumulé = 30s/heure = 0.5s/min
- Criticité : 1 (rapport admin)
- Effort : 5 (refonte complète)
→ Score = (0.5 × 1) / 5 = 0.1 ← BASSE PRIORITÉ
```

---

## Workflow d'analyse complet

### Vue d'ensemble du processus

```
┌─────────────────────────────────────────────────────────┐
│           WORKFLOW D'ANALYSE DE REQUÊTES                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [1] DÉTECTION                                          │
│      • Alertes monitoring                               │
│      • Plaintes utilisateurs                            │
│      • Analyse proactive                                │
│          ↓                                              │
│  [2] COLLECTE                                           │
│      • Activer slow query log                           │
│      • Capturer échantillon représentatif               │
│      • 24-48h de données minimum                        │
│          ↓                                              │
│  [3] AGRÉGATION                                         │
│      • pt-query-digest sur slow log                     │
│      • Grouper par pattern de requête                   │
│      • Calculer métriques agrégées                      │
│          ↓                                              │
│  [4] IDENTIFICATION                                     │
│      • Top 10-20 requêtes par temps cumulé              │
│      • Filtrer par criticité business                   │
│      • Scorer et prioriser                              │
│          ↓                                              │
│  [5] ANALYSE DÉTAILLÉE                                  │
│      • EXPLAIN ANALYZE de chaque requête                │
│      • Identifier bottleneck (scan, sort, join)         │
│      • Vérifier index existants                         │
│          ↓                                              │
│  [6] OPTIMISATION                                       │
│      • Stratégie : Index, réécriture, cache, etc.       │
│      • Implémenter en staging                           │
│      • Tester performance                               │
│          ↓                                              │
│  [7] VALIDATION                                         │
│      • Comparer métriques avant/après                   │
│      • Load testing                                     │
│      • Vérifier effets de bord                          │
│          ↓                                              │
│  [8] DÉPLOIEMENT                                        │
│      • Production progressive (canary)                  │
│      • Monitoring intensif                              │
│      • Rollback plan prêt                               │
│          ↓                                              │
│  [9] SUIVI                                              │
│      • Surveillance continue                            │
│      • Documenter les changements                       │
│      • Répéter le cycle                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Exemple de workflow appliqué

```sql
-- SCÉNARIO : Application web avec lenteurs signalées

-- [1] DÉTECTION
-- Alerte : Latence p95 passée de 200ms à 2s

-- [2] COLLECTE
-- Activer slow query log pour 24h
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;  -- 100ms
SET GLOBAL log_slow_verbosity = 'query_plan,explain';

-- [3] AGRÉGATION
-- Analyser avec pt-query-digest
$ pt-query-digest /var/log/mysql/slow.log > report.txt

-- [4] IDENTIFICATION
-- Top 3 requêtes identifiées :
/*
1. SELECT * FROM orders WHERE customer_id = ?
   Temps cumulé : 45% du total
   Exécutions : 50k/heure
   Temps moyen : 0.8s
   
2. SELECT p.* FROM products p JOIN inventory i WHERE i.qty < 10
   Temps cumulé : 18% du total
   Exécutions : 500/heure
   Temps moyen : 12s
   
3. SELECT COUNT(*) FROM logs WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 DAY)
   Temps cumulé : 8% du total
   Exécutions : 100/heure
   Temps moyen : 28s
*/

-- [5] ANALYSE DÉTAILLÉE
-- Requête #1
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE customer_id = 12345;
-- Résultat : Table scan, pas d'index sur customer_id !

-- [6] OPTIMISATION
-- Solution : Ajouter index
ALTER TABLE orders ADD INDEX idx_customer (customer_id);

-- [7] VALIDATION
-- Test en staging : 0.8s → 0.003s (250x plus rapide !)

-- [8] DÉPLOIEMENT
-- Production : Activer innodb_alter_copy_bulk
SET SESSION innodb_alter_copy_bulk = ON;
ALTER TABLE orders ADD INDEX idx_customer (customer_id);

-- [9] SUIVI
-- Vérifier amélioration après 24h
-- Latence p95 : 2s → 150ms ✅
```

---

## Comparaison des approches d'analyse

### Approche réactive vs proactive

```
┌────────────────────────────────────────────────────┐
│  RÉACTIVE (Fire-fighting)                          │
├────────────────────────────────────────────────────┤
│  Déclencheur : Problème en production              │
│  Timing : Urgence, pression                        │
│  Scope : Requête spécifique                        │
│  Risque : Décisions hâtives, régression            │
│                                                    │
│  Avantage : Focus sur impact immédiat              │
│  Inconvénient : Pas de vision globale              │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  PROACTIVE (Continuous improvement)                │
├────────────────────────────────────────────────────┤
│  Déclencheur : Audit régulier planifié             │
│  Timing : Réfléchi, méthodique                     │
│  Scope : Top 10-20 requêtes systématiquement       │
│  Risque : Faible, tests approfondis                │
│                                                    │
│  Avantage : Prévention, optimisation durable       │
│  Inconvénient : Nécessite discipline               │
└────────────────────────────────────────────────────┘

RECOMMANDATION : Combiner les deux
──────────────────────────────────

• Réactif : Résoudre urgences (10% temps)
• Proactif : Audit mensuel/trimestriel (90% temps)
• Automatiser la collecte et alerting
• Documenter toutes les optimisations
```

### Top-Down vs Bottom-Up

```sql
-- APPROCHE TOP-DOWN (Recommandée)
-- ─────────────────────────────────
-- Partir de la métrique business vers la requête

-- 1. Métrique business
--    "Temps de chargement page checkout = 5s (vs 1s cible)"

-- 2. Métrique applicative
--    "Endpoint /api/checkout prend 4.5s"

-- 3. Métriques DB
--    "3 requêtes SQL totalisent 4s"

-- 4. Requête spécifique
--    "SELECT * FROM cart_items WHERE cart_id = ? prend 3s"

-- 5. Plan d'exécution
--    "Full table scan sur 50M lignes"

-- 6. Optimisation
--    "Ajouter index sur cart_id"

-- Avantage : Focus sur l'impact utilisateur réel


-- APPROCHE BOTTOM-UP
-- ──────────────────
-- Partir des métriques techniques

-- 1. Monitoring DB
--    "Buffer pool hit rate = 85%"

-- 2. Investigation
--    "Beaucoup d'I/O sur table cart_items"

-- 3. Requêtes
--    "Scan complet sur cart_items fréquent"

-- 4. Impact business ?
--    "Pas clair... Est-ce vraiment un problème utilisateur ?"

-- Risque : Optimiser des choses non critiques
```

---

## Catégories de problèmes de performance

### Taxonomie des problèmes courants

```
┌────────────────────────────────────────────────────┐
│  CATÉGORIE 1 : INDEX MANQUANTS                     │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • Full table scan dans EXPLAIN                    │
│  • Rows examined >> Rows sent                      │
│  • Type: ALL dans EXPLAIN                          │
│                                                    │
│  Exemple :                                         │
│  SELECT * FROM orders WHERE customer_id = 123      │
│  → Table scan sur 10M lignes pour retourner 5      │
│                                                    │
│  Solution :                                        │
│  CREATE INDEX idx_customer ON orders(customer_id); │
│                                                    │
│  Difficulté : ⭐ (Facile)                          │
│  Impact : ⭐⭐⭐⭐⭐ (Très élevé)                 │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  CATÉGORIE 2 : INDEX NON OPTIMAUX                  │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • Index existe mais pas utilisé                   │
│  • Index utilisé partiellement                     │
│  • key_len dans EXPLAIN < optimal                  │
│                                                    │
│  Exemple :                                         │
│  Index : (customer_id)                             │
│  Requête : WHERE customer_id = ? AND status = ?    │
│  → Index sur customer_id seul, puis scan           │
│                                                    │
│  Solution :                                        │
│  CREATE INDEX idx_comp ON orders(customer_id, status);
│                                                    │
│  Difficulté : ⭐⭐ (Moyen)                         │
│  Impact : ⭐⭐⭐⭐ (Élevé)                         │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  CATÉGORIE 3 : REQUÊTES MAL ÉCRITES                │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • Subqueries non corrélées répétées               │
│  • SELECT * au lieu de colonnes spécifiques        │
│  • OR au lieu de UNION                             │
│  • Fonctions sur colonnes indexées dans WHERE      │
│                                                    │
│  Exemple :                                         │
│  WHERE DATE(created_at) = '2024-01-01'             │
│  → Fonction DATE() empêche utilisation index       │
│                                                    │
│  Solution :                                        │
│  WHERE created_at >= '2024-01-01'                  │
│    AND created_at < '2024-01-02'                   │
│                                                    │
│  Difficulté : ⭐⭐⭐ (Moyen-Difficile)             │
│  Impact : ⭐⭐⭐⭐ (Élevé)                         │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  CATÉGORIE 4 : PROBLÈMES DE JOINTURES              │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • Join sans index sur colonnes de jointure        │
│  • Produit cartésien accidentel                    │
│  • Trop de tables joinées (>5-6)                   │
│                                                    │
│  Exemple :                                         │
│  SELECT * FROM a JOIN b JOIN c JOIN d JOIN e       │
│  → Complexité explosive                            │
│                                                    │
│  Solution :                                        │
│  • Index sur colonnes de join                      │
│  • Dénormalisation partielle                       │
│  • Vues matérialisées                              │
│                                                    │
│  Difficulté : ⭐⭐⭐⭐ (Difficile)                 │
│  Impact : ⭐⭐⭐⭐⭐ (Très élevé)                  │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  CATÉGORIE 5 : PROBLÈMES DE TRI/AGRÉGATION         │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • "Using filesort" dans EXPLAIN                   │
│  • "Using temporary" dans EXPLAIN                  │
│  • GROUP BY/ORDER BY sur gros datasets             │
│                                                    │
│  Exemple :                                         │
│  SELECT customer_id, COUNT(*)                      │
│  FROM orders                                       │
│  GROUP BY customer_id                              │
│  ORDER BY COUNT(*) DESC                            │
│  → Tri de millions de groupes                      │
│                                                    │
│  Solution :                                        │
│  • Index couvrant avec ORDER BY                    │
│  • Pré-agrégation dans table summary               │
│  • LIMIT avec index                                │
│                                                    │
│  Difficulté : ⭐⭐⭐ (Moyen-Difficile)             │
│  Impact : ⭐⭐⭐⭐ (Élevé)                         │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  CATÉGORIE 6 : PROBLÈMES DE LOCKS                  │
├────────────────────────────────────────────────────┤
│  Symptôme :                                        │
│  • Lock wait timeout                               │
│  • High lock time dans slow log                    │
│  • Transactions longues                            │
│                                                    │
│  Exemple :                                         │
│  BEGIN;                                            │
│  SELECT ... FOR UPDATE;  -- Lock acquis            │
│  -- Application processing 30s                     │
│  UPDATE ...;                                       │
│  COMMIT;                                           │
│  → Lock tenu trop longtemps                        │
│                                                    │
│  Solution :                                        │
│  • Transactions plus courtes                       │
│  • Row-level locking approprié                     │
│  • Éviter SELECT ... FOR UPDATE si possible        │
│                                                    │
│  Difficulté : ⭐⭐⭐⭐ (Difficile)                 │
│  Impact : ⭐⭐⭐⭐⭐ (Très élevé)                  │
└────────────────────────────────────────────────────┘
```

---

## Dashboard de monitoring

### Vue SQL de santé des requêtes

```sql
-- Vue globale de la santé des performances requêtes
CREATE OR REPLACE VIEW v_query_health AS
SELECT 
    'Slow queries' as metric,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Slow_queries') as current_value,
    CASE 
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Slow_queries') > 1000 
        THEN 'WARNING'
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Slow_queries') > 10000 
        THEN 'CRITICAL'
        ELSE 'OK'
    END as status

UNION ALL

SELECT 
    'Slow queries ratio',
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Slow_queries') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Questions'), 0) * 100,
            2
        ),
        '%'
    ),
    CASE 
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Slow_queries') /
             NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Questions'), 0) * 100 > 5
        THEN 'CRITICAL'
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Slow_queries') /
             NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Questions'), 0) * 100 > 1
        THEN 'WARNING'
        ELSE 'OK'
    END

UNION ALL

SELECT 
    'Full table scans',
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Select_scan'),
    CASE 
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Select_scan') > 100000 
        THEN 'WARNING'
        ELSE 'OK'
    END

UNION ALL

SELECT 
    'Temp tables on disk',
    CONCAT(
        ROUND(
            (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
            NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
             WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100,
            2
        ),
        '%'
    ),
    CASE 
        WHEN (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Created_tmp_disk_tables') /
             NULLIF((SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
              WHERE VARIABLE_NAME = 'Created_tmp_tables'), 0) * 100 > 25
        THEN 'WARNING'
        ELSE 'OK'
    END;

-- Utiliser
SELECT * FROM v_query_health;
```

### Alertes automatiques

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE alert_slow_queries()
BEGIN
    DECLARE v_slow_queries BIGINT;
    DECLARE v_questions BIGINT;
    DECLARE v_ratio DECIMAL(10,2);
    
    SELECT VARIABLE_VALUE INTO v_slow_queries
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Slow_queries';
    
    SELECT VARIABLE_VALUE INTO v_questions
    FROM information_schema.GLOBAL_STATUS
    WHERE VARIABLE_NAME = 'Questions';
    
    SET v_ratio = (v_slow_queries / NULLIF(v_questions, 0)) * 100;
    
    IF v_ratio > 5 THEN
        SELECT CONCAT(
            'CRITICAL: ', v_ratio, '% des requêtes sont lentes (> 5%)',
            ' - Activer slow query log et analyser !'
        ) as alert;
    ELSEIF v_ratio > 1 THEN
        SELECT CONCAT(
            'WARNING: ', v_ratio, '% des requêtes sont lentes (> 1%)',
            ' - Surveiller de près'
        ) as alert;
    ELSE
        SELECT CONCAT(
            'OK: ', v_ratio, '% des requêtes sont lentes'
        ) as status;
    END IF;
END //
DELIMITER ;

-- Appeler périodiquement
CALL alert_slow_queries();
```

---

## Pièges courants à éviter

### Les 10 erreurs fréquentes

```
1. ❌ Optimiser sans mesurer
   ✅ Toujours établir baseline avant optimisation

2. ❌ Optimiser prématurément
   ✅ Attendre d'avoir des données de production réelles

3. ❌ Ajouter des index "au cas où"
   ✅ Index basés sur analyse de workload réel

4. ❌ Ignorer la fréquence d'exécution
   ✅ Temps cumulé = temps moyen × fréquence

5. ❌ Sur-optimiser des requêtes rares
   ✅ Focus sur impact business réel

6. ❌ Tester uniquement avec petits datasets
   ✅ Tester avec volumes production

7. ❌ Optimiser en production directement
   ✅ Staging → Tests → Production progressive

8. ❌ Ne pas documenter les changements
   ✅ Logs de toutes les optimisations

9. ❌ Oublier de monitorer post-déploiement
   ✅ Surveillance continue après changement

10. ❌ Traiter les symptômes, pas la cause
    ✅ Analyse root cause avec EXPLAIN
```

---

## Nouveautés MariaDB 11.8 pour l'analyse

### Améliorations du cost optimizer

```sql
-- Le cost optimizer 11.8 est SSD-aware
-- Impact sur l'analyse :

-- Avant 11.8
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
/*
Optimizer choisit parfois mal sur SSD
Car coûts calibrés pour HDD
*/

-- Avec 11.8 🆕
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
/*
Optimizer détecte SSD automatiquement
Plans d'exécution plus optimaux
Moins de "faux positifs" d'optimisation
*/

-- Vérifier le type de stockage détecté
SHOW VARIABLES LIKE 'optimizer_disk%';
```

### Performance Schema amélioré

```sql
-- MariaDB 11.8 : Instrumentation plus fine

-- Nouvelles métriques disponibles
SELECT * FROM performance_schema.setup_instruments
WHERE name LIKE '%statement%'
AND name LIKE '%11.8%';  -- Features 11.8

-- Overhead réduit : ~5% → ~2%
-- Plus de détails sans impact performance
```

---

## ✅ Points clés à retenir

- 📊 **Règle 80/20** : 20% des requêtes = 80% des ressources
- 🎯 **Mesurer d'abord** : Baseline avant toute optimisation
- 📈 **Temps cumulé** : Métrique #1 pour prioriser (fréquence × durée)
- 🔧 **Outils complémentaires** : Slow log + pt-query-digest + Performance Schema
- 🎬 **Workflow méthodique** : Détecter → Analyser → Optimiser → Valider
- 🆕 **MariaDB 11.8** : Cost optimizer SSD-aware améliore les plans
- ⚠️ **Éviter les pièges** : Pas d'optimisation prématurée
- 📝 **Documenter** : Toutes les optimisations et leurs impacts
- 🔍 **Root cause** : Comprendre pourquoi, pas juste quoi
- 🔄 **Continu** : L'optimisation est un processus, pas un événement

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Slow Query Log](https://mariadb.com/kb/en/slow-query-log/)
- [📖 Performance Schema](https://mariadb.com/kb/en/performance-schema/)
- [📖 sys Schema](https://mariadb.com/kb/en/sys-schema/)
- [📖 EXPLAIN](https://mariadb.com/kb/en/explain/)

### Outils externes

- [Percona Toolkit - pt-query-digest](https://docs.percona.com/percona-toolkit/pt-query-digest.html)
- [Percona Monitoring and Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management)

### Lectures recommandées

- [High Performance MySQL (O'Reilly)](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [SQL Performance Explained (Markus Winand)](https://sql-performance-explained.com/)

---

## ➡️ Sections suivantes

Les sections suivantes détaillent chaque outil d'analyse :

### **Section 15.7.1** : [Slow Query Log](/15-performance-tuning/07.1-slow-query-log.md)
*Configuration, activation, analyse du slow query log. Paramètres critiques et best practices.*

### **Section 15.7.2** : [pt-query-digest](/15-performance-tuning/07.2-pt-query-digest.md)
*Outil Percona pour analyser et agréger les logs. Rapports détaillés et priorisation.*

---

*L'analyse des requêtes lentes est un art autant qu'une science. Une méthodologie rigoureuse et les bons outils font toute la différence entre un DBA efficace et un "fire-fighter" perpétuel.*

⏭️ [Slow query log](/15-performance-tuning/07.1-slow-query-log.md)
