🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 — Maintenance des tables

Avec le temps, les tables se dégradent : les suppressions et mises à jour laissent des espaces inutilisés, les statistiques sur lesquelles s'appuie l'optimiseur se périment, et — surtout sur les moteurs non transactionnels — une corruption peut survenir à la suite d'un incident matériel ou d'un arrêt brutal. MariaDB propose quatre opérations de maintenance pour garder les tables saines : `CHECK`, `REPAIR`, `ANALYZE` et `OPTIMIZE`. Cette section les présente, explique en quoi leur comportement dépend du moteur de stockage, et décrit les outils pour les exécuter ; les sous-sections détaillent ensuite chacune d'elles.

## Pourquoi entretenir les tables

Trois phénomènes justifient la maintenance. La **fragmentation** apparaît quand les suppressions et mises à jour laissent des trous dans les fichiers de données et d'index : l'espace disque est gaspillé et les parcours ralentissent ; `OPTIMIZE` réorganise la table pour récupérer cet espace. Les **statistiques périmées** sont la deuxième cause : après un chargement massif ou de nombreuses modifications, la distribution réelle des données ne correspond plus à ce que connaît l'optimiseur, qui peut alors retenir de mauvais plans d'exécution ; `ANALYZE` rafraîchit ces statistiques. Enfin, la **corruption** — rare mais possible, en particulier sur MyISAM — se détecte avec `CHECK` et, sur les moteurs concernés, se corrige avec `REPAIR`.

Il faut toutefois relativiser : le moteur par défaut, InnoDB, est transactionnel et largement auto-entretenu (statistiques mises à jour automatiquement, récupération après crash intégrée). La maintenance manuelle y est donc bien plus légère qu'à l'époque où MyISAM dominait, sans pour autant disparaître complètement.

## Les quatre opérations

| Opération | Rôle | Moteurs concernés | Détaillée en |
|-----------|------|-------------------|--------------|
| `CHECK TABLE` | Détecter erreurs et corruption | Tous (verrou en lecture) | §11.6.3 |
| `REPAIR TABLE` | Réparer une table corrompue | MyISAM, Aria, ARCHIVE, CSV — **pas InnoDB** | §11.6.3 |
| `ANALYZE TABLE` | Mettre à jour les statistiques de l'optimiseur | Tous | §11.6.2 |
| `OPTIMIZE TABLE` | Défragmenter et récupérer l'espace | Tous (sur InnoDB : reconstruction) | §11.6.1 |

`CHECK` est un outil d'investigation : il vérifie l'intégrité d'une table sans la modifier. `REPAIR` ne s'applique qu'aux moteurs non transactionnels (MyISAM, Aria, ARCHIVE, CSV) ; InnoDB ne se répare pas ainsi (voir plus loin). `ANALYZE` recalcule la distribution des clés pour guider l'optimiseur (§5.7, chapitre 15). `OPTIMIZE`, enfin, défragmente : sur MyISAM ou Aria, il réorganise physiquement les fichiers ; sur InnoDB, il est traduit en une reconstruction de la table (de type `ALTER TABLE … FORCE`) qui compacte l'index *clustered* et met à jour les statistiques.

## Deux façons de les exécuter

Ces opérations existent d'abord sous forme d'**instructions SQL**, exécutables directement :

```sql
CHECK    TABLE commandes;
ANALYZE  TABLE commandes;
OPTIMIZE TABLE commandes;
REPAIR   TABLE commandes;   -- moteurs non transactionnels uniquement
```

Elles sont aussi accessibles via l'utilitaire en ligne de commande **`mariadb-check`** (anciennement `mysqlcheck`, toujours accessible sous cet ancien nom). C'est une interface pratique aux quatre instructions : il choisit la bonne selon l'option (`-c` pour *check*, `-r` pour *repair*, `-a` pour *analyze*, `-o` pour *optimize*) et l'envoie au serveur — lequel doit donc être **démarré**.

```bash
# Vérifier toutes les tables d'une base, et réparer automatiquement si besoin
mariadb-check --check --auto-repair ma_base -u root -p
```

Deux limites à connaître : `mariadb-check` **ne fonctionne pas avec les tables partitionnées** (pour celles-ci, on passe par les instructions SQL, qui acceptent une clause `PARTITION`), et c'est lui qu'invoque `mariadb-upgrade` pour vérifier puis réparer les tables après une mise à niveau. Pour une intervention serveur **arrêté**, il existe par ailleurs des outils hors ligne propres à chaque moteur : `myisamchk` pour MyISAM et `aria_chk` pour Aria.

## Le rôle déterminant du moteur de stockage

Le comportement de chaque opération dépend fortement du moteur, ce qui explique l'essentiel des différences que l'on rencontrera dans les sous-sections.

**InnoDB** (le moteur par défaut) protège chaque page par une somme de contrôle CRC32 et détecte donc la corruption en cours de fonctionnement normal. Il suit une philosophie de *fail fast* : par défaut, lorsqu'une corruption est détectée — y compris par un `CHECK TABLE` —, InnoDB provoque délibérément l'arrêt du serveur pour empêcher sa propagation, en journalisant l'erreur. (Ce comportement n'est pas réglable dans MariaDB moderne : la variable `innodb_corrupt_table_action`, héritée de XtraDB, est obsolète et a été retirée — elle n'existe plus en 12.3.) Une table InnoDB **ne se répare pas** avec `REPAIR` : la récupération passe par la restauration d'une sauvegarde ou, en dernier recours, par un redémarrage en mode `innodb_force_recovery`. En contrepartie, InnoDB demande très peu de maintenance d'intégrité manuelle.

**MyISAM** et **Aria**, à l'inverse, ne sont pas protégés de la même manière et sont plus exposés à la corruption après un arrêt brutal ; c'est pour eux que le couple `CHECK` / `REPAIR` prend tout son sens, complété par les outils hors ligne mentionnés plus haut.

## Verrouillage, réplication et impact

Ces opérations ne sont pas anodines sur une base active : chaque table traitée est verrouillée et indisponible aux autres sessions le temps du traitement (un simple verrou en lecture pour `CHECK`), et l'opération peut être longue sur de grandes tables. La reconstruction d'une table InnoDB par `OPTIMIZE` est en grande partie réalisable en ligne, mais reste gourmande en ressources et en espace temporaire.

En contexte de réplication, ces instructions sont par défaut journalisées, donc rejouées sur les réplicas. Comme une table corrompue sur le primaire ne l'est généralement pas sur ses réplicas, on exécute fréquemment la maintenance serveur par serveur, en ajoutant le mot-clé `LOCAL` (ou `NO_WRITE_TO_BINLOG`) pour éviter de la propager.

En pratique, on planifie ces tâches pendant des fenêtres de maintenance ou des heures creuses, on sauvegarde toujours avant un `REPAIR`, et l'on surveille la fragmentation et l'espace disque (§11.7 et chapitre 16).

## Plan de la section

Les sous-sections détaillent successivement `OPTIMIZE TABLE` et la récupération d'espace (§11.6.1), `ANALYZE TABLE` et les statistiques de l'optimiseur (§11.6.2), puis `CHECK TABLE` et `REPAIR TABLE`, y compris leur prise en charge des objets `SEQUENCE` (§11.6.3). Pour les sujets connexes, on se reportera à l'analyse des plans d'exécution (§5.7), au *tuning* des performances (chapitre 15), aux moteurs de stockage (chapitre 7), à la gestion de l'espace disque (§11.7) et à la supervision (chapitre 16).

⏭️ [OPTIMIZE TABLE](/11-administration-configuration/06.1-optimize-table.md)
