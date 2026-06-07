🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.11 Sharding et distribution horizontale

Le partitionnement, étudié aux §15.9 et §15.10, découpe une table en plusieurs morceaux **au sein d'un même serveur**. Le sharding pousse cette logique d'un cran plus loin : il distribue les données entre **plusieurs serveurs indépendants**, chacun détenant un sous-ensemble des lignes. L'ensemble du jeu de données est alors la réunion de tous les fragments — les *shards* —, et aucun serveur ne contient la totalité. C'est le levier ultime de mise à l'échelle horizontale, capable de dépasser les limites d'une seule machine en volume comme en débit d'écriture ; c'est aussi, de loin, le plus complexe à mettre en œuvre et à exploiter.

## Sharding, partitionnement et réplication : trois leviers distincts

Avant d'entrer dans le sharding proprement dit, il importe de le distinguer nettement de deux autres mécanismes auxquels on l'amalgame souvent, et qui répondent à des besoins différents.

| Mécanisme | Ce qu'il distribue | Ce qu'il fait monter en charge | Transparence |
|-----------|--------------------|--------------------------------|--------------|
| **Partitionnement** (§15.9-15.10) | Des morceaux d'une table sur **un seul serveur** | La gérabilité et l'élagage à la lecture | Total : géré par le moteur |
| **Réplication** (§13) | Des **copies identiques** sur plusieurs serveurs | Les **lectures** et la disponibilité | Géré par le serveur |
| **Sharding** | Des **sous-ensembles différents** sur plusieurs serveurs | Les **écritures** et le **volume total** | À la charge de l'application ou d'un intermédiaire |

La distinction essentielle est la suivante : la réplication multiplie les copies d'un même jeu de données — elle soulage les lectures et assure la résilience, mais n'augmente ni la capacité d'écriture ni le stockage total. Le sharding, lui, répartit des données **différentes** sur chaque nœud : il augmente à la fois la capacité d'écriture et le volume adressable, au prix d'une logique de routage que ni le serveur ni le moteur ne fournissent gratuitement.

## Quand envisager le sharding — et que faire avant

Le sharding doit être considéré comme un **dernier recours**, en raison de la complexité opérationnelle qu'il introduit. Avant d'y recourir, il convient d'épuiser les leviers plus simples : la mise à l'échelle verticale (un serveur plus puissant), les réplicas en lecture pour absorber la charge de lecture (§13), le partitionnement pour maîtriser la taille et bénéficier de l'élagage (§15.9), la mise en cache applicative, et bien sûr l'optimisation des requêtes et de l'indexation.

Le sharding ne se justifie que lorsque ces leviers sont épuisés et que l'un des deux seuils physiques est atteint : un seul serveur ne peut plus **contenir** le volume de données, ou son débit d'**écriture** dépasse ce qu'une seule machine peut soutenir. Tant que ce n'est pas le cas, la complexité du sharding l'emporte sur ses bénéfices.

## Les stratégies de sharding

La façon dont une ligne est affectée à un shard repose, comme pour le partitionnement, sur une **clé de sharding** et une stratégie de répartition. On retrouve d'ailleurs un parallèle direct avec les méthodes de partitionnement.

Le **sharding par intervalles** (range-based) répartit les lignes selon des plages de valeurs de la clé — par exemple les identifiants de 1 à 1 million sur le shard A, de 1 million à 2 millions sur le shard B. Il rend les requêtes par plage efficaces et facilite l'ajout de shards en fin de plage, mais expose aux points chauds : avec une clé séquentielle, les écritures récentes se concentrent toutes sur le dernier shard.

Le **sharding par hachage** (hash-based) affecte chaque ligne au shard `hash(clé) mod N`. Il assure une répartition uniforme et évite les points chauds, mais rend les requêtes par plage inefficaces (elles touchent tous les shards) et complique fortement le re-sharding : changer le nombre de shards `N` impose de re-hacher et de redéplacer une grande partie des données.

Le **sharding par annuaire** (directory-based) s'appuie sur une table de correspondance qui associe explicitement chaque clé — ou chaque locataire — à un shard. Très souple, il facilite le rééquilibrage, mais transforme cette table de correspondance en dépendance critique qui doit être hautement disponible et performante.

