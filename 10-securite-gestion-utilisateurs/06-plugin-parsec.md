🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 — Plugin d'authentification PARSEC

PARSEC est le plugin d'authentification **le plus moderne** de MariaDB, et celui qui est **destiné à devenir le plugin par défaut** dans une version future. Comme `ed25519` (10.5.2), il propose une authentification par mot de passe fondée sur la signature à courbe elliptique ; mais il en corrige les principales faiblesses en ajoutant du **sel**, une **dérivation de clé** et un **aléa côté client**. C'est aujourd'hui le choix recommandé par MariaDB pour les **nouveaux comptes** à forte exigence de sécurité. Cette section en présente le principe, le fonctionnement, l'installation et la compatibilité.

---

## PARSEC en bref

PARSEC signifie *Password Authentication using Response Signed with Elliptic Curve*. Il combine plusieurs ingrédients de cryptographie moderne :

- des **mots de passe salés** ;
- une **dérivation de clé** (KDF) par **PBKDF2** ;
- un **format de stockage extensible** (qui pourra évoluer sans changer de plugin) ;
- des *scrambles* (aléas) **côté serveur *et* côté client** ;
- une signature de la réponse d'authentification par **ed25519 standard**, tel que fourni par les bibliothèques courantes (OpenSSL, WolfSSL, GnuTLS).

C'est cette dernière différence avec `ed25519` qui est notable : là où le plugin `ed25519` utilise une implémentation propre à MariaDB, PARSEC s'appuie sur l'ed25519 **non modifié** des bibliothèques standard.

---

## Ce que PARSEC corrige par rapport à ed25519

L'intérêt de PARSEC se comprend mieux en le comparant à `ed25519`, dont il comble deux lacunes :

- **Absence de sel.** L'empreinte ed25519 n'est pas salée : deux mots de passe identiques produisent la même valeur stockée, exposée aux **tables arc-en-ciel**. PARSEC introduit un **sel**, rendant chaque empreinte unique.
- **Absence d'aléa côté client.** Dans le schéma ed25519, le serveur fournit seul le défi (*nonce*) ; un intermédiaire malveillant pouvait imposer un nonce prévisible (par exemple `0000…`) pour faciliter une attaque par interception. PARSEC ajoute un **scramble côté client** : le défi combine désormais l'aléa du serveur **et** celui du client.

À cela s'ajoute la **dérivation de clé par PBKDF2** (avec un grand nombre d'itérations), qui ralentit considérablement toute tentative de force brute hors ligne. Le résultat conforme aux recommandations NIST : résistance aux tables arc-en-ciel (sel), aux attaques par interception (double aléa) et au cassage hors ligne (étirement de clé).

---

## Comment ça fonctionne

Le serveur ne stocke pas le mot de passe, mais une **clé publique ed25519 dérivée** du mot de passe via PBKDF2, accompagnée des paramètres de dérivation. La chaîne d'authentification stockée a une forme telle que :

```text
P0:WW9sXaaL/o:vubFBzIrapbfHct1/J72dnUryz5VS7lA6XHH8sIx4TI
```

On y lit trois parties : le préfixe `P` indique l'algorithme de dérivation **PBKDF2**, et le chiffre qui suit (`0`) encode le **nombre d'itérations** (`0` correspond à 1024, `1` à 2048, etc., en doublant) ; vient ensuite le **sel** (`WW9sXaaL/o`) ; puis l'empreinte proprement dite — la **clé publique ed25519** dérivée de `PBKDF2(mot de passe, sel)`. L'ensemble « sel + paramètres » est désigné comme l'*ext-salt*.

L'échange d'authentification se déroule ainsi : le serveur envoie un aléa de 32 octets ; le client récupère l'*ext-salt* (fonction de hachage SHA-512 ou SHA-256, nombre d'itérations, sel, longueur de clé), recalcule sa **clé dérivée** par PBKDF2, puis envoie son propre aléa de 32 octets et la **concaténation des deux aléas signée en ed25519** avec cette clé. Le serveur vérifie la signature à l'aide de la clé publique stockée et répond « ok » ou « access denied ». Le mot de passe lui-même ne circule jamais.

---

## Installation

PARSEC est livré avec les versions récentes de MariaDB, mais **n'est ni installé ni utilisé par défaut**. Si l'on tente de l'utiliser sans l'avoir chargé, on obtient :

