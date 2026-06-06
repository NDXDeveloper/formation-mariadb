🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.10 — Password validation plugins et politiques

Lorsque l'authentification repose sur des mots de passe (10.5), la sécurité du serveur ne vaut que ce que valent ces mots de passe. MariaDB agit sur deux axes complémentaires : imposer la **robustesse** des mots de passe au moment où ils sont définis, au moyen de plugins de validation, et encadrer leur **cycle de vie** dans le temps — expiration, non-réutilisation, verrouillage des comptes. Cette section couvre ces deux volets, ainsi qu'un point de configuration décisif sans lequel toute la politique peut être contournée.

## Les plugins de validation de mots de passe

MariaDB est livré avec trois plugins de validation : `simple_password_check`, `cracklib_password_check` et `password_reuse_check`. Aucun n'est activé par défaut ; on les charge avec `INSTALL SONAME` (ou `INSTALL PLUGIN`). Dès qu'au moins un plugin de validation est chargé, **tout nouveau mot de passe est vérifié** et l'instruction qui le définit échoue s'il ne satisfait pas les contrôles. Plusieurs plugins peuvent être chargés simultanément : le mot de passe doit alors passer l'ensemble des vérifications de tous les plugins.

```sql
-- Charger un plugin de validation (non activé par défaut)
INSTALL SONAME 'simple_password_check';

-- Lister les plugins de validation chargés
SELECT plugin_name, plugin_status
FROM information_schema.all_plugins
WHERE plugin_type = 'PASSWORD VALIDATION';
```

Un point important : la validation n'intervient qu'au **moment où un mot de passe est défini ou modifié** — `CREATE USER`, `ALTER USER`, `SET PASSWORD`, ou `GRANT … IDENTIFIED BY`. Les mots de passe déjà en place ne sont pas vérifiés rétroactivement. Lorsqu'un mot de passe est refusé, le serveur renvoie l'erreur `1819 (HY000)` ; un `SHOW WARNINGS` apporte le détail de la raison, particulièrement utile avec CrackLib.

### simple_password_check

Ce plugin impose des règles de **complexité** quantitatives : longueur minimale et nombre minimal de caractères de chaque catégorie. À l'installation, il exige par défaut au moins huit caractères, dont au moins un chiffre, une majuscule, une minuscule et un caractère qui n'est ni une lettre ni un chiffre.

| Variable | Rôle | Défaut |
|----------|------|--------|
| `simple_password_check_minimal_length` | Longueur minimale | 8 |
| `simple_password_check_digits` | Nombre minimal de chiffres | 1 |
| `simple_password_check_letters_same_case` | Minimum de minuscules **et** de majuscules | 1 |
| `simple_password_check_other_characters` | Minimum de caractères « autres » | 1 |

```sql
-- Renforcer la politique de complexité
SET GLOBAL simple_password_check_minimal_length    = 12;
SET GLOBAL simple_password_check_digits            = 2;
SET GLOBAL simple_password_check_letters_same_case = 1;
SET GLOBAL simple_password_check_other_characters  = 1;

CREATE USER 'sally'@'10.0.1.%' IDENTIFIED BY 'short';
-- ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
```

### cracklib_password_check

Ce plugin s'appuie sur la bibliothèque **CrackLib** et son dictionnaire pour évaluer la qualité d'un mot de passe. Là où `simple_password_check` se contente de compter des catégories de caractères, CrackLib rejette qualitativement les mots de passe fondés sur des mots du dictionnaire, des suites simples ou dérivés du nom d'utilisateur. C'est un contrôle plus pertinent contre les mots de passe « complexes mais devinables ».

Le chemin du dictionnaire est fixé par `cracklib_password_check_dictionary` (souvent `/usr/share/cracklib/pw_dict` selon le système). Le plugin requiert la présence de la bibliothèque CrackLib, généralement fournie par un paquet distinct (par exemple `mariadb-plugin-cracklib-password-check`). Les deux plugins peuvent être combinés pour cumuler contrôle de complexité et contrôle par dictionnaire.

### password_reuse_check

Disponible depuis **MariaDB 10.7**, ce plugin empêche un utilisateur de **réutiliser un ancien mot de passe**, exigence de certaines politiques de sécurité. Les empreintes des anciens mots de passe sont conservées dans la table système `mysql.password_reuse_check_history`.

La durée de rétention est définie par `password_reuse_check_interval`, exprimée en jours. Sa valeur par défaut est `0`, ce qui signifie une **rétention illimitée** : aucun mot de passe déjà utilisé ne pourra plus jamais être repris. Une valeur positive limite l'interdiction aux mots de passe employés au cours de cette période.

```sql
INSTALL SONAME 'password_reuse_check';

-- Interdire la réutilisation d'un mot de passe utilisé dans les 365 derniers jours
SET GLOBAL password_reuse_check_interval = 365;
```

Si l'on doit exceptionnellement autoriser la réutilisation d'un mot de passe, il faut supprimer son empreinte de la table d'historique.

## Le contournement par empreinte : `strict_password_validation`

C'est le point le plus important de cette section. Un mot de passe peut être défini directement sous forme d'**empreinte précalculée** (un hash), plutôt qu'en clair. Or, défini ainsi, il **échappe à tous les plugins de validation** — à moins que la variable `strict_password_validation` ne soit active.

Par défaut, `strict_password_validation` est positionnée à `ON`. Tant qu'elle l'est et qu'au moins un plugin de validation est chargé, un mot de passe fourni sous forme d'empreinte qui ne peut être validé est rejeté ; le contournement est donc fermé. La désactiver (la passer à `OFF`) rouvre cette brèche : on pourrait alors contourner l'intégralité de la politique de complexité simplement en fournissant un hash. La recommandation est sans ambiguïté : la **laisser à `ON`**. À noter qu'elle n'a aucun effet si aucun plugin de validation n'est chargé.

