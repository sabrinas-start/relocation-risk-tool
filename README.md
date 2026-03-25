# Relocation Risk Tool

Outil de simulation de risque RH lié à un déménagement de locaux.

Une entreprise saisit une adresse candidate pour ses futurs locaux. L'outil recalcule les trajets domicile/travail de chaque salarié, calcule un score de risque individuel et global, et affiche les résultats dans une interface de décision.

> Projet portfolio — build no-code/IA réalisé dans le cadre d'une reconversion Product Builder.

---

## Contexte métier

Quand une entreprise déménage, elle risque de perdre ses profils techniques rares si l'allongement des temps de trajet devient inacceptable. Cet outil permet aux RH de simuler l'impact de plusieurs adresses candidates avant de prendre une décision.

---

## Fonctionnalités

**Simulation**
- Saisie d'une adresse candidate
- Calcul automatique des trajets via Google Maps (domicile → locaux actuels, domicile → adresse candidate)
- Score de risque individuel par salarié, trié par criticité
- Score de risque global avec jauge colorée
- Lecture croisée score de compétence × delta de trajet

**Import de la base salariés**
- Upload CSV depuis l'interface
- Merge sur identifiant salarié (insert + update, sans suppression)
- Rapport d'import détaillé (insérés / mis à jour / rejetés avec erreurs)
- Validation structure et contenu avant insertion

---

## Modèle de scoring

```
Risque individuel = score_compétence^1.5 × f(delta)

f(delta) :
  delta ≤ 0        → 0          (trajet raccourci ou identique)
  0 < delta ≤ 30   → delta / 30 (allongement progressif)
  delta > 30       → 1 + (delta - 30) / 10  (s'emballe au-delà du seuil)

Risque global = somme des risques individuels / nombre total de salariés
```

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 3 | 3 – 12 | > 12 |
| Risque global | 0 – 2 | 2 – 6 | > 6 |

> Le score de compétence est élevé à la puissance 1.5 pour amplifier l'écart entre profils rares et profils courants. Un profil score 5 pèse 11.18, un profil score 2 pèse 2.83.

---

## Stack technique

| Rôle | Outil |
|---|---|
| Base de données | Airtable |
| Orchestration + API | n8n |
| Calcul trajets | Google Maps Distance Matrix API |
| Interface | Lovable |
| Versionning | GitHub |

---

## Architecture

```
Lovable (front)
    │
    ├── POST /webhook/relocation-simulation  ──→  n8n workflow simulation
    │       { adresse_candidate: "..." }              │
    │                                                 ├── Airtable (lecture salariés)
    │                                                 ├── Google Maps API × 2 / salarié
    │                                                 └── Calcul scoring
    │       ← { risque_global, salaries: [...] }
    │
    └── POST /webhook/import-salaries        ──→  n8n workflow import
            { csv: "string brute" }                   │
                                                      ├── Parse + validation structure
                                                      ├── Validation contenu ligne par ligne
                                                      └── Airtable (insert / update par id_salarie)
            ← { inserts, updates, lignes_rejetees }
```

---

## Structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md                              — contexte, périmètre, décisions
    stack.md                              — détail technique par brique
    build-log.md                          — journal de build session par session
    workflow-n8n-documentation.docx       — doc du workflow simulation
    workflow-import-documentation.docx    — doc du workflow import CSV
  /n8n
    workflow-simulation.json              — export workflow n8n simulation
    workflow-import.json                  — export workflow n8n import CSV
  /data
    sample-data.csv                       — jeu de données fictif (10 salariés IDF)
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

---

## Contraintes et limites connues

- **Mono-client** — la base Airtable est partagée, pas d'isolation par client. Extension multi-tenant prévue en V2.
- **Lecture du tableau** — un delta très élevé sur un profil faible peut dépasser un profil critique modérément impacté. Lire le risque individuel en croisant avec le score de compétence.
- **Adresses approximées** — Google Maps peut géolocaliser au centre d'un arrondissement si l'adresse est incomplète. Aucune erreur remontée dans ce cas.

---

## Évolutions prévues

- Agent IA d'interprétation des résultats (API Anthropic) — synthèse en langage naturel post-simulation
- Mode cartographie — carte de chaleur des zones optimales (Phase 2)
- Multi-tenant — isolation des données par client (Phase 2)
