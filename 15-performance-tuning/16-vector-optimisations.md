🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.16 Vector : optimisations 12.3 (distance par extrapolation, calcul dans le moteur de stockage) 🆕

> **Cadrage.** MariaDB Vector — le type `VECTOR`, l'index **HNSW** et les fonctions `VEC_DISTANCE_EUCLIDEAN` / `VEC_DISTANCE_COSINE` — a été livré dans la **11.8** (voir §5.3 et §18.10 pour les aspects fonctionnels). La **12.3 LTS** n'introduit **aucune nouvelle API** : elle accélère le **cœur** de la recherche vectorielle — le calcul de distance — de façon **transparente** pour l'application. Cette section adopte l'angle *performance & tuning* ; pour la mécanique fonctionnelle (création d'index, requêtes), se reporter au §18.10. Fonctionnalité suivie sous **MDEV-36205** (« Faster vector distance calculations via extrapolation »), introduite dans la série rolling 12.x et consolidée dans la 12.3 LTS.

---

## Le goulot d'étranglement : le calcul de distance

Dans une recherche par plus proches voisins sur index HNSW, l'essentiel du temps n'est pas passé à parcourir le graphe mais à **mesurer des distances** entre le vecteur de requête et les vecteurs candidats. Trois constats motivent l'optimisation :

- le calcul de distance représente **80 à 90 %** du temps total d'une recherche vectorielle ;
- son coût est **linéaire en nombre de dimensions** : comparer des vecteurs de 1536 dimensions coûte deux fois plus cher que des vecteurs de 768 dimensions ;
- la tendance va vers des **vecteurs de plus en plus longs** : les derniers modèles d'OpenAI et de Gemini produisent des embeddings de **3072 dimensions**.

Autrement dit, à mesure que les embeddings s'allongent pour gagner en qualité sémantique, la recherche devient mécaniquement plus lente. Réduire le coût du calcul de distance est donc le levier de performance le plus rentable.

> **Rappel.** Côté SQL, la recherche vectorielle prend la forme d'un `ORDER BY VEC_DISTANCE_*(colonne, vecteur_requête) LIMIT n` : l'optimiseur exploite alors l'index HNSW. C'est exactement cette opération que la 12.3 accélère (détails au §18.10).

---

## La distance par extrapolation

### Le principe

Certains modèles (notamment OpenAI) produisent des **embeddings Matryoshka** : un vecteur long peut être **tronqué** à ses premières dimensions et rester exploitable pour la recherche — la recherche est alors plus rapide (moins de dimensions) mais le rappel se dégrade. L'approche manuelle classique consiste à faire du *shortlisting and reranking* : une première passe sur des vecteurs courts tronqués, puis un reclassement des résultats sur les vecteurs complets. C'est efficace mais **laborieux** : l'utilisateur doit raisonner en termes de Matryoshka et écrire des requêtes complexes.

La 12.3 **automatise** ce raisonnement. Plutôt que de calculer la distance euclidienne exacte sur l'intégralité des `N` dimensions, le serveur la calcule sur un **préfixe** de petite longueur `k` (avec `k` très inférieur à `N`), puis l'**extrapole** à la dimension complète :

```text
Distance exacte    :  d_exact  = Σ (aᵢ − bᵢ)²               pour i = 0 … N  
Distance extrapolée :  d_prefix = (N / k) · Σ (aᵢ − bᵢ)²     pour i = 0 … k   (k ≪ N)  
```

L'extrapolation n'est intéressante que si `d_prefix` reste proche de `d_exact`. Pour les embeddings Matryoshka, c'est le cas : le ratio `d_prefix / d_exact` est **quasiment égal à 1 dès environ 100 à 150 dimensions**. Calculer la distance sur un préfixe de cette taille suffit donc à estimer fidèlement la distance réelle.

### L'astuce décisive : le rejet anticipé

Se contenter de `d_prefix` reviendrait à faire du simple *shortlisting* sur vecteurs tronqués, sans jamais profiter de la précision des vecteurs complets. L'idée fine est de **basculer conditionnellement** sur la longueur totale.

Pendant la recherche, on ne s'intéresse pas à la distance *en tant que telle* : on veut seulement savoir si un vecteur candidat est **plus proche** de la requête que ceux déjà retenus, c'est-à-dire si sa distance est **inférieure à un seuil**. D'où la logique :

1. extrapoler la distance à partir d'un **court préfixe** des deux vecteurs ;
2. si l'extrapolation prédit une distance **supérieure au seuil**, le candidat est trop loin → **rejet immédiat**, sans calculer la suite ;
3. sinon, on ne peut pas conclure → on calcule la **distance exacte sur le vecteur complet**.

La grande majorité des candidats étant écartés dès l'étape 2 (en temps réduit), on n'effectue le calcul complet et coûteux que pour les rares vecteurs réellement prometteurs. Le **rappel est préservé**, puisque tout candidat potentiellement pertinent est vérifié sur sa distance exacte.

### Une activation automatique et sans risque

Cette optimisation n'est **pas réservée** aux embeddings Matryoshka, et il serait peu réaliste de demander à l'utilisateur de déclarer une option du type « mes vecteurs ont la propriété d'extrapolabilité ». MariaDB la **détecte donc automatiquement**.

Le critère retenu est l'**écart-type du ratio** `d_prefix / d_exact` : plus il est faible, plus l'extrapolation est fiable. Le mécanisme est prudent :

- pour les **10 000 premiers calculs de distance**, l'optimisation est **désactivée** : MariaDB calcule à la fois la distance extrapolée et la distance exacte, uniquement pour **collecter des statistiques** (il n'utilise pas encore le raccourci) ;
- si l'écart-type du ratio s'avère **inférieur au seuil**, l'optimisation est **activée** et le serveur commence à court-circuiter les calculs ;
- dans le cas contraire, elle est **totalement désactivée**.

