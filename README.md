# Relocation Risk Tool

> Quand une entreprise déménage, la question du risque RH est rarement traitée avec des données. Ce projet y répond : en saisissant une adresse candidate ou en étudiant des zones géographiques, les RH ou décideurs visualisent immédiatement un risque global et quels profils clés sont à risque de départ — et dans quelle mesure.

**Statut** : MVP en production  
**Stack** : Airtable · n8n · Google Maps API · Lovable (React / Leaflet)  
**Démo** : https://www.loom.com/share/fd4b3959d0e341e8abef2781ee4a50ab

---

## Contexte

Un déménagement de locaux peut provoquer des départs non anticipés, notamment chez les profils techniques rares pour lesquels l'expérience prime et le remplacement est long. Sans outil, l'évaluation de ce risque repose sur des estimations informelles et une collecte manuelle fastidieuse.

Ce projet propose une alternative : une simulation chiffrée, fondée sur les trajets réels de chaque salarié et sa valeur pour l'organisation, pour éclairer la décision avant qu'elle soit prise. 
L'outil tient également compte de la fragilité du trajet actuel : un salarié qui fait déjà 45 minutes de trajet est plus proche de sa limite de rupture qu'un salarié à 10 minutes. Ce seuil de tolérance est intégré au modèle.

L'outil couvre deux cas d'usage distincts :
- **Adresse candidate identifiée** → l'entreprise à déjà trouvé des locaux potentiels et souhaite connaître l'impact d'un déménagement sur cette adresse → Simulation précise avec scoring individuel et global *(page Simulation)*
- **Zone à explorer** → L'entreprise souhaite explorer des zones géographique sans adresse précise en tête → simulations groupées sur grille de 20 points classés par risque, pour orienter la recherche *(page Carte)*
 
Les deux pages forment un flux de décision continu : la carte permet d'identifier les zones prometteuses, et chaque point de résultat est directement cliquable pour lancer une simulation précise sur cette adresse — sans ressaisie.

---

## Fonctionnalités
 
### Simulation adresse précise *(page Simulation)*
 
L'entreprise a une adresse candidate identifiée et souhaite mesurer son impact précisément.
 
- Saisie de l'adresse candidate avec autocomplétion Google Places (adresse actuelle persistée en localStorage)
- Calcul automatique des trajets via Google Maps (domicile → locaux actuels, domicile → adresse candidate)
- Score de risque individuel par salarié, trié par criticité
- Score de risque global avec jauge colorée et légende des seuils
- Export XLS des résultats (résumé global + détail par salarié, nommé `simulation_YYYY-MM-DD.xlsx`)
 
### Exploration de zones *(page Carte)*
 
L'entreprise ne sait pas encore vers où chercher et souhaite orienter sa recherche immobilière.
 