Le **sharding géographique**, enfin, distribue les données par région pour des raisons de localité ou de conformité réglementaire — un sujet repris au §20.5 (géo-distribution).

## Le choix de la clé de sharding

Le choix de la clé de sharding est la décision la plus lourde de conséquences de toute l'architecture, car il est extrêmement coûteux à revoir une fois les données distribuées. Une bonne clé présente une **forte cardinalité** et une **distribution uniforme**, pour éviter les déséquilibres ; elle est **présente dans la majorité des requêtes**, afin que celles-ci puissent être routées vers un seul shard ; et elle est **stable**, car modifier la valeur de la clé d'une ligne reviendrait à la déplacer d'un shard à l'autre.

Une clé mal choisie a des effets en cascade : une clé de faible cardinalité ou monotone provoque des points chauds, tandis qu'une clé absente des requêtes courantes force des interrogations sur tous les shards. En pratique, on shardera par exemple sur l'identifiant de locataire dans une application multi-tenant, ou sur l'identifiant d'utilisateur dans une application sociale — des clés naturellement présentes dans presque toutes les requêtes.

## Les défis du sharding

Le sharding introduit une série de difficultés qui n'existent pas sur une base monolithique, et qu'il faut anticiper.

Les **requêtes inter-shards** sont le premier écueil : une requête qui ne mentionne pas la clé de sharding doit être diffusée à tous les shards puis agrégée (*scatter-gather*), opération coûteuse qui annule une partie du bénéfice de la distribution. Les **jointures inter-shards** sont plus délicates encore : joindre des données réparties sur des serveurs différents est difficile, et l'on cherche généralement à l'éviter en **co-localisant** les données liées sur un même shard (même clé de sharding) ou en dénormalisant.

Les **transactions distribuées** posent un problème de cohérence : une transaction couvrant plusieurs shards exige un protocole de validation en deux phases (XA, §6.8), plus lent et exposé à davantage de modes de défaillance ; beaucoup d'architectures s'imposent donc de ne jamais transiger à cheval sur plusieurs shards. L'**intégrité référentielle** ne peut pas davantage être garantie par le serveur entre shards : les clés étrangères ne franchissent pas les frontières, et leur vérification retombe au niveau applicatif.

Le **rééquilibrage** (re-sharding) — déplacer des données lors de l'ajout ou du retrait d'un shard, ou pour soulager un shard surchargé — est l'une des opérations les plus complexes, souvent synonyme de migration soigneusement orchestrée. Enfin, la **complexité opérationnelle** se diffuse partout : sauvegardes, supervision et changements de schéma doivent désormais être coordonnés sur l'ensemble des shards, et même une répartition uniforme des données n'exclut pas les **points chauds** en charge.

## Les approches dans l'écosystème MariaDB

MariaDB ne propose pas de solution de sharding « clé en main » entièrement automatisée comparable à certains systèmes distribués spécialisés ; il offre en revanche plusieurs approches, du tout-applicatif au transparent.

### Sharding applicatif

L'approche la plus répandue pour les grands systèmes sur mesure consiste à confier le routage à l'**application** elle-même — ou à une bibliothèque de sharding —, qui détermine le shard cible à partir de la clé. Elle s'appuie généralement sur une base de métadonnées (annuaire) enregistrant la correspondance clé → shard, assortie d'un mécanisme de verrouillage pour le rééquilibrage. Cette voie exige de bâtir la logique de routage, un mécanisme de re-sharding et une supervision par shard ; elle offre le contrôle maximal, au prix d'une responsabilité maximale.

### Moteur Spider : sharding transparent au niveau SQL

Le moteur **Spider** (§7.10.3) est un moteur de stockage doté de fonctionnalités de sharding intégrées. Une table Spider ne stocke pas elle-même les données : elle **établit un lien** vers une table sur un serveur MariaDB distant, cette table distante pouvant utiliser n'importe quel moteur. Spider permet ainsi de manipuler des tables réparties sur plusieurs instances comme si elles résidaient sur la même ; il s'agit d'une implémentation de la norme SQL/MED (ISO/IEC 9075-9) relative à l'accès aux données externes.