Conséquence importante : sur les jeux de données **non éligibles**, il n'y a **ni perte de rappel ni perte de performance** — le serveur retombe simplement sur le calcul exact habituel.

---

## Le calcul porté dans le moteur de stockage

Au-delà de l'algorithme, la 12.3 effectue ces opérations mathématiques **directement dans la couche de stockage**, là où réside l'index HNSW, au plus près des données. Pousser le calcul de distance au niveau du moteur (plutôt que de le réaliser plus haut dans la chaîne d'exécution) évite des allers-retours et du transfert de données inutiles : les candidats peuvent être élagués sur place, pendant le parcours du graphe.

Cette localisation se conjugue idéalement avec l'extrapolation : la distance sur préfixe est calculée en **temps constant** (la longueur du préfixe est fixe), **indépendamment de la dimension réelle** des vecteurs. C'est précisément pourquoi le bénéfice **croît avec la longueur des embeddings** — plus les vecteurs sont longs, plus on économise en évitant le calcul complet.

> Ces optimisations rejoignent les autres travaux bas niveau de MariaDB Vector (représentation mémoire, instructions SIMD AVX2/AVX-512/ARM/Power10 abordées au §18.10.5) : l'objectif d'ensemble est de faire de MariaDB à la fois la base relationnelle **et** la base vectorielle d'une même application, sans pile dédiée supplémentaire.

---

## Gains observés

D'après MariaDB, la 12.3 affiche **10 à 30 % de débit (QPS) en plus à rappel équivalent** par rapport à la 11.8. Et comme expliqué ci-dessus, cet écart **s'accentue** à mesure que les embeddings s'allongent (le coût du préfixe restant constant). Pour les charges RAG fondées sur des embeddings de 1536 ou 3072 dimensions, le gain est donc d'autant plus significatif.

---

## Ce que cela change concrètement (DBA / développeur)

- **C'est transparent.** Aucune réécriture de requête, aucune nouvelle syntaxe, aucun paramètre obligatoire : une simple **mise à niveau 11.8 → 12.3** suffit à en bénéficier sur les recherches `ORDER BY VEC_DISTANCE_* … LIMIT n`.
- **Vérifier le rappel sur ses propres données** après migration : l'optimisation est conçue pour préserver le rappel, mais un contrôle sur un jeu représentatif (mesure du recall@k avant/après) reste une bonne pratique de qualification en environnement sensible à la latence.
- **Maximiser le bénéfice** en privilégiant des modèles produisant des embeddings éligibles — typiquement les modèles Matryoshka (familles `text-embedding-3` d'OpenAI, embeddings Gemini) et les vecteurs longs.
- **Rendre un jeu non éligible compatible** (technique avancée, pour information) : multiplier tous les vecteurs par une **matrice orthogonale aléatoire** revient à une rotation dans l'espace ; cela **préserve toutes les distances et tous les angles** (donc les résultats de recherche), tout en stabilisant le ratio `d_prefix / d_exact` autour de 1. Exemple illustratif en Python :

```python
import scipy.stats  
rom = scipy.stats.ortho_group.rvs(dim=len(dataset[0]))  
for i in range(len(dataset)):  
    dataset[i] = rom.dot(dataset[i])
```

Cette transformation s'applique de façon cohérente aux vecteurs stockés **et** aux vecteurs de requête.

---

## À retenir

- La 12.3 accélère la recherche vectorielle **sans changer l'API** ; le levier visé est le calcul de distance, qui pèse **80–90 %** du temps de recherche (MDEV-36205).
- **Distance par extrapolation** : la distance est estimée sur un court préfixe puis extrapolée — fiable dès ~100–150 dimensions pour les embeddings Matryoshka.
- **Rejet anticipé** : si la distance extrapolée dépasse le seuil, le candidat est écarté sans calcul complet ; sinon, la distance exacte est calculée → **rappel préservé**.
- **Activation automatique** : 10 000 calculs de calibrage, puis activation si l'écart-type du ratio est sous le seuil ; **aucune pénalité** sur les jeux non éligibles.
- **Calcul dans la couche de stockage**, au plus près des données ; coût du préfixe **constant**, d'où un gain croissant avec la longueur des vecteurs.
- Résultat : **+10 à 30 % de QPS** à rappel égal vs 11.8, et davantage avec des embeddings longs (3072 dimensions).

---

## Voir aussi

- **§18.10** MariaDB Vector : recherche vectorielle pour l'IA/RAG (type `VECTOR`, fonctions, index HNSW)
- **§18.10.7** Optimisations 12.3 (même sujet, sous l'angle fonctionnel) et **§18.10.5** Optimisations SIMD
- **§5.3** Index VECTOR (HNSW) pour la recherche vectorielle
- **§7.7** Moteur Vector/HNSW
- **§20.9** Cas d'usage IA : RAG et recherche vectorielle

⏭️ [Partie 8 : DevOps, Cloud et Automatisation](/partie-08-devops-cloud-automatisation.md)
