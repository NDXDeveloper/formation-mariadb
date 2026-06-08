🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.9 Dynamic columns

## Des attributs variables dans une colonne

Les **colonnes dynamiques** (*dynamic columns*) permettent de stocker un ensemble **variable de paires clé-valeur** à l'intérieur d'une seule colonne, déclarée comme un `BLOB`. Chaque ligne peut ainsi porter des attributs différents, sans qu'il soit nécessaire de modifier le schéma de la table — une forme de souplesse « sans schéma » au sein d'un modèle relationnel.

Apparue dès MariaDB 5.3 (puis enrichie en 10.0 avec la prise en charge des **clés nommées**), cette fonctionnalité fut la réponse historique de MariaDB aux données semi-structurées, **antérieure au support natif du JSON** (§4.7). Ce contexte est essentiel pour bien situer son usage aujourd'hui, comme on le verra plus bas.

## Les fonctions de manipulation

On ne lit ni n'écrit directement le `BLOB` : on passe par une famille de fonctions `COLUMN_*`. La colonne se déclare comme un `BLOB` (ou `LONGBLOB` pour de gros volumes), puis :

```sql
CREATE TABLE produit (
  id        INT PRIMARY KEY,
  nom       VARCHAR(100),
  attributs BLOB              -- colonne dynamique
);

INSERT INTO produit (id, nom, attributs) VALUES
  (1, 'T-shirt', COLUMN_CREATE('couleur', 'rouge', 'taille', 'M')),
  (2, 'Mug',     COLUMN_CREATE('couleur', 'blanc', 'contenance_ml', 350));
```

Les principales fonctions sont :

- **`COLUMN_CREATE(clé, valeur, …)`** — construit un blob dynamique à partir de paires clé-valeur ;
- **`COLUMN_GET(blob, clé AS type)`** — lit une valeur, **obligatoirement** convertie vers un type ;
- **`COLUMN_ADD(blob, clé, valeur, …)`** — ajoute ou met à jour des paires et renvoie le blob modifié ;
- **`COLUMN_DELETE(blob, clé, …)`** — supprime des clés ;
- **`COLUMN_EXISTS(blob, clé)`** — teste la présence d'une clé ;
- **`COLUMN_LIST(blob)`** — renvoie la liste des clés ;
- **`COLUMN_CHECK(blob)`** — vérifie la validité du blob ;
- **`COLUMN_JSON(blob)`** — convertit le blob en JSON.

À l'usage :

```sql
SELECT id, nom, COLUMN_GET(attributs, 'couleur' AS CHAR) AS couleur FROM produit;

UPDATE produit SET attributs = COLUMN_ADD(attributs, 'promo', 1) WHERE id = 1;

SELECT COLUMN_LIST(attributs)  FROM produit WHERE id = 1;   -- couleur,taille,promo
SELECT COLUMN_JSON(attributs)  FROM produit WHERE id = 1;   -- {"couleur":"rouge","taille":"M","promo":1}
```

## Typage et imbrication

La lecture impose toujours de préciser le type attendu via `AS` (`CHAR`, `INTEGER`, `DOUBLE`, `DECIMAL`, `DATE`, `DATETIME`…), car le blob ne porte pas une typologie exploitable directement par SQL. Les colonnes dynamiques peuvent par ailleurs être **imbriquées** : la valeur d'une clé peut elle-même être un blob dynamique, ce qui autorise des structures arborescentes.

```sql
COLUMN_CREATE('dimensions', COLUMN_CREATE('largeur', 10, 'hauteur', 20))
```

## Indexation et recherche

Comme pour le JSON, on **n'indexe pas directement** l'intérieur d'une colonne dynamique, et filtrer avec `COLUMN_GET` dans un `WHERE` conduit à un parcours complet. La parade est identique à celle vue pour les colonnes générées (§18.4) : extraire la valeur dans une colonne générée, puis l'indexer.

```sql
ALTER TABLE produit
  ADD COLUMN couleur VARCHAR(30) AS (COLUMN_GET(attributs, 'couleur' AS CHAR)) VIRTUAL,
  ADD INDEX idx_couleur (couleur);

SELECT * FROM produit WHERE couleur = 'rouge';   -- exploite idx_couleur
```

## Colonnes dynamiques ou JSON ?

C'est le point décisif de cette section. Les colonnes dynamiques précèdent le **support natif du JSON** dans MariaDB, et pour tout **nouveau développement, c'est JSON qu'il faut privilégier**. Plusieurs raisons à cela :

- le **JSON est un format standard et portable**, interopérable avec les applications, les API et les autres bases ; le format binaire des colonnes dynamiques, lui, est propre à MariaDB et non échangeable ;
- le JSON dispose d'un **écosystème de fonctions riche et normalisé** : extraction et modification (§4.7), expressions de chemin avancées (§4.8), validation de schéma (§4.9), opérateur raccourci `->>`, prédicat `IS JSON`, et indexation via colonnes générées (§4.10) ;
- l'intégration avec les outils et les langages modernes est bien plus naturelle avec JSON.

Les colonnes dynamiques conservent donc surtout un intérêt pour **comprendre et maintenir du code existant** qui les utilise, plutôt que pour de nouveaux schémas.

## Migrer vers JSON

La transition est facilitée par la fonction **`COLUMN_JSON`**, qui restitue le contenu d'une colonne dynamique sous forme de document JSON. Elle constitue le pont naturel pour convertir des données historiques vers une colonne `JSON`, après quoi l'on bénéficie de tout l'outillage du chapitre 4.

```sql
SELECT COLUMN_JSON(attributs) FROM produit;
-- chaque blob dynamique est rendu sous forme de JSON, prêt à être stocké dans une colonne JSON
```

## Points clés à retenir

- Les colonnes dynamiques stockent des **paires clé-valeur variables** dans un `BLOB`, manipulé par la famille de fonctions `COLUMN_*` (`COLUMN_CREATE`, `COLUMN_GET … AS type`, `COLUMN_ADD`, `COLUMN_JSON`…).
- La lecture exige un **typage explicite** (`AS`), et les structures peuvent être **imbriquées**.
- Pas d'indexation directe : utiliser le motif **colonne générée + index** (§18.4) pour rechercher efficacement.
- **Fonctionnalité historique antérieure au JSON** : pour tout nouveau développement, **préférer JSON** (§4.7 à §4.10), standard, portable et richement outillé.
- `COLUMN_JSON` offre un **pont vers JSON** pour migrer les données existantes.

⏭️ [MariaDB Vector : Recherche vectorielle pour l'IA/RAG](/18-fonctionnalites-avancees/10-mariadb-vector.md)