```text
ERROR 1524 (HY000): Plugin 'parsec' is not loaded
```

Il suffit alors de l'installer **une seule fois** ; le serveur s'en souviendra même après un redémarrage ou une mise à niveau, sans avoir à ajouter d'option en ligne de commande ni de fichier de configuration :

```sql
INSTALL SONAME 'auth_parsec';
```

Le plugin s'appuie sur les bibliothèques cryptographiques standard (OpenSSL, GnuTLS…) déjà présentes pour ed25519, PBKDF2 et la génération d'aléas.

---

## Créer un compte

La déclaration suit le schéma habituel : le serveur calcule l'empreinte salée et dérivée à partir du mot de passe fourni.

```sql
CREATE USER 'alice'@'%' IDENTIFIED VIA parsec USING PASSWORD('mot_de_passe');
```

---

## Compatibilité client

Comme pour `ed25519`, le **client doit implémenter PARSEC**. Un atout de conception facilite cette prise en charge : PBKDF2 est disponible partout (Windows natif, Java, JavaScript, PHP, .NET), ce qui rend le plugin **portable** à implémenter. Concrètement, il faut des **connecteurs récents** :

- MariaDB Connector/C et MariaDB Connector/J disposent d'une implémentation de PARSEC ;
- côté .NET, **MySqlConnector** prend en charge PARSEC via le paquet NuGet `MySqlConnector.Authentication.Ed25519`, en appelant `ParsecAuthenticationPlugin.Install()` au démarrage de l'application.

Du côté des intermédiaires, **MaxScale** (chapitre 14) propose un authentificateur PARSEC (`parsecauth`), prévu pour MariaDB 12 et au-delà ; il requiert que le compte de service dispose du privilège `SET USER` (lié à la section 10.12). Comme toujours, on vérifiera la prise en charge effective des connecteurs et proxies avant de généraliser PARSEC.

---

## Quand l'utiliser

PARSEC est le choix **recommandé pour de nouveaux comptes** lorsqu'on vise la meilleure sécurité d'authentification par mot de passe disponible dans MariaDB, et que les clients le prennent en charge. Il se positionne au sommet de la progression vue dans ce chapitre :

- `mysql_native_password` (10.5.1) — compatibilité maximale, sécurité datée (SHA-1) ;
- `ed25519` (10.5.2) — sécurité renforcée, mais sans sel ni aléa client ;
- **PARSEC** — sel, dérivation de clé PBKDF2, double aléa, ed25519 standard : le plus robuste, et futur défaut.

À côté, `caching_sha2_password` (10.5.5) reste réservé à la **migration** depuis MySQL, et GSSAPI/Kerberos (10.5.4) à l'**authentification d'entreprise** sans mot de passe. Pour une instance MariaDB native et moderne, PARSEC est la cible à privilégier dès que l'écosystème client le permet.

---

## À retenir

**PARSEC** (*Password Authentication using Response Signed with Elliptic Curve*) est le plugin d'authentification par mot de passe le plus avancé de MariaDB, **destiné à devenir le défaut**. Il améliore `ed25519` en ajoutant un **sel**, une **dérivation de clé PBKDF2** (étirement) et un **aléa côté client** (contre les attaques par interception), tout en signant avec de l'**ed25519 standard**. Le serveur stocke une **clé publique dérivée** du mot de passe (jamais le mot de passe). Il s'installe en une fois (`INSTALL SONAME 'auth_parsec'`), persiste après redémarrage, mais **n'est pas activé par défaut**. Sa contrepartie est la nécessité d'un **client compatible** (connecteurs récents). C'est le choix recommandé pour les **nouveaux comptes** à forte sécurité.

---

> 🔔 **Note de version.** PARSEC est disponible depuis **MariaDB Community Server 11.6** (et Enterprise Server 11.8), via MDEV-32618. Dans **MariaDB 12.3 LTS**, il est présent mais **n'est pas encore installé ni utilisé par défaut**, bien qu'il soit appelé à le devenir dans une version future. Un seul `INSTALL SONAME 'auth_parsec'` suffit à l'activer durablement. Sources : *MariaDB Server Documentation — Authentication Plugin PARSEC* et *Pluggable Authentication Overview*.

⏭️ [Chiffrement des connexions (SSL/TLS)](/10-securite-gestion-utilisateurs/07-chiffrement-ssl-tls.md)
