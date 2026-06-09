🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.6 Hybrid cloud et multi-cloud

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La section §20.5 a traité de la répartition des données dans l'espace. Cette répartition pose immédiatement une question d'infrastructure : sur **quel** environnement faire reposer ces données ? Un seul fournisseur cloud, plusieurs fournisseurs, ou une combinaison de centres de données privés et de cloud public ?

C'est l'objet du **cloud hybride** et du **multi-cloud**. Ces modèles répondent à des préoccupations de coût, de résilience, de souveraineté et — surtout — d'indépendance vis-à-vis d'un fournisseur unique. Sur ce dernier point, MariaDB possède un atout structurel que cette section met en avant : sa **portabilité**.

---

## Définitions : hybride et multi-cloud

Deux notions voisines mais distinctes :

- **Cloud hybride** : combinaison d'une infrastructure **sur site** (centre de données privé) et de **cloud public**. La base de données, ou une partie de celle-ci, s'étend sur les deux.
- **Multi-cloud** : recours simultané à **plusieurs fournisseurs de cloud public** (par exemple AWS, Azure, GCP).

Les deux partagent une même logique : ne pas dépendre d'un environnement unique et répartir les charges sur des infrastructures hétérogènes. Elles se combinent d'ailleurs souvent (sur site + plusieurs clouds).

---

## Pourquoi adopter ces modèles

Plusieurs motivations conduisent à ces architectures :

- **Éviter l'enfermement propriétaire** (*vendor lock-in*) : ne pas être captif d'un fournisseur, de sa tarification et de ses services spécifiques.
- **Optimiser les coûts** : placer chaque charge là où elle est la moins chère, ou valoriser un investissement sur site existant.
- **Renforcer la résilience** : survivre à la panne d'un fournisseur entier en disposant d'une redondance au niveau du fournisseur lui-même.
- **Respecter la souveraineté des données** : conserver les données sensibles sur site ou chez un fournisseur précis (en lien avec §20.5).
- **Migrer progressivement** : l'hybride sert souvent d'**état transitoire** lors d'une migration du sur site vers le cloud.
- **Tirer parti du meilleur de chaque fournisseur** (*best-of-breed*) : utiliser les forces respectives de chacun.

---

## L'atout de MariaDB : la portabilité

C'est le point déterminant de cette section. De nombreuses bases de données managées du cloud sont **propriétaires** et indissociables de leur fournisseur : adopter l'une d'elles, c'est s'y enfermer, car migrer ailleurs implique de réécrire une partie de l'application.

MariaDB échappe à cette logique. **Logiciel libre**, il s'exécute de manière identique sur site, sur n'importe quel cloud (en auto-géré sur machines virtuelles ou conteneurs) et sur Kubernetes. Le **même moteur** tourne partout, avec les mêmes outils, la même syntaxe et le même comportement. Cette uniformité est précisément ce qui rend l'hybride et le multi-cloud naturels avec MariaDB :

- on peut **déplacer** une base d'un environnement à un autre sans la reconcevoir ;
- on peut **répliquer** entre environnements hétérogènes (voir plus loin) ;
- on conserve une **liberté de négociation** vis-à-vis des fournisseurs.

