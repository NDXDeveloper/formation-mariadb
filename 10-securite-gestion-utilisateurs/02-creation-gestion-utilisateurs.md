🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 — Création et gestion des utilisateurs

La gestion des comptes constitue la première opération concrète de sécurisation d'une instance MariaDB. Elle s'appuie sur un petit ensemble d'instructions dédiées — `CREATE USER`, `ALTER USER`, `RENAME USER`, `SET PASSWORD` et `DROP USER` — qu'il faut toujours préférer à la modification directe des tables système. Ces instructions mettent à jour automatiquement les tables de droits *et* leurs copies en mémoire, sans nécessiter de `FLUSH PRIVILEGES`. Comme vu en 10.1, chaque compte est identifié par le couple `'utilisateur'@'hôte'` ; cette identité se retrouve dans toutes les commandes ci-dessous.

---

## Créer un compte : `CREATE USER`

La forme la plus simple crée un compte assorti d'un mot de passe :

```sql
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'mot_de_passe';
```

Le mot de passe fourni en clair après `IDENTIFIED BY` est haché par le serveur avant d'être stocké ; il n'apparaît jamais en clair dans les tables de droits. Si l'on omet la partie hôte (`CREATE USER 'alice'`), MariaDB la complète implicitement par `'%'`, c'est-à-dire « depuis n'importe quel hôte » — un comportement à connaître, car il est rarement souhaitable. La bonne pratique est de **toujours préciser l'hôte explicitement**.

Pour éviter une erreur lorsque le compte pourrait déjà exister, on utilise `IF NOT EXISTS` ; à l'inverse, `CREATE OR REPLACE USER` (propre à MariaDB) supprime puis recrée le compte :

```sql
CREATE USER IF NOT EXISTS 'bob'@'10.0.0.%' IDENTIFIED BY 'mot_de_passe';
CREATE OR REPLACE USER 'bob'@'10.0.0.%' IDENTIFIED BY 'autre_mot_de_passe';
```

### Choisir la méthode d'authentification

Plutôt qu'un mot de passe classique, on peut associer le compte à un **plugin d'authentification** via `IDENTIFIED VIA`. MariaDB autorise même plusieurs méthodes alternatives pour un même compte, séparées par `OR` : la connexion réussit si l'une d'elles est satisfaite.

```sql
-- Authentification moderne par ed25519
CREATE USER 'carol'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('mot_de_passe');

-- Deux méthodes acceptées : ed25519, ou le socket Unix (identité système)
CREATE USER 'svc'@'localhost'
  IDENTIFIED VIA ed25519 USING PASSWORD('mot_de_passe')
  OR unix_socket;
```

Le panorama complet des plugins disponibles est détaillé aux sections 10.5 et 10.6 ; on se contente ici de savoir que le choix de la méthode se fait dès la création du compte.

### Définir plusieurs comptes à la fois

Une même instruction peut créer plusieurs comptes :

```sql
CREATE USER 'lecture'@'%'  IDENTIFIED BY '...',
            'ecriture'@'%' IDENTIFIED BY '...';
```

---

## Les options de compte

`CREATE USER` (et `ALTER USER`) acceptent, après la méthode d'authentification, plusieurs clauses qui encadrent l'usage du compte. Elles s'écrivent dans l'ordre : exigences TLS (`REQUIRE`), limites de ressources (`WITH`), puis options de mot de passe et de verrouillage.

```sql
CREATE USER 'app'@'%'
  IDENTIFIED BY '...'
  REQUIRE SSL
  WITH MAX_USER_CONNECTIONS 50
  PASSWORD EXPIRE INTERVAL 90 DAY;
```

### Exigences de chiffrement (`REQUIRE`)

La clause `REQUIRE` impose des conditions sur la connexion TLS : `REQUIRE NONE` (aucune exigence), `REQUIRE SSL` (connexion chiffrée obligatoire), `REQUIRE X509` (certificat client valide), ou des contraintes plus fines sur l'émetteur, le sujet ou le chiffrement (`ISSUER`, `SUBJECT`, `CIPHER`). La configuration du chiffrement côté serveur et la gestion des certificats sont traitées à la section 10.7.

