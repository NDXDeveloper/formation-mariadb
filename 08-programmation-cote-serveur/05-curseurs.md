🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5 Curseurs

Le SQL est par nature *ensembliste* : une requête manipule un ensemble de lignes en une seule opération. Il arrive pourtant qu'un traitement doive examiner les lignes d'un résultat **une par une**, par exemple pour appliquer à chacune une logique procédurale qui ne s'exprime pas simplement par une instruction ensembliste. C'est le rôle du *curseur* : un mécanisme, utilisable à l'intérieur d'un programme stocké (procédure, fonction, trigger ou event), qui parcourt séquentiellement les lignes d'un jeu de résultats.

## Le principe : traiter ligne à ligne

Un curseur s'associe à une requête `SELECT`. Une fois ouvert, il pointe successivement sur chaque ligne du résultat : on lit la ligne courante dans des variables, on la traite, puis on avance à la suivante, jusqu'à épuisement. Le curseur fait ainsi le pont entre le monde ensembliste du SQL et la logique séquentielle du langage procédural.

## Le cycle de vie : `DECLARE`, `OPEN`, `FETCH`, `CLOSE`

L'utilisation d'un curseur suit quatre étapes :

- `DECLARE nom CURSOR FOR requête` — déclare le curseur en l'associant à une requête (qui n'est pas encore exécutée).
- `OPEN nom` — exécute la requête et positionne le curseur avant la première ligne.
- `FETCH nom INTO var1, var2, …` — récupère la ligne courante dans des variables (dont le nombre et l'ordre doivent correspondre aux colonnes), puis avance d'une ligne.
- `CLOSE nom` — libère le curseur une fois le parcours terminé.

## Détecter la fin des données : `NOT FOUND`

Lorsque `FETCH` est appelé au-delà de la dernière ligne, il déclenche la condition `NOT FOUND`. Sans précaution, cette condition interromprait le programme. On déclare donc un gestionnaire qui l'intercepte — typiquement pour positionner un indicateur signalant la fin :

```sql
DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fini = 1;
```

On teste ensuite cet indicateur après chaque `FETCH` pour sortir de la boucle. La gestion des erreurs et des conditions, dont ce gestionnaire est un cas particulier, est détaillée en 8.6.

## Un parcours complet

L'exemple suivant rassemble ces éléments : un curseur qui parcourt les commandes en attente et applique à chacune un traitement procédural.

```sql
DELIMITER $$

CREATE OR REPLACE PROCEDURE traiter_commandes_en_attente()
BEGIN
    DECLARE v_fini INT DEFAULT 0;
    DECLARE v_id   INT;

    DECLARE cur CURSOR FOR
        SELECT id FROM commandes WHERE statut = 'en_attente';

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fini = 1;

    OPEN cur;

    boucle: LOOP
        FETCH cur INTO v_id;
        IF v_fini = 1 THEN
            LEAVE boucle;
        END IF;

        -- logique procédurale appliquée à chaque commande
        CALL traiter_commande(v_id);
    END LOOP;

    CLOSE cur;
END $$

DELIMITER ;
```

On y reconnaît les quatre étapes, encadrées par le gestionnaire `NOT FOUND` et une boucle `LOOP` étiquetée que l'on quitte par `LEAVE` (les structures de contrôle sont vues en 8.7).

## Caractéristiques et bon usage

Les curseurs de MariaDB présentent quelques caractéristiques à connaître. Ils sont **en lecture seule** — on ne peut pas modifier les données à travers eux — et **unidirectionnels** : le parcours ne va que de l'avant, une ligne à la fois, sans retour en arrière ni saut. Par ailleurs, les déclarations à l'intérieur d'un bloc suivent un ordre imposé : les variables et conditions d'abord, puis les curseurs, enfin les gestionnaires (voir 8.1.1).

Un point de méthode, enfin : un curseur traite les lignes une à une, ce qui est presque toujours **plus lent** qu'une opération ensembliste équivalente. Avant d'écrire un curseur, il faut donc se demander si le traitement ne pourrait pas s'exprimer en une seule instruction SQL. Le curseur ne se justifie que lorsque la logique appliquée à chaque ligne est véritablement procédurale — comme l'appel d'une procédure par ligne dans l'exemple ci-dessus.

## Les sous-sections : au-delà du curseur classique

Les deux sections qui suivent étendent ce mécanisme avec des nouveautés de la série 12.x :

- **8.5.1 — Curseurs sur prepared statements** 🆕 : associer un curseur à une instruction préparée, construite dynamiquement, plutôt qu'à une requête figée.
- **8.5.2 — `SYS_REFCURSOR` et `max_open_cursors`** 🆕 : les *curseurs de référence* (cursor variables), que l'on peut passer entre routines dans l'esprit d'Oracle, et la limite du nombre de curseurs ouverts simultanément.

---

Commençons par la première extension : les **[curseurs sur prepared statements](05.1-curseurs-prepared-statements.md)** 🆕.

⏭️ [Curseurs sur prepared statements](/08-programmation-cote-serveur/05.1-curseurs-prepared-statements.md)
