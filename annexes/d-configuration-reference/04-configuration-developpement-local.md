🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.4 — Développement local

> 💻 Un poste de travail : empreinte réduite, confort et rapidité d'itération priment sur la performance et la durabilité.

Le profil **développement local** vise la machine d'un développeur : ressources modestes, partagées avec l'IDE, le navigateur et d'autres conteneurs, et des données le plus souvent reproductibles (jeux de test, fixtures). On y privilégie une faible empreinte, des opérations rapides et une mise au point aisée — au détriment de la durabilité, de la concurrence et de la haute disponibilité, qui n'ont pas d'intérêt ici.

> ⚠️ **Règle d'or** : les réglages de cette section sont volontairement relâchés. Ils ne doivent **jamais** être appliqués en production.

---

## Configuration de référence (`[mysqld]`)

```ini
[mysqld]
# ============================================================
#  Profil développement local — MariaDB 12.3
#  ⚠️ Réglages relâchés : NE JAMAIS utiliser en production.
# ============================================================

# --- Mémoire (empreinte réduite) ---
innodb_buffer_pool_size        = 256M       # selon la RAM ; rester modeste
innodb_log_file_size           = 128M

# --- Durabilité relâchée (vitesse > sécurité, données reproductibles) ---
innodb_flush_log_at_trx_commit = 0          # le plus rapide ; perte possible des dernières transactions

# --- Concurrence faible (pas de thread pool nécessaire) ---
max_connections                = 50

# --- Confort de développement ---
local_infile                   = 1          # autorise LOAD DATA LOCAL INFILE (jeux de test)
# Décommenter pour voir TOUTES les requêtes (verbeux) :
# general_log                  = 1
# general_log_file             = /var/log/mysql/general.log
```

## Empreinte mémoire réduite

Sur un poste partagé, MariaDB ne doit pas monopoliser la mémoire. Un `innodb_buffer_pool_size` de quelques centaines de mégaoctets suffit largement pour des bases de test, et le journal redo peut rester petit. L'objectif est inverse de celui de la production : on cherche le strict nécessaire, pas le maximum.

## Durabilité : on peut la relâcher (en dev seulement)

C'est le compromis caractéristique du développement. Avec `innodb_flush_log_at_trx_commit = 0`, MariaDB n'effectue plus de `fsync` à chaque validation, ce qui accélère nettement les écritures — au prix de la perte possible des toutes dernières transactions en cas de crash ou de coupure. Sur un poste de développement, ce risque est sans conséquence : les données se régénèrent depuis les scripts d'amorçage.

> ⚠️ Ce relâchement est **réservé au développement**. En production, on revient impérativement à `innodb_flush_log_at_trx_commit = 1` (voir [D.1](01-configuration-oltp.md)).

## Garder le `sql_mode` aligné sur la production

Voici en revanche un réglage qu'il ne faut **pas** relâcher. Le `sql_mode` du poste de développement doit rester identique à celui de la production (mode strict). Un environnement de dev plus permissif laisse passer des erreurs de type ou de contrainte qui n'éclateront qu'en production — le classique « ça marchait sur ma machine ». La bonne pratique est donc de ne pas surcharger le `sql_mode` par défaut, ou, si la production en impose un, de le reproduire à l'identique. *Voir Ch. 11.3.*

## Confort de développement

Quelques réglages facilitent le travail au quotidien. `local_infile = 1` autorise `LOAD DATA LOCAL INFILE`, pratique pour charger des jeux de données de test. Pour le débogage, activer `general_log` permet de voir passer chaque instruction envoyée au serveur — sortie verbeuse, à n'activer que ponctuellement (une alternative est le slow query log avec `long_query_time = 0`, qui journalise toutes les requêtes avec leur durée). Le `PERFORMANCE_SCHEMA`, désactivé par défaut, peut rester éteint pour économiser de la mémoire, sauf si l'on souhaite précisément l'explorer.

## Cas du conteneur Docker

Beaucoup d'environnements de développement exécutent MariaDB dans Docker. L'image officielle se paramètre par variables d'environnement (`MARIADB_ROOT_PASSWORD`, `MARIADB_DATABASE`, `MARIADB_USER`, `MARIADB_PASSWORD`) et accepte un `my.cnf` monté en volume ; il faut veiller à la persistance des données via un volume dédié. *Voir Ch. 16.3 (conteneurisation) et Ch. 16.3.2 (Docker Compose pour le développement).*

## Ce qu'il n'est pas utile de configurer

En développement, plusieurs éléments de production sont superflus. La **journalisation binaire** et la réplication n'ont d'intérêt que si l'on teste précisément la réplication ; les omettre épargne de l'espace disque et des E/S. Le **thread pool** est inutile à faible concurrence. Et l'on évite, comme toujours, le Query Cache et les variables dépréciées ou retirées (`innodb_buffer_pool_instances`, `innodb_buffer_pool_chunk_size`, les retirées de la série 12.x…).

## Pour aller plus loin

Les environnements de développement sont traités au [Ch. 17.7](../../17-integration-developpement/README.md), la conteneurisation et Docker Compose au [Ch. 16.3](../../16-devops-automatisation/README.md), et les modes SQL au [Ch. 11.3](../../11-administration-configuration/README.md). Les profils de production se trouvent en [D.1](01-configuration-oltp.md), [D.2](02-configuration-olap.md) et [D.3](03-configuration-mixed-workload.md).

---

⬅️ [D.3 — Mixed workload](03-configuration-mixed-workload.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [Annexe E — Checklist de Performance](../e-checklist-performance/README.md)

⏭️ [Checklist de Performance](/annexes/e-checklist-performance/README.md)
