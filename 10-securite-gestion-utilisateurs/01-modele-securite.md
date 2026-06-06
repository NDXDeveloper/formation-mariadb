🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1 — Modèle de sécurité MariaDB

Avant de créer le moindre compte ou d'attribuer le moindre privilège, il est essentiel de comprendre comment MariaDB raisonne sur la sécurité. Le modèle repose sur quelques principes simples mais structurants : une protection organisée en couches, un contrôle d'accès en deux phases, une identité de compte fondée sur le couple utilisateur/hôte, et un stockage des droits dans des tables système dédiées. Cette section pose ce cadre conceptuel ; les mécanismes qu'elle évoque (privilèges, rôles, plugins d'authentification, chiffrement) sont approfondis dans les sections suivantes du chapitre.

---

## Une protection en couches : la défense en profondeur

MariaDB n'oppose pas une barrière unique à un éventuel attaquant, mais plusieurs lignes de défense complémentaires. Si l'une cède, les autres restent en place. On distingue classiquement cinq couches :

- **Réseau** : chiffrement des connexions par TLS/SSL, restriction des interfaces d'écoute (`bind-address`), pare-feu en amont.
- **Authentification** : vérification de l'identité du client lors de la connexion, via un plugin d'authentification.
- **Autorisation** : contrôle, pour chaque opération, des privilèges du compte authentifié, éventuellement organisés en rôles.
- **Données** : chiffrement des données au repos (*encryption at rest*) pour protéger les fichiers du serveur.
- **Audit** : journalisation des connexions et des opérations sensibles, à des fins de traçabilité et de conformité.

Aucune de ces couches n'est suffisante isolément. Un mot de passe robuste ne protège pas une connexion transmise en clair sur le réseau ; un chiffrement TLS irréprochable ne compense pas un compte applicatif doté de tous les privilèges. La sécurité résulte de leur combinaison.

---

## Le contrôle d'accès en deux phases

C'est le cœur du modèle. Lorsqu'un client interagit avec le serveur, MariaDB applique deux vérifications distinctes, qu'il faut toujours garder séparées à l'esprit.

