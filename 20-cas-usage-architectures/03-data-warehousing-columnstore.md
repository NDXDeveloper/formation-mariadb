🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.3 Data warehousing avec ColumnStore

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La section §20.1 a posé la distinction entre charges transactionnelles (OLTP) et analytiques (OLAP), et désigné **ColumnStore** comme le moteur de MariaDB taillé pour l'analytique. Cette section concrétise ce constat : comment bâtir un véritable **entrepôt de données** (*data warehouse*) avec ColumnStore, depuis la modélisation jusqu'au chargement, en passant par l'articulation avec les systèmes transactionnels.

L'enjeu n'est plus d'exécuter une requête analytique ponctuelle, mais de concevoir un système dédié à l'analyse de grands volumes de données historiques — la brique au cœur de l'informatique décisionnelle (BI) et du reporting.

---

## Qu'est-ce qu'un entrepôt de données ?

Un entrepôt de données est un **dépôt centralisé** réunissant des données intégrées, provenant de plusieurs sources opérationnelles, et organisées pour l'analyse plutôt que pour la production. Il se distingue d'une base opérationnelle sur plusieurs points :

- **Orienté sujet** : les données sont structurées autour de grands thèmes métier (les ventes, les clients, les stocks) plutôt qu'autour des transactions.
- **Intégré** : il consolide des données issues de systèmes hétérogènes, harmonisées en un modèle cohérent.
- **Historisé** : il conserve l'historique sur de longues périodes, là où une base opérationnelle ne reflète souvent que l'état courant.
- **Non volatil** : les données y sont surtout ajoutées et lues, rarement modifiées en place.

Cet usage massivement en lecture, sur de grands volumes et avec de fortes agrégations, est précisément le terrain où le stockage en colonnes excelle.

---

## Pourquoi ColumnStore pour l'analytique

Les avantages du stockage en colonnes, détaillés en §20.1, prennent tout leur sens à l'échelle d'un entrepôt :

- **Moins d'entrées/sorties** : une requête analytique ne touche souvent que quelques colonnes (une mesure, une ou deux dimensions). ColumnStore ne lit que ces colonnes, en ignorant tout le reste — une économie décisive sur des tables de plusieurs centaines de colonnes.
- **Compression élevée** : les valeurs d'une même colonne, de type homogène et souvent répétitives, se compressent fortement, ce qui réduit l'empreinte disque d'un historique volumineux.
- **Traitement massivement parallèle** : ColumnStore est conçu pour distribuer le travail de balayage et d'agrégation, et peut s'étendre sur plusieurs nœuds pour absorber de très grands volumes.

---

## L'architecture de ColumnStore

ColumnStore est, depuis MariaDB 10.5, intégré comme **moteur de stockage enfichable** : une table se déclare simplement avec `ENGINE=ColumnStore`, et cohabite dans le même serveur avec des tables InnoDB. Plusieurs caractéristiques le distinguent radicalement d'InnoDB.

### Stockage par colonnes et extents

Chaque colonne est stockée séparément et découpée en unités appelées **extents** — de grands blocs contigus de valeurs. Cette organisation est la base des optimisations qui suivent.

### L'élimination d'extents : pourquoi il n'y a pas d'index

