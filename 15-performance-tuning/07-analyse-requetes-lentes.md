🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.7 Analyse des requêtes lentes

C'est souvent dans l'optimisation des requêtes — bien plus que dans le réglage des paramètres du serveur — que se trouvent les gains de performance les plus importants. Sur la plupart des bases, une **poignée de requêtes concentre l'essentiel de la charge** : les identifier puis les corriger produit un effet disproportionné. Cette section pose la méthode et le panorama des outils permettant de trouver et d'analyser ces requêtes. Les deux moyens principaux sont ensuite détaillés dans les sous-sections : le *slow query log* (§15.7.1) et `pt-query-digest` (§15.7.2).

---

## Pourquoi analyser les requêtes lentes

Le principe est conforme à la méthodologie générale d'optimisation (§15.1) : on mesure, on identifie le goulot d'étranglement, puis on corrige le point au plus fort impact. Or, à matériel et configuration équivalents, c'est presque toujours une requête mal indexée ou mal écrite qui domine la consommation de ressources. Une seule requête correctement optimisée peut réduire la charge du serveur davantage que des heures passées à ajuster des variables système.

---

## « Lente » ou « coûteuse » ? La distinction décisive

C'est le concept le plus important de cette section. Il faut distinguer :

- une requête **lente**, dont une exécution isolée prend beaucoup de temps (forte latence) ;
- une requête **coûteuse**, dont le coût *agrégé* est élevé — c'est-à-dire **fréquence × latence**.

Une requête qui dure 0,5 seconde mais qui est exécutée des milliers de fois par minute consomme bien plus de ressources qu'une requête qui dure 5 secondes mais n'est lancée qu'une fois par heure. La conséquence pratique est essentielle : il ne faut pas seulement traquer les requêtes les plus lentes individuellement, mais **classer les requêtes par temps total consommé** (somme des latences sur toutes leurs exécutions).

C'est aussi pourquoi un *slow query log* configuré avec un seuil élevé peut induire en erreur : il ne capture que les requêtes dépassant ce seuil et **ignore les requêtes rapides mais très fréquentes**, qui sont pourtant souvent les vrais responsables. D'où l'intérêt d'agréger les requêtes par **« empreinte » (*digest*)** — leur forme normalisée, valeurs littérales retirées —, ce que font à la fois `pt-query-digest` et le Performance Schema.

---

## La démarche : un cycle itératif

L'analyse des requêtes lentes suit un cycle que l'on répète :

1. **Identifier** les requêtes les plus coûteuses (par temps total consommé).
2. **Analyser** pourquoi elles le sont, notamment via leur plan d'exécution (EXPLAIN / EXPLAIN ANALYZE, cf. §5.7).
3. **Optimiser** : ajout ou révision d'un index (cf. §5.5), réécriture de la requête, ou ajustement du schéma.
4. **Vérifier** en mesurant avant et après.

Comme l'optimisation du principal consommateur déplace le goulot d'étranglement vers le suivant, on recommence le cycle jusqu'à atteindre un niveau de performance satisfaisant.

---

## Les signaux à observer

Quelques indicateurs trahissent une requête problématique et orientent vers sa cause :

- le **temps d'exécution** et sa distribution (moyenne, mais aussi p95/p99 et maximum, qui révèlent les pics) ;
- la **fréquence d'exécution**, indispensable pour calculer le coût agrégé ;
- le rapport **lignes examinées / lignes renvoyées** : c'est le signal le plus parlant. Une requête qui parcourt des millions de lignes pour n'en retourner qu'une poignée souffre presque toujours d'une **mauvaise sélectivité** — index manquant ou balayage complet de table ;
- le **temps de verrouillage** (*lock time*), révélateur de contention ;
- l'usage de **tables temporaires** (surtout sur disque) et de **tris de fichiers** (*filesort*) ;
- les **balayages complets** de table ou d'index.

Ces signaux pointent vers le levier de correction approprié (indexation, réécriture, schéma) et se confirment ensuite à l'aide d'`EXPLAIN` (§5.7).

---

## Où trouver les requêtes : le panorama des outils

Plusieurs sources complémentaires permettent de repérer les requêtes coûteuses ; elles se combinent plutôt qu'elles ne se remplacent.

- **Le *slow query log*** (§15.7.1) est le journal historique, présent partout : un fichier texte recensant les requêtes dont la durée dépasse un seuil (`long_query_time`), et éventuellement celles n'utilisant pas d'index. Classique et universel, il dépend toutefois du seuil choisi et engendre des E/S disque lorsque ce seuil est très bas. Brut, il s'exploite avec des outils d'analyse.

- **Les analyseurs de journal** transforment ce log en rapport synthétique : `mariadb-dumpslow` (fourni avec le serveur) pour un résumé rapide, et surtout `pt-query-digest` du Percona Toolkit (§15.7.2) pour une analyse agrégée avancée, classée par temps total et par empreinte de requête.

- **Le Performance Schema et le sys schema** (§15.8, et §9.7.2) offrent une approche en mémoire, sans fichier ni surcoût d'E/S, et qui capture **toutes** les requêtes — y compris les rapides. La table `events_statements_summary_by_digest` agrège les statistiques par empreinte (nombre d'exécutions, temps total/moyen/maximum, temps de verrouillage, lignes examinées et renvoyées…), ce qui en fait l'outil idéal pour un classement par coût agrégé. La vue `sys.statement_analysis` en présente une version directement lisible.

- **`EXPLAIN` et `EXPLAIN ANALYZE`** (§5.7) interviennent une fois la requête fautive identifiée, pour comprendre son plan d'exécution : utilisation des index, type de balayage, ordre des jointures, et écart entre lignes estimées et réelles.

- **Les requêtes en cours** s'observent en temps réel via `SHOW [FULL] PROCESSLIST` ou la table `performance_schema.events_statements_current`, utiles pour repérer une requête bloquée ou anormalement longue à l'instant *t*.

En résumé, le *slow query log* et ses analyseurs offrent une vue **rétrospective sur fichier**, le Performance Schema et le sys schema une **agrégation en mémoire** continue, et `EXPLAIN` une **analyse fine** requête par requête.

---

## Au-delà de l'analyse : prévenir les requêtes incontrôlées

L'analyse vise à corriger les requêtes coûteuses, mais MariaDB permet aussi de **borner** une requête trop gourmande : l'indication d'optimisation `MAX_EXECUTION_TIME` (§15.15.5) limite la durée d'exécution d'un `SELECT`, empêchant une requête emballée de monopoliser les ressources. C'est un complément utile à la démarche d'analyse présentée ici.

---

## Feuille de route de la section

Les sous-sections suivantes détaillent les deux outils centraux de l'analyse :

- **§15.7.1 — *Slow query log*** : sa configuration (seuil, options de journalisation, échantillonnage) et sa lecture.
- **§15.7.2 — `pt-query-digest`** : l'analyse agrégée et avancée des journaux de requêtes lentes.

⏭️ [Slow query log](/15-performance-tuning/07.1-slow-query-log.md)
