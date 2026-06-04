🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.7 JSON dans MariaDB

> **Chapitre 4 — Concepts Avancés SQL** · Niveau : Avancé  
> Alias JSON disponible depuis MariaDB 10.2.7 · Référence : **MariaDB 12.3 LTS**

Toutes les données ne tiennent pas commodément dans des colonnes fixes. Des attributs qui varient d'un enregistrement à l'autre, des structures imbriquées, des charges utiles d'API, des métadonnées ou des configurations évolutives se modélisent mal dans un schéma rigide. Le format **JSON** (*JavaScript Object Notation*) répond à ce besoin : il permet de stocker, dans une seule colonne, un document semi-structuré — objets, tableaux, valeurs imbriquées — au sein d'une base relationnelle. MariaDB sait stocker, interroger, valider et indexer du JSON, faisant le pont entre le modèle relationnel et le modèle document.

Cette section introduit le sujet ; ses sous-sections en détaillent ensuite chaque aspect.

## 🎯 Objectif de la section

Comprendre l'intérêt du JSON dans une base relationnelle, connaître la particularité de son implémentation en MariaDB, savoir quand l'employer plutôt que des colonnes normalisées, et situer les outils que les sous-sections approfondiront.

## JSON dans un monde relationnel

Le JSON s'avère pertinent lorsque la structure des données est **souple, éparse ou imbriquée** : préférences utilisateur aux clés variables, fiches produit aux caractéristiques hétérogènes, métadonnées d'audit, réponses de services externes, paramètres applicatifs. Plutôt que de multiplier les colonnes (souvent nulles) ou les tables annexes, on regroupe ces données dans un document unique, lisible et auto-descriptif.

## La particularité de MariaDB : `JSON` est un alias de `LONGTEXT`

C'est le point déterminant, qui distingue MariaDB de MySQL. En MariaDB, le type `JSON` n'est **pas** un type binaire dédié : c'est un **alias de `LONGTEXT`** (avec la collation `utf8mb4_bin`). Ce choix a été fait pour la compatibilité avec MySQL — réplication et lecture de *dumps* — et parce qu'un type JSON natif s'écarterait du standard SQL ; les mesures de MariaDB indiquent une performance au moins équivalente.

Trois conséquences pratiques en découlent, détaillées au § 4.7.1 :

- **Validation automatique** : déclarer une colonne `JSON` ajoute automatiquement une contrainte `CHECK (JSON_VALID(colonne))`, qui rejette tout document mal formé à l'insertion ou à la mise à jour.
- **Stockage verbatim** : le texte est conservé **tel quel** — espaces, ordre des clés, voire clés en double sont préservés. MySQL, qui stocke une forme binaire normalisée, se comporte différemment ; cette divergence a des implications lors d'une migration (chapitre 19).
- **Performance et indexation** : comme la donnée est du texte, le moteur **analyse la chaîne entière** à chaque accès et n'indexe pas directement les chemins JSON. Pour interroger efficacement un champ enfoui, on l'extrait dans une colonne virtuelle que l'on indexe (§ 4.10).

## Quand utiliser JSON ?

Le JSON n'est pas un substitut universel aux colonnes : c'est un arbitrage entre **souplesse** et **rigueur**.

Le JSON convient lorsque la structure est variable ou imprévisible, que les attributs sont nombreux mais rarement renseignés, que les données sont imbriquées, ou qu'elles ne servent qu'à être lues en bloc (métadonnées, journaux).

Les **colonnes normalisées** restent préférables dès que les données sont structurées, fréquemment filtrées ou triées, soumises à des contraintes (types, clés étrangères, unicité) ou reliées à d'autres tables. Elles bénéficient pleinement de l'indexation et de l'intégrité référentielle.

En pratique, une approche **hybride** est souvent la meilleure : conserver en colonnes les champs « chauds » (interrogés, contraints, indexés) et réserver le JSON aux données accessoires ou évolutives — quitte à extraire ponctuellement un champ JSON vers une colonne indexable (§ 4.10).

## La boîte à outils JSON

MariaDB fournit un ensemble complet d'outils pour manipuler le JSON, répartis dans les sous-sections suivantes ainsi que dans la suite du chapitre :

- le **stockage et le type** lui-même (§ 4.7.1) ;
- un riche jeu de **fonctions** d'extraction, de modification et d'inspection — `JSON_EXTRACT`, `JSON_SET`, `JSON_ARRAY`, `JSON_OBJECT`, etc. (§ 4.7.2) ;
- des **opérateurs raccourcis** plus concis que les fonctions (§ 4.7.3) ;
- le prédicat **`IS JSON`** et la levée de la limite de profondeur, nouveautés de la série 12.x (§ 4.7.4) ;
- les **expressions de chemin** avancées pour cibler précisément un élément (§ 4.8) ;
- la **validation par schéma** pour contraindre la structure d'un document (§ 4.9) ;
- l'**indexation** via colonnes virtuelles, pour concilier souplesse et performance (§ 4.10).

## 🗺️ Plan de la section

- **4.7.1 — Stockage et type de données JSON** : l'alias `LONGTEXT`, la validation, le stockage verbatim.
- **4.7.2 — Fonctions JSON** : extraire, construire, modifier et inspecter un document.
- **4.7.3 — Opérateur raccourci (`->>`)** : une syntaxe condensée pour l'extraction.
- **4.7.4 — Prédicat `IS JSON` et suppression de la limite de profondeur** 🆕 : les apports de la 12.x.

## Points clés à retenir

Le JSON permet de stocker des données **semi-structurées** dans une colonne, comblant l'écart entre modèle relationnel et modèle document. En MariaDB, le type `JSON` est un **alias de `LONGTEXT`** (collation `utf8mb4_bin`) : la validité est garantie par une contrainte `JSON_VALID` ajoutée automatiquement, le texte est stocké **verbatim**, et l'indexation passe par des colonnes virtuelles plutôt que par les chemins JSON directement. Le JSON s'emploie pour des données **souples, éparses ou imbriquées** ; les colonnes normalisées restent préférables pour des données structurées, filtrées et contraintes — l'approche hybride combinant souvent le meilleur des deux.

---

**Section précédente :** [4.6 — Gestion des valeurs `NULL` : Logique ternaire](06-gestion-valeurs-null.md)  
**Section suivante :** [4.7.1 — Stockage et type de données JSON](07.1-stockage-type-json.md)  

⏭️ [Stockage et type de données JSON](/04-concepts-avances-sql/07.1-stockage-type-json.md)
