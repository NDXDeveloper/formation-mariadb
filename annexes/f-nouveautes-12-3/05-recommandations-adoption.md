🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.5 — Recommandations d'adoption

Cette page de clôture transforme l'inventaire des sections précédentes en guide de décision : faut-il passer à la 12.3, selon quel profil, à quel moment et avec quelle méthode. Il s'agit d'un cadre à adapter à votre contexte plutôt que de règles absolues ; les détails d'une migration donnée sont traités au chapitre 19 (détail des versions et de leurs échéances en [Annexe G](../g-versions-reference/README.md)).

## La 12.3 comme socle recommandé

Pour le cas général, la 12.3 est le choix par défaut. C'est la **LTS courante** (GA 12.3.2, supportée jusqu'en juin 2029), qui consolide la série *rolling* 12.0 → 12.2 en un socle stable. Elle apporte des gains tangibles — performance en écriture et contrôle de l'optimiseur ([F.2](02-features-phares.md)), compatibilité renforcée Oracle et MySQL ([F.3](03-compatibilite-oracle-mysql.md)) — pour un nombre limité de changements de comportement à anticiper ([F.4](04-impact-migration-compatibilite.md)). Pour tout nouveau déploiement, c'est la version naturelle.

## Adopter selon votre point de départ

La décision dépend largement de la version dont vous partez :

- **Nouveau déploiement** — choisir directement la 12.3 : LTS courante, fenêtre de support la plus longue (juin 2029) et ensemble des acquis 12.x.
- **Depuis la 11.8 LTS** (support 2028) — aucune urgence, mais la 12.3 apporte des gains réels et une année de support supplémentaire. Planifier une migration à un rythme maîtrisé, en la priorisant si le débit en écriture ou la compatibilité Oracle/MySQL sont des enjeux.
- **Depuis une LTS plus ancienne** — la 10.6 (fin de support 2026) appelle une migration sans tarder ; la 10.11 (2028) dispose d'un peu de marge. La 11.4 partage l'échéance 2029 de la 12.3 : le passage se justifie alors par les fonctionnalités, non par la fenêtre de support.
- **Depuis une *rolling* 12.x** — basculer vers la 12.3 LTS pour bénéficier des garanties de support, les versions *rolling* n'étant maintenues que jusqu'à la version suivante.

## Calendrier : quand franchir le pas

La 12.3 est récente — GA 12.3.2 fin mai 2026 (la 12.3.1 ayant servi de *release candidate*). Elle est donc *production-ready*, mais comme pour toute nouvelle LTS, les systèmes les plus critiques gagnent à être éprouvés d'abord en pré-production, voire à attendre une première version de maintenance pour la posture la plus prudente.

Tant que vous êtes sur une LTS encore supportée, il n'y a pas d'urgence : le bon moment est dicté par vos besoins — gains de performance, compatibilité, fenêtre de support — plutôt que par une échéance imminente. La principale exception est la 10.6, en fin de support, pour laquelle la migration ne doit pas attendre.

## Méthode et précautions

La démarche recommandée tient en quatre temps :

1. **Recenser les changements de comportement** ([F.4](04-impact-migration-compatibilite.md)) qui touchent votre installation : variables système retirées, noms de contraintes FK, packaging Galera, etc.
2. **Valider en pré-production** avec une charge représentative.
3. **Suivre la procédure de migration** ([§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md)), choisir un chemin de mise à jour — en place ou logique ([§19.4](../../19-migration-compatibilite/04-strategies-mise-a-jour.md)) — et préparer un plan de repli ([§19.7](../../19-migration-compatibilite/07-rollback-contingence.md)).
4. **Adopter les nouveautés optionnelles ensuite**, une fois le socle stable.

Pour ce dernier point, procéder par paliers. Le **binlog InnoDB** offre le gain d'écriture le plus net, mais ne l'activez qu'après avoir validé les outils qui consomment le binlog (réplication, *Change Data Capture*, sauvegarde). Les **optimizer hints** s'emploient de façon ciblée sur les requêtes problématiques, documentés et re-testés (cf. [F.2](02-features-phares.md) et l'audit de requêtes, Annexe E.3). Les **fonctionnalités de compatibilité** (mode Oracle, `caching_sha2_password`) s'activent selon la source de migration. Cette approche garde la mise à jour du socle à faible risque tout en capturant les gains au fil de l'eau.

## Et après : la série 13.x

Pour la production, privilégier la stabilité de la LTS : la 12.3 couvre vos besoins jusqu'en juin 2029. La série **13.x** ouvre le cycle suivant : sa première version, **13.0**, est en préversion puis version candidate (RC) à la date de rédaction, sa GA n'étant pas encore parue. Ce canal *rolling* s'adresse à ceux qui veulent les nouveautés rapidement, au prix de mises à jour fréquentes et d'un support limité à la version *rolling* suivante. Il est utile d'en suivre l'évolution pour anticiper la prochaine LTS, sans pour autant précipiter un passage en *rolling* sur des systèmes de production. La stratégie LTS contre *rolling* est détaillée aux §1.5 à §1.7 et au §19.3.

## En résumé

- La 12.3 LTS est le **socle recommandé** : choix par défaut des nouveaux déploiements et cible de migration.
- Sur 11.8, **planifier une migration** à rythme maîtrisé, pour les gains et l'année de support supplémentaire.
- **Tester les changements de comportement** ([F.4](04-impact-migration-compatibilite.md)) et migrer via le [§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md), plan de repli à l'appui.
- **Activer progressivement** les nouveautés optionnelles (binlog InnoDB, optimizer hints, compatibilité).
- **Garder la production sur LTS** et surveiller la 13.x sans s'y précipiter.

⏭️ [Versions de Référence](/annexes/g-versions-reference/README.md)
