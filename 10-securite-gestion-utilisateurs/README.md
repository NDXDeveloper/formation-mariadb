🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 10 — Sécurité et Gestion des Utilisateurs

*Partie 5 : Sécurité et Administration (DBA)*

---

La sécurité d'une base de données n'est pas une fonctionnalité que l'on active à la fin d'un projet : c'est une propriété transversale qui se construit couche après couche, depuis le réseau jusqu'à la donnée elle-même. Une instance MariaDB mal protégée expose non seulement ses tables, mais aussi l'ensemble du système d'information qu'elle alimente. Pour un administrateur, maîtriser le modèle de sécurité est donc une compétence aussi fondamentale que savoir écrire une requête.

MariaDB applique le principe de **défense en profondeur** : plutôt que de reposer sur une unique barrière, la protection s'organise en plusieurs niveaux complémentaires. Le **chiffrement des connexions** (SSL/TLS) protège les données en transit ; l'**authentification** vérifie l'identité de celui qui se connecte ; le **système de privilèges et de rôles** détermine ce que chaque compte est autorisé à faire ; l'**audit** trace les accès et les opérations sensibles ; enfin, la **validation des mots de passe** et la sécurité applicative renforcent l'ensemble. Ce chapitre parcourt ces couches dans l'ordre, du contrôle d'accès aux mécanismes les plus avancés.

Le contrôle d'accès de MariaDB repose sur deux principes structurants. D'abord, un compte n'est pas seulement un nom d'utilisateur : il est défini par le couple **`utilisateur@hôte`**, ce qui permet d'accorder des droits différents selon l'origine de la connexion. Ensuite, l'accès se déroule en **deux phases** : la phase de connexion (le client peut-il se connecter, et son identité est-elle vérifiée ?) puis la phase d'autorisation (l'utilisateur authentifié a-t-il le droit d'exécuter telle opération sur tel objet ?). Garder ces deux phases distinctes à l'esprit aide à diagnostiquer la plupart des problèmes d'accès.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez en mesure de :

- Comprendre le **modèle de sécurité** de MariaDB et la distinction entre authentification et autorisation.
- Créer, modifier et supprimer des **comptes utilisateurs** en tenant compte du périmètre `utilisateur@hôte`.
- Attribuer et révoquer des **privilèges** au bon niveau de granularité (global, base, table, colonne) et appliquer le principe du moindre privilège.
- Concevoir une politique d'accès maintenable à l'aide des **rôles**.
- Choisir et configurer le **plugin d'authentification** adapté à votre contexte (moderne, annuaire d'entreprise, compatibilité MySQL).
- Mettre en place le **chiffrement TLS** des connexions et gérer les certificats associés.
- Activer et configurer l'**audit** pour répondre aux exigences de traçabilité et de conformité.
- Identifier les **changements de comportement** introduits par MariaDB 12.3 LTS en matière de sécurité.

---

## Prérequis

Ce chapitre s'adresse à un public orienté administration. Il suppose une connaissance des bases du SQL (Partie 1), une familiarité avec la ligne de commande Unix/Linux, et la compréhension des notions d'installation et de configuration vues au chapitre 1. Aucune expérience préalable en sécurité des SGBD n'est nécessaire : les concepts sont introduits progressivement.

---

## Au programme

Le chapitre est organisé des fondations conceptuelles vers les mécanismes les plus spécialisés :

