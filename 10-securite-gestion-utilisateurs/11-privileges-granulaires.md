🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.11 — Privilèges granulaires (options 11.8)

## Du privilège `SUPER` aux privilèges fins

Historiquement, une grande partie des capacités d'administration de MariaDB était regroupée sous un unique privilège très large : `SUPER`. Accorder `SUPER` à un utilisateur pour lui permettre une seule opération revenait à lui confier, du même coup, une foule de pouvoirs sans rapport — gérer la réplication, contourner le mode lecture seule, administrer le binlog, modifier des variables système sensibles, etc. C'est l'antithèse du principe du moindre privilège.

À partir de **MariaDB 10.5.2** (MDEV-21743), le privilège `SUPER` a été **découpé** en un ensemble de privilèges administratifs fins, dans un esprit comparable aux *dynamic privileges* de MySQL 8. Ce modèle de contrôle d'accès granulaire a ensuite été enrichi au fil des versions ; la **11.8 LTS** l'a établi comme socle standard — repris tel quel dans la 12.3 — au point d'en faire l'un de ses arguments de sécurité. L'intérêt est direct : on accorde **exactement la capacité d'administration nécessaire**, et rien de plus, ce qui réduit la surface d'attaque et clarifie la responsabilité de chaque compte.

## Le catalogue des privilèges issus du découpage de `SUPER`

Ces privilèges sont **globaux** : ils se concèdent au niveau serveur, avec `*.*` comme portée. Le tableau ci-dessous récapitule les principaux et la capacité que chacun ouvre.

| Privilège | Capacité accordée |
|-----------|-------------------|
| `BINLOG ADMIN` | Administrer le journal binaire : `PURGE BINARY LOGS` et réglage des variables système associées |
| `BINLOG MONITOR` | Consulter l'état du binlog (`SHOW BINLOG STATUS`, `SHOW BINARY LOGS`) ; nouveau nom de `REPLICATION CLIENT`, dont l'alias reste accepté |
| `BINLOG REPLAY` | Rejouer le binlog via l'instruction `BINLOG` (générée par `mariadb-binlog`) |
| `CONNECTION ADMIN` | Outrepasser `max_connections`, `max_user_connections` et `max_password_errors`, et fermer les connexions d'autres utilisateurs |
| `FEDERATED ADMIN` | Exécuter `CREATE SERVER`, `ALTER SERVER`, `DROP SERVER` |
| `READ_ONLY ADMIN` | Piloter la variable `read_only` et écrire malgré `read_only = ON` |
| `REPLICATION MASTER ADMIN` | Administrer un serveur primaire (`SHOW REPLICA HOSTS`, variables `gtid_binlog_state`, `gtid_domain_id`, `server_id`…) |
| `REPLICATION SLAVE ADMIN` | Administrer une réplique (`START/STOP SLAVE`, `CHANGE MASTER`…) |
| `SLAVE MONITOR` (alias `REPLICA MONITOR`) | Exécuter `SHOW SLAVE STATUS` / `SHOW REPLICA STATUS` (non couvert par `BINLOG MONITOR`) |
| `SET USER` | Fixer le `DEFINER` lors de la création de triggers, vues et routines stockées |