## Politiques de cycle de vie : l'expiration des mots de passe

Au-delà de la robustesse à l'instant T, on peut imposer le **renouvellement** périodique des mots de passe. Deux variables système gouvernent l'expiration.

`default_password_lifetime` définit la politique globale, en jours. La valeur `0` (par défaut) désactive l'expiration automatique ; une valeur positive `N` impose le changement du mot de passe tous les `N` jours. Les comptes qui ne portent pas de réglage propre héritent de cette valeur globale.

`disconnect_on_expired_password` détermine le comportement face à un mot de passe expiré : soit la connexion est refusée, soit le client est admis en **mode restreint** (« sandbox »), limité à un sous-ensemble de requêtes destinées à réinitialiser le mot de passe, en particulier `SET PASSWORD` et `SET`.

L'expiration se règle aussi **par compte**, ce qui prime sur la politique globale, au moyen des clauses `PASSWORD EXPIRE` de `CREATE USER` et `ALTER USER`.

```sql
-- Politique globale : renouvellement tous les 90 jours
SET GLOBAL default_password_lifetime = 90;

-- Réglages par compte (priment sur la politique globale)
ALTER USER 'app_boutique'@'10.0.1.%' PASSWORD EXPIRE NEVER;        -- jamais d'expiration
ALTER USER 'analyste'@'10.0.1.%'     PASSWORD EXPIRE INTERVAL 60 DAY;
ALTER USER 'app_boutique'@'10.0.1.%' PASSWORD EXPIRE DEFAULT;      -- suit la valeur globale
ALTER USER 'temporaire'@'10.0.1.%'   PASSWORD EXPIRE;              -- expiration immédiate
```

Sur le plan de la politique, il convient de noter que la rotation périodique forcée est aujourd'hui discutée : les recommandations modernes (notamment celles du NIST) privilégient des mots de passe longs et la prévention de la réutilisation plutôt qu'un changement fréquent imposé, ce dernier poussant souvent les utilisateurs vers des variations faibles et prévisibles. L'expiration garde tout son sens pour les comptes à privilèges, en réaction à un départ ou à une suspicion de compromission, mais une expiration courte et systématique sur l'ensemble des comptes n'est plus une bonne pratique universelle.

## Verrouillage des comptes

### Verrouillage administratif (`ACCOUNT LOCK`)

MariaDB permet de **verrouiller** un compte pour en interdire la connexion tout en le conservant intact, avec ses privilèges. C'est le moyen approprié pour suspendre un accès — départ d'un collaborateur, compte suspecté compromis — sans le supprimer.

```sql
ALTER USER 'parti'@'10.0.1.%' ACCOUNT LOCK;
ALTER USER 'parti'@'10.0.1.%' ACCOUNT UNLOCK;
```

### Protection contre la force brute (`max_password_errors`)

Pour limiter les attaques par force brute, MariaDB s'appuie sur la variable système globale `max_password_errors`. Elle fixe le nombre maximal de tentatives de connexion en échec **pour cause de mot de passe invalide** au-delà duquel l'utilisateur est bloqué pour toute connexion ultérieure — **y compris avec le bon mot de passe** :

```text
ERROR 4150 (HY000): User is blocked because of too many credential errors; unblock with 'ALTER USER / FLUSH PRIVILEGES'
```

Comme l'indique le message, **deux moyens** débloquent le compte : un `FLUSH PRIVILEGES` (global) ou un `ALTER USER` sur le compte concerné. Cette limite ne s'applique pas aux utilisateurs disposant du privilège `SUPER` ou, depuis MariaDB 10.5.2, du privilège `CONNECTION ADMIN`, lorsqu'ils se connectent depuis `localhost`, `127.0.0.1` ou `::1`.

```ini
[mariadb]
max_password_errors = 5
```

Un point d'attention important, en particulier lors d'une **migration depuis MySQL** : ce mécanisme est **global au serveur**, et non paramétrable par compte. Les clauses `FAILED_LOGIN_ATTEMPTS` et `PASSWORD_LOCK_TIME` de `CREATE USER`/`ALTER USER`, qui dans MySQL 8 permettent un verrouillage temporaire par compte après un nombre configurable d'échecs, **n'existent pas dans MariaDB**. Une politique de blocage par compte ou par adresse IP devra donc être obtenue autrement — au niveau réseau ou système (par exemple via un outil comme fail2ban), ou par une couche de proxy en amont.

## Synthèse

Une politique de mots de passe robuste dans MariaDB combine plusieurs leviers. On charge au moins un plugin de validation — `simple_password_check` pour la complexité, idéalement complété par `cracklib_password_check` pour le contrôle par dictionnaire, et `password_reuse_check` si la non-réutilisation est requise — tout en **conservant `strict_password_validation` à `ON`** pour fermer le contournement par empreinte. On définit une expiration cohérente (en gardant à l'esprit que la prévention de la réutilisation et la longueur priment désormais sur la rotation systématique), on utilise `ACCOUNT LOCK` pour suspendre un accès, et `max_password_errors` pour freiner la force brute, en sachant que ce dernier est global et non par compte. Enfin, le moyen le plus sûr de réduire la surface liée aux mots de passe reste de **diminuer la dépendance aux mots de passe statiques** : les plugins d'authentification vus en 10.5 — `ed25519`, PAM/LDAP centralisé, GSSAPI/Kerberos — déplacent une partie de ces préoccupations vers des mécanismes mieux maîtrisés.

⏭️ [Privilèges granulaires (options 11.8)](/10-securite-gestion-utilisateurs/11-privileges-granulaires.md)
