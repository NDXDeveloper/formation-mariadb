🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.7 Vector/HNSW : recherche vectorielle pour l'IA

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS  
> Cette section présente la recherche vectorielle sous l'angle du **stockage**. Le détail complet de la fonctionnalité est traité en §18.10, l'index en §5.3, et les cas d'usage IA/RAG au chapitre 20 (§20.9).

## Une mise au point : une fonctionnalité intégrée, pas un moteur distinct

Malgré son intitulé, **« Vector/HNSW » n'est pas un moteur de stockage séparé** au sens d'InnoDB, d'Aria ou de ColumnStore. C'est une **fonctionnalité intégrée au serveur** MariaDB : un type de données `VECTOR` et un type d'index vectoriel, utilisés **par-dessus un moteur de stockage classique** — en pratique **InnoDB**.

Ce choix d'architecture a des conséquences importantes. Les vecteurs sont stockés dans une **table InnoDB ordinaire**, et l'index vectoriel est implémenté comme un « index de haut niveau » dont la conception s'inspire des index full-text d'InnoDB. Il s'ajoute de façon **indépendante du moteur** et **hérite de toutes les propriétés du moteur sous-jacent** : il fait partie de la même transaction, suit les mêmes règles d'isolation, bénéficie de la récupération après incident et du verrouillage. Le **graphe HNSW en mémoire** n'est, lui, qu'un **cache** de l'index destiné à la performance.

Autrement dit : MariaDB offre la recherche vectorielle **avec des garanties transactionnelles complètes, dans la même base**, sans qu'il faille déployer un magasin vectoriel séparé. C'est ce qui distingue cette section des véritables moteurs du chapitre.

## À quoi sert la recherche vectorielle

Un modèle d'IA (texte, image, audio) transforme une donnée en un **vecteur** d'*embedding* — une suite de nombres qui capture son sens. Des données sémantiquement proches produisent des vecteurs proches selon une **fonction de distance**. La recherche vectorielle consiste donc à retrouver, pour un vecteur de requête, les enregistrements dont les vecteurs sont les plus **proches** : c'est le socle de la recherche sémantique, du **RAG** (génération augmentée par la récupération), des moteurs de recommandation ou de la détection d'anomalies.

La recherche vectorielle a été introduite en **MariaDB 11.7**, intégrée à la **11.8 GA LTS** (premier LTS doté du support vectoriel) et est présente dans la **12.3 LTS**. Les cas d'usage IA sont développés en §20.9.

## Le type `VECTOR` et l'index HNSW

Tout part de deux éléments déclarés dans le `CREATE TABLE` :

- le type **`VECTOR(n)`**, où `n` est le **nombre de dimensions** — il doit correspondre exactement à la sortie du modèle d'*embedding* utilisé ;
- l'index **`VECTOR INDEX (colonne)`**, avec deux paramètres optionnels : `M` (connectivité du graphe HNSW) et `DISTANCE` (la fonction de distance, `euclidean` ou `cosine`).

```sql
CREATE TABLE documents (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  proprietaire_id BIGINT UNSIGNED NOT NULL,       -- ex. multi-tenant : à qui appartient le document
  contenu         TEXT          NOT NULL,
  embedding       VECTOR(1536)  NOT NULL,         -- dimensions = sortie du modèle
  VECTOR INDEX (embedding) M=6 DISTANCE=cosine
) ENGINE = InnoDB;                                -- table InnoDB ordinaire
```

L'algorithme de recherche est une version **modifiée de HNSW** (*Hierarchical Navigable Small Worlds*), qui résout le problème du **plus proche voisin approché** (*Approximate Nearest Neighbor*) — on échange une exactitude parfaite contre une rapidité considérable. Par défaut, la distance utilisée est euclidienne. En interne, MariaDB stocke les valeurs de l'index en `int16` (15 bits utiles), un compromis entre précision et compacité.

## Insérer et interroger des vecteurs

Les vecteurs s'insèrent à partir d'un **tableau JSON de flottants** converti par `VEC_FromText()` (la fonction inverse, `VEC_ToText()`, reconvertit un vecteur en texte) :

```sql
INSERT INTO documents (proprietaire_id, contenu, embedding)
VALUES (1234, 'Texte du document…', VEC_FromText('[0.12, -0.04, 0.88, …]'));
```

La recherche des **k plus proches voisins** s'exprime avec un `ORDER BY` sur une fonction de distance, suivi d'un `LIMIT` :

```sql
SELECT id, contenu
FROM documents
ORDER BY VEC_DISTANCE_COSINE(embedding, VEC_FromText('[…vecteur de la requête…]'))
LIMIT 10;
```

Point essentiel : la **fonction de distance employée dans la requête doit correspondre** à celle avec laquelle l'index a été construit (`DISTANCE=`).

## Les fonctions de distance

MariaDB propose deux mesures de distance, plus une variante automatique :

- **`VEC_DISTANCE_EUCLIDEAN`** : distance en ligne droite (L2). Adaptée aux données brutes non normalisées, mais sensible à la magnitude — donc moins indiquée pour des *embeddings* normalisés de grande dimension.
- **`VEC_DISTANCE_COSINE`** : distance cosinus, fondée sur l'angle entre les vecteurs et insensible à la magnitude — souvent préférée pour les *embeddings* normalisés.
- **`VEC_DISTANCE`** : choisit automatiquement la comparaison appropriée pour l'index concerné.

Le détail de ces fonctions figure en §18.10.3.

## Configuration : les variables `mhnsw_`

