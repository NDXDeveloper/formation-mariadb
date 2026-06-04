🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 — Index VECTOR (HNSW) pour la recherche vectorielle

> **Chapitre 5 — Index et Performance** · Section 5.3  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Les types d'index vus jusqu'ici répondent à des questions de la forme « est-ce égal ? », « est-ce dans cette plage ? » ou « ce mot est-il présent ? ». L'index **VECTOR**, lui, répond à une question d'une nature entièrement différente : « **qu'est-ce qui ressemble le plus à ceci ?** ». C'est la recherche **par similarité**, au cœur des applications d'intelligence artificielle modernes — recherche sémantique, RAG (*Retrieval-Augmented Generation*), moteurs de recommandation. Disponible depuis MariaDB 11.7 et stabilisée dans la 11.8 LTS, la recherche vectorielle est aujourd'hui une fonctionnalité standard de MariaDB, que la série 12.x a encore optimisée.

Cette section traite l'**aspect index** de la fonctionnalité : la structure HNSW, sa création et son interrogation efficace. Le traitement complet de MariaDB Vector — fonctions de conversion, optimisations SIMD, intégration avec les LLMs — fait l'objet de la [section 18.10](../18-fonctionnalites-avancees/10-mariadb-vector.md) ; les cas d'usage IA sont développés au [chapitre 20](../20-cas-usage-architectures/09-use-cases-ia-rag.md).

---

## Le problème : la recherche par similarité

Un modèle d'IA (modèle d'*embeddings*) transforme un contenu — un texte, une image, un son — en un **vecteur** : une liste de plusieurs centaines ou milliers de nombres décimaux qui en capture le « sens » sous forme numérique. La propriété fondamentale de ces vecteurs : **deux contenus de sens proche produisent des vecteurs géométriquement proches** dans l'espace. « Médecin » et « docteur » aboutiront à des vecteurs voisins ; « médecin » et « bicyclette », à des vecteurs éloignés.

La recherche sémantique consiste alors à : transformer la requête en vecteur, puis **trouver les vecteurs stockés les plus proches** de celui-ci. Ce n'est ni une égalité (on ne cherche jamais un vecteur *identique*), ni une plage : c'est une recherche des **k plus proches voisins** (*k-Nearest Neighbors*, k-NN). Aucune des structures précédentes ne sait la traiter — d'où une structure d'index dédiée.

---

## ANN et l'algorithme HNSW

Calculer la distance entre la requête et **chacun** des vecteurs stockés donnerait le résultat exact, mais ce serait un balayage complet : impraticable sur des millions de vecteurs de grande dimension. On accepte donc un compromis : renoncer à l'exactitude absolue pour gagner énormément en vitesse. C'est le principe de la recherche **approximative** des plus proches voisins (*Approximate Nearest Neighbor*, **ANN**).

MariaDB met en œuvre une variante de l'algorithme **HNSW** (*Hierarchical Navigable Small Worlds*). Son principe : les vecteurs forment un **graphe**, dans lequel chaque vecteur (nœud) est relié par des arêtes à ses voisins. Une recherche **part d'un nœud d'entrée et « marche » de proche en proche** le long des arêtes, en se rapprochant à chaque pas du vecteur recherché, jusqu'à converger vers son voisinage. L'organisation en couches hiérarchiques permet de franchir d'abord de grandes distances, puis d'affiner localement. Le résultat est *quasi* toujours correct, pour un coût de recherche très faible.

Détail d'implémentation notable : pour chaque index vectoriel, MariaDB maintient une **table interne « fantôme »**, invisible de l'utilisateur, qui stocke les vecteurs et, pour chacun, la liste de ses voisins. C'est ce choix qui confère à l'index vectoriel les garanties **transactionnelles** d'un vrai index de SGBD (voir plus bas), là où des solutions ajoutées par-dessus une base ne les offrent pas toujours.

---

## Le type `VECTOR` et l'index HNSW

MariaDB fournit un type de données dédié, **`VECTOR(N)`**, où `N` est le **nombre de dimensions** — qui doit correspondre à la sortie du modèle d'*embeddings* utilisé (par exemple 1536 dimensions pour certains modèles OpenAI). La colonne stocke des nombres flottants **32 bits** (IEEE 754).

L'index se déclare avec **`VECTOR INDEX`**, et obéit à deux contraintes strictes :

- la colonne indexée doit être **`NOT NULL`** ;
- il ne peut y avoir qu'**un seul index vectoriel par table**.

