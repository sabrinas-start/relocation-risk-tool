# Relocation Risk Tool

> Quand une entreprise déménage, la question du risque RH est rarement traitée avec des données. Ce projet y répond : en saisissant une adresse candidate, les RH visualisent immédiatement quels profils clés sont à risque de départ — et dans quelle mesure.

**Statut** : MVP en production  
**Stack** : Airtable · n8n · Google Maps API · Lovable (React)  
**Démo** : [Loom — à insérer]

---

## Contexte

Un déménagement de locaux peut provoquer des départs non anticipés, notamment chez les profils techniques rares pour lesquels l'expérience prime et le remplacement est long. Sans outil, l'évaluation de ce risque repose sur des estimations informelles et une collecte manuelle fastidieuse.

Ce projet propose une alternative : une simulation chiffrée, fondée sur les trajets réels de chaque salarié et sa valeur pour l'organisation, pour éclairer la décision avant qu'elle soit prise.

---

## Fonctionnalités

**Simulation**
- Saisie d'une adresse candidate + adresse actuelle des locaux (persistée en localStorage)
- Calcul automatique des trajets via Google Maps (domicile → locaux actuels, domicile → adresse candidate)
- Score de risque individuel par salarié, trié par criticité
- Score de risque global avec jauge colorée et légende des seuils
- Export XLS des résultats (résumé global + détail par salarié)

**Gestion de la base salariés**
- Import CSV avec merge sur identifiant (`id_salarie`) — insert + update, sans suppression
- Limite MVP : 50 lignes par import — imports successifs possibles pour les bases plus grandes
- Rapport d'import détaillé (insérés / mis à jour / rejetés avec détail erreurs ligne par ligne)
- Compteur salariés en base affiché en temps réel
- Bouton "Vider la base" pour repartir de zéro avant un réimport complet
- Export du template CSV pré-formaté

---

## Modèle de scoring — conçu de zéro

Le modèle repose sur deux dimensions indépendantes, additionnées avant d'être amplifiées par le delta de trajet.

**Valeur du profil** — le `score_competence` (1 à 5, décimales acceptées) est élevé à la puissance 1.5 pour amplifier l'écart entre profils rares et profils courants, sans l'écraser.

**Fragilité du trajet actuel** — un salarié qui fait déjà 40 minutes de trajet est plus proche de sa limite de rupture qu'un salarié à 10 minutes. Une fonction de tolérance capture cette fragilité et s'additionne au score de compétence.

**Amplification par le delta** — la somme est multipliée par `f(delta)`, linéaire jusqu'à +30 min d'allongement, puis croissante rapidement au-delà. Ce seuil correspond au point de bascule au-delà duquel le risque de départ augmente fortement.

```
Risque individuel = (score_compétence^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  ≤ 10 min        → 0
  10–20 min       → 0.5
  20–30 min       → 1
  > 30 min        → 1 + (trajet - 30) / 15

f(delta) :
  delta ≤ 0        → 0              (trajet raccourci ou identique)
  0 < delta ≤ 30   → delta / 30     (allongement progressif)
  delta > 30       → 1 + (delta - 30) / 10   (s'emballe au-delà du seuil)

Risque global = somme des risques individuels / nombre total de salariés
```

> Note de lecture : un delta très élevé sur un profil faible peut dépasser un profil critique avec un delta modéré. Croiser `risque_individuel` et `score_competence` pour identifier les vrais profils à risque.

**Seuils de lecture**

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 2 | 2 – 8 | > 8 |
| Risque global | 0 – 1.5 | 1.5 – 4 | > 4 |
| Tolérance | 0 – 1 | 1 – 2 | > 2 |

---

## Stack technique

| Rôle | Outil |
|---|---|
| Base de données | Airtable |
| Orchestration + API | n8n (4 workflows) |
| Calcul trajets | Google Maps Distance Matrix API |
| Interface | Lovable (React) |
| Versionning | GitHub |

---

## Architecture

```
Lovable (front)
    │
    ├── POST /webhook/relocation-simulation  ──→  n8n workflow simulation
    │       { adresse_candidate: "...",             │
    │         adresse_actuelle: "..." }             ├── Airtable (lecture salariés)
    │                                              ├── Google Maps API × 2 / salarié
    │                                              └── Calcul scoring
    │       ← { risque_global, salaries: [...] }
    │
    ├── POST /webhook/import-salaries        ──→  n8n workflow import CSV
    │       { csv: "string brute" }                │
    │                                              ├── Parse + validation structure
    │                                              ├── Sub-workflow résolution ids
    │                                              └── Airtable (insert / update)
    │       ← { statut, inserts, updates, rejetes, lignes_rejetees }
    │
    ├── POST /webhook/vider-base             ──→  n8n workflow vider la base
    │       ← { statut, deleted: N }
    │
    └── GET  /webhook/stats-base             ──→  n8n workflow stats
            ← { nb_salaries: N }
```

---

## Format du CSV d'import

4 colonnes exactes, dans cet ordre :

```
id_salarie,adresse_domicile,mode_transport,score_competence
EMP001,"14 rue de la Roquette, 75011 Paris",transit,5
EMP002,"3 avenue du Général de Gaulle, 92100 Boulogne-Billancourt",driving,3.5
```

Valeurs acceptées pour `mode_transport` : `driving` / `transit` / `bicycling` / `walking`

Valeurs acceptées pour `score_competence` : nombre entre 1 et 5, décimales acceptées (ex. 3.5)

> Limite MVP : 50 lignes par import. Pour une base plus grande, effectuer des imports successifs — les lignes déjà présentes sont mises à jour, les nouvelles sont insérées.

---

## Structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md
    build-log.md
  /n8n
    workflow-simulation.json
    workflow-import.json
    workflow-vider-base.json
    workflow-stats-base.json
  /screenshots
```

---

## Contraintes et limites connues

- **Mono-client** — la base Airtable est partagée, pas d'isolation par client. Extension multi-tenant prévue en V2.
- **Import limité à 50 lignes** — timeout Lovable à 2 min. Imports successifs possibles.
- **Adresse actuelle persistée en localStorage** — liée au navigateur. Re-saisie requise si changement de machine. Persistance côté serveur prévue en V2.
- **Lecture du tableau** — croiser `risque_individuel` et `score_competence` pour identifier les vrais profils critiques.
- **Adresses approximées** — Google Maps peut géolocaliser au centre d'un arrondissement sans remonter d'erreur si l'adresse est incomplète.

---

## Compétences mobilisées

`Product thinking` · `Conception d'un modèle de scoring` · `Modélisation de données` · `Automatisation n8n` · `API REST` · `Google Maps API` · `Webhooks` · `Logique métier RH` · `React (Lovable)` · `Gestion d'erreurs` · `Tests structurés`

---

## Évolutions prévues (V2)

- **Simulation avec horaires réels** — `departure_time` = `heure_debut_service` - 40 min, deux nouveaux champs Airtable
- **Agent IA recommandation zones** — suggestions cliquables basées sur la distribution géographique des domiciles + scores, clic déclenche une simulation directe
- **Multi-tenant** — isolation des données par client, adresse actuelle persistée côté serveur
- **Import > 50 lignes** — bulk insert en une seule passe
