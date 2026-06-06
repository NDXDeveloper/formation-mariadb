🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.9 — Sécurité au niveau application

Les sections précédentes ont décrit la sécurité telle que l'administrateur la configure sur le serveur : modèle de sécurité (10.1), comptes (10.2), privilèges (10.3), rôles (10.4), authentification (10.5–10.6), chiffrement du transport (10.7) et audit (10.8). Mais une part importante des incidents ne provient pas du moteur lui-même : elle naît de la **manière dont l'application se connecte à la base et l'utilise**. Cette section adopte le point de vue de la frontière application–base de données et rassemble, sous l'angle de la sécurité applicative, des mécanismes vus ailleurs dans la formation.

Le principe directeur est celui de la **défense en profondeur** : la base de données doit être considérée comme la dernière ligne de défense et partir du postulat que l'application peut être compromise. L'objectif n'est pas seulement d'empêcher l'intrusion, mais de **limiter le rayon d'impact** si une couche cède — credentials divulgués, faille applicative, injection SQL réussie.

## Le principe du moindre privilège pour les comptes applicatifs

La première règle est qu'une application ne se connecte **jamais** avec `root` ni avec un compte d'administration disposant de privilèges étendus. Chaque application — idéalement chaque service — dispose d'un compte dédié, restreint exactement aux bases, tables et opérations dont elle a besoin.

Il est sain de séparer les comptes par fonction : un compte transactionnel en lecture/écriture pour le fonctionnement courant, un compte de reporting en lecture seule, et un compte de migration disposant des droits DDL mais utilisé uniquement au moment des déploiements. Ainsi, une fuite du compte de reporting ne permet pas de modifier les données, et le compte applicatif courant ne peut pas altérer le schéma.

```sql
-- Compte applicatif transactionnel, restreint à sa base et à son sous-réseau
CREATE USER 'app_boutique'@'10.0.1.%' IDENTIFIED VIA ed25519 USING PASSWORD('secret_fort');
GRANT SELECT, INSERT, UPDATE, DELETE ON boutique.* TO 'app_boutique'@'10.0.1.%';

-- Compte de reporting en lecture seule
CREATE USER 'app_reporting'@'10.0.1.%' IDENTIFIED BY 'autre_secret';
GRANT SELECT ON boutique.* TO 'app_reporting'@'10.0.1.%';
```

À l'inverse, un compte applicatif ne doit jamais détenir de privilèges qui ouvriraient la voie à une escalade : `GRANT OPTION` (qui permettrait de s'octroyer d'autres droits), `SUPER`, `FILE`, `PROCESS`, `SHUTDOWN`, `CREATE USER`. Le privilège `FILE` mérite une attention particulière : il autorise la lecture et l'écriture de fichiers sur le serveur via `LOAD DATA INFILE` et `SELECT … INTO OUTFILE`, ce qui en fait un vecteur d'exfiltration ou de dépôt de fichiers ; lorsqu'il est nécessaire, on le confine avec la variable `secure_file_priv`, qui restreint ces opérations à un répertoire désigné. Lorsque le nombre de comptes et de droits devient important, les **rôles** (10.4) permettent de factoriser ces ensembles de privilèges et d'en garantir la cohérence.

## Restreindre l'origine et les ressources des connexions

Le compte MariaDB est identifié par le couple `utilisateur@hôte`. Lier un compte applicatif à l'hôte ou au sous-réseau précis depuis lequel il se connecte (`'app_boutique'@'10.0.1.%'`) plutôt qu'à `'app'@'%'` réduit fortement la surface d'attaque : des identifiants divulgués ne peuvent alors pas être réutilisés depuis une machine arbitraire. Ce contrôle applicatif se complète par un durcissement réseau côté serveur — `bind-address` pour limiter les interfaces d'écoute, règles de pare-feu et non-exposition publique du port 3306, voire `skip-networking` pour un usage strictement local.

Il est par ailleurs prudent de **plafonner les ressources** consommées par un compte, afin de contenir une application défaillante ou compromise (fuite de connexions, volume de requêtes anormal) et d'éviter qu'un seul service ne monopolise le serveur. Ces limites se posent directement sur le compte :

