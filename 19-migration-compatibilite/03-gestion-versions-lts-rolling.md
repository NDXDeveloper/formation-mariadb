🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.3 Gestion des versions : Stratégie LTS vs Rolling

> **Chapitre 19 — Migration et Compatibilité** · Section 19.3  
> Niveau : DBA / Architecte

---

## Un choix stratégique, pas seulement une version

Migrer vers MariaDB — ou planifier son usage sur la durée — ne consiste pas seulement à choisir un numéro de version, mais aussi un **type de publication** : **LTS** (Long Term Support) ou **Rolling**. Ce choix conditionne pour plusieurs années le **rythme des mises à jour**, le **niveau de stabilité** et l'**accès aux nouvelles fonctionnalités**. C'est donc une décision d'architecture autant que d'exploitation, qu'il vaut mieux prendre en amont de toute migration.

Cette section présente le modèle de publication de MariaDB et les critères pour arbitrer entre les deux canaux. Les mécanismes concrets de mise à jour, eux, font l'objet de la section suivante (19.4).

---

## Le modèle de publication de MariaDB

MariaDB Community Server publie une nouvelle version **environ quatre fois par an** (tous les trois mois). Le rythme combine un cycle **annuel** (les versions LTS) et un cycle **trimestriel** (les versions Rolling). Il existe deux types de versions **GA** (*Generally Available*, c'est-à-dire prêtes pour la production) :

- les **versions Rolling** (`X.0`, `X.1`, `X.2`) apportent de nouvelles fonctionnalités à chaque trimestre ;
- la **version LTS** (`X.3`) conclut le cycle d'une série majeure et privilégie la stabilité.

Pour la série 12, par exemple, les versions **12.0, 12.1 et 12.2** sont des publications Rolling, et **12.3 est la LTS** qui clôt la série ; la suivante ouvrira la série **13.x**. La règle se résume ainsi : la version `.3` de chaque version majeure est une LTS.

Au sein d'une version, le cheminement de pré-publication suit un schéma régulier : `X.Y.0` est une **préversion** (maturité alpha, destinée à montrer les fonctionnalités à venir), `X.Y.1` une **version candidate** (RC), et `X.Y.2` la **première GA**. C'est précisément le cas de la 12.3 : la 12.3.1 était une RC, et la **12.3.2** constitue la GA (fin mai 2026).

---

## Le canal LTS (Long Term Support)

Une version LTS est un **instantané consolidé** d'un cycle de publication complet. Une fois publiée, elle ne reçoit plus que des **corrections de bogues, des correctifs de sécurité et des améliorations de stabilité critiques** : son code « ne bouge plus » sous les pieds de l'application. C'est la **stabilité plutôt que la vélocité**.

Côté durée de maintenance, le modèle a évolué : les versions LTS communautaires sont désormais maintenues **trois ans** après leur GA (depuis la **11.8**). Les LTS antérieures, comme la **10.11** et la **11.4**, conservaient une maintenance de **cinq ans**. Une **souscription Enterprise** prolonge par ailleurs ce support (deux années supplémentaires, jusqu'à cinq avec l'option étendue).

Ce canal s'adresse aux organisations qui ont besoin de **fenêtres de maintenance prévisibles** et d'un socle figé : production, applications critiques, environnements où l'on ne souhaite pas absorber de nouvelles fonctionnalités à chaque trimestre. C'est le choix le plus courant en entreprise — et la raison pour laquelle cette formation prend **12.3 LTS** comme socle de référence (supportée jusqu'en juin 2029). Pour le détail des versions et de leurs échéances, voir l'Annexe G.

La contrepartie est qu'on ne bénéficie des nouveautés qu'à la **LTS suivante** (rythme annuel), et non en continu.

---

## Le canal Rolling (Rolling GA)

Une version Rolling est une publication GA trimestrielle qui livre **les nouvelles fonctionnalités au fil de l'eau**, avec une qualité de production. Son modèle de support est radicalement différent de celui des LTS :

- elle **ne reçoit aucun correctif après sa GA** (la version GA est finale) ;
- elle n'est **supportée que jusqu'à la publication de la version Rolling suivante**, soit environ trois mois.

Concrètement, pour obtenir un correctif, on ne reçoit pas de version de maintenance : on **migre vers la version Rolling suivante**, qui combine corrections et nouvelles fonctionnalités. Ce canal convient aux équipes qui veulent **les dernières capacités dès leur sortie** et qui sont en mesure de **mettre à jour fréquemment**, en acceptant un changement permanent.

La contrepartie est un **rythme de mise à jour exigeant** et une **fenêtre de support très courte** par version : ce n'est pas un canal adapté à un déploiement de type « installer et oublier ».

---

## Comparaison synthétique

| Critère | LTS (`X.3`) | Rolling (`X.0`/`X.1`/`X.2`) |
|---|---|---|
| Cadence | Annuelle | Trimestrielle |
| Objectif | Stabilité | Nouvelles fonctionnalités |
| Correctifs après GA | Oui (bogues, sécurité, stabilité) | **Aucun** |
| Durée de support | **3 ans** (communautaire, depuis 11.8) | Jusqu'à la version Rolling suivante (~3 mois) |
| Pour obtenir un correctif | Version de maintenance | Migrer vers la Rolling suivante |
| Public visé | Production, applications critiques | Équipes orientées fonctionnalités, mises à jour fréquentes |
| Charge de mise à jour | Faible et planifiable | Élevée et continue |

---

## Choisir sa stratégie

Le bon arbitrage dépend du contexte :

- **Pour la plupart des cibles de production et de migration**, le canal **LTS** s'impose : stabilité, fenêtre de maintenance de trois ans, mises à jour planifiables. C'est le choix par défaut pour un système critique — et la cible recommandée d'une migration (par exemple **12.3 LTS**).
- **Pour le développement, les tests ou les équipes pilotées par les fonctionnalités**, le canal **Rolling** offre l'accès anticipé aux nouveautés, à condition de mettre en place un **processus de mise à jour trimestriel**.
- Une **approche hybride** est souvent pertinente : exploiter les versions Rolling en développement ou en pré-production pour évaluer les fonctionnalités à venir, tout en maintenant une **LTS en production**.

Plusieurs critères guident la décision : la **durée de support** au regard du cycle de vie du projet, la **tolérance au changement**, la **capacité opérationnelle** à absorber des mises à jour fréquentes, et le **besoin éventuel d'une fonctionnalité récente** disponible uniquement dans une version Rolling.

Un point important : **choisir une LTS ne signifie pas ne jamais mettre à jour**. Le cycle LTS étant annuel et la fenêtre de support communautaire de trois ans, il faut planifier le passage vers une LTS ultérieure **avant la fin de vie** de la version en place — soit, en pratique, un saut de LTS à LTS tous les deux à trois ans.

---

## Implications pour la migration et le long terme

Pour une **migration vers MariaDB**, le réflexe recommandé est de viser une **version LTS comme version d'arrivée**, sauf besoin spécifique d'une fonctionnalité disponible uniquement en Rolling. On gagne ainsi un socle stable et une fenêtre de maintenance connue.

Au-delà de la migration initiale, il convient de **gérer le cycle de vie des versions** dans la durée : suivre les dates de fin de vie (voir l'Annexe G), et budgéter les **montées de version de LTS en LTS**, dont les mécanismes sont détaillés en 19.4. Le passage spécifique de la LTS précédente (11.8) à la LTS actuelle (12.3), avec ses changements de comportement, est traité en 19.10. Enfin, le choix entre éditions **Community** et **Enterprise** influe sur la durée de support disponible.

---

## En résumé

MariaDB propose deux canaux aux logiques opposées :

- le canal **LTS** (`X.3`, annuel) privilégie la **stabilité** : correctifs de bogues et de sécurité uniquement, fenêtre de support communautaire de **trois ans** (depuis la 11.8), code figé — c'est le **choix par défaut en production** ;
- le canal **Rolling** (`X.0`/`X.1`/`X.2`, trimestriel) privilégie les **fonctionnalités** : nouveautés en continu, mais **aucun correctif après la GA** et un support limité à la version suivante — réservé aux équipes capables de **mettre à jour fréquemment**.

Cette formation s'appuie sur **12.3 LTS** comme socle de production stable. La section suivante, **19.4 (Stratégies de mise à jour et upgrade paths)**, aborde la mécanique concrète des montées de version.

⏭️ [Stratégies de mise à jour et upgrade paths](/19-migration-compatibilite/04-strategies-mise-a-jour.md)