Toutes les variables de réglage de la recherche vectorielle commencent par **`mhnsw_`** (le « m » se lit « MariaDB » ou « Modified HNSW ») :

- **`mhnsw_max_cache_size`** : la taille du **cache mémoire** du graphe HNSW (défaut **16 Mo**). C'est le paramètre clé : pour des performances optimales, l'index devrait tenir entièrement en cache (la recherche reste correcte sinon, mais plus lente).
- **`mhnsw_default_m`** : la valeur **M** par défaut des nouveaux index (connectivité du graphe HNSW) ; le `M` d'un index précis se fixe dans la clause `VECTOR INDEX`.
- **`mhnsw_ef_search`** : correspond au paramètre **ef** (largeur de la recherche).
- **`mhnsw_default_distance`** : la fonction de distance par défaut (`euclidean`).

```ini
[mariadb]
mhnsw_max_cache_size = 1G   # cache de l'index HNSW ; idéalement l'index entier (défaut : 16 Mo)
```

L'arbitrage est classique : un `M` et un `ef` plus élevés améliorent le **rappel** (la qualité des résultats) au prix de plus de mémoire et de temps de calcul.

## Performance : le cache et le SIMD

Deux facteurs gouvernent la vitesse de recherche. Le premier est la **mise en cache** de l'index en mémoire (graphe HNSW), évoquée ci-dessus. Le second est le **calcul de distance lui-même** : il représente l'essentiel du temps de recherche — de l'ordre de **80 à 90 %** — et croît **linéairement avec la longueur du vecteur** (les modèles récents produisent des vecteurs de 1536, voire 3072 dimensions). C'est pourquoi MariaDB Vector exploite les jeux d'instructions **SIMD** (AVX2, AVX512, ARM, IBM Power10) pour accélérer ces calculs, comme détaillé en §18.10.5.

> 🆕 La série 12.x apporte des **optimisations supplémentaires** sur ce point — notamment le calcul de distance **par extrapolation** et **au plus près du moteur de stockage** —, détaillées en §15.16 et §18.10.7.

## Intégration transactionnelle et recherche hybride

Parce que l'index vit dans une table InnoDB, MariaDB Vector prend en charge les **lectures et écritures concurrentes** et **tous les niveaux d'isolation**, avec des garanties ACID complètes. Surtout, les vecteurs cohabitent avec les données relationnelles : on peut **combiner, dans une seule requête**, un filtre SQL classique et une recherche de similarité — c'est la **recherche hybride** (vecteurs + SQL relationnel, §20.9.4) :

```sql
SELECT id, contenu
FROM documents
WHERE proprietaire_id = 1234                                   -- filtre relationnel
ORDER BY VEC_DISTANCE_COSINE(embedding, VEC_FromText('[…]'))   -- proximité vectorielle
LIMIT 10;
```

C'est l'un des principaux atouts d'une base relationnelle dotée de vecteurs face à un magasin vectoriel isolé : pas de synchronisation entre deux systèmes, et des filtres relationnels appliqués nativement.

## Quand l'utiliser

La recherche vectorielle de MariaDB s'impose lorsqu'on veut de la **recherche sémantique, du RAG, de la recommandation ou de la détection d'anomalies** **intégrés aux données relationnelles** et à leurs transactions — sans déployer ni synchroniser une base vectorielle dédiée. La table sous-jacente étant InnoDB, on conserve toutes les garanties transactionnelles habituelles. Les autres moteurs du chapitre répondent à d'autres besoins ; la grille de décision figure en §7.8.

## Liens avec d'autres chapitres

- La fonctionnalité **MariaDB Vector** est traitée en détail en §18.10 (type, index, fonctions, SIMD, intégration LLM), et l'**index VECTOR (HNSW)** sous l'angle indexation en §5.3.
- Les **cas d'usage IA/RAG** (recherche sémantique, recommandation, détection d'anomalies, recherche hybride) sont développés en §20.9.
- Les **optimisations 12.x** du calcul de distance sont détaillées en §15.16 et §18.10.7.
- Le moteur **InnoDB** qui porte les tables vectorielles est traité en §7.2, et le type **`VECTOR`** parmi les types MariaDB en §2.2.5.

## Ce qu'il faut retenir

- « Vector/HNSW » n'est **pas un moteur distinct** : c'est une **fonctionnalité du serveur** (type `VECTOR`, index vectoriel) utilisée **sur une table InnoDB**, dont elle hérite des garanties **ACID, transactionnelles et de récupération**.
- On déclare un type **`VECTOR(n)`** (n = dimensions du modèle) et un **`VECTOR INDEX`** (`M`, `DISTANCE=euclidean|cosine`) ; la recherche utilise un **HNSW modifié** (plus proche voisin approché).
- On insère via **`VEC_FromText()`** et on cherche les k plus proches voisins avec **`ORDER BY VEC_DISTANCE_*(…) LIMIT k`** — la fonction de distance devant **correspondre à l'index**.
- Le réglage passe par les variables **`mhnsw_`** (`mhnsw_max_cache_size` en tête, plus `mhnsw_default_m`, `mhnsw_ef_search`) ; la performance dépend du **cache en mémoire** et du **SIMD**, le calcul de distance étant le point chaud.
- L'intégration relationnelle permet la **recherche hybride** (filtre SQL + similarité) en une requête — détails en §18.10, §5.3 et §20.9.

⏭️ [Comparaison et choix du moteur approprié](/07-moteurs-de-stockage/08-comparaison-choix-moteur.md)