Des services **managés** de MariaDB existent par ailleurs (y compris l'offre cloud de MariaDB), pour qui souhaite déléguer l'exploitation ; mais c'est l'option **auto-gérée** qui offre la portabilité maximale, en s'affranchissant de toute dépendance à un fournisseur. C'est un avantage que les bases propriétaires ne peuvent, par construction, pas offrir.

---

## Conteneurs et Kubernetes : le socle de la portabilité

La portabilité du moteur se concrétise par la **conteneurisation** (chapitre 16). Empaqueter MariaDB dans un conteneur garantit un déploiement identique quel que soit l'environnement, et **Kubernetes** fournit un modèle d'orchestration commun à tous les clouds et au sur site.

- Le **mariadb-operator** (§16.5) automatise le provisionnement, la réplication et les sauvegardes de la même manière, quel que soit le substrat.
- L'**Infrastructure as Code** avec Terraform (§16.2.2), capable de cibler plusieurs fournisseurs, et le **GitOps** (§16.12) assurent une configuration cohérente et reproductible d'un environnement à l'autre.
- Les **StatefulSets** et **PersistentVolumes** (§16.4) abstraient le stockage sous-jacent, propre à chaque fournisseur.

Cet outillage est ce qui transforme la portabilité théorique du moteur en portabilité opérationnelle réelle.

---

## Relier les environnements : la réplication

Pour faire communiquer des environnements distincts, on s'appuie sur la **réplication** (chapitre 13), exactement comme pour la géo-distribution (§20.5) — car un lien sur site ↔ cloud ou cloud ↔ cloud est, par nature, un lien longue distance (WAN).

- La **réplication asynchrone** relie un primaire (sur site ou dans un cloud) à des réplicas situés dans d'autres environnements, avec les mêmes propriétés que celles vues en §20.5 : faible latence d'écriture, mais cohérence à terme.
- **Galera** s'y applique avec les mêmes réserves : sa nature synchrone le rend sensible à la latence WAN ; on privilégiera des **clusters par environnement reliés en asynchrone** (§13.11) plutôt qu'un cluster unique étendu.
- **MaxScale** (§14.4) route les requêtes vers le bon environnement (lecture locale, écriture vers le primaire).

Les patterns de cohérence et de gestion des conflits de §20.5 (notamment le **partitionnement des écritures** pour un actif-actif sans conflit) restent pleinement valables ici.

---

## Stockage portable et stockage objet

Le **stockage objet compatible S3** constitue une couche particulièrement adaptée à l'hybride et au multi-cloud, car cette interface est commune à de nombreux environnements : les stockages objets des grands clouds, mais aussi **MinIO** sur site ou ailleurs. MariaDB en tire parti à plusieurs niveaux :

- le **moteur S3** (§7.6) pour archiver des données froides en lecture seule sur un stockage objet ;
- le **stockage objet de ColumnStore** (§20.3), qui découple calcul et données ;
- les **sauvegardes vers le stockage objet** (§12.8), portables d'un environnement à l'autre.

S'appuyer sur une interface S3 standard, plutôt que sur un service de stockage propriétaire, préserve la portabilité jusque dans la couche de stockage.

---

## Les défis : réseau, coûts et sécurité

L'hybride et le multi-cloud ne sont pas sans contreparties, et certaines sont souvent sous-estimées :

- **Réseau** : la latence et la fiabilité des liens entre environnements conditionnent tout (mêmes contraintes WAN qu'en §20.5). Une mauvaise interconnexion ruine les bénéfices attendus.
- **Coûts de sortie de données** (*data egress*) : c'est le piège le plus fréquent. Les fournisseurs cloud facturent les données qui **quittent** leur réseau. Or répliquer entre clouds, ou entre cloud et sur site, génère un trafic sortant continu — et donc une facture d'egress qui peut devenir importante. Ce coût doit être anticipé dès la conception.
- **Sécurité** : faire transiter des données entre environnements élargit la surface d'attaque. Le **chiffrement des connexions** (TLS, chapitre 10.7) devient indispensable dès que le trafic traverse des réseaux non maîtrisés, de même que le chiffrement au repos.
- **Complexité opérationnelle** : maintenir une configuration, une sécurité et une supervision cohérentes sur des environnements hétérogènes demande de la rigueur et un outillage adapté.
- **Effet « plus petit dénominateur commun »** : pour rester portable, on tend à éviter les services managés spécifiques à un fournisseur, ce qui implique davantage d'auto-gestion. C'est le prix de l'indépendance.

---

## Patterns courants

En pratique, plusieurs montages reviennent :

- **Hybride pour la migration** : répliquer du sur site vers le cloud, valider, puis basculer — une voie vers une migration sans interruption (§19.8).
- **Hybride pour la reprise** : production sur site et réplica de secours dans le cloud (ou l'inverse), souvent plus économique qu'un second centre de données physique.
- **Hybride pour l'élasticité** : conserver une base de référence sur site et étendre la capacité de lecture vers le cloud lors des pics.
- **Multi-cloud pour la résilience** : actif dans deux clouds, reliés en asynchrone, avec partitionnement des écritures par région pour éviter les conflits.
- **Multi-cloud pour la souveraineté ou le best-of-breed** : placer certaines données chez un fournisseur précis, ou exploiter les services propres à chacun.

---

## Synthèse

La portabilité de MariaDB — moteur libre, identique partout, orchestrable par conteneurs et Kubernetes — en fait un candidat naturel pour l'hybride et le multi-cloud, là où les bases propriétaires enferment. Les mécanismes sont ceux déjà rencontrés (réplication, Galera, MaxScale, stockage objet), appliqués à des environnements hétérogènes plutôt qu'à de simples régions.

Le choix relève d'un arbitrage entre **indépendance et coût/complexité** : éviter l'enfermement et gagner en résilience, mais au prix d'une exploitation plus exigeante, de coûts de sortie de données et de contraintes réseau. Une fois l'infrastructure choisie, reste à décider **comment** faire croître la capacité de la base — par la montée en puissance d'un nœud ou par la multiplication des nœuds. C'est la question du **passage à l'échelle vertical et horizontal**, qu'aborde la section suivante (§20.7).

---

## Navigation

⬅️ Section précédente : [20.5 Géo-distribution](05-geo-distribution.md)  
➡️ Section suivante : [20.7 Scaling vertical vs horizontal](07-scaling-vertical-horizontal.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Scaling vertical vs horizontal](/20-cas-usage-architectures/07-scaling-vertical-horizontal.md)
