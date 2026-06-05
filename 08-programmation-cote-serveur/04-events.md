🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.4 Events (tâches planifiées)

Un *event* (événement planifié) est une tâche nommée, stockée dans la base, que le serveur exécute **automatiquement à des moments définis**. C'est l'équivalent d'un planificateur de tâches — un *cron* — intégré au SGBD : on lui confie un traitement SQL et un calendrier, et il s'exécute seul, sans intervention ni ordonnanceur externe.

## Une routine déclenchée par le temps

Les events complètent la famille des routines côté serveur selon leur mode de déclenchement. Une procédure s'exécute sur appel explicite (`CALL`) ; un trigger réagit à une modification de données ; un event, lui, est déclenché par le **temps**. C'est la bonne réponse chaque fois qu'un traitement doit se répéter à intervalle régulier ou survenir à une échéance précise, indépendamment de toute requête utilisateur.

## Ponctuelles ou récurrentes

Un event suit l'un de deux modes de planification. Il peut être **ponctuel** — exécuté une seule fois, à un instant donné — ou **récurrent** — exécuté à intervalle régulier (toutes les heures, chaque nuit, chaque semaine…), éventuellement borné par une date de début et de fin. La syntaxe précise de cette planification est détaillée en 8.4.1.

## À quoi servent les events ?

Les events brillent dans les tâches de maintenance et d'automatisation récurrentes. Les usages les plus fréquents sont la **purge de données** (suppression des lignes périmées : sessions expirées, jetons, journaux anciens), l'**archivage** vers des tables d'historique ou un stockage froid, le **rafraîchissement de données agrégées** (recalcul périodique d'une table de synthèse, à la manière d'une vue matérialisée que MariaDB ne propose pas nativement), ou encore la production de **rapports et instantanés** à heure fixe. Tout ce qu'on confierait à une tâche planifiée du système peut, lorsque c'est pertinent, vivre directement dans la base sous forme d'event — et voyager avec elle.

## Le planificateur, la réplication et autres précautions

Plusieurs particularités conditionnent le bon fonctionnement des events. La plus importante : un event ne s'exécute que si le **planificateur d'événements** (*Event Scheduler*) est activé. Ce thread de fond, contrôlé par la variable `event_scheduler`, est souvent désactivé par défaut — un event défini mais oublié reste alors inerte, sans erreur ni avertissement. Le planificateur fait l'objet de la section 8.4.2.

Par ailleurs, un event dépend de la disponibilité du serveur : si celui-ci est arrêté au moment prévu, l'exécution est manquée et n'est pas rattrapée rétroactivement. Enfin, en contexte de **réplication**, les events ne s'exécutent que sur le serveur primaire ; sur les réplicas, ils sont automatiquement désactivés (statut `SLAVESIDE_DISABLED`) afin d'éviter une double exécution, les effets du traitement étant déjà propagés par la réplication.

## Création, gestion et sécurité

Un event appartient à une base. On le crée avec `CREATE EVENT`, on le modifie avec `ALTER EVENT` (pour changer son calendrier, l'activer ou le désactiver) et on le supprime avec `DROP EVENT` ; la syntaxe est introduite en 8.4.1. Les events existants se consultent avec `SHOW EVENTS` ou via la vue `INFORMATION_SCHEMA.EVENTS`, et `SHOW CREATE EVENT` en restitue la définition. Leur création requiert le privilège `EVENT` sur la base concernée et, comme les autres routines, un event possède un définisseur (`DEFINER`) qui détermine le contexte de privilèges de son exécution.

## Plan de la section

- **8.4.1 — `CREATE EVENT` et planification** : la définition d'un event et l'expression de son calendrier (exécution ponctuelle `AT`, récurrente `EVERY`, bornes `STARTS` / `ENDS`).
- **8.4.2 — Event Scheduler** : le planificateur de fond qui exécute les events, son activation et sa supervision.

---

Commençons par définir un event et son calendrier : **[`CREATE EVENT` et planification](04.1-create-event.md)**.

⏭️ [CREATE EVENT et planification](/08-programmation-cote-serveur/04.1-create-event.md)
