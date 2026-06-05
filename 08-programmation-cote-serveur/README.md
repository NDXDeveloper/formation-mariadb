🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Programmation Côté Serveur

Dans les chapitres précédents, MariaDB a surtout été abordé comme un dépôt de données interrogé depuis l'extérieur : une application envoie des requêtes SQL et exploite les résultats. Ce chapitre adopte une perspective différente. La *programmation côté serveur* consiste à confier au serveur de base de données lui-même une partie de la logique applicative, sous forme de routines, de déclencheurs et de tâches planifiées qui s'exécutent au plus près des données.

MariaDB propose pour cela un véritable langage procédural, dérivé du standard SQL/PSM (*Persistent Stored Modules*), qui enrichit le SQL déclaratif de variables, de structures de contrôle, de gestion d'erreurs et de curseurs. Ce langage permet d'écrire des traitements complets — validation de règles métier, calculs, agrégations, automatisations — sans jamais quitter le moteur.

## Pourquoi programmer côté serveur ?

Déplacer de la logique dans la base de données répond à plusieurs préoccupations concrètes. La première est la performance : un traitement exécuté côté serveur évite les allers-retours réseau répétés entre l'application et la base ; une procédure qui parcourt et met à jour des milliers de lignes le fait localement, là où une application devrait rapatrier puis renvoyer chaque ligne. Vient ensuite la centralisation des règles métier — une contrainte ou un calcul implémenté dans une routine ou un déclencheur s'applique quelle que soit l'application cliente (web, traitement par lots, outil d'administration), sans pouvoir être contournée ni dupliquée de façon incohérente. Les déclencheurs servent aussi l'intégrité des données en garantissant qu'une action systématique (journalisation, mise à jour d'un champ dérivé, contrôle de cohérence) accompagne chaque modification, indépendamment de la rigueur du code applicatif. Sur le plan de la sécurité, une routine peut encapsuler des opérations sensibles et n'exposer que des points d'entrée contrôlés, sans accorder aux utilisateurs de privilèges directs sur les tables sous-jacentes. Enfin, les *events* ouvrent la voie à l'automatisation de tâches récurrentes — purge d'archives, recalcul de statistiques, opérations de maintenance — directement depuis le serveur, sans ordonnanceur externe.

Cette approche a aussi ses limites, abordées en fin de chapitre : une logique trop dispersée entre application et base devient difficile à tester, à versionner et à faire évoluer. Tout l'enjeu est de trouver le bon équilibre.

## Compatibilité Oracle et nouveautés 12.3

Le langage procédural de MariaDB peut fonctionner selon deux dialectes. Par défaut, il suit la syntaxe historique proche de MySQL. En activant le mode Oracle (`SET sql_mode = ORACLE`), il adopte une syntaxe largement compatible avec PL/SQL, ce qui facilite la migration d'applications existantes.

MariaDB 12.3 LTS renforce nettement cette compatibilité et modernise plusieurs aspects du langage. Le chapitre signale ces nouveautés par le marqueur 🆕 : les tableaux associatifs de type Oracle dans les procédures (8.1.4), les déclencheurs multi-événements — un seul *trigger* couvrant `INSERT`, `UPDATE` et `DELETE` (8.3.4) —, ainsi que les curseurs sur instructions préparées (8.5.1) et le type `SYS_REFCURSOR`, un curseur de référence faible encadré par `max_open_cursors` (8.5.2).

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- écrire, appeler et maintenir des procédures et des fonctions stockées, en maîtrisant le passage de paramètres ;
- automatiser des réactions aux modifications de données grâce aux déclencheurs ;
- planifier des tâches récurrentes à l'aide du planificateur d'événements ;
- parcourir des jeux de résultats ligne à ligne avec des curseurs ;
- structurer vos traitements avec les instructions de contrôle de flux et gérer proprement les erreurs et exceptions ;
- appliquer les bonnes pratiques qui rendent ce code lisible, performant et maintenable.

## Prérequis

Ce chapitre suppose une bonne aisance avec le SQL des chapitres 2 à 4 (requêtes, jointures, sous-requêtes) ainsi qu'avec les notions de transactions et de niveaux d'isolation du chapitre 6, car les procédures et les déclencheurs s'exécutent dans un contexte transactionnel.

## Structure du chapitre

Le chapitre progresse des briques fondamentales vers les mécanismes les plus avancés :

- **8.1 — Procédures stockées** : la brique de base des traitements côté serveur, leur création (`CREATE PROCEDURE`), le passage de paramètres (`IN`, `OUT`, `INOUT`) et leur appel (`CALL`), y compris les tableaux associatifs hérités d'Oracle. 🆕
- **8.2 — Fonctions stockées** : des routines qui retournent une valeur et s'intègrent dans les expressions SQL, avec leurs caractéristiques (`DETERMINISTIC`, `NO SQL`, etc.).
- **8.3 — Triggers** : des déclencheurs exécutés automatiquement `BEFORE` ou `AFTER` une opération `INSERT`, `UPDATE` ou `DELETE`, manipulant les pseudo-enregistrements `OLD` et `NEW`, jusqu'aux déclencheurs multi-événements. 🆕
- **8.4 — Events** : la planification de tâches récurrentes et le fonctionnement de l'*Event Scheduler*.
- **8.5 — Curseurs** : le traitement ligne à ligne d'un jeu de résultats, étendu aux instructions préparées et au type `SYS_REFCURSOR`. 🆕
- **8.6 — Gestion des erreurs et exceptions** : la déclaration de gestionnaires (`DECLARE … HANDLER`) pour intercepter et traiter les conditions d'erreur.
- **8.7 — Variables et contrôle de flux** : les instructions `IF`, `CASE`, `LOOP`, `WHILE` et `REPEAT` qui donnent au langage sa dimension procédurale.
- **8.8 — Bonnes pratiques de programmation** : les recommandations transversales pour un code robuste et maintenable.

---

Place maintenant à la première brique de la programmation côté serveur : les **[procédures stockées](01-procedures-stockees.md)**.

⏭️ [Procédures stockées](/08-programmation-cote-serveur/01-procedures-stockees.md)