- **10.1 — Modèle de sécurité MariaDB.** Les principes de base du contrôle d'accès, le rôle des tables système et l'articulation des couches de protection.
- **10.2 — Création et gestion des utilisateurs.** Les commandes `CREATE USER`, `ALTER USER` et `DROP USER`, ainsi que le comportement de `DROP USER` en présence de sessions actives 🆕.
- **10.3 — Système de privilèges.** L'attribution (`GRANT`) et la révocation (`REVOKE`) des droits, et les différents niveaux concernés : global, base, table et colonne.
- **10.4 — Rôles.** La gestion centralisée des droits via `CREATE ROLE`, `SET ROLE` et `DEFAULT ROLE`, pour éviter la dispersion des privilèges compte par compte.
- **10.5 — Authentification : plugins.** Le panorama des mécanismes d'authentification, de `mysql_native_password` à `ed25519`, en passant par PAM/LDAP, GSSAPI/Kerberos et la compatibilité `caching_sha2_password` avec MySQL 8 🆕.
- **10.6 — Plugin d'authentification PARSEC.** Une méthode d'authentification moderne fondée sur la cryptographie à courbes elliptiques.
- **10.7 — Chiffrement des connexions (SSL/TLS).** La configuration côté serveur, la gestion des certificats et de l'autorité de certification, le TLS « zéro-configuration » et les clés protégées par passphrase 🆕.
- **10.8 — Audit et logging.** Le *Server Audit Plugin*, l'audit des connexions et des requêtes, et les options récentes de journalisation bufferisée 🆕.
- **10.9 — Sécurité au niveau application.** Les bonnes pratiques côté code pour réduire la surface d'attaque.
- **10.10 — Password validation plugins et politiques.** L'imposition de règles de robustesse sur les mots de passe.
- **10.11 — Privilèges granulaires.** Les options de contrôle d'accès affinées héritées de la 11.8.
- **10.12 — `SET SESSION AUTHORIZATION`.** L'exécution d'actions sous l'identité d'un autre utilisateur, utile au diagnostic et à l'administration 🆕.

---

## Nouveautés 12.3 LTS abordées dans ce chapitre

La série 12.x apporte plusieurs évolutions en matière de sécurité, signalées par le marqueur 🆕 :

- **`DROP USER` et sessions actives** : avertissement lorsqu'un compte supprimé possède encore des sessions ouvertes, avec échec de la commande en mode Oracle (10.2.1).
- **`caching_sha2_password`** : prise en charge du plugin d'authentification par défaut de MySQL 8, pour faciliter les migrations et les environnements mixtes (10.5.5).
- **Clés SSL protégées par passphrase** : possibilité de déchiffrer une clé privée protégée via `ssl_passphrase` (10.7.4).
- **Audit bufferisé** : journalisation plus performante grâce à `server_audit_file_buffer_size`, avec prise en charge des destinations `HOST:PORT` et du paramètre `tls_version` (10.8.3).
- **`SET SESSION AUTHORIZATION`** : exécution d'opérations sous l'identité d'un autre utilisateur (10.12).

Par ailleurs, plusieurs fonctionnalités **apparues au fil de la série 11.x et consolidées par la 11.8 LTS** sont désormais considérées comme du contenu standard : le plugin **PARSEC** (10.6, depuis la 11.6), le **TLS zéro-configuration** (10.7.3, depuis la 11.4) et les **privilèges granulaires** (10.11).

---

## Liens avec d'autres chapitres

La sécurité de MariaDB dépasse le cadre de la gestion des utilisateurs. Plusieurs sujets connexes sont traités ailleurs dans la formation : le **chiffrement des données au repos** (*encryption at rest*) au chapitre 18.7, la **prévention des injections SQL** et l'usage des *prepared statements* au chapitre 17.8, ainsi que le **masquage de données** via les vues au chapitre 9.5. La protection au niveau réseau et infrastructure (pare-feu applicatif de MaxScale) est abordée au chapitre 14.4.4.

---

> 🔔 **Note de version.** Ce chapitre est rédigé pour **MariaDB 12.3 LTS** (GA fin mai 2026, supportée jusqu'en juin 2029). La **11.8 LTS** sert de point de comparaison pour les comportements antérieurs. Les éléments marqués 🆕 désignent les nouveautés de la série 12.x par rapport à la 11.8.

⏭️ [Modèle de sécurité MariaDB](/10-securite-gestion-utilisateurs/01-modele-securite.md)
