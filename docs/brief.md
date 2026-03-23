# Brief — Relocation Risk Tool

## Contexte

Une entreprise doit déménager ses locaux. Elle emploie des profils techniques rares pour lesquels l'expérience prime. Le risque principal : que l'allongement des temps de trajet pousse des salariés clés à ne pas suivre, sans possibilité de remplacement à court terme.

## Besoin

Disposer d'un outil permettant de simuler l'impact d'un déménagement sur la rétention des salariés, en tenant compte de la valeur individuelle de chaque profil et de l'évolution de son temps de trajet.

## Approche solution

Un outil en deux modes :

**Mode simulation** — L'utilisateur saisit une adresse candidate pour les futurs locaux. L'outil recalcule le temps de trajet domicile/travail pour chaque salarié selon son mode de transport, compare avec le trajet actuel, et produit un score de risque individuel et global.

**Mode cartographie** *(phase 2)* — L'outil calcule le score de risque sur une grille géographique et affiche une carte de chaleur des zones optimales pour le déménagement.

## Modèle de scoring

- Chaque salarié dispose d'un **score de compétence** (1 à 5) défini par les RH
- Le **delta de trajet** = temps de trajet futur − temps de trajet actuel
- Le **risque individuel** = score de compétence × fonction du delta (s'emballe au-delà d'un seuil défini, ex. +30 min)
- Le **risque global** = somme pondérée des risques individuels

## Périmètre MVP

- Base de données salariés : identifiant, code postal domicile, mode de transport, score de compétence
- Simulation adresse par adresse
- Résultat : tableau trié par niveau de risque + score global
- Interface simple, utilisable par un profil RH non technique

## Hors périmètre MVP

- Mode cartographie (phase 2)
- Authentification et gestion des droits
- Synchronisation RH automatique