```sql
ALTER USER 'app_boutique'@'10.0.1.%'
  WITH MAX_USER_CONNECTIONS 50
       MAX_QUERIES_PER_HOUR 200000;
```

Les clauses disponibles incluent `MAX_USER_CONNECTIONS` (connexions simultanées), `MAX_CONNECTIONS_PER_HOUR`, `MAX_QUERIES_PER_HOUR` et `MAX_UPDATES_PER_HOUR`. Elles s'articulent avec les limites globales du serveur (`max_connections`, `max_user_connections`), qui fixent les plafonds d'ensemble.

## Encapsuler l'accès aux données : vues et procédures stockées

Plutôt que d'accorder à l'application un accès direct aux tables, on peut l'obliger à passer par une **interface contrôlée**. Deux constructions, étudiées par ailleurs dans la formation, servent cet objectif.

Les **vues** (chapitre 9) exposent un sous-ensemble de colonnes ou de lignes : on masque ainsi les colonnes sensibles et on filtre les enregistrements visibles, le compte ne recevant des privilèges que sur la vue, pas sur la table sous-jacente — c'est le mécanisme de masquage de données abordé en 9.5. Les **procédures stockées** (chapitre 8) vont plus loin : l'application reçoit uniquement le privilège `EXECUTE` sur des procédures, et ne peut donc exécuter aucune instruction DML arbitraire sur les tables de base.

```sql
-- L'application n'a aucun droit direct sur les tables : seulement une interface métier
GRANT EXECUTE ON PROCEDURE boutique.passer_commande TO 'app_boutique'@'10.0.1.%';
```

Cette encapsulation a deux vertus : elle impose les règles métier côté serveur et elle réduit l'impact d'une injection SQL, puisque la requête détournée ne peut pas atteindre directement les tables. Un point de vigilance accompagne cette approche : la clause `SQL SECURITY` des routines et des vues. Par défaut (`DEFINER`), une procédure s'exécute avec les privilèges de son créateur, ce qui permet à un compte applicatif peu privilégié de déclencher une opération privilégiée encadrée — c'est puissant, mais cela fait du définisseur une cible : il doit lui-même être le moins privilégié possible. À l'inverse, `SQL SECURITY INVOKER` exécute la routine avec les privilèges de l'appelant.

## Protéger les données sensibles

Il faut distinguer deux niveaux de chiffrement. Le **chiffrement au repos** (18.7), transparent, protège les fichiers de données et les sauvegardes contre un accès au support de stockage, mais reste invisible pour quiconque dispose d'une connexion légitime au serveur. Le **chiffrement applicatif** consiste, lui, à chiffrer certains champs sensibles dans l'application avant de les stocker : il protège alors même contre une compromission de la base ou un accès administrateur, au prix de la perte d'indexation et de recherche directe sur ces champs.

Pour les secrets que l'application gère elle-même — par exemple les mots de passe de ses propres utilisateurs finaux, à ne pas confondre avec l'authentification des comptes MariaDB —, la règle est d'appliquer un **hachage fort et salé** (bcrypt, Argon2) côté application : jamais de stockage en clair ni de chiffrement réversible.

Enfin, les données sensibles peuvent fuir par les **journaux**. Le journal d'audit (10.8.2) consigne le texte des requêtes, et le slow query log peut capturer des clauses `WHERE` contenant des données personnelles ; le masquage des mots de passe par le plugin d'audit ne couvre pas ces cas. Il faut donc protéger ces fichiers (permissions restreintes) et réfléchir à ce qui s'y retrouve, en cohérence avec les recommandations de la section 10.8.

## Gérer les secrets de connexion

Les identifiants de connexion ne doivent **jamais** être codés en dur dans le code source ni versionnés dans un dépôt. On les externalise : variables d'environnement, fichiers de configuration à permissions restreintes, ou — préférable — gestionnaire de secrets dédié (HashiCorp Vault, gestionnaires de secrets cloud, *Secrets* Kubernetes), avec une rotation régulière. Cette discipline rejoint celle décrite pour la gestion des clés de chiffrement en 18.7.2.