### Limites de ressources (`WITH`)

La clause `WITH` plafonne la consommation d'un compte :

- `MAX_QUERIES_PER_HOUR` — nombre de requêtes par heure.
- `MAX_UPDATES_PER_HOUR` — nombre d'instructions de modification par heure.
- `MAX_CONNECTIONS_PER_HOUR` — nombre de connexions par heure.
- `MAX_USER_CONNECTIONS` — nombre de connexions **simultanées**.
- `MAX_STATEMENT_TIME` — durée maximale (en secondes) d'une requête, propre à MariaDB.

Ces garde-fous limitent l'impact d'un compte mal configuré ou d'une application emballée.

### Expiration des mots de passe (`PASSWORD EXPIRE`)

MariaDB permet d'imposer un renouvellement périodique des mots de passe :

- `PASSWORD EXPIRE` — expire le mot de passe immédiatement (l'utilisateur devra le changer à la prochaine connexion).
- `PASSWORD EXPIRE INTERVAL n DAY` — expiration tous les *n* jours.
- `PASSWORD EXPIRE NEVER` — pas d'expiration.
- `PASSWORD EXPIRE DEFAULT` — applique la politique globale définie par `default_password_lifetime`.

### Verrouillage de compte (`ACCOUNT LOCK`)

Un compte peut être verrouillé sans être supprimé : il subsiste avec ses privilèges, mais ne peut plus établir de connexion. C'est le moyen idéal de **désactiver temporairement** un accès.

```sql
ALTER USER 'app'@'%' ACCOUNT LOCK;    -- désactive le compte
ALTER USER 'app'@'%' ACCOUNT UNLOCK;  -- le réactive
```

---

## Modifier un compte : `ALTER USER`

`ALTER USER` modifie un compte existant : mot de passe, méthode d'authentification, exigences TLS, limites de ressources, expiration ou verrouillage. La clause `IF EXISTS` évite l'erreur si le compte est absent.

```sql
-- Changer le mot de passe
ALTER USER 'alice'@'localhost' IDENTIFIED BY 'nouveau_mot_de_passe';

-- Forcer l'expiration au prochain login
ALTER USER IF EXISTS 'bob'@'10.0.0.%' PASSWORD EXPIRE;
```

Pour viser son propre compte sans le nommer explicitement, on utilise `CURRENT_USER()` (ou le mot-clé `CURRENT_USER`), qui désigne le compte de la session courante ; contrairement à MySQL, MariaDB n'accepte pas la fonction `USER()` à cet endroit :

```sql
ALTER USER CURRENT_USER() IDENTIFIED BY 'mon_nouveau_mot_de_passe';
```

Comme toute commande `ALTER USER`, cette forme requiert le privilège `CREATE USER` (il n'existe pas d'exception pour son propre compte). Pour qu'un utilisateur change **son propre** mot de passe sans privilège particulier, on emploie plutôt `SET PASSWORD` (ci-dessous).

---

## Changer un mot de passe : `SET PASSWORD`

`SET PASSWORD` offre une alternative ciblée pour la seule modification du mot de passe :

```sql
SET PASSWORD FOR 'alice'@'localhost' = PASSWORD('nouveau_mot_de_passe');
SET PASSWORD = PASSWORD('mon_nouveau_mot_de_passe');  -- pour le compte courant
```

La forme **sans `FOR`** — pour le compte courant — ne demande **aucun privilège particulier** : c'est le moyen pour tout utilisateur de changer son propre mot de passe. La forme **`FOR …`**, qui vise un autre compte, requiert le privilège `UPDATE` sur la base `mysql`.

Pour les comptes modernes, `ALTER USER ... IDENTIFIED BY` est généralement préféré, car il couvre aussi le changement de plugin et les autres options dans une syntaxe unifiée.

---

## Renommer un compte : `RENAME USER`

`RENAME USER` change le nom ou l'hôte d'un compte. Point important : **les privilèges suivent le compte**, ils ne sont pas perdus lors du renommage.

```sql
-- Déplacer le compte d'un hôte local vers une adresse précise
RENAME USER 'alice'@'localhost' TO 'alice'@'10.0.0.5';
```

---

## Supprimer un compte : `DROP USER`

`DROP USER` supprime un ou plusieurs comptes et retire l'ensemble de leurs privilèges de toutes les tables de droits. La clause `IF EXISTS` évite l'erreur si le compte n'existe pas.

```sql
DROP USER 'alice'@'localhost';
DROP USER IF EXISTS 'bob'@'10.0.0.%', 'carol'@'%';
```

La suppression efface définitivement le compte et ses droits. En revanche, le comportement de `DROP USER` vis-à-vis des **sessions encore ouvertes** au moment de la suppression — ainsi que son cas particulier en mode Oracle — fait l'objet de la section suivante (10.2.1).

---

## Inspecter les comptes

Pour retrouver la définition d'un compte existant, `SHOW CREATE USER` reconstitue l'instruction de création (sans révéler le mot de passe en clair) :

```sql
SHOW CREATE USER 'alice'@'localhost';
```

On peut aussi lister l'ensemble des comptes en interrogeant la vue `mysql.user` :

```sql
SELECT User, Host FROM mysql.user;
```

L'examen des privilèges effectivement accordés à un compte se fait avec `SHOW GRANTS`, présenté avec le système de privilèges à la section 10.3.

---

## Bonnes pratiques

Quelques principes simples rendent la gestion des comptes plus sûre et plus lisible :

- **Préciser systématiquement l'hôte** plutôt que de laisser MariaDB compléter par `'%'`.
- **Distinguer les comptes selon les usages** (lecture seule, écriture, administration) et selon les environnements, en appliquant le principe du moindre privilège vu en 10.1.
- **Verrouiller plutôt que supprimer** lorsqu'on souhaite désactiver temporairement un accès : `ACCOUNT LOCK` préserve la configuration et les droits.
- **Privilégier les méthodes d'authentification modernes** (ed25519, et les plugins vus en 10.5/10.6) pour les nouveaux comptes.
- **Imposer une politique de mot de passe** cohérente via `PASSWORD EXPIRE` et `default_password_lifetime` lorsque le contexte l'exige.

---

## À retenir

La gestion des comptes repose sur cinq instructions : `CREATE USER` (création, avec `IF NOT EXISTS` ou `CREATE OR REPLACE`), `ALTER USER` (modification, avec `IF EXISTS`), `RENAME USER` (renommage, les privilèges suivent), `SET PASSWORD` (changement de mot de passe) et `DROP USER` (suppression définitive). Les options `REQUIRE`, `WITH`, `PASSWORD EXPIRE` et `ACCOUNT LOCK` encadrent l'usage du compte. Ces instructions actualisant automatiquement les tables et la mémoire, elles dispensent du `FLUSH PRIVILEGES`.

---

> 🔔 **Note de version.** Les instructions décrites ici sont stables dans **MariaDB 12.3 LTS** : `ALTER USER` et `SHOW CREATE USER` existent depuis la 10.2, `IF [NOT] EXISTS` depuis la 10.1, le verrouillage de compte (`ACCOUNT LOCK`) et l'expiration des mots de passe depuis la 10.4, et la prise en charge de plusieurs méthodes d'authentification par compte (`OR`) depuis la 10.4. La nouveauté 12.x relative à `DROP USER` (gestion des sessions actives) est traitée en 10.2.1.

⏭️ [DROP USER : avertissement si sessions actives (échec en mode Oracle)](/10-securite-gestion-utilisateurs/02.1-drop-user-sessions-actives.md)
