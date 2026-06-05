🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Procédures stockées

Une *procédure stockée* est un programme nommé, conservé de façon permanente dans la base de données, qui regroupe une ou plusieurs instructions SQL accompagnées d'une logique procédurale (variables, conditions, boucles, gestion d'erreurs). Une fois créée, elle s'exécute à la demande par un simple appel `CALL`, depuis une application, un script ou une autre routine. Contrairement à une requête isolée, elle encapsule tout un traitement — souvent composé de plusieurs étapes — derrière un point d'entrée unique et réutilisable.

C'est la première brique de la programmation côté serveur, et la plus polyvalente : une procédure peut aussi bien insérer un jeu de données cohérent dans plusieurs tables au sein d'une même transaction que produire un rapport, parcourir des lignes une à une ou orchestrer une opération de maintenance.

## Procédure ou fonction ?

MariaDB distingue deux types de routines stockées qu'il ne faut pas confondre. Une **procédure** est appelée explicitement par `CALL` ; elle ne renvoie pas de valeur directement exploitable dans une expression, mais peut communiquer des résultats par l'intermédiaire de paramètres de sortie ou en renvoyant un ou plusieurs jeux de résultats au client. Une **fonction**, en revanche, retourne une valeur unique et s'utilise au cœur d'une expression SQL, exactement comme une fonction native (`SELECT ma_fonction(x) FROM …`).

En résumé : on appelle une procédure pour *faire quelque chose*, on évalue une fonction pour *obtenir une valeur*. Les fonctions font l'objet de la section 8.2 ; ce chapitre commence par les procédures car elles imposent moins de restrictions et illustrent plus largement les possibilités du langage.

## Pourquoi encapsuler un traitement dans une procédure ?

Au-delà des avantages généraux de la programmation côté serveur évoqués en introduction du chapitre, les procédures apportent des bénéfices propres. Elles regroupent une séquence d'opérations en une unité logique appelable d'un seul ordre, ce qui réduit le trafic réseau : là où une application enchaînerait plusieurs requêtes et autant d'allers-retours, un unique `CALL` suffit. Elles offrent aussi une forme d'interface stable à la base — les applications appellent des procédures aux signatures connues sans dépendre de la structure exacte des tables, ce qui facilite l'évolution du schéma. Enfin, elles constituent un point naturel pour regrouper, au sein d'une même transaction, plusieurs modifications qui doivent réussir ou échouer ensemble.

## Cycle de vie et stockage

Une procédure appartient à une base de données précise : son nom complet est de la forme `nom_base.nom_procedure`, et elle n'est visible que dans le contexte de cette base (sauf à la préfixer). Elle est créée par `CREATE PROCEDURE` (détaillé en 8.1.1), modifiable par `ALTER PROCEDURE` et supprimée par `DROP PROCEDURE`.

Les routines existantes se consultent sans avoir à mémoriser leur code : la vue `INFORMATION_SCHEMA.ROUTINES` recense toutes les procédures et fonctions, `SHOW PROCEDURE STATUS` en donne un aperçu rapide, et `SHOW CREATE PROCEDURE nom_procedure` restitue leur définition complète.

Concrètement, `SHOW CREATE PROCEDURE` reconstruit la définition stockée — en y faisant apparaître le `DEFINER` et les caractéristiques, **même s'ils n'avaient pas été écrits explicitement** à la création :

```
SHOW CREATE PROCEDURE nb_commandes_client\G
*************************** 1. row ***************************
       Procedure: nb_commandes_client
Create Procedure: CREATE DEFINER=`root`@`localhost` PROCEDURE `nb_commandes_client`(
                      IN p_client_id INT, OUT p_total INT)
                      READS SQL DATA
                      COMMENT 'Renvoie le nombre de commandes d''un client'
                  BEGIN
                      SELECT COUNT(*) INTO p_total
                        FROM commandes WHERE client_id = p_client_id;
                  END
```

