🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.8 Bonnes pratiques de programmation

La programmation côté serveur est un outil puissant, mais à double tranchant : mal employée, elle disperse la logique, complique la maintenance et dégrade les performances. Cette section rassemble les bonnes pratiques évoquées tout au long du chapitre, pour écrire des routines à la fois utiles, sûres et durables.

## Choisir le bon niveau

La première décision n'est pas *comment* écrire une routine, mais *s'il faut* l'écrire. Le SGBD est le bon endroit pour ce qui touche à l'intégrité des données et gagne à être au plus près d'elles : contraintes complexes via des triggers, opérations encapsulées réduisant les allers-retours réseau, accès contrôlé à des données sensibles. À l'inverse, une logique métier riche, amenée à évoluer souvent, est généralement mieux placée dans l'application, où elle se teste, se versionne et se déploie plus facilement. Les triggers, en particulier, sont à employer avec parcimonie : leur exécution invisible peut surprendre, et une cascade de triggers devient vite difficile à suivre. La bonne question : ce traitement gagne-t-il vraiment à vivre dans la base ?

## Privilégier l'ensembliste sur le procédural

C'est la règle cardinale, déjà soulignée à propos des curseurs (8.5) : le SQL est un langage *ensembliste*, et une opération exprimée en une seule instruction est presque toujours plus rapide qu'une boucle qui traite les lignes une à une. Avant d'écrire un curseur ou une boucle, il faut se demander si un `UPDATE … JOIN`, un `INSERT … SELECT` ou une requête bien pensée ne suffirait pas. De même, une fonction appelée dans une requête est évaluée potentiellement à chaque ligne (8.2) : son coût se paie sur l'ensemble du résultat. Le procédural se justifie pour ce qui ne s'exprime pas en ensembliste — pas pour reproduire à la main ce que le moteur fait mieux.

## Soigner la robustesse : erreurs et transactions

Une routine qui modifie des données doit prévoir l'échec. Dans une transaction, le réflexe est un gestionnaire `EXIT HANDLER FOR SQLEXCEPTION` qui annule (`ROLLBACK`) puis propage l'erreur (`RESIGNAL`), comme vu en 8.6 — afin de ne jamais laisser de modifications partielles. À l'inverse, il faut se méfier des gestionnaires qui *avalent* silencieusement les erreurs : un `CONTINUE HANDLER` qui ne journalise ni ne relève la condition masque les problèmes au lieu de les traiter. Pour rejeter une donnée invalide, on emploie `SIGNAL` avec un message clair, plutôt que de laisser passer une valeur incohérente. Enfin, on garde les transactions courtes et l'on reste attentif aux instructions provoquant une validation implicite (le DDL, notamment) au milieu d'un traitement.

## Penser à la sécurité

Le contexte d'exécution mérite une décision consciente : avec `SQL SECURITY DEFINER` (la valeur par défaut), une routine s'exécute avec les privilèges de son définisseur. Bien employé, ce mécanisme permet d'accorder un simple droit `EXECUTE` sur une procédure sans donner d'accès direct aux tables — la routine devient une porte d'entrée contrôlée. La vigilance s'impose dès qu'on construit du SQL dynamique (8.5.1) : concaténer une entrée utilisateur dans une requête ouvre la voie à l'injection. On valide alors les identifiants contre une liste blanche et l'on paramètre les valeurs (voir 17.8).

## Déclarer avec exactitude, viser la performance

Quelques détails techniques ont des effets bien concrets. Déclarer honnêtement les caractéristiques d'une fonction (`DETERMINISTIC`, niveau d'accès aux données) n'est pas une formalité : cela conditionne des optimisations et, sous journalisation binaire, la création même de la fonction (8.2.2). Côté performance, on indexe les requêtes que les routines exécutent, on évite les triggers coûteux qui s'exécutent à chaque ligne, et l'on reste conscient du comportement en réplication (events désactivés sur les réplicas, déterminisme des fonctions, exécution des triggers en réplication par lignes).

## Rendre le code maintenable

Une routine se lit bien plus souvent qu'elle ne s'écrit. Des conventions de nommage cohérentes (un préfixe distinguant procédures, fonctions et triggers ; des préfixes pour les paramètres et les variables locales), une mise en forme soignée et un `COMMENT` décrivant le rôle de la routine facilitent grandement la reprise. On garde chaque routine *focalisée* sur une responsabilité, plutôt que d'y accumuler une logique tentaculaire. Enfin, le code stocké reste du code : il a vocation à être versionné dans un gestionnaire de sources (sous forme de scripts DDL), déployé de façon reproductible grâce à `CREATE OR REPLACE`, et — bien que ce chapitre ne propose pas d'exercices — testé comme n'importe quel composant logiciel.

## En résumé

La programmation côté serveur n'est ni à fuir ni à généraliser : c'est un outil à employer à bon escient. Les meilleures routines sont celles qui font une chose utile, le font de manière sûre et lisible, et restent simples. Ce chapitre a parcouru l'ensemble des objets — procédures, fonctions, triggers, events, curseurs — ainsi que la gestion des erreurs et le contrôle de flux qui les animent. Le chapitre suivant poursuit avec un autre moyen d'organiser et d'exposer les données : les vues.

⏭️ [Vues et Données Virtuelles](/09-vues-et-donnees-virtuelles/README.md)
