🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 — Rôles (CREATE ROLE, SET ROLE, DEFAULT ROLE)

Attribuer les privilèges compte par compte, comme on l'a fait avec `GRANT` en 10.3.1, devient vite ingérable dès que les utilisateurs se multiplient. Les **rôles** apportent la réponse : un rôle est un **ensemble nommé de privilèges**, que l'on accorde une fois à des comptes. Modifier les droits d'un groupe d'utilisateurs revient alors à modifier ceux du rôle, sans toucher à chaque compte individuellement. C'est le fondement du contrôle d'accès basé sur les rôles (RBAC). Cette section couvre la création des rôles, l'attribution de privilèges, l'affectation aux comptes, l'activation (`SET ROLE`) et l'activation automatique (`SET DEFAULT ROLE`).

---

## Pourquoi des rôles ?

Imaginons une dizaine de comptes « rédacteur » devant tous disposer des mêmes droits sur une base. Sans rôle, accorder ou retirer un privilège à ce groupe oblige à répéter l'opération sur chaque compte — et la moindre divergence devient une source d'erreurs. Avec un rôle `redacteur`, on règle les privilèges **au niveau du rôle** ; tous les comptes qui le portent en bénéficient immédiatement. Les rôles centralisent ainsi la gestion des droits et la rendent lisible et cohérente.

Un rôle se distingue d'un compte ordinaire : il est identifié par un **nom seul** (sans partie hôte), n'a pas de mot de passe et **ne peut pas servir à se connecter**. Il vit dans la table `mysql.user` (avec un indicateur `is_role`) mais n'existe que comme conteneur de privilèges.

---

## Créer un rôle : `CREATE ROLE`

```sql
CREATE ROLE lecteur;
CREATE ROLE IF NOT EXISTS redacteur WITH ADMIN CURRENT_USER;
```

La clause `IF NOT EXISTS` évite l'erreur si le rôle existe déjà. La clause `WITH ADMIN` désigne qui pourra **administrer** le rôle (l'attribuer, le révoquer) — par défaut, le compte qui l'a créé (`CURRENT_USER`). On peut aussi confier cette administration à un autre compte ou à un autre rôle.

---

## Donner des privilèges à un rôle : `GRANT … TO role`

On accorde des privilèges à un rôle exactement comme à un compte, avec la syntaxe `GRANT` vue en 10.3.1 — le bénéficiaire est simplement un nom de rôle :

```sql
GRANT SELECT ON boutique.* TO lecteur;
GRANT SELECT, INSERT, UPDATE, DELETE ON boutique.* TO redacteur;
```

Tous les niveaux de portée (10.3.3) et tous les privilèges du catalogue (10.3) sont utilisables. Le rôle accumule les privilèges qu'on lui accorde.

---

## Attribuer un rôle à un compte : `GRANT role TO user`

Une fois le rôle garni, on l'attribue aux comptes concernés :

```sql
GRANT lecteur  TO 'alice'@'localhost';
GRANT redacteur TO 'bob'@'10.0.0.%' WITH ADMIN OPTION;
```

La clause `WITH ADMIN OPTION` permet au bénéficiaire d'attribuer ce rôle à son tour à d'autres comptes. Un rôle peut aussi être attribué à **un autre rôle**, ce qui construit une hiérarchie (voir plus bas).

---

## Activer un rôle : `SET ROLE`

Point essentiel et **spécifique à MariaDB** : les rôles attribués à un compte ne sont **pas actifs par défaut**. Tant qu'aucun rôle n'est activé, le compte ne dispose que de ses privilèges propres. On active un rôle pour la session courante avec `SET ROLE` :

```sql
SET ROLE redacteur;        -- active le rôle pour cette session
SELECT CURRENT_ROLE;       -- affiche 'redacteur'
SET ROLE NONE;             -- désactive tout rôle (CURRENT_ROLE redevient NULL)
```

Une particularité distingue nettement MariaDB de MySQL : **un seul rôle peut être actif à la fois**. `SET ROLE` *remplace* le rôle courant, il ne l'ajoute pas à une liste — c'est le comportement conforme au standard SQL. Là où MySQL autorise plusieurs rôles simultanément, MariaDB n'en garde qu'un.

Tenter d'activer un rôle qui n'existe pas, ou qui n'a pas été attribué au compte, déclenche l'erreur `ERROR 1959 (OP000): Invalid role specification`. La fonction `CURRENT_ROLE` (ou `SELECT CURRENT_ROLE;`) indique à tout moment le rôle actif.

---

## Activation automatique : `SET DEFAULT ROLE`

Pour éviter d'imposer un `SET ROLE` manuel à chaque connexion, on définit un **rôle par défaut** : il est alors activé automatiquement dès la connexion (un `SET ROLE` implicite est exécuté juste après l'authentification).

```sql
SET DEFAULT ROLE lecteur FOR 'alice'@'localhost';
SET DEFAULT ROLE NONE    FOR 'alice'@'localhost';   -- retire le rôle par défaut
```

Sans la clause `FOR`, l'instruction s'applique au compte courant. Quelques règles encadrent cette commande :

