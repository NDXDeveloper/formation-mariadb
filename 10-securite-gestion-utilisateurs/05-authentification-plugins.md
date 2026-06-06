🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5 — Authentification : Plugins

L'authentification est la **phase 1** du contrôle d'accès vue en 10.1 : vérifier l'identité d'un client au moment où il se connecte. MariaDB ne fige pas une méthode unique pour cela ; il s'appuie sur une **architecture enfichable** (*pluggable authentication*) où chaque compte est associé à un **plugin d'authentification** qui détermine *comment* le client prouve son identité — par un mot de passe haché, un défi cryptographique, l'identité du système d'exploitation, ou un service externe comme un annuaire. Cette section présente le principe et le panorama des plugins ; chacun est ensuite détaillé dans les sous-sections 10.5.1 à 10.5.5 (et 10.6 pour PARSEC).

---

## Le principe de l'authentification enfichable

Plutôt que de coder en dur une seule façon de s'authentifier, MariaDB délègue cette vérification à des **plugins serveur**. Cela permet de faire cohabiter, sur une même instance, des comptes authentifiés par mot de passe classique, par cryptographie moderne ou via un système d'entreprise.

Deux catégories de plugins coexistent :

- les plugins **intégrés statiquement** au serveur, toujours disponibles (c'est le cas de `mysql_native_password`) ;
- les plugins **chargeables dynamiquement**, distribués avec MariaDB mais non installés par défaut (c'est le cas d'`ed25519` ou de `caching_sha2_password`), qu'il faut activer explicitement.

Certaines méthodes nécessitent en outre un **plugin côté client** compatible : le client calcule et transmet sa preuve d'identité selon le protocole attendu par le plugin serveur. C'est un point d'attention pour la compatibilité des pilotes et connecteurs.

---

## Comment un compte est associé à un plugin

Le plugin se choisit dès la création du compte, avec la clause `IDENTIFIED VIA` déjà rencontrée en 10.2, et se modifie ensuite avec `ALTER USER` :

```sql
-- À la création
CREATE USER 'carol'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('mot_de_passe');

-- En modification
ALTER USER 'bob'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('mot_de_passe');

-- Plusieurs méthodes acceptées (la connexion réussit si l'une convient)
CREATE USER 'svc'@'localhost'
  IDENTIFIED VIA ed25519 USING PASSWORD('mot_de_passe')
  OR unix_socket;
```

À l'inverse, `IDENTIFIED BY 'mot_de_passe'` (sans `VIA`) emploie le **plugin par défaut** du compte :

```sql
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'mot_de_passe';
```

Le plugin associé à chaque compte et la donnée d'authentification correspondante (empreinte, clé…) sont consignés dans `mysql.user` — concrètement, dans `mysql.global_priv` depuis la 10.4 — via les colonnes `plugin` et `authentication_string`.

---

## Installer et lister les plugins

Un plugin chargeable s'active avec `INSTALL SONAME` (ou `INSTALL PLUGIN`), et se désactive avec `UNINSTALL SONAME` / `UNINSTALL PLUGIN`. On peut aussi le charger au démarrage via les options `--plugin-load` ou `--plugin-load-add` dans un fichier de configuration.

```sql
INSTALL SONAME 'auth_ed25519';     -- active le plugin ed25519
UNINSTALL SONAME 'auth_ed25519';   -- le désactive
```

Pour savoir quels plugins sont chargés et de quel type ils sont :

```sql
SHOW PLUGINS;                                         -- tous les plugins chargés
SELECT PLUGIN_NAME, PLUGIN_STATUS
  FROM information_schema.PLUGINS
  WHERE PLUGIN_TYPE = 'AUTHENTICATION';               -- seulement l'authentification

SELECT User, Host, plugin FROM mysql.user;            -- plugin associé à chaque compte
```

---

## Le plugin par défaut

Lorsqu'un compte est créé sans préciser de plugin, MariaDB utilise **`mysql_native_password`** : c'est le plugin par défaut, fondé sur un hachage SHA-1, doté de la plus large compatibilité avec les clients et pilotes existants. Sur la plupart des installations Linux, le compte `root@localhost` fait toutefois exception et utilise par défaut `unix_socket`, qui authentifie d'après l'identité du système d'exploitation (via `sudo`) plutôt qu'avec un mot de passe stocké.

Un point important pour la sécurité comme pour la migration : `mysql_native_password` reste le défaut **de MariaDB**, alors que **MySQL** l'a déprécié (depuis la 8.0.34) au profit de `caching_sha2_password`. MariaDB le conserve pour la compatibilité, mais en **déconseille l'usage pour les nouvelles installations** exigeant une forte sécurité — au profit d'`ed25519` ou de PARSEC.

---

## Panorama des plugins

| Plugin | Usage | Détaillé en |
|---|---|---|
| **`mysql_native_password`** | Mot de passe conventionnel (SHA-1), **plugin par défaut**, large compatibilité client | 10.5.1 |
| **`ed25519`** | Mot de passe à sécurité renforcée (courbe elliptique, comme OpenSSH) | 10.5.2 |
| **PAM / LDAP** | Délégation de l'authentification au système ou à un annuaire | 10.5.3 |
| **GSSAPI / Kerberos** | Authentification unique (SSO) en environnement d'entreprise | 10.5.4 |
| **`caching_sha2_password`** 🆕 | Compatibilité avec l'authentification de MySQL 8 (aide à la migration) | 10.5.5 |
| **PARSEC** | Authentification moderne (ed25519 salé), futur plugin par défaut | 10.6 |
| **`unix_socket`** | Identité du système d'exploitation (souvent `root@localhost`) | — |

À ces plugins s'ajoute `mysql_old_password`, vestige de l'authentification antérieure à MySQL 4.1, conservé pour compatibilité mais à proscrire pour toute nouvelle utilisation.

On peut résumer les grandes familles :

- **Mot de passe** (`mysql_native_password`, `ed25519`, PARSEC, `caching_sha2_password`) — le client présente un mot de passe, vérifié selon un algorithme plus ou moins robuste.
- **Identité externe** (PAM, LDAP, GSSAPI/Kerberos, `unix_socket`) — la vérification est déléguée au système d'exploitation, à un annuaire ou à un service d'authentification.

---

## Choisir un plugin

Le choix dépend des contraintes de sécurité, de compatibilité et d'intégration :

- pour de **nouveaux comptes à forte sécurité**, privilégier `ed25519` ou PARSEC, qui remplacent avantageusement le SHA-1 vieillissant ;
- pour une **large compatibilité** avec d'anciens clients ou pilotes, `mysql_native_password` reste le choix le plus universel (au prix d'une sécurité moindre) ;
- pour une **intégration au SI** (comptes système, annuaire, SSO), s'orienter vers PAM/LDAP ou GSSAPI/Kerberos ;
- pour une **migration depuis MySQL 8** sans changer les mots de passe, recourir à `caching_sha2_password`, en le considérant comme une aide transitoire plutôt qu'une cible définitive.

