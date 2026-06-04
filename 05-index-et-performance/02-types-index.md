🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 — Types d'index

> **Chapitre 5 — Index et Performance** · Section 5.2  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La [section précédente](01-fonctionnement-index.md) a présenté le B-Tree comme la structure d'indexation de référence. Mais MariaDB ne se limite pas à une seule structure : il en propose **plusieurs**, car aucune n'est universellement optimale. Rechercher un mot dans un long texte, trouver les points géographiques les plus proches d'une position, ou retrouver les vecteurs sémantiquement similaires à une requête sont des problèmes de nature très différente — et chacun appelle une structure de données spécialement conçue pour lui.

Choisir le bon type d'index, c'est donc d'abord identifier la **nature de la recherche** que l'on souhaite accélérer, puis sélectionner la structure qui y répond. Cette section dresse le panorama des types disponibles dans MariaDB et fournit les repères nécessaires pour s'orienter ; les sous-sections suivantes les approfondiront un à un.

---

## Deux sens du mot « type d'index »

Avant tout, levons une ambiguïté de vocabulaire. L'expression « type d'index » recouvre deux notions distinctes :

1. **La structure de données sous-jacente** — B-Tree, Hash, index inversé (*full-text*), R-Tree (spatial), HNSW (vectoriel). C'est elle qui détermine *quelles opérations* l'index sait accélérer. **C'est l'objet de cette section.**
2. **La classification fonctionnelle** — un index peut par ailleurs être *unique* ou non, *composite* (sur plusieurs colonnes), *préfixe* (sur les premiers caractères d'une chaîne), *couvrant*, ou encore *invisible*. Ces qualificatifs ne changent pas la structure de l'index, mais la façon de l'utiliser. Ils sont traités dans les sections dédiées : stratégies d'indexation ([5.5](05-strategies-indexation.md)), index composites ([5.6](06-index-composites.md)), index couvrants ([5.9](09-index-covering.md)) et index invisibles ([5.10](10-invisible-progressive-indexes.md)).

Dans toute la suite de cette section, « type d'index » désigne donc la **structure de données**.

---

## Panorama des structures d'index

### B-Tree — le généraliste

C'est la structure **par défaut** et de très loin la plus utilisée. Parce qu'elle maintient les valeurs **triées**, elle répond efficacement à l'égalité, aux plages (`BETWEEN`, `>`, `<`), aux recherches par préfixe (`LIKE 'abc%'`) et aux tris (`ORDER BY`). Dans la quasi-totalité des cas courants, c'est le bon choix — au point qu'on la considère comme l'option « par défaut tant qu'il n'y a pas de raison de faire autrement ». Elle est détaillée en [section 5.2.1](02.1-btree.md).

### Hash — l'égalité pure

Un index Hash applique une **fonction de hachage** à la clé pour calculer directement l'emplacement de la valeur. Il offre ainsi un accès en temps quasi constant (**O(1)**) pour les recherches d'**égalité stricte** (`=`, `IN`). En contrepartie, comme il ne conserve aucun ordre, il est **incapable** de servir une recherche de plage, un tri ou un préfixe. Son usage est donc de niche. Dans MariaDB, c'est la structure **native du moteur MEMORY** ; InnoDB, lui, n'expose pas d'index Hash défini par l'utilisateur — il dispose en revanche d'un *Adaptive Hash Index* interne, géré automatiquement (voir [chapitre 15](../15-performance-tuning/13-adaptive-hash-index.md)). Détails en [section 5.2.2](02.2-hash.md).

### Full-Text — la recherche textuelle

Pour rechercher des **mots à l'intérieur** d'un texte long (articles, descriptions de produits, commentaires), un index B-Tree est inopérant : une condition comme `LIKE '%terme%'` ne peut pas l'exploiter. L'index *full-text* répond à ce besoin grâce à un **index inversé**, qui associe chaque mot aux documents qui le contiennent. On l'interroge via la syntaxe `MATCH (...) AGAINST (...)`, avec un mode *langage naturel* ou *booléen*, et un **score de pertinence**. Il est pris en charge par InnoDB, MyISAM et Aria. Détails en [section 5.2.3](02.3-full-text.md).

### Spatial (R-Tree) — les données géographiques

Pour les données **géométriques** (`GEOMETRY`, `POINT`, `POLYGON`…), MariaDB propose l'index spatial fondé sur une structure **R-Tree**. Celle-ci indexe les rectangles englobants (*Minimum Bounding Rectangle*) des objets afin de répondre rapidement aux questions de relation spatiale : qu'y a-t-il **à proximité** ? quel polygone **contient** ce point ? quels objets **s'intersectent** ? C'est l'outil des applications de cartographie, de géolocalisation et de SIG. Il est pris en charge par InnoDB, MyISAM et Aria. Détails en [section 5.2.4](02.4-spatial.md).

### Vector (HNSW) — la similarité pour l'IA

Enfin, pour la **recherche par similarité** sur des vecteurs (les *embeddings* produits par les modèles d'IA), MariaDB fournit l'index **HNSW** (*Hierarchical Navigable Small Worlds*), fondé sur une structure de **graphe navigable**. Il permet de retrouver les *k* vecteurs les plus proches d'un vecteur de requête, et constitue le socle des cas d'usage de RAG, de recherche sémantique et de recommandation. En raison de son importance pour l'IA, il fait l'objet d'une section à part entière : la [section 5.3](03-index-vector-hnsw.md).

---

## Tableau comparatif

| Structure | À quoi elle sert | Opérations clés | Types de données | Section |
|-----------|------------------|-----------------|------------------|---------|
| **B-Tree** | Usage général | `=`, plages, préfixe, `ORDER BY`, min/max | La plupart des types scalaires | [5.2.1](02.1-btree.md) |
| **Hash** | Égalité stricte | `=`, `IN` (pas de plage ni de tri) | Types scalaires (MEMORY) | [5.2.2](02.2-hash.md) |
| **Full-Text** | Recherche de mots dans du texte | `MATCH … AGAINST` (pertinence) | `CHAR`, `VARCHAR`, `TEXT` | [5.2.3](02.3-full-text.md) |
| **Spatial (R-Tree)** | Relations géométriques | Proximité, contenance, intersection | `GEOMETRY` et sous-types | [5.2.4](02.4-spatial.md) |
| **Vector (HNSW)** | Similarité de vecteurs (IA) | Plus proches voisins (*k-NN*) | `VECTOR(n)` | [5.3](03-index-vector-hnsw.md) |

---

## Dépendance au moteur de stockage et syntaxe

La disponibilité d'un type d'index **dépend du moteur de stockage** de la table. Tous ne prennent pas en charge toutes les structures :

| Structure | InnoDB | MyISAM | Aria | MEMORY |
|-----------|:------:|:------:|:----:|:------:|
| B-Tree | ✅ *(B+Tree)* | ✅ | ✅ | ✅ |
| Hash | interne uniquement¹ | ❌ | ❌ | ✅ *(défaut)* |
| Full-Text | ✅ | ✅ | ✅ | ❌ |
| Spatial (R-Tree) | ✅ | ✅ | ✅ | ❌ |
| Vector (HNSW) | ✅ | ✅ | ✅ | ❌ |

> ¹ InnoDB ne propose pas d'index Hash défini par l'utilisateur, mais un *Adaptive Hash Index* construit et géré automatiquement par le moteur.

Côté **syntaxe**, le choix du type se traduit ainsi à la création :

- Par **défaut**, un index est un **B-Tree** (on peut l'expliciter avec `USING BTREE`).
- Le moteur **MEMORY** crée des index **Hash** par défaut ; on peut y forcer un B-Tree avec `USING BTREE`.
- Les index **full-text**, **spatiaux** et **vectoriels** requièrent un mot-clé explicite : respectivement `FULLTEXT`, `SPATIAL` et `VECTOR`.

La syntaxe complète de création et de gestion (`CREATE INDEX`, `ALTER TABLE … ADD INDEX`, etc.) est présentée en [section 5.4](04-creation-gestion-index.md).

---

## Guide de choix rapide

Pour orienter le choix en fonction du besoin :

| Besoin | Type recommandé |
|--------|-----------------|
| Recherche par égalité, plage, ou tri sur des colonnes classiques | **B-Tree** (le réflexe par défaut) |
| Égalité stricte uniquement, sur des tables temporaires en mémoire | **Hash** (moteur MEMORY) |
| Trouver des **mots** dans un texte long | **Full-Text** |
| Interroger des **coordonnées** ou des formes géographiques | **Spatial** |
| Trouver les éléments **sémantiquement proches** (embeddings, IA) | **Vector (HNSW)** |

En pratique, **le B-Tree couvre la grande majorité des situations**. Les autres structures répondent à des besoins spécialisés et bien identifiés ; on les choisit délibérément, lorsque la nature de la donnée ou de la recherche l'exige.

---

> ### 📝 À retenir  
>  
> - MariaDB propose **plusieurs structures d'index**, chacune adaptée à une nature de recherche : **B-Tree**, **Hash**, **Full-Text**, **Spatial (R-Tree)** et **Vector (HNSW)**.  
> - « Type d'index » désigne ici la **structure de données** ; les notions d'index *unique*, *composite*, *préfixe* ou *couvrant* relèvent d'une autre classification, traitée dans des sections distinctes.  
> - Le **B-Tree** reste le **choix par défaut** et couvre l'essentiel des besoins ; les autres types sont réservés à des cas spécialisés.  
> - La disponibilité d'un type **dépend du moteur de stockage** : InnoDB couvre B-Tree, Full-Text, Spatial et Vector ; le moteur MEMORY est le seul à exposer des index Hash utilisateur.  
> - Les types `FULLTEXT`, `SPATIAL` et `VECTOR` exigent un **mot-clé explicite** à la création ; le B-Tree est implicite.

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.1 Fonctionnement des index : structure B-Tree](01-fonctionnement-index.md)
- ➡️ Section suivante : [5.2.1 B-Tree : le standard](02.1-btree.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [B-Tree : Le standard](/05-index-et-performance/02.1-btree.md)