```sql
-- Illustration simplifiée : une table Spider reliée à un serveur distant
CREATE TABLE commandes (
    id      BIGINT,
    montant DECIMAL(10,2),
    PRIMARY KEY (id)
) ENGINE = SPIDER
COMMENT = 'host "192.168.0.2", user "spider", port "3306"';
```

Pour réaliser un véritable sharding, on combine Spider avec le partitionnement : chaque partition de la table Spider pointe vers un serveur dorsal différent, de sorte que les lignes sont physiquement distribuées entre les nœuds tout en restant accessibles via une table unique. L'atout déterminant de Spider par rapport aux solutions de routage est qu'il **autorise les jointures inter-shards** et prend en charge les **transactions XA** pour la cohérence distribuée. En contrepartie, il introduit une surcharge de performance et une configuration non triviale.

### MaxScale schemarouter : sharding par schéma

MaxScale (§14.4) propose un sharding **par schéma** au moyen de son module *schemarouter* : chaque schéma (base de données) réside sur un serveur distinct, et le schemarouter les agrège derrière un point d'accès logique unique. Les requêtes sont routées d'après la base de données qu'elles ciblent, et l'état de session est préservé lors du passage d'un shard à l'autre, ce qui rend la bascule transparente pour le client.

Cette approche est particulièrement adaptée aux architectures multi-tenant en *database-per-tenant* (§20.4). Sa limite principale est l'**impossibilité des jointures inter-shards** — une contrainte généralement acceptable en contexte multi-tenant, où chaque locataire est isolé ; si de telles jointures sont nécessaires, c'est vers Spider qu'il faut se tourner. À la différence du sharding applicatif, le schemarouter est transparent pour l'application, qui ne voit qu'un seul serveur.

### ProxySQL et autres proxies

ProxySQL (§14.9) peut également implémenter une forme de sharding par le biais de **règles de routage de requêtes** — routage par schéma, par motif de requête ou par indication explicite. Il s'agit d'une approche au niveau de la couche proxy, transparente pour l'application en ce qui concerne le routage.

### ColumnStore : un cas distinct de distribution

Il convient enfin de ne pas confondre le sharding transactionnel avec la distribution analytique de **ColumnStore** (§7.5, §20.3). ColumnStore répartit des données en colonnes sur plusieurs modules pour la montée en charge analytique (OLAP) ; c'est une distribution au service de l'analyse de gros volumes, et non un sharding OLTP. Les deux relèvent de la distribution horizontale, mais répondent à des usages opposés.

## Sharding ou alternatives : synthèse

Le choix entre ces mécanismes se résume à la nature du goulet d'étranglement. Pour absorber un surcroît de **lectures**, les réplicas (§13) sont la réponse la plus simple. Pour maîtriser la **taille** et accélérer les requêtes ciblées, le partitionnement (§15.9) suffit le plus souvent et reste cantonné à un seul serveur. Le sharding ne s'impose que pour franchir les limites d'**écriture** et de **stockage** d'un nœud unique — et son coût en complexité doit être justifié par un besoin réel et démontré, conformément aux arbitrages de mise à l'échelle examinés au §20.7.

## En conclusion

Le sharding constitue la frontière de la mise à l'échelle horizontale : il échange la simplicité d'exploitation contre une capacité quasi illimitée. Pour la grande majorité des situations où c'est le volume qui pose problème, le partitionnement étudié dans les sections précédentes demeure la réponse la plus raisonnable sur un nœud unique ; le sharding ne devient pertinent que lorsqu'un seul serveur est véritablement insuffisant. Quelle que soit la stratégie retenue, son intérêt réel ne se présume pas : il se **mesure**. C'est précisément l'objet du §15.12 (Benchmarking), qui fournit les outils pour quantifier l'effet des optimisations et des choix d'architecture présentés tout au long de ce chapitre.

⏭️ [Benchmarking](/15-performance-tuning/12-benchmarking.md)