```sql
CREATE TABLE documents (
  id        BIGINT UNSIGNED PRIMARY KEY,
  contenu   TEXT,
  embedding VECTOR(1536) NOT NULL,              -- 1536 dimensions
  VECTOR INDEX (embedding) M=8 DISTANCE=cosine  -- index HNSW, métrique cosinus
) ENGINE = InnoDB;
```

Deux **options** configurent l'index :

| Option | Rôle | Détails |
|--------|------|---------|
| **`M`** | Nombre maximal d'arêtes par nœud du graphe HNSW | Plage **3 à 200**. Une valeur plus élevée améliore la **précision** (le rappel) au prix d'un index plus volumineux, de plus de mémoire et d'`INSERT`/`SELECT` plus lents. **Fixée à la création**, elle n'est pas modifiable ensuite. |
| **`DISTANCE`** | Fonction de distance pour laquelle l'index est construit | `euclidean` (par défaut) ou `cosine`. **Point capital** : une recherche utilisant une **autre** fonction de distance **ne pourra pas exploiter l'index**. |

La dimensionnalité de l'index est **fixe** : tout vecteur inséré doit avoir exactement `N` dimensions, sous peine d'erreur.

---

## Insérer des vecteurs

Un vecteur s'insère depuis sa représentation textuelle — un tableau JSON de nombres — convertie en binaire par la fonction **`VEC_FromText()`** (la fonction inverse, `VEC_ToText()`, restitue le texte). Ces fonctions de conversion sont détaillées en [section 18.10.4](../18-fonctionnalites-avancees/10.4-fonctions-conversion.md).

```sql
INSERT INTO documents (id, contenu, embedding)
VALUES (1, 'Texte du document…',
        VEC_FromText('[0.012, -0.034, 0.561, … ]'));   -- 1536 valeurs
```

---

## Interroger : les k plus proches voisins

La recherche s'exprime en **triant par la distance** à un vecteur de requête et en limitant le nombre de résultats. La fonction de distance doit correspondre à celle de l'index :

- `VEC_DISTANCE_EUCLIDEAN(colonne, vecteur)` pour un index `DISTANCE=euclidean` ;
- `VEC_DISTANCE_COSINE(colonne, vecteur)` pour un index `DISTANCE=cosine` ;
- ou, plus simple et plus sûr, la fonction générique **`VEC_DISTANCE(colonne, vecteur)`**, qui choisit **automatiquement** la métrique de l'index — évitant tout risque de discordance entre la requête et l'index.

```sql
SET @q = VEC_FromText('[0.01, -0.02, … ]');   -- vecteur de la requête

SELECT id, contenu
FROM documents
ORDER BY VEC_DISTANCE_COSINE(embedding, @q)   -- distance croissante = du plus proche…
LIMIT 5;                                       -- … aux 5 plus proches voisins
```

---

## ⚠️ La condition d'utilisation de l'index

C'est le point **le plus important** de cette section, et une source d'erreurs fréquente. L'index HNSW n'est exploité **que** lorsque la requête respecte une forme **précise** :

> un `ORDER BY` portant sur l'appel `VEC_DISTANCE_*(colonne, vecteur)` **littéral** (ou son alias), trié en ordre **croissant**, **accompagné d'un `LIMIT`**.

Deux formulations très courantes **désactivent silencieusement** l'index et provoquent un **balayage complet** :

**1. Envelopper la distance dans une expression.** Trier par un score de similarité (par exemple `1 - distance_cosinus`) empêche l'optimiseur de reconnaître l'appel de distance. La solution consiste à calculer le score dans une **requête englobante**, en conservant à l'intérieur un `ORDER BY` sur la distance « nue » :

```sql
-- ✅ L'index est utilisé : l'ORDER BY interne porte sur la distance nue (via son alias)
SELECT id, 1 - d AS score
FROM (
  SELECT id, VEC_DISTANCE_COSINE(embedding, @q) AS d
  FROM documents
  ORDER BY d
  LIMIT 5
) AS proches
ORDER BY score DESC;
```

**2. Filtrer par un seuil de distance.** Une condition de la forme `WHERE VEC_DISTANCE_COSINE(...) < 0.2`, sans `ORDER BY … LIMIT`, est un **prédicat de plage** que l'index HNSW ne sait pas piloter :

