# Relocation Risk Tool

> Remplace une décision de localisation prise à l'aveugle par une simulation chiffrée — en saisissant une adresse candidate, les RH voient immédiatement quels profils clés sont à risque de départ.

**Statut** : MVP en production  
**Stack** : Airtable · n8n · Google Maps API · Lovable (React)

---

## Contexte métier

Quand une entreprise déménage, elle risque de perdre ses profils techniques rares si l'allongement des temps de trajet devient inacceptable. Sans outil, la question *"est-ce qu'on peut déménager là ?"* demande des semaines de collecte manuelle et reste sans réponse fiable. Cet outil permet aux RH de simuler l'impact de plusieurs adresses candidates en quelques secondes, avant de prendre une décision.

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

## Modèle de scoring

```
Risque individuel = (score_compétence^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  trajet_actuel ≤ 10        → t = 0
  10 < trajet_actuel ≤ 20   → t = 0.5
  20 < trajet_actuel ≤ 30   → t = 1
  trajet_actuel > 30         → t = 1 + (trajet_actuel - 30) / 15

f(delta) :
  delta ≤ 0        → 0          (trajet raccourci ou identique)
  0 < delta ≤ 30   → delta / 30 (allongement progressif)
  delta > 30       → 1 + (delta - 30) / 10  (s'emballe au-delà du seuil)

Risque global = somme des risques individuels / nombre total de salariés
```

> La tolérance capture le fait qu'un salarié déjà à la limite de son temps de trajet acceptable est plus fragile face à un allongement supplémentaire. Elle s'additionne au score de compétence avant d'être appliquée au delta — les deux dimensions sont indépendantes.

> Le score de compétence est élevé à la puissance 1.5 pour amplifier l'écart entre profils rares et profils courants. Lire le tableau en croisant risque individuel et score de compétence.

> Seuils calibrés sur un scénario banlieue → banlieue (~15km max). À revoir si le périmètre de simulation change.

---

## Stack technique

| Rôle | Outil |
|---|---|
| Base de données | Airtable |
| Orchestration + API | n8n (4 workflows) |
| Calcul trajets | Google Maps Distance Matrix API |
| Interface | Lovable |
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
EMP002,"3 avenue du Général de Gaulle, 92100 Boulogne-Billancourt",driving,4
```

Valeurs acceptées pour `mode_transport` : `driving` / `transit` / `bicycling` / `walking`

Valeurs acceptées pour `score_competence` : entier entre 1 et 5

> Limite MVP : 50 lignes par import. Pour une base de plus de 50 salariés, effectuer des imports successifs — les lignes déjà présentes sont mises à jour, les nouvelles sont insérées.

---

## Structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md
    build-log.md
    workflow-n8n-documentation.docx
    workflow-import-documentation.docx
  /n8n
    workflow-simulation.json
    workflow-import.json
    workflow-vider-base.json
    workflow-stats-base.json
  /data
    sample-data.csv
```

---

## Contraintes et limites connues

- **Mono-client** — la base Airtable est partagée, pas d'isolation par client. Extension multi-tenant prévue en V2.
- **Import limité à 50 lignes** — timeout Lovable à 2 min. Imports successifs possibles.
- **Adresse actuelle persistée en localStorage** — liée au navigateur. Si l'utilisateur change de machine, re-saisie requise une fois. Persistance côté serveur prévue en V2 (multi-tenant).
- **Lecture du tableau** — un delta très élevé sur un profil faible peut dépasser un profil critique modérément impacté. Croiser risque individuel et score de compétence.
- **Adresses approximées** — Google Maps peut géolocaliser au centre d'un arrondissement si l'adresse est incomplète, sans remonter d'erreur.

---

## Compétences mobilisées

`Product thinking` · `Modélisation de données` · `Automatisation n8n` · `API REST` · `Google Maps API` · `Webhooks` · `Logique métier RH` · `React (Lovable)` · `Gestion d'erreurs` · `Tests structurés`

---

## Évolutions prévues

- Simulation avec horaires réels — `departure_time` = `heure_debut_service` - 40 min par salarié, deux nouveaux champs Airtable (`heure_debut_service`, `heure_fin_service`) — Phase 2
- Agent IA recommandation zones optimales — suggestions cliquables basées sur distribution géographique des domiciles + scores, clic déclenche simulation directe — Phase 2
- Mode cartographie — carte de chaleur des zones optimales — Phase 2
- Multi-tenant — isolation des données par client, adresse actuelle persistée côté serveur — Phase 2
- Import > 50 lignes en une seule passe — bulk insert — Phase 2