Quelle que soit la méthode, on veillera à ce que les **connecteurs côté application** prennent en charge le plugin retenu : un pilote trop ancien ne sait pas dialoguer avec `ed25519`, par exemple.

---

## À retenir

L'authentification de MariaDB repose sur une **architecture enfichable** : chaque compte est rattaché à un plugin qui détermine comment vérifier son identité. Le plugin se choisit avec `IDENTIFIED VIA` (et plusieurs méthodes peuvent coexister via `OR`), tandis que `IDENTIFIED BY` emploie le plugin par défaut, **`mysql_native_password`** (SHA-1, conservé pour compatibilité mais déconseillé en haute sécurité). Les plugins chargeables s'activent avec `INSTALL SONAME` ; `SHOW PLUGINS` et `information_schema.PLUGINS` permettent de les lister. Le catalogue va du mot de passe conventionnel à la cryptographie moderne (`ed25519`, PARSEC), en passant par l'identité externe (PAM/LDAP, GSSAPI) et la compatibilité MySQL (`caching_sha2_password`). Les sous-sections suivantes détaillent chacun de ces plugins.

---

> 🔔 **Note de version.** Dans **MariaDB 12.3 LTS**, le plugin par défaut reste `mysql_native_password` ; `ed25519` est disponible depuis la 10.1.22 / 10.2.5, PARSEC depuis la 11.6 (destiné à devenir le futur plugin par défaut), et `caching_sha2_password` depuis la 12.1 (aide à la migration depuis MySQL, traité en 10.5.5). Sources : *MariaDB Server Documentation — Pluggable Authentication Overview* et fiches des plugins associés.

⏭️ [mysql_native_password](/10-securite-gestion-utilisateurs/05.1-mysql-native-password.md)