- Carte interactive avec clusters de domiciles salariés
- Curseur repositionnable — simulation batch sur une grille de 20 points autour de la zone ciblée
- Classement des zones par risque global croissant, badges Faible / Moyen / Élevé / Partiel
- Export XLS par simulation
- Historique des simulations avec persistance Airtable (survit au changement d'appareil)
- Cercles permanents numérotés et colorés sur la carte — cliquables pour recharger les résultats
 
### Gestion de la base salariés
 
- Import CSV avec merge sur identifiant (`id_salarie`) — insert + update, sans suppression
- Limite : 50 lignes par import — imports successifs possibles pour les bases plus grandes
- Rapport d'import détaillé : insérés ✅ / mis à jour 🔄 / rejetés ⚠️ avec détail erreur ligne par ligne
- Compteur salariés en base affiché en temps réel
- Bouton "Vider la base" avec confirmation
- Export du template CSV pré-formaté
 
### Accès et quotas
 
- Écran de login — deux niveaux : client / admin
- Quota d'utilisation configurable par type de simulation (table `config` Airtable)
- Mode admin : bypass quota, accès illimité
 
---
 
## Modèle de scoring — conçu de zéro
 
Le modèle repose sur deux dimensions indépendantes, additionnées avant d'être amplifiées par le delta de trajet.
 
**Valeur du profil** — le `score_competence` (1 à 5) est élevé à la puissance 1.5 pour amplifier l'écart entre profils rares et profils courants, sans l'écraser.
 
**Fragilité du trajet actuel** — un salarié qui fait déjà 40 minutes de trajet est plus proche de sa limite de rupture qu'un salarié à 10 minutes. Une fonction de tolérance capture cette fragilité et s'additionne au score de compétence.
 
**Amplification par le delta** — la somme est multipliée par `f(delta)`, linéaire jusqu'à +30 min d'allongement, puis croissante rapidement au-delà. Ce seuil correspond au point de bascule au-delà duquel le risque de départ augmente fortement.
 
```
Risque individuel = (score_compétence^1.5 + tolérance(trajet_actuel)) × f(delta)
 
tolérance(trajet_actuel) :
  ≤ 20 min        → 0
  20–45 min       → 1
  > 45 min        → 2
 
f(delta) :
  delta ≤ 0        → 0              (trajet raccourci ou identique)
  0 < delta ≤ 30   → delta / 30     (allongement progressif)
  delta > 30       → 1 + (delta - 30) / 10   (s'emballe au-delà du seuil)
 
Risque global = somme des risques individuels / nombre total de salariés
```
 
> Note de lecture : un delta très élevé sur un profil faible peut dépasser un profil critique avec un delta modéré. Croiser `risque_individuel` et `score_competence` pour identifier les vrais profils à risque.
 
**Seuils de lecture**
 
| Indicateur | Faible | Moyen | Élevé |
|---|---|---|---|
| Risque individuel | 0 – 2 | 2 – 8 | > 8 |
| Risque global | 0 – 1.5 | 1.5 – 4 | > 4 |
| Tolérance | 0 – 1 | 1 – 2 | > 2 |
 
---
 
## Stack technique
 
| Rôle | Outil |
|---|---|
| Base de données | Airtable |
| Orchestration + API | n8n Cloud (11 workflows) |
| Calcul trajets | Google Maps Distance Matrix API |
| Géocodage adresses | Google Maps Geocoding API |
| Autocomplétion adresses | Google Places Autocomplete |
| Interface | Lovable (React / Leaflet) |
| Export tableur | SheetJS |
| Versionning | GitHub |
 
---
 
## Architecture
 
```
Lovable (front)
    │
    ├── POST /webhook/relocation-simulation    →  Simulation adresse précise
    │       { adresse_candidate, adresse_actuelle, admin }
    │       ← { risque_global, salaries: [...], quota_restant }
    │
    ├── POST /webhook/simulate-batch           →  Simulation grille 20 points
    │       { adresse_actuelle, lat_actuelle, lng_actuelle,
    │         lat_curseur, lng_curseur, rayon_override, admin }
    │       ← { points: [...], couleur_global, niveau_global, simulation_id }
    │           └── appelle simulate-one-point (sub-workflow) × 20
    │
    ├── POST /webhook/import-salaries          →  Import CSV
    │       { csv: "string brute" }
    │       ← { statut, inserts, updates, rejetes, lignes_rejetees }
    │
    ├── POST /webhook/geocode-salaries         →  Géocodage batch salariés
    ├── POST /webhook/vider-base               →  Suppression complète base
    ├── POST /webhook/get-simulations          →  Historique simulations batch
    ├── GET  /webhook/stats-base               →  Compteur salariés
    ├── GET  /webhook/stats-agent              →  Stats page Carte
    ├── GET  /webhook/search-salarie           →  Recherche salarié par id
    ├── GET  /webhook/clusters-carte           →  Clusters domiciles pour carte
    └── GET  /webhook/get-quota                →  Quotas en cours
```
 
---
 
## Format du CSV d'import
 
4 colonnes exactes, dans cet ordre :
 
```
id_salarie,adresse_domicile,mode_transport,score_competence
EMP001,"14 rue de la Roquette, 75011 Paris",transit,5
EMP002,"3 avenue du Général de Gaulle, 92100 Boulogne-Billancourt",driving,4
```
 
Valeurs acceptées pour `mode_transport` : `driving` / `transit` / `bicycling` / `walking`  
Valeurs acceptées pour `score_competence` : entier entre 1 et 5
 
> Limite : 50 lignes par import. Pour une base plus grande, effectuer des imports successifs — les lignes déjà présentes sont mises à jour, les nouvelles sont insérées.
 
---
 
## Structure du repo
 
```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md
    build-log.md
    build-log-V2.md
    case-study.md
  /n8n
    workflow-relocation-simulation.json
    workflow-import-salaries.json
    workflow-vider-base.json
    workflow-stats-base.json
    workflow-geocode-salaries.json
    workflow-stats-agent.json
    workflow-search-salarie.json
    workflow-simulate-one-point.json
    workflow-simulate-batch.json
    workflow-clusters-carte.json
    workflow-get-quota.json
    workflow-get-simulations.json
  /data
    sample-data.csv
  /screenshots
```
 
---
 
## Contraintes et limites connues
 
- **Mono-client** — la base Airtable est partagée, pas d'isolation par client. Extension multi-tenant prévue en V3.
- **Import limité à 50 lignes par passe** — timeout Lovable. Imports successifs possibles.
- **Limite 300 salariés (simulation batch)** — au-delà, le temps d'exécution dépasse le timeout synchrone.
- **Correspondance exacte adresse** sur l'historique — normalisée via Google Places, risque marginal si format change.
- **Adresses approximées** — Google Maps peut géolocaliser au centre d'un arrondissement sans remonter d'erreur si l'adresse est incomplète.
 
---
 
## Compétences mobilisées
 
`Product thinking` · `Conception d'un modèle de scoring` · `Modélisation de données` · `Automatisation n8n` · `API REST` · `Google Maps API` · `Webhooks` · `Architecture no-code` · `Logique métier RH` · `React / Leaflet (Lovable)` · `Gestion d'erreurs` · `Tests structurés`
 
---
 
## Évolutions prévues (V3+)
 
- **Multi-tenant** — isolation des données par client, adresse actuelle persistée côté serveur
- **Simulation avec horaires réels** — `departure_time` = `heure_debut_service` - 40 min, deux nouveaux champs Airtable
- **Page paramètres** — seuils configurables via UI, stockés en Airtable
- **Import > 50 lignes** — pattern asynchrone avec polling
