🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.12 — SET SESSION AUTHORIZATION : exécuter des actions en tant qu'un autre utilisateur 🆕

## Présentation

Introduite dans **MariaDB 12.0** (MDEV-20299), l'instruction `SET SESSION AUTHORIZATION` permet à un utilisateur suffisamment privilégié d'**endosser l'identité d'un autre compte** pour la durée de la session — l'équivalent d'un « sudo » à l'intérieur de la session SQL. À partir de la bascule, les instructions s'exécutent avec les privilèges et sous l'identité de l'utilisateur cible. Il s'agit d'une instruction du standard SQL, que l'on retrouve notamment dans PostgreSQL.

Il est important de comprendre que cette instruction n'introduit pas un pouvoir réellement nouveau. Le privilège `SET USER` (voir 10.11) autorisait déjà à créer des triggers et des routines avec un `DEFINER` arbitraire, c'est-à-dire à faire exécuter du code au nom de n'importe quel utilisateur. `SET SESSION AUTHORIZATION` ne fait que rendre cette capacité **directement utilisable**, sans passer par la création d'un objet intermédiaire.

## Syntaxe et effet sur la session

La syntaxe prend une spécification de compte `utilisateur@hôte` :

```sql
SELECT USER(), CURRENT_USER(), DATABASE();
-- +--------------------+--------------------+------------+
-- | msandbox@localhost | msandbox@localhost | test       |
-- +--------------------+--------------------+------------+

SET SESSION AUTHORIZATION foo@localhost;

SELECT USER(), CURRENT_USER(), DATABASE();
-- +---------------+----------------+------------+
-- | foo@localhost | foo@%          | NULL       |
-- +---------------+----------------+------------+
```

Après la bascule, la session adopte entièrement l'identité cible. On notera trois effets dans cet exemple. D'abord, `USER()` reflète l'utilisateur demandé (`foo@localhost`). Ensuite, `CURRENT_USER()` indique le **compte réellement apparié** dans le système de privilèges (`foo@%`, dont le motif d'hôte correspond) : c'est cette identité qui sert désormais aux vérifications de droits. Enfin, `DATABASE()` repasse à `NULL` — la base de données par défaut est réinitialisée par la bascule, et il faut donc refaire un `USE` si nécessaire.

## Privilège requis et contrôles contournés

Le changement d'identité requiert le privilège **`SET USER`**. La seule cible qu'un utilisateur peut endosser **sans** ce privilège est sa propre identité authentifiée — celle que renvoie `USER()`. Toute autre cible l'exige, **y compris son propre compte** lorsque celui-ci a été apparié via un motif d'hôte plus large que l'adresse de connexion : `USER()` valant alors `utilisateur@<adresse_réelle>`, désigner le compte `utilisateur@%` est interprété comme une bascule vers un autre compte et renvoie `ERROR 1873 (28000): Access denied trying to change to user …`.

Le point déterminant pour la sécurité est que cette bascule **contourne plusieurs contrôles** : l'authentification, le verrouillage de compte (`ACCOUNT LOCK`), l'expiration du mot de passe et l'exigence `REQUIRE SSL`. Autrement dit, les identifiants, l'état de verrouillage ou les contraintes de connexion du compte cible **ne sont pas revérifiés** : la racine de confiance est le privilège `SET USER` de l'émetteur, pas les justificatifs de la cible.

Il en découle une conséquence pratique forte : `SET USER` équivaut, de fait, au **droit de devenir n'importe quel utilisateur**. Puisqu'il permettait déjà l'exécution sous un `DEFINER` arbitraire, et qu'il ouvre désormais directement `SET SESSION AUTHORIZATION`, ce privilège doit être considéré comme quasi équivalent à un super-privilège et n'être accordé qu'avec une extrême parcimonie, en cohérence avec l'approche granulaire de la section 10.11.

## Restrictions d'emploi

`SET SESSION AUTHORIZATION` **ne fonctionne pas à l'intérieur d'une transaction, d'un prepared statement ni d'une procédure stockée**. La restriction sur les transactions découle du standard SQL ; celle sur les prepared statements et les routines tient au fait que changer l'identité courante en cours d'exécution n'aurait pas de sémantique sûre. L'instruction doit donc être émise de façon autonome, en dehors de toute transaction.

## Revenir à l'identité d'origine