**Phase 1 — Vérification de la connexion (authentification).** À l'ouverture de la connexion, le serveur détermine si le client est autorisé à se connecter. La décision dépend de trois éléments : le nom d'utilisateur, l'**hôte** depuis lequel le client se connecte, et les informations d'identification fournies (mot de passe ou autre, selon le plugin d'authentification). Si aucune correspondance n'existe pour ce couple utilisateur/hôte, ou si les informations d'identification sont invalides, la connexion est refusée — sans même considérer ce que le compte pourrait faire ensuite.

**Phase 2 — Vérification des requêtes (autorisation).** Une fois connecté, le client peut émettre des instructions. Pour chacune, le serveur vérifie que le compte dispose des privilèges suffisants sur l'objet visé (base, table, colonne, routine). Une opération non autorisée est rejetée, même si la connexion, elle, était parfaitement légitime.

Cette séparation explique la plupart des situations de diagnostic : un « *access denied* » à la connexion relève de la phase 1 (compte, hôte ou mot de passe), tandis qu'un refus survenant sur une instruction précise relève de la phase 2 (privilèges manquants).

---

## L'identité d'un compte : `utilisateur@hôte`

Dans MariaDB, un compte n'est pas seulement un nom d'utilisateur : il est identifié par le couple **`'utilisateur'@'hôte'`**. Deux comptes portant le même nom mais associés à des hôtes différents sont donc deux entités distinctes, avec potentiellement des mots de passe et des privilèges différents :

```sql
-- Deux comptes distincts, malgré un nom d'utilisateur identique
CREATE USER 'app'@'localhost' IDENTIFIED BY '...';
CREATE USER 'app'@'10.0.0.%'  IDENTIFIED BY '...';
```

La partie hôte accepte des caractères génériques : `%` correspond à n'importe quelle chaîne (y compris vide), `_` à un seul caractère, et une notation de masque réseau est également possible. Ce mécanisme permet un contrôle fin de l'origine des connexions — par exemple, accorder à un compte des droits étendus en local mais restreints depuis un sous-réseau distant.

Lorsque plusieurs comptes pourraient correspondre à une même connexion, MariaDB retient le plus **spécifique** : un hôte littéral l'emporte sur un motif générique. La distinction entre l'identité revendiquée par le client et l'identité finalement retenue par le serveur se reflète dans deux fonctions utiles :

```sql
SELECT USER();          -- ce que le client a fourni à la connexion
SELECT CURRENT_USER();  -- le compte des tables de droits réellement utilisé
```

Ces deux valeurs diffèrent dès qu'une connexion a été acceptée via un motif générique ou un compte anonyme.

---

## Où sont stockés les droits : les tables système

Les informations de sécurité — comptes, mots de passe, plugins d'authentification et privilèges — résident dans la base de données système `mysql`. Les principales tables de droits (*grant tables*) sont :

- **`mysql.global_priv`** : privilèges globaux et informations d'authentification de chaque compte.
- **`mysql.db`** : privilèges au niveau d'une base de données.
- **`mysql.tables_priv`** : privilèges au niveau d'une table.
- **`mysql.columns_priv`** : privilèges au niveau d'une colonne.
- **`mysql.procs_priv`** : privilèges sur les procédures et fonctions stockées.
- **`mysql.roles_mapping`** : associations entre comptes et rôles.

Un point propre à MariaDB mérite attention : depuis la version 10.4, la table de référence pour les comptes et les privilèges globaux est **`mysql.global_priv`**, dont les droits et les données d'authentification sont stockés dans une colonne au format JSON. La table historique **`mysql.user`** est désormais une **vue** de compatibilité construite sur `mysql.global_priv`. On peut toujours l'interroger comme avant :

```sql
SELECT User, Host FROM mysql.user;
```

Au démarrage, le serveur charge ces tables en mémoire et fonde ses vérifications de privilèges sur ces copies, pour des raisons de performance. Les instructions d'administration des comptes (`GRANT`, `REVOKE`, `CREATE USER`, `SET PASSWORD`, etc.) mettent à jour **automatiquement** les tables *et* leurs copies mémoire. En revanche, une modification directe des tables par `INSERT`/`UPDATE`/`DELETE` n'est pas prise en compte tant qu'on n'a pas exécuté :

```sql
FLUSH PRIVILEGES;
```

En pratique, on évite de manipuler directement les tables de droits : les instructions dédiées (vues aux sections 10.2 à 10.4) sont plus sûres et n'exigent pas de rechargement manuel.

---

## Les niveaux de privilèges

L'autorisation s'organise selon une hiérarchie de portées, du plus large au plus précis : **global** (`*.*`), **base de données** (`base.*`), **table** (`base.table`), **colonne**, et **routine**. Lorsqu'il évalue une opération, le serveur additionne les privilèges applicables à chaque niveau : un droit accordé globalement vaut pour toutes les bases, un droit accordé sur une base vaut pour toutes ses tables, et ainsi de suite. Le détail des privilèges disponibles et de leur attribution fait l'objet de la section 10.3.

---

## L'authentification par plugins

MariaDB n'impose pas une méthode d'authentification unique : chaque compte est associé à un **plugin d'authentification**. Cette conception ouverte permet de combiner mots de passe classiques, cryptographie moderne et intégration à l'annuaire d'entreprise au sein d'une même instance. Sur de nombreuses distributions Linux, le compte `root@localhost` utilise par défaut le plugin `unix_socket` : l'authentification repose alors sur l'identité de l'utilisateur système (via `sudo`) plutôt que sur un mot de passe stocké. Le panorama complet des plugins — `mysql_native_password`, `ed25519`, PAM/LDAP, GSSAPI/Kerberos, PARSEC, `caching_sha2_password` — est présenté aux sections 10.5 et 10.6.

---

## Le principe du moindre privilège

Au-delà des mécanismes, une règle de conception domine le modèle : n'accorder à chaque compte que les privilèges strictement nécessaires à son usage. Un compte applicatif n'a presque jamais besoin de `ALL PRIVILEGES` ni de droits d'administration ; il lui suffit le plus souvent de `SELECT`, `INSERT`, `UPDATE` et `DELETE` sur les tables de son périmètre. Réserver les comptes à fort privilège aux opérations d'administration, et distinguer les comptes selon les applications et les environnements, limite considérablement l'impact d'une compromission ou d'une erreur.

---

## Sécuriser une installation neuve

Une instance fraîchement installée n'est pas sécurisée par défaut. La première opération consiste à exécuter l'utilitaire fourni :

```bash
sudo mariadb-secure-installation
```

Il guide l'administrateur dans les actions de durcissement essentielles : définir (ou verrouiller) le mot de passe du compte d'administration, supprimer les **comptes anonymes** (`''@'localhost'`), interdire la connexion distante du compte `root`, et retirer la base de test `test` accessible à tous. Ces quelques étapes éliminent les vecteurs d'attaque les plus courants avant toute mise en service.

---

## À retenir

Le modèle de sécurité de MariaDB se résume à quelques idées clés : une protection **en couches**, un contrôle d'accès en **deux phases** (authentification puis autorisation), une identité de compte fondée sur le couple **`utilisateur@hôte`**, des droits stockés dans des **tables système** (avec `mysql.global_priv` comme table de référence depuis la 10.4) et chargés en mémoire, et une **hiérarchie de privilèges** que l'on applique selon le principe du **moindre privilège**. Ce cadre étant posé, les sections suivantes détaillent successivement la gestion des comptes, l'attribution des privilèges, les rôles, l'authentification et le chiffrement.

---

> 🔔 **Note de version.** Cette section décrit le modèle de sécurité de **MariaDB 12.3 LTS**, stable dans ses grands principes depuis plusieurs versions. La structure `mysql.global_priv` / vue `mysql.user` est en place depuis la 10.4. Les nouveautés propres à la série 12.x concernent des mécanismes spécifiques (gestion des sessions sur `DROP USER`, `caching_sha2_password`, clés SSL avec passphrase, audit bufferisé, `SET SESSION AUTHORIZATION`) abordés dans les sections dédiées de ce chapitre.

⏭️ [Création et gestion des utilisateurs (CREATE USER, ALTER USER, DROP USER)](/10-securite-gestion-utilisateurs/02-creation-gestion-utilisateurs.md)
