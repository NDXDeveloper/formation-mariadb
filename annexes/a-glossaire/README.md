🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A — Glossaire des Termes Techniques

> 📖 Référence transversale de la formation MariaDB 12.3 LTS — à consulter au fil des chapitres.

Cette annexe rassemble les termes et acronymes qui reviennent tout au long de la formation. Elle joue le rôle d'un dictionnaire de référence : on ne la lit pas de façon linéaire, on la consulte ponctuellement, dès qu'un terme rencontré dans un chapitre demande une définition rapide ou un rappel.

Les définitions sont cadrées dans le contexte de **MariaDB 12.3 LTS** (et, lorsque c'est pertinent, de la LTS précédente 11.8, qui sert de point de comparaison pour la migration). Lorsqu'un terme est lié à une nouveauté de la série 12.x, il est signalé par le marqueur 🆕, comme dans le reste de la formation.

## Objectif

Le glossaire poursuit trois buts. D'abord lever les ambiguïtés : un même sigle peut recouvrir des réalités différentes selon le contexte (SGBD, réseau, cloud), et cette annexe fixe le sens retenu ici. Ensuite servir de point d'entrée : chaque entrée renvoie, quand c'est utile, vers le chapitre qui développe le concept en profondeur. Enfin accélérer la lecture : un profil débutant peut avancer sans rester bloqué sur un terme technique, tandis qu'un profil confirmé y trouve un rappel express.

## Comment utiliser ce glossaire

Pour la **définition d'un concept** (ACID, MVCC, moteur de stockage, etc.), consultez la section A.1. Pour la **signification d'un sigle** (FK, PK, GTID, SST, etc.), reportez-vous à la section A.2. En cas de doute entre un concept et une abréviation, commencez par A.2 : l'acronyme y est développé, puis renvoyé vers le terme correspondant en A.1 lorsqu'une définition plus complète existe.

## Contenu de l'annexe

### A.1 — [Termes MariaDB essentiels](01-termes-mariadb-essentiels.md)

Définitions des notions de fond manipulées dans la formation : propriétés transactionnelles (ACID), contrôle de concurrence (MVCC), moteur par défaut (InnoDB), cluster synchrone (Galera), ainsi que l'ensemble du vocabulaire récurrent du modèle relationnel et de l'écosystème MariaDB. Chaque terme est défini de façon autonome et accompagné, le cas échéant, d'un renvoi vers le chapitre où il est approfondi.

### A.2 — [Acronymes courants](02-acronymes-courants.md)

Développement des sigles fréquents : FK (clé étrangère), PK (clé primaire), CTE (expression de table commune), GTID (identifiant de transaction global), SST et IST (transferts d'état Galera), parmi d'autres abréviations qui parsèment la documentation et les chapitres. Le format y est volontairement compact : sigle, forme développée, puis courte mise en contexte.

## Où ces notions sont-elles développées ?

Le glossaire reste synthétique par nature. Pour approfondir, les principaux concepts sont traités en détail dans les chapitres suivants.

| Notion | Chapitre de référence |
|--------|----------------------|
| ACID, MVCC, niveaux d'isolation, verrous | [Ch. 6 — Transactions et Concurrence](../../06-transactions-et-concurrence/README.md) |
| InnoDB, Aria, ColumnStore et autres moteurs | [Ch. 7 — Moteurs de Stockage](../../07-moteurs-de-stockage/README.md) |
| GTID, réplication, failover | [Ch. 13 — Réplication](../../13-replication/README.md) |
| Galera, SST / IST, quorum, MaxScale | [Ch. 14 — Haute Disponibilité](../../14-haute-disponibilite/README.md) |
| CTE, window functions, JSON | [Ch. 4 — Concepts Avancés SQL](../../04-concepts-avances-sql/README.md) |

## Conventions

Le marqueur 🆕 signale un terme ou un comportement introduit (ou modifié) par la série 12.x depuis la 11.8. Les noms d'objets SQL, variables système et commandes sont notés en `police à chasse fixe`. Les renvois entre entrées sont introduits par la mention « *voir* … ».

---

⬅️ [Chapitre 20 — Cas d'Usage et Architectures](../../20-cas-usage-architectures/README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [A.1 — Termes MariaDB essentiels](01-termes-mariadb-essentiels.md)

⏭️ [Termes MariaDB essentiels (ACID, MVCC, InnoDB, Galera, etc.)](/annexes/a-glossaire/01-termes-mariadb-essentiels.md)
