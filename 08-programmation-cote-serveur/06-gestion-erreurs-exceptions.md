🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.6 Gestion des erreurs et exceptions (`DECLARE HANDLER`)

Lorsqu'une instruction, dans un programme stocké, rencontre un problème, elle lève une *condition* : une erreur, un avertissement, ou l'absence de données. Sans traitement, une erreur interrompt le programme et remonte à l'appelant. Gérer les erreurs, c'est *intercepter* ces conditions pour y réagir — journaliser, annuler une transaction, poursuivre, ou transformer l'erreur — au moyen de gestionnaires déclarés avec `DECLARE … HANDLER`.

## Conditions : `SQLSTATE` et codes d'erreur

Chaque condition est identifiée de deux façons complémentaires : par un **`SQLSTATE`**, code standard de cinq caractères, et par un **numéro d'erreur** propre à MariaDB. Les deux premiers caractères du `SQLSTATE` (sa *classe*) en indiquent la nature : `00` signale le succès, `01` un avertissement, `02` l'absence de données (*not found*), et les autres classes signalent des erreurs. La classe `45` est réservée aux exceptions définies par l'utilisateur.

## `DECLARE HANDLER` : intercepter une condition

Un *gestionnaire* associe une réaction à une ou plusieurs conditions :

```sql
DECLARE { CONTINUE | EXIT } HANDLER
    FOR valeur_condition [, valeur_condition] …
    instruction
```

Le **type** détermine ce qui se passe une fois le gestionnaire exécuté : `CONTINUE` reprend l'exécution à l'instruction suivant celle qui a échoué ; `EXIT` quitte le bloc `BEGIN … END` dans lequel le gestionnaire est déclaré. (Le type `UNDO` prévu par le standard SQL n'est pas pris en charge par MariaDB.)

Les **valeurs de condition** peuvent être un `SQLSTATE`, un numéro d'erreur MariaDB, une condition nommée (voir ci-dessous), ou l'un des trois raccourcis : `NOT FOUND` (toutes les conditions de classe `02`), `SQLWARNING` (classe `01`) et `SQLEXCEPTION` (toutes les erreurs non couvertes par les deux précédents). L'instruction associée peut être unique ou un bloc `BEGIN … END`. Le gestionnaire `NOT FOUND` rencontré en 8.5 pour détecter la fin d'un curseur est donc un cas particulier de ce mécanisme.

L'exemple suivant illustre un usage central : garantir l'intégrité d'une transaction en l'annulant à la moindre erreur, puis en propageant celle-ci.

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE transferer(IN p_de INT, IN p_vers INT, IN p_montant DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;
        UPDATE comptes SET solde = solde - p_montant WHERE id = p_de;
        UPDATE comptes SET solde = solde + p_montant WHERE id = p_vers;
    COMMIT;
END $$

DELIMITER ;
```

## Conditions nommées : `DECLARE CONDITION`

Pour gagner en lisibilité, on peut donner un nom à une condition — un `SQLSTATE` ou un numéro d'erreur — avec `DECLARE … CONDITION`, puis employer ce nom dans les gestionnaires et dans `SIGNAL` :

```sql
DECLARE prix_invalide CONDITION FOR SQLSTATE '45000';
```

## Lever une erreur : `SIGNAL` et `RESIGNAL`

`SIGNAL` déclenche explicitement une condition, typiquement pour rejeter une donnée invalide. On lui associe un `SQLSTATE` (par convention `45000` pour une exception applicative) et des informations comme le message (`MESSAGE_TEXT`) ou un numéro d'erreur (`MYSQL_ERRNO`) :

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE definir_prix(IN p_id INT, IN p_prix DECIMAL(10,2))
BEGIN
    IF p_prix < 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Le prix ne peut pas être négatif';
    END IF;
    UPDATE produits SET prix = p_prix WHERE id = p_id;
END $$

DELIMITER ;
```

`RESIGNAL`, utilisable uniquement à l'intérieur d'un gestionnaire, **relève** la condition courante, éventuellement modifiée. C'est ce qui permet, dans l'exemple de transaction plus haut, d'annuler localement puis de propager l'erreur à l'appelant après l'avoir traitée.

## Inspecter l'erreur : `GET DIAGNOSTICS`

À l'intérieur d'un gestionnaire, on ne connaît pas a priori le détail de la condition interceptée. `GET DIAGNOSTICS` permet de l'obtenir : le nombre de conditions, et, pour une condition donnée, son numéro (`MYSQL_ERRNO`), son `SQLSTATE` (`RETURNED_SQLSTATE`) et son message (`MESSAGE_TEXT`). C'est indispensable pour journaliser une erreur ou adapter le traitement à sa nature.

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE inserer_avec_log(IN p_valeur INT)
BEGIN
    DECLARE v_errno INT;
    DECLARE v_msg   TEXT;

    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1
            v_errno = MYSQL_ERRNO, v_msg = MESSAGE_TEXT;
        INSERT INTO erreurs_log (numero, message, survenu_le)
        VALUES (v_errno, v_msg, NOW());
    END;

    INSERT INTO valeurs (valeur) VALUES (p_valeur);
END $$

DELIMITER ;
```

## Ordre et portée des gestionnaires

Comme les curseurs, les gestionnaires obéissent à l'ordre des déclarations dans un bloc : les variables et conditions d'abord, puis les curseurs, enfin les gestionnaires (voir 8.1.1). Un gestionnaire couvre les conditions levées dans le bloc où il est déclaré, y compris dans les blocs imbriqués — ce qui permet de placer un gestionnaire général au niveau le plus externe et des gestionnaires plus spécifiques dans des sous-blocs.

## En mode Oracle : exceptions PL/SQL

En mode Oracle (`SET sql_mode = ORACLE`), MariaDB propose en complément une gestion des exceptions à la manière de PL/SQL : un bloc peut comporter une section `EXCEPTION` introduisant des clauses `WHEN … THEN`, l'instruction `RAISE` permet de déclencher une exception, et les fonctions `SQLCODE` et `SQLERRM` renvoient respectivement le code et le message de la dernière erreur. Ce style facilite la reprise de code PL/SQL existant, mais recouvre les mêmes besoins que les gestionnaires standard décrits ci-dessus.

---

Les gestionnaires s'appuient sur des variables et des structures de contrôle — `IF`, `LOOP`, `LEAVE` — que l'on a utilisées dans les exemples sans les détailler. La section suivante les présente : **[variables et contrôle de flux](07-variables-flow-control.md)**.

⏭️ [Variables et flow control (IF, CASE, LOOP, WHILE, REPEAT)](/08-programmation-cote-serveur/07-variables-flow-control.md)
