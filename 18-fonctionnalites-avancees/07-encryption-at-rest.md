🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.7 Encryption at rest

## Chiffrer les données au repos

Le **chiffrement au repos** (*encryption at rest*) protège les données telles qu'elles sont stockées sur le disque. L'objectif est qu'un tiers ayant mis la main sur les **fichiers** de la base — disque dérobé, copie de sauvegarde égarée, support mal mis au rebut, accès non autorisé au système de fichiers — ne puisse rien en tirer sans détenir les clés de déchiffrement.

C'est une réponse directe à toute une famille de menaces *physiques* ou *hors-base* : vol de matériel, fuite de sauvegardes, recyclage de disques. MariaDB chiffre alors le contenu des fichiers avec un algorithme symétrique de type **AES**, et ne les déchiffre qu'au moment de les charger en mémoire pour les exploiter.

## Une protection transparente

Un atout majeur du chiffrement au repos est sa **transparence** pour les applications. Le chiffrement et le déchiffrement s'opèrent au niveau de la couche de stockage : les requêtes, les schémas et le code applicatif restent rigoureusement inchangés. Une fois la fonctionnalité activée et configurée, personne, côté application, n'a à se soucier de la cryptographie sous-jacente.

## Ce que le chiffrement au repos ne protège pas

Il est essentiel de cerner les limites de cette protection, car elle est souvent mal comprise.

Le chiffrement au repos **ne protège pas les données en transit** entre le client et le serveur : c'est le rôle du chiffrement des connexions par SSL/TLS (§10.7), un sujet distinct et complémentaire. Il **ne protège pas non plus les données en cours d'utilisation** : une fois chargées dans le buffer pool, elles y sont en clair. Et surtout, il **n'oppose aucune barrière à un utilisateur disposant d'identifiants valides** : celui-ci interroge la base et en obtient les résultats déchiffrés normalement — la confidentialité vis-à-vis des comptes légitimes relève du modèle de privilèges (§10), des vues de masquage (§9.5) et de l'audit (§10.8).

Le chiffrement au repos n'est donc pas une protection autosuffisante, mais **une couche parmi d'autres** dans une stratégie de défense en profondeur.

## Deux piliers : chiffrer, et gérer les clés

La mise en œuvre repose sur deux composants indissociables.

Le premier est le **chiffrement du stockage** proprement dit : quelles données sont chiffrées, avec quel algorithme, et comment l'activer. C'est l'objet de la sous-section 18.7.1.

Le second est la **gestion des clés**. MariaDB délègue la fourniture et la gestion des clés de chiffrement à un *plugin* dédié, que l'on choisit selon le contexte : une clé conservée dans un fichier sur le serveur, ou un gestionnaire de clés externe (KMS) tel que HashiCorp Vault ou un service cloud. Ce point est loin d'être secondaire : **le chiffrement ne vaut que ce que vaut la protection des clés.** Une clé mal gardée annule tout le bénéfice ; une clé perdue rend les données définitivement illisibles. C'est l'objet de la sous-section 18.7.2.

## Ce qui peut être chiffré

Le périmètre du chiffrement au repos dépasse les seules tables. Il peut couvrir, au choix et indépendamment :

- les **tablespaces InnoDB**, table par table ou pour l'ensemble des tables ;
- le **redo log** d'InnoDB ;
- les **fichiers temporaires** générés par le serveur ;
- les **journaux binaires** (binlog) ;
- les tables du moteur **Aria**.

Chiffrer les tables sans chiffrer les journaux ou les fichiers temporaires laisserait en effet des données en clair sur le disque ; la cohérence du périmètre fait partie des points traités en §18.7.1.

## Performance et compromis

Sur le plan des performances, le coût du chiffrement AES est aujourd'hui **modéré** grâce à l'accélération matérielle (jeu d'instructions AES-NI) présente sur les processeurs modernes. La véritable difficulté n'est pas le calcul, mais la **gestion des clés** : leur stockage sécurisé, leur rotation, leur sauvegarde et la garantie de ne jamais les perdre.

Le chiffrement au repos se **combine** par ailleurs avec la compression de tables (§18.6) : les données sont compressées *puis* chiffrées, ce qui permet de cumuler économie d'espace et confidentialité.

## Organisation de cette section

La suite se décline en deux sous-sections :

1. **[Data at Rest Encryption](07.1-data-at-rest-encryption.md)** (§18.7.1) — ce que l'on chiffre et comment l'activer : tablespaces, journaux, fichiers temporaires, et les variables de configuration associées.
2. **[Key management](07.2-key-management.md)** (§18.7.2) — la gestion des clés : plugins disponibles (dont `file_key_management`), rotation, et intégration avec des gestionnaires externes comme HashiCorp Vault.

⏭️ [Data at Rest Encryption](/18-fonctionnalites-avancees/07.1-data-at-rest-encryption.md)