La page de référence MariaDB documente la bascule mais **pas de forme dédiée de retour**. Dans le standard SQL et dans des moteurs comme PostgreSQL, on revient à l'utilisateur initialement authentifié avec `SET SESSION AUTHORIZATION DEFAULT` ou `RESET SESSION AUTHORIZATION`, formes exécutables par n'importe quel utilisateur. Sur MariaDB, il faut garder à l'esprit que, du point de vue de l'identité endossée, revenir à l'utilisateur d'origine revient à basculer vers « un autre utilisateur », ce qui requiert à nouveau `SET USER` — privilège que l'identité endossée ne possède pas forcément. En pratique, le moyen sûr de mettre fin à l'usurpation est donc de **fermer puis rouvrir la connexion** : **vérifié sur 12.3.2, la forme `SET SESSION AUTHORIZATION DEFAULT` n'est pas reconnue** (erreur de syntaxe `1064`) — il n'existe donc pas, à ce stade, de forme de retour dédiée dans MariaDB.

À noter également pour l'audit : comme `USER()` et `CURRENT_USER()` adoptent tous deux l'identité endossée, le connecteur d'origine n'est plus visible au travers de ces fonctions après la bascule.

## Cas d'usage

Trois usages se dégagent. Le premier est l'**impersonation administrative** : exécuter une opération de maintenance ou une correction sous un compte précis, afin de la réaliser avec les droits et la propriété attendus. Le deuxième est le **test de privilèges** : pour vérifier ce qu'un compte peut réellement faire — par exemple s'assurer qu'un compte applicatif est bien limité à `SELECT` sur certaines tables — il est plus fiable de prendre son identité et de tenter les opérations que de déduire ses droits à partir des `GRANT`. Le troisième est l'**attribution derrière un mandataire de confiance** : une couche applicative intermédiaire détenant `SET USER` peut endosser l'identité de chaque utilisateur final, de sorte que les instructions suivantes s'exécutent avec les droits de cet utilisateur et apparaissent sous son identité dans `CURRENT_USER()` et dans la piste d'audit (10.8).

## Une fonction d'administration, pas une frontière de sécurité multi-locataire

Ce dernier cas appelle une mise en garde essentielle, qui précise la discussion sur le pooling de connexions amorcée en 10.9. Parce qu'elle exige `SET USER` — un privilège quasi super-utilisateur — et qu'elle contourne l'authentification, cette instruction est un **outil d'impersonation administrative, non un mécanisme d'isolation**. Le standard SQL déconseille de s'en servir pour faire partager une même connexion à des utilisateurs qui ne se font pas mutuellement confiance : dès lors qu'une connexion peut changer d'autorisation, les identités endossées héritent à leur tour de cette capacité.

La distinction est donc la suivante. Légitime : une couche applicative **entièrement de confiance** qui attribue et restreint les actions par utilisateur final. À proscrire : s'appuyer sur ce mécanisme comme frontière de sécurité entre locataires non fiables partageant une connexion mutualisée. `SET SESSION AUTHORIZATION` facilite l'attribution et le cantonnement des privilèges sous un mandataire de confiance, mais il ne remplace pas une authentification propre à chaque utilisateur lorsque ceux-ci ne se font pas confiance.

## Synthèse

`SET SESSION AUTHORIZATION` (MariaDB 12.0) permet à un compte disposant de `SET USER` d'endosser l'identité d'un autre utilisateur pour la session : les instructions suivantes s'exécutent alors avec les privilèges et sous l'identité de ce compte, la bascule contournant les contrôles d'authentification, de verrouillage, d'expiration et de TLS de la cible. L'instruction s'emploie hors transaction, prepared statement et procédure stockée, et le retour à l'identité initiale passe en pratique par une reconnexion. Le privilège `SET USER` qui la conditionne doit être traité comme une capacité de très haut niveau. On réservera l'usage de cette instruction à l'impersonation administrative, au test de privilèges et à l'attribution sous mandataire de confiance — jamais comme barrière de sécurité entre locataires non fiables. Elle s'articule avec le privilège `SET USER` et les privilèges granulaires (10.11), la sécurité applicative et le pooling (10.9), et l'audit (10.8), qui enregistre l'identité effective des actions.

⏭️ [Administration et Configuration](/11-administration-configuration/README.md)
