🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Types de données MariaDB

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

## Introduction

Toute colonne d'une table possède un **type de données** qui définit la nature des valeurs qu'elle peut contenir : un nombre, du texte, une date, un document JSON, etc. Le type n'est pas une simple étiquette : il détermine la place occupée sur le disque, les opérations autorisées, les contrôles appliqués à chaque insertion et, *in fine*, la performance des requêtes. Le choix du type est donc l'une des décisions de conception les plus structurantes — et l'une des plus délicates à corriger une fois la table en production et remplie de données.

MariaDB propose un éventail de types riche, qui va des types standard du SQL (numériques, texte, temporels, binaires) à des types plus spécifiques tournés vers des usages modernes (JSON, adresses réseau, identifiants universels, vecteurs pour l'IA). Cette section offre une vue d'ensemble de ces familles et des principes qui guident leur choix ; chaque famille est ensuite détaillée dans sa propre sous-section.

## Les grandes familles de types

MariaDB organise ses types en quelques grandes familles, que ce chapitre parcourt l'une après l'autre :

| Catégorie | Principaux types | Détaillée en |
|---|---|---|
| **Numériques** | `INT`, `BIGINT`, `DECIMAL`, `FLOAT`, `DOUBLE`… | §2.2.1 |
| **Texte** | `VARCHAR`, `TEXT`, `CHAR`, `ENUM`, `SET` | §2.2.2 |
| **Temporels** | `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR` | §2.2.3 |
| **Binaires** | `BLOB`, `BINARY`, `VARBINARY` | §2.2.4 |
| **Spécifiques MariaDB** | `JSON`, `UUID`, `INET6`, `VECTOR` | §2.2.5 |
| **XML** 🆕 | `XMLTYPE` (compatibilité Oracle) | §2.2.6 |

À ces familles s'ajoutent les types géographiques (spatial, `GEOMETRY`, `POINT`…), abordés sous l'angle de l'indexation au chapitre 5, ainsi que des raccourcis hérités de MySQL/MariaDB comme `BOOLEAN` (synonyme de `TINYINT(1)`) ou `SERIAL` (alias d'un entier auto-incrémenté unique). À noter dès maintenant : sous MariaDB, le type `JSON` n'est pas un type binaire dédié mais un **alias de `LONGTEXT`** assorti d'une validation automatique du contenu — une particularité détaillée en section 2.2.5 et au chapitre 4.

## Pourquoi le choix du type est important

Le type d'une colonne agit sur trois plans. Sur le plan de l'**intégrité**, il filtre les valeurs admissibles : une colonne `DATE` refuse `'2026-13-40'`, une colonne numérique refuse une valeur textuelle. Sur le plan du **stockage**, il conditionne l'espace occupé : un `BIGINT` consomme 8 octets là où un `SMALLINT` n'en réclame que 2, et utiliser le second lorsqu'il suffit allège considérablement une grande table. Sur le plan de la **performance**, des lignes plus compactes tiennent en plus grand nombre dans une page de données, ce qui réduit les entrées/sorties, rend les index plus petits et accélère les comparaisons.

Quelques règles empiriques en découlent. On privilégie le type le plus compact couvrant l'amplitude réellement attendue des valeurs. On distingue les nombres exacts (`DECIMAL`, pour les montants) des nombres approchés (`FLOAT` / `DOUBLE`, pour les mesures scientifiques). On stocke une date dans un type temporel plutôt que dans une chaîne, afin de bénéficier des contrôles et des fonctions de date (chapitre 3). Enfin, on n'élargit un type qu'en cas de besoin avéré, car modifier une colonne sur une table volumineuse peut être coûteux — d'où l'intérêt de l'`ALTER TABLE` en ligne abordé en section 18.11.

## Attributs et modificateurs communs

Indépendamment de leur famille, la plupart des types acceptent des attributs qui précisent leur comportement. On les introduit ici, car ils reviendront tout au long du chapitre :

- `NULL` / `NOT NULL` — autoriser ou interdire l'absence de valeur (voir §2.5).
- `DEFAULT` — valeur attribuée en l'absence de valeur explicite, y compris une expression comme `CURRENT_TIMESTAMP` (voir §2.5).
- `AUTO_INCREMENT` — génération automatique d'un entier croissant, typiquement pour une clé primaire.
- `UNSIGNED` — restreint un type numérique aux valeurs positives et double son amplitude maximale (voir §2.2.1).
- `CHARACTER SET` et `COLLATE` — jeu de caractères et ordre de comparaison d'une colonne textuelle (voir §2.2.2).

Depuis la 11.8, le jeu de caractères par défaut est passé à `utf8mb4` (avec des collations UCA 14.0.0), couvrant ainsi l'intégralité d'Unicode, émojis compris ; ce point est développé en sections 2.3 et 11.11.

## Plan de la section

- **2.2.1 [Numériques](02.1-types-numeriques.md)** — `INT`, `BIGINT`, `DECIMAL`, `FLOAT`, `DOUBLE` : entiers, décimaux exacts et flottants, l'attribut `UNSIGNED` et les pièges de l'arithmétique flottante.
- **2.2.2 [Texte](02.2-types-texte.md)** — `VARCHAR`, `TEXT`, `CHAR`, `ENUM`, `SET` : longueur fixe ou variable, jeux de caractères et collations, listes de valeurs énumérées.
- **2.2.3 [Temporels](02.3-types-temporels.md)** — `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR`, fuseaux horaires et extension de la plage `TIMESTAMP` au-delà de 2038.
- **2.2.4 [Binaires](02.4-types-binaires.md)** — `BLOB`, `BINARY`, `VARBINARY` pour les données non textuelles (fichiers, hachages, données brutes).
- **2.2.5 [Spécifiques MariaDB](02.5-types-specifiques-mariadb.md)** — `JSON` (alias de `LONGTEXT` validé), `UUID`, `INET6` et `VECTOR`, ce dernier ouvrant la voie aux usages IA et à la recherche vectorielle.
- **2.2.6 [Type XML basique](02.6-type-xml.md)** 🆕 — prise en charge du type `XMLTYPE` au titre de la compatibilité avec Oracle.

## Note de version

La taxonomie des types relève en grande partie du tronc commun du SQL et est stable de longue date. Trois éléments se distinguent toutefois au regard de la 12.3 : le **type XML basique** (§2.2.6), marqué 🆕, fait partie des nouveautés de la série 12.x ; le type **`VECTOR`**, dédié à la recherche vectorielle pour l'IA, est désormais du contenu standard (hérité de la 11.8, approfondi en §18.10) ; enfin, le passage à `utf8mb4` par défaut et l'extension de la plage `TIMESTAMP` au-delà de 2038, également hérités de la 11.8, sont aujourd'hui acquis.

---

← Section précédente : [2.1 Introduction au langage SQL](01-introduction-langage-sql.md) · [Sommaire du chapitre](README.md) · Sous-section suivante : [2.2.1 Types numériques](02.1-types-numeriques.md) →

⏭️ [Numériques (INT, BIGINT, DECIMAL, FLOAT, DOUBLE)](/02-bases-du-sql/02.1-types-numeriques.md)