ColumnStore conserve, pour chaque extent, les **valeurs minimale et maximale** de la colonne dans une structure de métadonnées (l'*Extent Map*). Lorsqu'une requête filtre sur une colonne, le moteur consulte cette carte et **ignore purement et simplement** tout extent dont l'intervalle [min, max] ne peut contenir la valeur recherchée.

Ce mécanisme, appelé **élimination d'extents**, joue un rôle comparable à un élagage de partitions, mais de façon automatique et au niveau du stockage. C'est la raison pour laquelle ColumnStore **n'utilise pas d'index B-Tree secondaires** : le balayage columnaire combiné à l'élimination d'extents remplace l'indexation traditionnelle. On ne crée donc pas d'index sur une table ColumnStore — et c'est voulu.

### Exécution distribuée

ColumnStore peut répartir le traitement d'une requête — filtrage, agrégation — sur plusieurs unités de calcul, voire plusieurs nœuds. Cette architecture *MPP* (*Massively Parallel Processing*) lui permet de monter en charge horizontalement, en ajoutant des nœuds plutôt qu'en grossissant une seule machine.

---

## Modélisation dimensionnelle

Comme évoqué en §20.1, un entrepôt s'appuie rarement sur un schéma normalisé. Le modèle de référence est le **schéma en étoile** : une **table de faits** centrale, volumineuse, qui enregistre les mesures (les ventes, les événements), entourée de **tables de dimensions** plus petites qui en décrivent le contexte (le temps, le produit, le magasin, le client).

Cette dénormalisation réduit le nombre de jointures et s'accorde parfaitement avec ColumnStore, dont les forces résident dans le balayage et l'agrégation de grandes tables de faits.

```sql
-- Table de faits (volumineuse) sur ColumnStore : aucune clé ni aucun index
CREATE TABLE faits_ventes (
    date_id      INT,
    produit_id   INT,
    magasin_id   INT,
    client_id    INT,
    quantite     INT,
    montant      DECIMAL(12,2),
    remise       DECIMAL(12,2)
) ENGINE=ColumnStore;

-- Tables de dimensions, également sur ColumnStore
CREATE TABLE dim_temps (
    date_id   INT,
    jour      DATE,
    mois      TINYINT,
    trimestre TINYINT,
    annee     SMALLINT
) ENGINE=ColumnStore;

CREATE TABLE dim_produit (
    produit_id INT,
    libelle    VARCHAR(120),
    categorie  VARCHAR(60),
    marque     VARCHAR(60)
) ENGINE=ColumnStore;
```

Une requête analytique typique croise faits et dimensions, puis agrège :

```sql
SELECT t.annee, t.trimestre, p.categorie,
       SUM(f.montant)  AS chiffre_affaires,
       SUM(f.quantite) AS volume
FROM faits_ventes f
JOIN dim_temps   t ON t.date_id    = f.date_id
JOIN dim_produit p ON p.produit_id = f.produit_id
GROUP BY t.annee, t.trimestre, p.categorie
ORDER BY t.annee, chiffre_affaires DESC;
```

Sur une telle requête, ColumnStore ne lit que les colonnes mobilisées et écarte, par élimination d'extents, les blocs hors périmètre — sans qu'aucun index n'ait à être défini ni maintenu.

---

## Alimenter l'entrepôt : ETL et chargement

Un entrepôt se nourrit des données des systèmes opérationnels au travers de processus **ETL** (*Extract, Transform, Load*) ou **ELT** : on extrait les données des sources, on les transforme et les harmonise, puis on les charge dans l'entrepôt, généralement par lots périodiques.

Côté chargement, ColumnStore privilégie le **chargement en masse** plutôt que les insertions ligne à ligne :

- **`cpimport`** est l'outil de chargement haute performance dédié à ColumnStore. Il contourne la couche SQL pour atteindre des débits élevés et constitue la voie recommandée pour les volumes importants.

  ```bash
  # Chargement massif d'un fichier dans une table ColumnStore
  cpimport entrepot faits_ventes /data/ventes_2026.csv -s ','
  ```

- **`LOAD DATA INFILE`** et **`INSERT ... SELECT`** restent utilisables, par exemple pour transférer des données depuis une table InnoDB du même serveur vers une table ColumnStore.

La règle d'or est d'**accumuler par lots** : ColumnStore est optimisé pour ingérer de gros volumes d'un coup, non pour encaisser un flot de petites écritures unitaires.

---

## Articulation avec l'OLTP : le pattern HTAP

L'un des atouts de MariaDB est de permettre la cohabitation, **dans un même serveur**, de tables InnoDB transactionnelles et de tables ColumnStore analytiques. Cela ouvre la voie aux architectures **HTAP** introduites en §20.1, où l'on souhaite analyser des données quasi fraîches sans pénaliser la production.

Plusieurs montages sont possibles :

- **Réplication d'InnoDB vers ColumnStore** : on peut alimenter un réplica ColumnStore à partir d'un primaire InnoDB (chapitre 13), constituant ainsi un entrepôt mis à jour en continu à partir des données transactionnelles, tout en isolant la charge analytique de la base de production.
- **Pipelines orientés événements** : la capture de changements (CDC) à partir du journal binaire, couplée à Kafka et Debezium (voir §20.8), permet de déverser en flux les changements opérationnels vers l'entrepôt.
- **ETL classique par lots** : des chargements planifiés transforment et déversent périodiquement les données dans ColumnStore via `cpimport`.

Le principe directeur reste de **séparer les charges** : laisser InnoDB servir l'OLTP, et ColumnStore servir l'OLAP, en synchronisant les données entre les deux.

---

## Ce que ColumnStore n'est pas

Pour faire un choix d'architecture éclairé, il faut connaître les limites du moteur. ColumnStore est spécialisé, et ce qui fait sa force en analytique le rend inadapté à d'autres usages :

- **Ce n'est pas un moteur transactionnel** : les insertions, mises à jour et suppressions **ligne à ligne** sont coûteuses. ColumnStore vise les modèles en ajout massif et lecture intensive, non les écritures unitaires fréquentes.
- **Pas de clés étrangères ni d'index secondaires** : l'intégrité référentielle déclarative et l'indexation traditionnelle ne font pas partie de son modèle (l'élimination d'extents remplace les index).
- **Faible adéquation à la forte concurrence transactionnelle** : il n'est pas conçu pour des milliers de petites transactions concurrentes, mais pour un nombre plus restreint de requêtes lourdes.

En clair : ColumnStore est le moteur de l'**entrepôt**, pas celui de l'**application opérationnelle**. C'est précisément pourquoi il cohabite avec InnoDB plutôt qu'il ne le remplace.

---

## Déploiement et mise à l'échelle

ColumnStore se déploie selon deux grands modes :

- **Mono-nœud** : le moteur tourne au sein d'une seule instance MariaDB. C'est la configuration la plus simple, suffisante pour de nombreux entrepôts de taille moyenne, et idéale pour démarrer ou pour des montages HTAP avec InnoDB sur le même serveur.
- **Multi-nœuds (MPP)** : pour les très grands volumes, ColumnStore se distribue sur plusieurs nœuds afin de paralléliser davantage le traitement et de croître horizontalement.

Du côté du stockage, ColumnStore peut s'appuyer sur un stockage local ou sur un **stockage objet compatible S3**, permettant de découpler le calcul et les données — une approche pertinente dans les déploiements cloud (chapitre 16). Cette capacité propre à ColumnStore est à distinguer du **moteur S3** générique (§7.6), dédié à l'archivage en lecture seule de données froides.

Pour la configuration de référence d'une charge analytique, on se reportera utilement à l'annexe D (profil OLAP).

---

## Quand utiliser ColumnStore

ColumnStore s'impose lorsque :

- l'on construit un **entrepôt de données** ou un système de **reporting/BI** sur de grands volumes historiques ;
- les requêtes sont **analytiques** : agrégations, regroupements et balayages portant sur quelques colonnes parmi beaucoup ;
- les données arrivent par **lots** plutôt que par écritures unitaires fréquentes ;
- l'on souhaite **isoler** la charge analytique de la base transactionnelle, idéalement en alimentant ColumnStore depuis InnoDB.

À l'inverse, pour une charge transactionnelle, des écritures fréquentes ligne à ligne ou un besoin d'intégrité référentielle, InnoDB reste le moteur approprié.

ColumnStore est ainsi la réponse de MariaDB au volet analytique du couple OLTP/OLAP, et un pilier des architectures décisionnelles. La section suivante aborde un autre axe de cloisonnement — non plus par type de charge, mais par client — avec l'**architecture multi-tenant** (§20.4).

---

## Navigation

⬅️ Section précédente : [20.2.2 Shared database pattern](02.2-shared-database.md)  
➡️ Section suivante : [20.4 Architecture multi-tenant](04-architecture-multi-tenant.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Architecture multi-tenant](/20-cas-usage-architectures/04-architecture-multi-tenant.md)
