🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Introduction au langage SQL

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

## Qu'est-ce que le SQL ?

SQL, pour *Structured Query Language* (langage de requête structuré), est le langage normalisé qui permet de dialoguer avec une base de données relationnelle. C'est par son intermédiaire que l'on définit la structure des données, qu'on les insère, qu'on les interroge, qu'on les modifie et qu'on en gère les accès. Sous MariaDB, comme sous tout SGBD relationnel, chaque interaction avec les données passe, directement ou indirectement, par du SQL — y compris lorsqu'un ORM ou une interface graphique le génère pour vous.

La caractéristique fondamentale du SQL est d'être un langage **déclaratif** : on décrit *ce que l'on veut obtenir*, et non *comment l'obtenir*. En écrivant une requête, on n'indique pas au serveur l'ordre dans lequel parcourir les tables ni les index à emprunter ; on exprime un résultat souhaité, et c'est l'**optimiseur** de MariaDB qui détermine la meilleure stratégie d'exécution (sujet approfondi au chapitre 5). Cette approche distingue le SQL des langages impératifs classiques (C, Java, Python…), dans lesquels le développeur décrit pas à pas la suite des opérations à effectuer.

Le SQL repose sur le **modèle relationnel**, présenté au chapitre 1 : les données y sont organisées en tables (relations), composées de lignes (enregistrements) et de colonnes (attributs). L'essentiel de ce chapitre consiste précisément à apprendre à créer ces structures puis à agir sur leur contenu.

## Un bref historique

Le SQL est né au début des années 1970 dans les laboratoires d'IBM, dans le cadre du projet de recherche System R. Conçu par Donald Chamberlin et Raymond Boyce, il s'appelait à l'origine SEQUEL (*Structured English Query Language*) avant d'être renommé SQL pour des raisons de marque déposée. Son objectif était d'offrir un moyen pratique d'interroger les bases relationnelles théorisées quelques années plus tôt par Edgar F. Codd.

Le langage a été normalisé une première fois par l'ANSI en 1986, puis par l'ISO en 1987. Plusieurs révisions ont suivi et continuent d'enrichir le standard : SQL-92 (longtemps la version de référence), SQL:1999 (requêtes récursives et déclencheurs), SQL:2003 (fonctions de fenêtrage et séquences), SQL:2011 (tables temporelles), SQL:2016 (prise en charge native du JSON) et SQL:2023 (nouvelles fonctions JSON et requêtes de graphes de propriétés). MariaDB implémente l'essentiel de ces apports : on retrouvera les requêtes récursives et les *window functions* au chapitre 4, les tables temporelles au chapitre 18, et le JSON aux chapitres 4 et 18.

## Un standard, plusieurs dialectes

Bien qu'il existe une norme ISO, aucun SGBD ne l'applique à la lettre, et chacun y ajoute ses propres extensions. On parle ainsi de *dialectes* SQL : celui de MariaDB, hérité de MySQL, diffère sur certains points de ceux de PostgreSQL, d'Oracle ou de SQL Server. La syntaxe de base — `SELECT`, `INSERT`, `JOIN`, etc. — reste très largement portable d'un système à l'autre, mais les types de données, les fonctions intégrées, la gestion des dates ou le comportement face aux valeurs `NULL` peuvent varier.

MariaDB pousse la souplesse plus loin que la plupart de ses concurrents grâce à la variable `sql_mode`, qui ajuste finement le comportement du serveur : rigueur de la validation des données, interprétation des guillemets, compatibilité avec d'autres SGBD… MariaDB propose même un **mode Oracle**, qui active une partie de la syntaxe PL/SQL et de nombreuses fonctions de compatibilité — un atout précieux pour les migrations, détaillé au chapitre 19. La variable `sql_mode` fait l'objet de la section 11.3.

## Les sous-langages du SQL

On a coutume de regrouper les instructions SQL en plusieurs catégories selon leur rôle. Cette classification n'a pas de portée syntaxique stricte, mais elle aide à structurer son apprentissage.

| Sous-langage | Rôle | Commandes principales |
|---|---|---|
| **DDL** (*Data Definition Language*) | Définir et modifier la structure des objets (bases, tables…) | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` |
| **DML** (*Data Manipulation Language*) | Manipuler les données contenues dans les tables | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** (*Data Query Language*) | Interroger les données (souvent rattaché au DML) | `SELECT` |
| **DCL** (*Data Control Language*) | Gérer les droits et les accès | `GRANT`, `REVOKE` |
| **TCL** (*Transaction Control Language*) | Délimiter et contrôler les transactions | `START TRANSACTION` / `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

