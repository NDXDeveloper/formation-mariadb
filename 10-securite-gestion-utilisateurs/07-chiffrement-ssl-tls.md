🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 — Chiffrement des connexions (SSL/TLS)

Le chiffrement des connexions est la **couche réseau** du modèle de défense en profondeur introduit en 10.1. Là où l'authentification vérifie *qui* se connecte et les privilèges déterminent *ce qu'il peut faire*, le chiffrement TLS protège *les données qui circulent* entre le client et le serveur. Sans lui, tout le trafic — y compris les échanges d'authentification et le contenu des requêtes — transite **en clair**, à la merci d'une écoute du réseau. Cette section présente les principes du chiffrement TLS et la manière de l'imposer ; les sous-sections détaillent ensuite la configuration serveur (10.7.1), les certificats (10.7.2), le TLS « zéro-configuration » (10.7.3) et les clés protégées par passphrase (10.7.4).

---

## Pourquoi chiffrer les connexions ?

Plusieurs mécanismes vus dans ce chapitre **dépendent directement** de TLS pour leur sécurité :

- le plugin **PAM** (10.5.3) transmet le mot de passe au serveur **en clair** : TLS est alors indispensable ;
- **`caching_sha2_password`** (10.5.5) exige une **connexion sécurisée** (ou un échange RSA plus contraignant) ;
- même **`mysql_native_password`** (10.5.1) reste vulnérable si un attaquant capture l'échange d'authentification sur le réseau.

Au-delà de l'authentification, le chiffrement protège l'ensemble des données échangées. TLS apporte trois garanties :

- **confidentialité** : les données sont illisibles pour un tiers qui intercepterait le trafic ;
- **intégrité** : les données ne peuvent être altérées en transit sans être détectées ;
- **authenticité** : l'identité du serveur — et, en option, celle du client — est vérifiée à l'aide de certificats.

---

## SSL ou TLS ?

Un point de vocabulaire. « SSL » (*Secure Sockets Layer*) est le nom **historique** du protocole ; il a été remplacé par **TLS** (*Transport Layer Security*), seul réellement utilisé aujourd'hui (les versions SSL étant obsolètes et non sûres). Pour des raisons d'héritage, MariaDB conserve le terme « ssl » dans le nom de ses options et variables (`ssl_cert`, `ssl_key`, `--ssl`, `REQUIRE SSL`…), mais le protocole employé est bien **TLS**. Dans ce chapitre, « SSL » et « TLS » désignent donc la même chose.

---

## Ce que TLS protège — et ne protège pas

Il est essentiel de bien situer le périmètre : TLS chiffre les données **en transit**, c'est-à-dire pendant leur circulation sur le réseau. Il **ne chiffre pas** les données **au repos** (les fichiers stockés sur le serveur). La protection des données sur disque relève d'un mécanisme distinct, le **chiffrement au repos** (*encryption at rest*), traité au chapitre 18.7. Les deux sont complémentaires et répondent à des menaces différentes.

---

## Les composants du chiffrement TLS

Une connexion TLS s'appuie sur quelques éléments cryptographiques, détaillés en 10.7.1 et 10.7.2 :

- une **clé privée** et un **certificat** côté serveur, qui permettent au client de vérifier l'identité du serveur et d'établir un canal chiffré ;
- une **autorité de certification (CA)**, qui signe les certificats et sert de référence de confiance ;
- en option, des **certificats côté client**, lorsqu'on souhaite que le serveur authentifie aussi le client par certificat (TLS mutuel, *mTLS*).

---

## Imposer le chiffrement : deux niveaux

MariaDB permet d'exiger le chiffrement à deux échelles complémentaires.

**À l'échelle du serveur**, la variable `require_secure_transport` impose que **toute** connexion utilise un transport sécurisé (TLS, ou un canal local comme le socket Unix) :

```ini
[mariadb]
require_secure_transport = ON
```

**À l'échelle du compte**, la clause `REQUIRE` (posée dès `CREATE USER` / `ALTER USER`, voir 10.2) définit des exigences par utilisateur :

