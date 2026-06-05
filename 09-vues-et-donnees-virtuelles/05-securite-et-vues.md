🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 · Sécurité et vues : masquage de données

> **Chapitre 9 — Vues et Données Virtuelles** · Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

Au-delà de la simplification des requêtes, les vues constituent un **outil de contrôle d'accès** : en interposant une couche d'indirection entre les utilisateurs et les tables, elles permettent de n'exposer que ce qui est autorisé. Cette section présente les quatre leviers de sécurité offerts par les vues — restreindre les **colonnes**, restreindre les **lignes**, **masquer** les valeurs sensibles, et contrôler le **contexte d'exécution** via `DEFINER` / `SQL SECURITY` — puis expose le motif d'usage classique et ses limites.

MariaDB ne dispose pas de fonctionnalité native de « masquage dynamique » au niveau SQL (à la manière du `MASKED WITH` de SQL Server). Les vues sont donc l'**approche idiomatique** pour masquer des données ; un masquage centralisé au niveau du proxy reste possible via le filtre de masquage de **MaxScale** (§14.4), évoqué plus loin.

## Restreindre les colonnes (sécurité verticale)

Le premier levier consiste à n'exposer qu'un **sous-ensemble des colonnes** d'une table, en omettant celles qui sont sensibles. Une vue « annuaire » peut ainsi révéler l'identité des employés sans jamais montrer leur salaire :

```sql
CREATE VIEW v_annuaire AS
SELECT id, nom, prenom, dept_id
FROM employes;
```

La colonne `salaire` n'apparaît pas : un utilisateur disposant d'un accès à `v_annuaire` — mais pas à la table `employes` — ne peut en aucune façon la consulter à travers cette vue. C'est précisément la combinaison avec le système de privilèges (présentée plus bas) qui rend ce cloisonnement effectif.

## Restreindre les lignes (sécurité horizontale)

Le deuxième levier filtre les **lignes visibles** via la clause `WHERE`. On peut limiter une vue à un périmètre métier (un département, une région, les enregistrements actifs…) ou, plus finement, à l'**utilisateur réellement connecté**.

Pour ce dernier cas, on s'appuie sur `SESSION_USER()` (équivalent à `USER()`), qui renvoie toujours le **compte client courant**. En supposant que la table `employes` comporte une colonne `login` reliant chaque ligne à un compte de connexion :

```sql
CREATE VIEW v_mon_dossier AS
SELECT id, nom, prenom, salaire, date_embauche
FROM employes
WHERE login = SUBSTRING_INDEX(SESSION_USER(), '@', 1);
```

Chaque utilisateur interrogeant `v_mon_dossier` ne voit alors que **sa propre ligne**.

> **Quelle fonction d'identité utiliser ?** Pour filtrer selon l'utilisateur *réellement connecté*, employez `SESSION_USER()` ou `USER()`, qui renvoient toujours le compte client. Évitez `CURRENT_USER()` dans ce contexte : cette fonction reflète le **contexte de privilèges**, qui — avec une vue en `SQL SECURITY DEFINER` (voir ci-dessous) — peut correspondre au *définisseur* et non à l'appelant.

Côté **écriture**, la sécurité horizontale se complète de `WITH CHECK OPTION` (§9.4), qui empêche un utilisateur d'insérer ou de modifier des lignes en dehors de son périmètre autorisé — mécanisme particulièrement utile en architecture multi-tenant (§20.4).

## Masquer les valeurs sensibles (data masking)

Plutôt que de supprimer purement et simplement une colonne, on peut en **transformer** la valeur pour n'en révéler qu'une partie, ou la rendre méconnaissable. C'est le **masquage de données** : la vue applique une expression de transformation à la volée.

En supposant que `employes` comporte aussi les colonnes `email` et `telephone` :

```sql
CREATE VIEW v_employes_masque AS
SELECT
    id,
    nom,
    CONCAT(LEFT(prenom, 1), '***')                  AS prenom,
    CONCAT('***@', SUBSTRING_INDEX(email, '@', -1)) AS email,      -- ***@domaine.fr
    CONCAT('******', RIGHT(telephone, 4))           AS telephone,  -- ******1234
    NULL                                            AS salaire     -- masquée
FROM employes;
```

Plusieurs techniques sont illustrées ici : **troncature partielle** (initiale du prénom, quatre derniers chiffres du téléphone), **conservation du domaine** d'une adresse e-mail tout en masquant l'identifiant, et **suppression complète** d'une valeur numérique. Selon les besoins, on peut aussi recourir au hachage (`SHA2()`) pour produire une valeur stable mais non réversible, ou à la pseudonymisation.

> **Important.** Le masquage par une vue ne transforme que ce qui est **lu à travers la vue**. La donnée brute demeure intacte dans la table de base : quiconque y accède directement — ou accède aux sauvegardes et aux réplicas — voit les valeurs réelles. Le masquage n'est donc une protection qu'**en conjonction** avec une restriction stricte des privilèges sur les tables.

## Le cœur du dispositif : `DEFINER` et `SQL SECURITY`

Tous les leviers précédents ne deviennent réellement sécurisants que grâce au **contexte d'exécution** de la vue, gouverné par deux clauses (introduites mais reportées en §9.1) :