Ce chapitre se concentre sur le DDL (sections 2.3 à 2.5) et le DML/DQL (sections 2.6 à 2.8). Le DCL est traité au chapitre 10 (sécurité) et le TCL au chapitre 6 (transactions). À noter : `TRUNCATE` est rangé ici dans le DDL car, sous MariaDB, vider une table de cette manière constitue une opération de définition (validation implicite, réinitialisation de l'`AUTO_INCREMENT`, impossibilité de revenir en arrière) ; il est néanmoins abordé en section 2.8 aux côtés de `DELETE`, dont il partage la finalité.

## Conventions de syntaxe sous MariaDB

**Terminaison des instructions.** Chaque instruction se termine par un point-virgule (`;`). Dans le client en ligne de commande `mariadb`, ce séparateur indique la fin de la commande et en déclenche l'exécution ; on peut aussi utiliser `\g` (équivalent du `;`) ou `\G` pour un affichage vertical, plus lisible lorsque les lignes comportent de nombreuses colonnes.

**Sensibilité à la casse.** Les mots-clés et les noms de fonctions sont insensibles à la casse : `SELECT`, `select` et `SeLeCt` sont équivalents. Par convention de lisibilité, on écrit les mots-clés en majuscules et les noms d'objets en minuscules. En revanche, les **noms de tables et de bases** sont, sous Linux, sensibles à la casse par défaut (le comportement dépend du système de fichiers et de la variable `lower_case_table_names`), tandis que les noms de colonnes, d'index et d'alias ne le sont jamais. C'est un point de vigilance lors d'une migration entre Windows et Linux.

**Commentaires.** MariaDB reconnaît trois formes de commentaires :

```sql
-- commentaire sur une ligne (un espace est requis après les deux tirets)
# commentaire sur une ligne (extension MariaDB/MySQL)
/* commentaire
   pouvant s'étendre sur plusieurs lignes */
```

Il existe par ailleurs des « commentaires exécutables » de la forme `/*! ... */`, dont le contenu n'est interprété que par MariaDB/MySQL : ils permettent d'écrire du SQL compatible avec d'autres outils tout en activant des fonctionnalités spécifiques.

**Chaînes, nombres et identifiants.** Les chaînes de caractères s'entourent de guillemets simples (`'Paris'`) ; un guillemet simple interne se double (`'l''adresse'`) ou s'échappe par un antislash (`'l\'adresse'`). Les nombres s'écrivent sans guillemets (`42`, `3.14`) et les dates sous forme de chaîne au format ISO (`'2026-06-03'`). Par défaut, MariaDB accepte aussi les guillemets doubles pour délimiter une chaîne, mais ce comportement change si le mode `ANSI_QUOTES` est actif : les guillemets doubles servent alors, comme dans le standard, à délimiter des identifiants. Pour entourer un identifiant (nom de table ou de colonne) contenant des caractères spéciaux ou correspondant à un mot réservé, MariaDB utilise les accents graves (*backticks*), par exemple `` `order` `` ou `` `nom complet` ``.

**La valeur `NULL`.** La valeur spéciale `NULL` représente l'**absence** de valeur ; elle se distingue de zéro comme de la chaîne vide. Sa manipulation obéit à une logique ternaire particulière, traitée en détail à la section 4.6.

**Mise en forme.** Le SQL est insensible aux espaces : tabulations, espaces et sauts de ligne supplémentaires sont ignorés. On en profite pour indenter et aérer les requêtes, ce qui améliore nettement leur lisibilité sans rien changer à leur exécution.

## Un premier aperçu

L'exemple suivant illustre, en quelques lignes, comment les sous-langages se combinent en pratique : on définit une structure (DDL), on y insère une donnée (DML), puis on l'interroge (DQL).

```sql
-- Définition de la structure (DDL)
CREATE TABLE produit (
    id       INT AUTO_INCREMENT PRIMARY KEY,
    nom      VARCHAR(100)  NOT NULL,
    prix     DECIMAL(10,2) NOT NULL,
    cree_le  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ajout d'une donnée (DML)
INSERT INTO produit (nom, prix) VALUES ('Clavier mécanique', 89.90);

-- Interrogation (DQL)
SELECT id, nom, prix
FROM produit
WHERE prix < 100
ORDER BY prix DESC;
```

Chacune de ces instructions sera reprise et détaillée dans les sections suivantes : les types de données (`INT`, `VARCHAR`, `DECIMAL`, `TIMESTAMP`) en 2.2, la création de tables en 2.4, les contraintes (`PRIMARY KEY`, `NOT NULL`) en 2.5, l'insertion en 2.6 et la sélection (`SELECT`, `WHERE`, `ORDER BY`) en 2.7.

## En résumé

Le SQL est un langage déclaratif et normalisé, organisé en sous-langages (DDL, DML, DQL, DCL, TCL) qui en couvrent les différents usages. MariaDB en propose un dialecte hérité de MySQL, largement portable mais doté d'extensions propres et d'une variable `sql_mode` capable d'en moduler le comportement, jusqu'à un mode de compatibilité Oracle. Quelques conventions de syntaxe — point-virgule terminal, insensibilité à la casse des mots-clés, formes de commentaires, guillemets simples pour les chaînes et accents graves pour les identifiants — suffisent pour aborder sereinement les sections suivantes, qui passent à la pratique du langage.

---

← [Chapitre 2 — Bases du SQL](README.md) · Section suivante : [2.2 Types de données MariaDB](02-types-de-donnees.md) →

⏭️ [Types de données MariaDB](/02-bases-du-sql/02-types-de-donnees.md)