- `REQUIRE NONE` — aucune exigence (défaut) ;
- `REQUIRE SSL` — la connexion doit être chiffrée ;
- `REQUIRE X509` — le client doit présenter un **certificat valide** signé par une CA de confiance ;
- `REQUIRE CIPHER '…'` — un chiffrement précis est imposé ;
- `REQUIRE ISSUER '…'` / `REQUIRE SUBJECT '…'` — le certificat client doit avoir un émetteur ou un sujet déterminé.

Ces conditions se combinent avec `AND` :

```sql
CREATE USER 'app'@'%'   IDENTIFIED BY '...' REQUIRE SSL;
CREATE USER 'admin'@'%' IDENTIFIED BY '...' REQUIRE X509;
ALTER  USER 'svc'@'%'   REQUIRE SUBJECT '/CN=svc.example.com' AND ISSUER '/CN=Mon-CA';
```

On combine souvent les deux niveaux : `require_secure_transport` garantit un plancher pour toutes les connexions, tandis que `REQUIRE X509` durcit l'exigence pour les comptes les plus sensibles.

---

## Côté client

Le client peut demander, voire vérifier, le chiffrement. Avec les outils en ligne de commande, les options `--ssl`, `--ssl-ca` (certificat de la CA) et `--ssl-verify-server-cert` permettent d'établir une connexion TLS **et de vérifier l'identité du serveur** :

```bash
mariadb --ssl --ssl-ca=ca.pem --ssl-verify-server-cert -u app -p
```

Sans vérification du certificat, le client peut chiffrer la connexion de façon « opportuniste », mais reste alors exposé à une usurpation de serveur ; la vérification (avec la CA) est ce qui protège contre l'attaque par interception.

---

## Vérifier qu'une connexion est chiffrée

Pour savoir si la session en cours est chiffrée, on interroge la variable d'état `Ssl_cipher` : vide, la connexion n'est pas chiffrée ; renseignée, elle indique le chiffrement TLS négocié.

```sql
SHOW SESSION STATUS LIKE 'Ssl_cipher';
-- valeur vide  => connexion en clair
-- valeur (ex. TLS_AES_256_GCM_SHA384) => connexion chiffrée
```

La commande `\s` (status) du client `mariadb` affiche également les informations TLS de la session.

---

## Au programme

Les sous-sections suivantes détaillent la mise en œuvre :

- **10.7.1 — Configuration serveur SSL** : activer TLS sur le serveur (clé, certificat, variables, versions de protocole).
- **10.7.2 — Certificats et CA** : créer et gérer les certificats serveur et client, et l'autorité de certification.
- **10.7.3 — TLS zéro-configuration** : le chiffrement disponible « clé en main » sans mise en place manuelle des certificats.
- **10.7.4 — Clés SSL protégées par passphrase** 🆕 : déchiffrer une clé privée protégée par mot de passe.

---

## À retenir

Le chiffrement **SSL/TLS** protège les connexions MariaDB en garantissant **confidentialité, intégrité et authenticité** des échanges — il est indispensable dès lors que des plugins comme PAM ou `caching_sha2_password` transmettent ou exposent des informations d'authentification. « SSL » et « TLS » désignent ici le même protocole (TLS, le seul utilisé). TLS protège les données **en transit**, pas au repos (chiffrement au repos : 18.7). Le chiffrement repose sur une **clé**, un **certificat** et une **CA**. On l'impose à deux niveaux : globalement avec `require_secure_transport`, et par compte avec la clause `REQUIRE` (`SSL`, `X509`, `CIPHER`, `ISSUER`, `SUBJECT`). On vérifie une session chiffrée via la variable d'état `Ssl_cipher`.

---

> 🔔 **Note de version.** Les mécanismes décrits ici — clause `REQUIRE`, variable `require_secure_transport`, variables d'état `Ssl_*` — sont stables dans **MariaDB 12.3 LTS**. Le TLS « zéro-configuration » (10.7.3) est devenu un acquis depuis la série 11.x, et la prise en charge des clés privées protégées par passphrase (10.7.4) est une nouveauté de la série 12.x. La protection des **données au repos** est traitée séparément au chapitre 18.7.

⏭️ [Configuration serveur SSL](/10-securite-gestion-utilisateurs/07.1-configuration-serveur-ssl.md)
