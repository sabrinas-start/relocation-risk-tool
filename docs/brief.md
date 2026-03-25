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
- Le **risque individuel** = `score^1.5 × f(delta)` — s'emballe au-delà d'un seuil de +30 min
- Le **risque global** = somme des risques individuels / nombre total de salariés (moyenne simple)
 
### Fonction f(delta)
 
```
si delta <= 0       → f = 0
si 0 < delta <= 30  → f = delta / 30
si delta > 30       → f = 1 + (delta - 30) / 10   ← s'emballe
```
 
### Seuils de lecture
 
| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 3 | 3 – 12 | > 12 |
| Risque global | 0 – 2 | 2 – 6 | > 6 |
 
## Périmètre MVP
 
- Import de la base salariés via fichier CSV depuis l'interface — merge sur `id_salarie` (insert + update, sans suppression)
- Base salariés : identifiant anonymisé, adresse domicile complète, mode de transport, score de compétence
- Simulation adresse par adresse
- Résultat : tableau trié par niveau de risque + score global
- Interface simple, utilisable par un profil RH non technique
 
## Évolutions prévues post-MVP
 
**Agent IA d'interprétation** *(backlog immédiat)* — Après chaque simulation, un agent basé sur l'API Anthropic produit une synthèse en langage naturel des résultats : profils à risque, lecture croisée score/delta, recommandations. Intégré directement dans l'interface Lovable.
 
**Mode cartographie** *(phase 2)* — Calcul du score de risque sur une grille géographique, affichage d'une carte de chaleur des zones optimales pour le déménagement.
 
## Décisions d'architecture
 
**Trajets calculés dynamiquement** — Le temps de trajet actuel n'est pas stocké en base. n8n appelle Google Maps deux fois par salarié à chaque simulation (domicile → locaux actuels, domicile → adresse candidate). La donnée est toujours fraîche et cohérente.
 
**Import CSV avec merge** — L'`id_salarie` est la clé de merge. Une ligne existante est mise à jour, une ligne nouvelle est insérée, une ligne absente du fichier est conservée. L'`id_salarie` est immuable — un changement d'id crée un doublon.
 
**Usage mono-client** — L'outil est conçu pour un seul client à la fois. La base Airtable est partagée et non isolée par client. Le multi-tenant (isolation par `client_id` + authentification) est identifié comme évolution V2.
 
**Données RGPD** — Aucun nom, prénom ou donnée identifiante. L'`id_salarie` anonymisé est la seule référence. Le build s'effectue sur données fictives géographiquement réalistes.
 
**Score^1.5** — Le score de compétence est élevé à la puissance 1.5 avant multiplication par f(delta), pour amplifier l'écart entre profils rares et profils courants.
 
> ⚠️ Limite connue : un delta très élevé sur un profil faible peut dépasser un profil critique modérément impacté. À signaler aux RH : lire le tableau avec le score de compétence, pas seulement le risque individuel.
 
## Hors périmètre MVP
 
- Mode cartographie (phase 2)
- Authentification et gestion des droits
- Multi-tenant — isolation des données par client
- Synchronisation RH automatique
- Simulation heure de pointe (`traffic_model` Maps API)
- Suppression de lignes à l'import