- le rôle doit avoir été **préalablement attribué** au compte, et l'on doit avoir le droit de l'activer (si l'on ne peut pas faire `SET ROLE X`, on ne peut pas faire `SET DEFAULT ROLE X`) ;
- définir le rôle par défaut d'un **autre** compte exige un accès en écriture à la base `mysql` ;
- le rôle par défaut est stocké dans la colonne `default_role` de `mysql.user`.

Un détail à connaître : l'enregistrement du rôle par défaut **n'est pas effacé** si le rôle est supprimé ou révoqué. Si ce rôle est ensuite recréé ou ré-attribué, il redevient automatiquement le rôle par défaut du compte.

---

## Hiérarchie de rôles

Puisqu'un rôle peut être attribué à un autre rôle, on peut composer une **hiérarchie**. Activer un rôle active aussi, par transitivité, les rôles qui lui ont été accordés.

```sql
CREATE ROLE consultation;
GRANT SELECT ON boutique.* TO consultation;

-- 'redacteur' hérite des privilèges de 'consultation'
GRANT consultation TO redacteur;
```

Désormais, `SET ROLE redacteur` confère à la fois les privilèges propres de `redacteur` **et** ceux de `consultation`. C'est d'ailleurs la bonne façon de « combiner » plusieurs ensembles de droits dans MariaDB : comme un seul rôle peut être actif à la fois, on ne cumule pas deux rôles en les activant tous les deux — on construit un rôle parent qui agrège les rôles enfants.

---

## Rôles et vues/routines `DEFINER`

Lorsqu'un compte a activé un rôle, il dispose en quelque sorte de **deux identités** : ses propres privilèges et ceux du rôle. Or une vue ou une routine créée avec `SQL SECURITY DEFINER` n'a qu'un seul définisseur. On précise donc, à la création, si le définisseur doit être `CURRENT_USER` (la vue n'utilise alors aucun privilège du rôle) ou `CURRENT_ROLE` (la vue utilise les privilèges du rôle, mais aucun de ceux du compte). Ce choix est à faire avec soin, car une combinaison mal pensée peut produire une vue inutilisable.

---

## Supprimer et révoquer

On retire un rôle d'un compte avec `REVOKE`, et on supprime le rôle lui-même avec `DROP ROLE` :

```sql
REVOKE redacteur FROM 'bob'@'10.0.0.%';   -- le compte perd le rôle
DROP ROLE IF EXISTS redacteur;            -- supprime le rôle et ses attributions
```

`DROP ROLE` retire le rôle de tous les comptes auxquels il était attribué.

---

## Inspecter les rôles

Plusieurs commandes permettent d'examiner la situation :

```sql
SELECT CURRENT_ROLE;                                 -- rôle actif dans la session
SELECT * FROM information_schema.APPLICABLE_ROLES;   -- rôles attribués au compte courant
SELECT * FROM information_schema.ENABLED_ROLES;      -- rôles actuellement actifs
SHOW GRANTS;                                         -- inclut les rôles attribués et leurs droits
```

`SHOW GRANTS` est particulièrement utile, mais une subtilité mérite d'être connue. Interrogé pour **un compte précis**, il affiche les privilèges **directs** du compte et les **rôles qui lui sont attribués** — mais **pas le détail** des privilèges portés par ces rôles :

```text
SHOW GRANTS FOR 'bob'@'10.0.0.%';
GRANT `redacteur` TO `bob`@`10.0.0.%`
GRANT USAGE ON *.* TO `bob`@`10.0.0.%` IDENTIFIED BY PASSWORD '*…'
GRANT SELECT ON `boutique`.`clients` TO `bob`@`10.0.0.%`
```

On y lit donc l'attribution du rôle (`GRANT redacteur TO bob`), la ligne `USAGE` qui matérialise le compte, et ses droits directs. Pour voir les privilèges **hérités**, deux moyens : inspecter le rôle lui-même (`SHOW GRANTS FOR 'redacteur'`), ou **activer** le rôle dans sa propre session — un simple `SHOW GRANTS` (sans `FOR`) y joint alors les droits des rôles **actifs**, sous la forme de lignes `GRANT … TO 'redacteur'`.

---

## Bonnes pratiques

- **Nommer les rôles par fonction** (`lecteur`, `redacteur`, `admin_boutique`) pour refléter les responsabilités plutôt que les personnes.
- **Centraliser les droits dans les rôles** plutôt que de les disperser sur les comptes : c'est tout l'intérêt du RBAC.
- **Définir un rôle par défaut** pour les comptes applicatifs, afin que les privilèges soient actifs dès la connexion sans `SET ROLE` explicite.
- **Composer par hiérarchie** lorsqu'un compte doit cumuler plusieurs ensembles de droits — un seul rôle pouvant être actif à la fois dans MariaDB.
- **Réserver `WITH ADMIN OPTION`** à un nombre restreint de comptes, comme pour `WITH GRANT OPTION` (10.3.1).

---

## À retenir

Un rôle est un ensemble nommé de privilèges, sans mot de passe ni connexion. On le crée avec `CREATE ROLE`, on le garnit avec `GRANT … TO role`, on l'attribue avec `GRANT role TO user`, on l'active avec `SET ROLE` et on l'active automatiquement à la connexion avec `SET DEFAULT ROLE`. **Spécificité MariaDB** : un seul rôle est actif à la fois (`SET ROLE` remplace le rôle courant) ; pour cumuler des droits, on construit une **hiérarchie** de rôles. `CURRENT_ROLE` indique le rôle actif, `SHOW GRANTS` et les vues `APPLICABLE_ROLES`/`ENABLED_ROLES` permettent l'inspection. Les rôles sont l'outil de référence pour une gestion des privilèges maintenable.

---

> 🔔 **Note de version.** Les rôles existent dans MariaDB depuis la 10.0.5 et les rôles par défaut (`SET DEFAULT ROLE`) depuis la 10.1.1 ; leur comportement est stable dans **MariaDB 12.3 LTS**. La limite d'**un seul rôle actif à la fois** est un choix de conception conforme au standard SQL, qui distingue durablement MariaDB de MySQL. Sources : *MariaDB Server Documentation — SET ROLE / SET DEFAULT ROLE / Roles Overview*.

⏭️ [Authentification : Plugins](/10-securite-gestion-utilisateurs/05-authentification-plugins.md)