(La commande renvoie aussi, dans d'autres colonnes du `\G`, le `sql_mode` et le jeu de caractères enregistrés avec la routine.) Pour un **inventaire tabulaire**, on interroge `INFORMATION_SCHEMA.ROUTINES`, qui expose en colonnes les caractéristiques de chaque routine :

```sql
SELECT ROUTINE_NAME, ROUTINE_TYPE, DATA_TYPE,
       IS_DETERMINISTIC, SQL_DATA_ACCESS, SECURITY_TYPE
  FROM information_schema.ROUTINES
 WHERE ROUTINE_SCHEMA = 'boutique';
```

| ROUTINE_NAME | ROUTINE_TYPE | DATA_TYPE | IS_DETERMINISTIC | SQL_DATA_ACCESS | SECURITY_TYPE |
|---|---|---|:---:|---|---|
| `nb_commandes_client` | PROCEDURE | | NO | READS SQL DATA | DEFINER |
| `prix_ttc` | FUNCTION | decimal | YES | CONTAINS SQL | DEFINER |

Une **fonction** y porte un `DATA_TYPE` de retour et un `IS_DETERMINISTIC` significatif, là où ces colonnes restent vides ou neutres pour une **procédure** — un moyen rapide de distinguer les deux et de repérer les routines mal déclarées.

Côté droits, créer une procédure requiert le privilège `CREATE ROUTINE`, la modifier ou la supprimer le privilège `ALTER ROUTINE`, et l'appeler le privilège `EXECUTE` — sauf pour son propriétaire, qui les détient implicitement sur ses propres routines.

## Contexte d'exécution et sécurité

Un aspect essentiel, souvent sous-estimé, est le contexte de privilèges dans lequel une procédure s'exécute. Chaque routine possède un *définisseur* (clause `DEFINER`, par défaut l'utilisateur qui l'a créée) et un mode de sécurité (`SQL SECURITY`) qui vaut `DEFINER` par défaut. Concrètement, la procédure s'exécute alors avec les privilèges de son définisseur, et non ceux de l'appelant.

Ce mécanisme est très utile : il permet d'accorder à un utilisateur le droit d'appeler une procédure (`EXECUTE`) sans lui donner d'accès direct aux tables qu'elle manipule — la procédure devient une porte d'entrée contrôlée. À l'inverse, le mode `SQL SECURITY INVOKER` fait s'exécuter la routine avec les droits de l'appelant. Le choix entre les deux est une décision de conception aux implications directes sur la sécurité ; il faut en avoir conscience dès la création.

## Le corps procédural

Le cœur d'une procédure est un bloc d'instructions, généralement délimité par `BEGIN … END`, à l'intérieur duquel cohabitent ordres SQL et constructions procédurales : déclaration de variables locales, structures de contrôle (`IF`, `CASE`, boucles), curseurs pour le parcours ligne à ligne et gestionnaires d'erreurs. Ces éléments sont approfondis plus loin dans le chapitre — curseurs en 8.5, gestion des erreurs en 8.6, contrôle de flux en 8.7 — et ne sont mentionnés ici que pour situer ce qu'une procédure peut contenir.

À noter dès maintenant un point pratique : comme le corps d'une procédure contient lui-même des points-virgules, on redéfinit temporairement le séparateur d'instructions (commande `DELIMITER`) lorsqu'on saisit une définition dans le client en ligne de commande. Ce détail est traité avec la syntaxe en 8.1.1.

## Syntaxe historique et mode Oracle

Comme l'ensemble du langage procédural, les procédures s'écrivent selon deux dialectes. Le mode par défaut suit la syntaxe historique, proche de MySQL. Le mode Oracle (`SET sql_mode = ORACLE`) adopte une écriture compatible avec PL/SQL et débloque des constructions supplémentaires, dont les **tableaux associatifs** (`DECLARE TYPE … TABLE OF … INDEX BY`), abordés en 8.1.4 🆕 et précieux pour migrer du code Oracle existant vers MariaDB 12.3.

## Plan de la section

Les sous-sections suivantes détaillent la mise en œuvre des procédures stockées :

- **8.1.1 — Syntaxe `CREATE PROCEDURE`** : la définition d'une procédure, du bloc `BEGIN … END` à la gestion du délimiteur.
- **8.1.2 — Paramètres `IN`, `OUT`, `INOUT`** : les trois modes de passage des paramètres et leurs usages respectifs.
- **8.1.3 — Appel avec `CALL`** : l'invocation d'une procédure et la récupération de ses résultats.
- **8.1.4 — Tableaux associatifs Oracle** 🆕 : les collections indexées du mode Oracle, pour la compatibilité PL/SQL.

---

Commençons par la définition d'une procédure : la syntaxe de **[`CREATE PROCEDURE`](01.1-syntaxe-create-procedure.md)**.

⏭️ [Syntaxe CREATE PROCEDURE](/08-programmation-cote-serveur/01.1-syntaxe-create-procedure.md)