```sql
-- ❌ Pas d'ORDER BY … LIMIT → l'index n'est pas utilisé, balayage complet
SELECT id
FROM documents
WHERE VEC_DISTANCE_COSINE(embedding, @q) < 0.2;
```

Pour obtenir un comportement indexé, il faut revenir à la forme « tri sur la distance nue + `LIMIT` ». Comme toujours, on vérifie l'usage effectif de l'index avec `EXPLAIN` ([section 5.7](07-analyse-plans-execution.md)).

---

## Caractère approximatif et réglage du compromis

Parce que la recherche est **approximative**, elle s'ajuste sur un curseur entre **vitesse** et **rappel** (la proportion de « vrais » plus proches voisins effectivement retrouvés) :

- à la **création**, le paramètre `M` agit sur la densité du graphe (voir plus haut) ;
- à l'**exécution**, des variables système préfixées `mhnsw_` permettent d'affiner le compromis — notamment un cache dédié et un multiplicateur élargissant l'exploration du graphe pour gagner en précision. Leur réglage relève du tuning et est abordé aux [sections 15.16](../15-performance-tuning/16-vector-optimisations.md) et [18.10](../18-fonctionnalites-avancees/10-mariadb-vector.md).

---

## Intégration transactionnelle

Un atout différenciant de l'approche MariaDB : l'index vectoriel est **pleinement transactionnel**. Les **lectures et écritures concurrentes** ainsi que **tous les niveaux d'isolation** sont pris en charge. Les vecteurs cohabitent avec les colonnes relationnelles classiques dans une **même table**, ce qui permet de combiner naturellement filtrage SQL et recherche de similarité (recherche hybride, voir [section 20.9.4](../20-cas-usage-architectures/09.4-hybrid-search.md)) — sans introduire de base de données spécialisée, ni de dialecte ou d'API tiers.

---

## 🆕 Optimisations de la série 12.x

La série 12.x, consolidée dans la **12.3 LTS**, a renforcé les performances de la recherche vectorielle, notamment par un **calcul de distance directement au niveau du moteur de stockage** et une **estimation de distance par extrapolation** qui réduit le nombre de calculs exacts nécessaires. Ces optimisations sont détaillées aux [sections 15.16](../15-performance-tuning/16-vector-optimisations.md) et [18.10.7](../18-fonctionnalites-avancees/10.7-optimisations-12-3.md).

---

## Pour aller plus loin

- **Fonctionnalité complète** (conversion, SIMD, LLMs) : [section 18.10](../18-fonctionnalites-avancees/10-mariadb-vector.md)
- **Le moteur Vector/HNSW** : [section 7.7](../07-moteurs-de-stockage/07-moteur-vector-hnsw.md)
- **Cas d'usage IA (RAG, recommandation, recherche hybride)** : [chapitre 20](../20-cas-usage-architectures/09-use-cases-ia-rag.md)
- **Recherche par mots-clés** (complémentaire de la recherche sémantique) : [section 5.2.3](02.3-full-text.md)

---

> ### 📝 À retenir  
>  
> - L'index VECTOR répond à la recherche **par similarité** (les **k plus proches voisins**), au service des applications d'IA — un problème distinct de l'égalité, de la plage ou du mot-clé.  
> - Il s'appuie sur l'algorithme **HNSW** (un **graphe** parcouru de proche en proche) pour une recherche **approximative** (ANN), rapide au prix d'une exactitude non garantie.  
> - On déclare un type **`VECTOR(N)`** et un **`VECTOR INDEX`** ; contraintes : colonne **`NOT NULL`** et **un seul index vectoriel par table**. Options : **`M`** (3–200, précision vs coût, fixée à la création) et **`DISTANCE`** (`euclidean` par défaut, ou `cosine`).  
> - La **fonction de distance de la requête doit correspondre** à celle de l'index, sinon l'index est ignoré.  
> - **Forme indexée obligatoire** : `ORDER BY VEC_DISTANCE_*(colonne, vecteur)` **nu**, **croissant**, **avec `LIMIT`**. Envelopper la distance dans une expression ou filtrer par un seuil **désactive l'index**.  
> - L'index est **transactionnel** (lectures/écritures concurrentes, tous niveaux d'isolation) ; la **12.x** y ajoute des optimisations de calcul de distance.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.2.4 Spatial : données géographiques](02.4-spatial.md)
- ➡️ Section suivante : [5.4 Création et gestion des index](04-creation-gestion-index.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Création et gestion des index](/05-index-et-performance/04-creation-gestion-index.md)