- **`DEFINER = compte`** désigne le compte au nom duquel la vue est définie. En l'absence de précision, il s'agit de l'utilisateur qui crée la vue (`CURRENT_USER`).
- **`SQL SECURITY { DEFINER | INVOKER }`** détermine *avec les privilèges de qui* la requête sous-jacente s'exécute :
  - **`DEFINER`** (valeur **par défaut**) : la vue s'exécute avec les privilèges du **définisseur**. L'appelant n'a donc *pas besoin* de droits sur les tables de base — il lui suffit d'avoir le droit d'accéder à la **vue**.
  - **`INVOKER`** : la vue s'exécute avec les privilèges de **l'appelant**, qui doit alors posséder lui-même les droits requis sur les tables sous-jacentes.

| | `SQL SECURITY DEFINER` (défaut) | `SQL SECURITY INVOKER` |
|---|---|---|
| Privilèges utilisés à l'exécution | ceux du **définisseur** | ceux de **l'appelant** |
| L'appelant doit-il avoir accès aux tables de base ? | **Non** | **Oui** |
| Usage typique | masquage, accès contrôlé sans droits sur les tables | vue respectant les droits propres de l'appelant |
| Risque principal | escalade de privilèges si la vue est mal conçue | — |

C'est le mode `DEFINER` qui rend possible le motif fondamental du masquage : un compte privilégié définit la vue, et des comptes sans aucun droit sur les tables peuvent néanmoins lire la vue — strictement dans les limites que celle-ci impose.

## Le motif « accès par la vue uniquement »

Assemblons les pièces. Un administrateur, qui dispose des droits de lecture sur `employes`, crée la vue avec une sécurité de type définisseur :

```sql
CREATE DEFINER = 'admin'@'localhost' SQL SECURITY DEFINER
VIEW v_annuaire AS
SELECT id, nom, prenom, dept_id
FROM employes;
```

On accorde ensuite à un compte de lecture le droit d'interroger **la vue**, sans jamais lui donner accès à **la table** :

```sql
GRANT SELECT ON ma_base.v_annuaire TO 'rh_lecteur'@'%';
-- 'rh_lecteur' ne reçoit aucun privilège sur ma_base.employes
```

Le résultat, du point de vue de `rh_lecteur` :

```sql
SELECT * FROM v_annuaire;   -- OK : exécutée avec les droits d'« admin »
SELECT * FROM employes;     -- ERROR : accès refusé à la table de base
```

En pratique, on rattache plutôt ces droits à un **rôle** (chapitre 10) qu'à des comptes individuels, et l'on combine librement les trois leviers : une même vue peut **projeter** un sous-ensemble de colonnes, **filtrer** les lignes par utilisateur ou par tenant, et **masquer** les valeurs résiduelles sensibles — le tout exécuté en contexte `DEFINER`, les tables de base restant hors d'atteinte directe.

## Limites et précautions

Les vues offrent une protection **commode mais grossière**, à manier avec lucidité :

- **Ce n'est pas une frontière infranchissable.** Le cloisonnement ne tient que si les utilisateurs **n'ont pas** d'accès direct aux tables de base. Le motif repose autant sur la *non-attribution* de privilèges sur les tables que sur la vue elle-même.
- **La donnée brute subsiste.** Masquage et filtrage n'agissent qu'à la lecture via la vue ; les tables, sauvegardes et réplicas contiennent les valeurs réelles. Pour une protection en profondeur, on combine ces vues avec le **chiffrement au repos** (§18.7), le **chiffrement des connexions** (§10.7) et le principe de moindre privilège.
- **Le définisseur doit exister.** Une vue en `SQL SECURITY DEFINER` dont le compte définisseur a été supprimé devient **inutilisable** : son invocation échoue (« the user specified as a definer does not exist »). MariaDB avertit d'ailleurs dès la création si le définisseur est inconnu.
- **Le mode `DEFINER` est puissant — donc sensible.** Une vue en contexte définisseur équivaut à une délégation, étroite et contrôlée, des droits de ce compte. Il faut donc **auditer** ce que chaque vue expose et **éviter** de choisir un compte trop privilégié (comme `root`) comme définisseur sans nécessité.
- **Fuites par inférence.** Même filtrée ou masquée, une vue peut laisser deviner des informations via des agrégats, des jointures, l'ordre des résultats ou les messages d'erreur. Les vues procurent une protection logique, pas une garantie cryptographique.
- **Coût en performance.** Des expressions de masquage complexes ou des filtres élaborés ajoutent une charge à chaque interrogation — un aspect à considérer au regard des algorithmes d'exécution étudiés à la section suivante.

Pour un masquage **centralisé et indépendant des requêtes**, le filtre de masquage de **MaxScale** (§14.4) constitue une alternative ou un complément : il obscurcit les colonnes désignées directement dans les résultats transitant par le proxy, quel que soit le SQL émis.

## En résumé

Les vues sécurisent l'accès aux données selon quatre leviers : **restriction des colonnes** (sécurité verticale), **restriction des lignes** (sécurité horizontale, éventuellement par utilisateur via `SESSION_USER()`), **masquage** des valeurs sensibles, et **contexte d'exécution** via `DEFINER` / `SQL SECURITY`. Le mode `SQL SECURITY DEFINER` (par défaut) autorise le motif « accès par la vue uniquement », où des comptes sans droits sur les tables consultent néanmoins des données filtrées et masquées. Cette protection reste toutefois conditionnée à une stricte gestion des privilèges et ne remplace ni le chiffrement ni les contrôles applicatifs.

Plusieurs mécanismes évoqués ici — filtres, expressions de masquage, contexte de sécurité — influent sur la **manière dont MariaDB exécute** une vue. C'est précisément l'objet de la **section 9.6 — Performance des vues : `MERGE` vs `TEMPTABLE`**.

⏭️ [Performance des vues : MERGE vs TEMPTABLE](/09-vues-et-donnees-virtuelles/06-performance-vues.md)