Côté MariaDB, plusieurs réflexes s'imposent. Les clients lisent volontiers les identifiants depuis des **fichiers d'options** (`~/.my.cnf`, `--defaults-file` de `mariadb-dump`, etc.) : ces fichiers doivent être strictement protégés.

```bash
# Un fichier d'options contenant un mot de passe doit être illisible par les autres
chmod 600 ~/.my.cnf
```

Il faut par ailleurs **éviter de passer un mot de passe sur la ligne de commande**, où il apparaîtrait dans la liste des processus et l'historique du shell. Plus structurellement, le recours à des plugins d'authentification limitant les mots de passe statiques partagés — `ed25519` (10.5.2), PAM/LDAP centralisé (10.5.3) ou GSSAPI/Kerberos (10.5.4) — réduit la prolifération de secrets à gérer.

## Sécuriser le transport depuis l'application

Le chiffrement du transport (10.7) doit être imposé pour les connexions applicatives, et pas seulement rendu possible. La clause `REQUIRE` sur le compte contraint le client à se connecter en TLS :

```sql
-- Exiger une connexion chiffrée
ALTER USER 'app_boutique'@'10.0.1.%' REQUIRE SSL;

-- Variante plus stricte : exiger un certificat client valide
ALTER USER 'app_boutique'@'10.0.1.%' REQUIRE X509;
```

La chaîne de connexion de l'application doit activer TLS et, idéalement, vérifier le certificat du serveur pour se prémunir d'une attaque de l'intercepteur. Le champ `tls_version` ajouté à l'audit des connexions (10.8.3) permet ensuite de **vérifier** que cette politique est effectivement respectée.

## Se prémunir contre l'injection SQL (rappel)

L'injection SQL reste la vulnérabilité applicative la plus répandue côté base de données. Sa parade définitive est l'usage de **requêtes paramétrées et de prepared statements**, traités en détail dans la partie développement (17.8 et 17.9). Du point de vue de l'architecture de sécurité, deux principes vus ci-dessus en limitent par ailleurs la portée : appliquer le moindre privilège, pour qu'une injection réussie atteigne le moins d'objets possible, et encapsuler l'accès derrière des procédures et des vues, pour qu'une requête détournée ne touche pas directement les tables. La validation des entrées côté application complète ce dispositif.

## Comptes techniques et attribution des connexions

Les **pools de connexions** partagent un même compte de base entre de nombreux utilisateurs finaux. La base ne voit alors qu'une seule identité, ce qui fait perdre l'attribution par utilisateur et complique la traçabilité et l'application de privilèges différenciés. La série 12.x introduit `SET SESSION AUTHORIZATION` (10.12), qui permet à une connexion mutualisée d'endosser l'identité d'un autre utilisateur le temps d'une session ou d'une instruction, sans réauthentification — restaurant ainsi l'attribution et le contrôle des privilèges au niveau de l'utilisateur réel. C'est un compromis classique entre l'efficacité du pooling et la granularité de l'audit et des droits.

## Synthèse

La sécurité au niveau application consiste à traiter la base comme la dernière ligne de défense et à durcir tout ce qui l'entoure. Concrètement : connecter chaque application avec un compte au **moindre privilège**, **lié à des hôtes connus** et **plafonné en ressources** ; passer par une **interface contrôlée** (vues, procédures) plutôt que par un accès direct aux tables ; imposer le **transport chiffré** ; **externaliser et protéger les secrets** de connexion ; **valider les entrées** et recourir aux **requêtes paramétrées** ; et préserver l'**attribution** des actions, y compris derrière un pool de connexions. Chacun de ces leviers renvoie à une fonctionnalité détaillée ailleurs dans la formation, mais c'est leur combinaison, à la frontière entre l'application et le serveur, qui constitue la sécurité applicative.

⏭️ [Password validation plugins et politiques](/10-securite-gestion-utilisateurs/10-password-validation.md)