On note au passage deux changements de terminologie issus de cette refonte : `REPLICATION CLIENT` est devenu `BINLOG MONITOR` (l'ancien nom restant utilisable), et la consultation de l'état d'une réplique relève désormais d'un privilège distinct, **`SLAVE MONITOR`** (dont `REPLICA MONITOR` est un alias accepté en entrée, canonicalisé en `SLAVE MONITOR` par le serveur — c'est ce dernier nom qu'affichent `SHOW PRIVILEGES` et `SHOW GRANTS`), qui n'est pas inclus dans `BINLOG MONITOR`.

## Voir la définition d'une routine sans en être propriétaire

Parmi les ajouts plus récents à ce modèle figure le privilège **`SHOW CREATE ROUTINE`**, disponible **depuis MariaDB 11.3.0**. Il autorise un utilisateur à consulter l'instruction de définition d'une routine — par exemple via `SHOW CREATE FUNCTION` ou `SHOW CREATE PROCEDURE` — **même lorsqu'il n'en est pas le propriétaire**, sans qu'il faille pour autant lui accorder des droits étendus. C'est précisément le type de privilège ciblé que recherche un auditeur ou un exploitant devant inspecter des définitions de routines dans une démarche de revue, sans pouvoir les modifier ni accéder au reste. À noter que ce privilège ne s'applique pas aux rôles.

## Séparer `READ_ONLY ADMIN` de `SUPER`

Le privilège `READ_ONLY ADMIN` est inclus dans `SUPER`, mais il peut en être **séparé** : il est possible de le retirer à l'ensemble des comptes, y compris à ceux qui détenaient `SUPER`. Cette séparation a une vertu opérationnelle nette : sur une réplique ou pendant une fenêtre de maintenance, elle permet de **garantir réellement** le mode lecture seule, en s'assurant qu'aucun compte — pas même un compte historiquement très privilégié — ne pourra écrire tant que `read_only` est actif.

## Mettre en œuvre les privilèges granulaires

S'agissant de privilèges globaux, ils se concèdent avec `GRANT … ON *.*`. La démarche consiste à raisonner par fonction et à n'attribuer que les privilèges strictement requis pour cette fonction.

Un compte de **supervision** qui doit observer l'état de la réplication et du binlog, mais ne rien pouvoir modifier, se limite ainsi aux privilèges de consultation :

```sql
CREATE USER 'supervision'@'10.0.2.%' IDENTIFIED VIA ed25519 USING PASSWORD('secret_fort');
GRANT BINLOG MONITOR, SLAVE MONITOR ON *.* TO 'supervision'@'10.0.2.%';
```

Un compte d'**exploitation de la réplication**, chargé de piloter une réplique sans détenir les autres pouvoirs d'un `SUPER`, combine l'administration et la surveillance des répliques :

```sql
GRANT REPLICATION SLAVE ADMIN, SLAVE MONITOR ON *.* TO 'repl_ops'@'10.0.2.%';
```

Lorsque ces ensembles se répètent d'un compte à l'autre, on les factorise au moyen des **rôles** (10.4), qui regroupent un jeu de privilèges granulaires sous un nom unique et en garantissent la cohérence. La vérification s'effectue avec `SHOW GRANTS` pour un compte donné, et `SHOW PRIVILEGES` pour lister l'ensemble des privilèges reconnus par le serveur.

## Incidence sur la migration depuis MySQL

Ce modèle granulaire est conceptuellement proche des *dynamic privileges* de MySQL 8, mais les **noms diffèrent** et la correspondance n'est pas exacte : MySQL emploie par exemple `BINLOG_ADMIN`, `CONNECTION_ADMIN` ou `REPLICATION_SLAVE_ADMIN` (avec des tirets bas et des périmètres parfois distincts). Lors d'une migration, les privilèges fins doivent donc être **re-concédés avec la nomenclature MariaDB**, sans s'attendre à un transfert littéral. Le changement de nom de `REPLICATION CLIENT` en `BINLOG MONITOR` illustre par ailleurs qu'un même besoin peut porter des intitulés différents d'un moteur à l'autre.

## Synthèse

Les privilèges granulaires sont la traduction concrète du moindre privilège appliqué à l'administration : plutôt que d'accorder `SUPER`, on attribue la capacité précise attendue — administrer le binlog, surveiller une réplique, contourner ou non `read_only`, fixer un `DEFINER`. On construit ainsi des comptes et des rôles de supervision, d'exploitation de la réplication ou de maintenance taillés au plus juste. Ce modèle, largement issu du découpage de `SUPER` en 10.5.2 et enrichi depuis — notamment par `SHOW CREATE ROUTINE` en 11.3 —, constitue le contrôle d'accès fin standard du socle 11.8 LTS et de la 12.3. Il s'articule naturellement avec les rôles (10.4), la sécurité au niveau application (10.9) et le mécanisme `SET SESSION AUTHORIZATION` étudié à la section suivante (10.12).

⏭️ [SET SESSION AUTHORIZATION : exécuter des actions en tant qu'un autre utilisateur](/10-securite-gestion-utilisateurs/12-set-session-authorization.md)
