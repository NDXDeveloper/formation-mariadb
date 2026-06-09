🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe G — Versions de Référence

Cette annexe sert de référence rapide pour les versions de MariaDB pertinentes dans le cadre de cette formation : leur type, leur date de disponibilité générale (GA) et leur horizon de support. La version cible est la **12.3 LTS** ; la **11.8 LTS** sert de point de comparaison pour la migration.

## Tableau de référence

| Version | Type | Date GA | Support jusqu'à |
|---------|------|---------|-----------------|
| 13.1 🧪 | Développement | — | — |
| 13.0 | Rolling | RC — GA à venir | Version rolling suivante |
| **12.3** ⭐ | LTS | Mai 2026 | Juin 2029 |
| **11.8** | LTS | Juin 2025 | 2028 |
| **11.4** | LTS | Mai 2024 | 2029 |
| 10.11 | LTS | Fév 2023 | 2028 |
| 10.6 | LTS | Juil 2021 | 2026 |

> **Version cible de cette formation : 12.3 LTS** (GA = 12.3.2, fin mai 2026 — la 12.3.1 était une RC). Supportée jusqu'en **juin 2029**, elle consolide la série rolling 12.0 → 12.2. La **11.8** reste un socle largement déployé (support 2028) et sert de référence pour la migration. La série **13.x** ouvre le cycle suivant : la 13.0 est en préversion (mars 2026) puis *release candidate* (13.0.1, mai 2026) — sa GA n'est pas encore parue — et la 13.1 en développement.  
> Rappel du cycle : **LTS** annuelle, supportée 3 ans (depuis la 11.8 ; les LTS antérieures, dont la 11.4, conservaient 5 ans) ; **rolling** trimestrielles, supportées jusqu'à la version suivante.

**Légende du tableau** : ⭐ version cible de la formation · 🧪 version en développement · **caractères gras** = version LTS.

## Les trois types de version

MariaDB suit trois régimes de publication, détaillés aux [§1.5](../../01-introduction-fondamentaux/05-politique-versions-lts-rolling.md) et [§1.6](../../01-introduction-fondamentaux/06-cycle-support-lts.md) :

- **LTS (*Long Term Support*)** — une version par an, privilégiant la stabilité. Supportée 3 ans depuis la 11.8 (5 ans pour les LTS antérieures). C'est le régime recommandé en production ; la 12.3, la 11.8, la 11.4, la 10.11 et la 10.6 en relèvent.
- **Rolling** — une version par trimestre, apportant les fonctionnalités les plus récentes, mais supportée seulement jusqu'à la version *rolling* suivante. Elle s'adresse à ceux qui veulent le dernier état de l'art et peuvent suivre des mises à jour fréquentes ; la 13.0 (actuellement en *release candidate*, GA à venir) inaugure la série 13.x.
- **Développement (🧪)** — version en cours d'élaboration, non destinée à la production : ici la 13.1.

## Échéances de support

La colonne « Support jusqu'à » indique la date au-delà de laquelle une version ne reçoit plus de correctifs. Deux points méritent l'attention.

D'abord, la **10.6 arrive en fin de support en 2026** : les installations qui en dépendent doivent planifier une migration sans tarder (voir le [§19.10](../../19-migration-compatibilite/10-migration-11-8-vers-12-3.md) si la cible est la 12.3).

Ensuite, un effet du raccourcissement de la fenêtre LTS de cinq à trois ans, intervenu à partir de la 11.8 : la **11.4, pourtant plus ancienne, est supportée plus longtemps (2029) que la 11.8 (2028)**, car elle bénéficiait encore de l'ancien horizon de cinq ans. La 11.4 partage ainsi l'échéance 2029 de la 12.3.

## Choisir sa version

Pour la production, le choix par défaut est la **dernière LTS, soit la 12.3** : stabilité et fenêtre de support la plus longue (juin 2029). Le recours à une version *rolling* (13.x) ne se justifie que pour bénéficier immédiatement des nouveautés, en acceptant un support court et des mises à jour rapprochées. Les versions en développement sont à réserver aux tests.

Les critères détaillés et le calendrier d'adoption figurent en [F.5](../f-nouveautes-12-3/05-recommandations-adoption.md) ; la stratégie LTS contre *rolling* est développée au [§19.3](../../19-migration-compatibilite/03-gestion-versions-lts-rolling.md) et dans la roadmap [§1.7](../../01-introduction-fondamentaux/07-roadmap-serie-13.md). Pour les nouveautés apportées par la version cible, voir l'[Annexe F](../f-nouveautes-12-3/README.md).

⏭️ [Ressources et Documentation](/annexes/h-ressources-documentation/README.md)
