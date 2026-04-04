# Build Log — Relocation Risk Tool
## MVP V1 — Mars / Avril 2026

---

## Session 1 — Architecture, setup, build workflow simulation
*23 mars 2026*

### Contexte
Démarrage du projet avec stack validée en amont (Airtable + n8n + Google Maps + Lovable). Objectif : construire le workflow simulation de A à Z et le tester en production.

### Architecture définie

```
Webhook → Set → Airtable → Loop
                              ↓
                         HTTP Maps #1 (trajet actuel)
                              ↓
                         HTTP Maps #2 (trajet candidat)
                              ↓
                         Code node (calculs)
                              ↓
                    Aggregate → Calcul risque global
                              ↓
                    Respond to Webhook
```

### Workflow simulation construit

| # | Node | Rôle |
|---|---|---|
| 1 | Webhook - Lovable | Reçoit `adresse_candidate` via POST |
| 2 | Ajout Adresse Actuelle | Injecte `adresse_actuelle` hardcodée dans le flux |
| 3 | Search records | Récupère les salariés Airtable |
| 4 | Boucle par salarié | Loop Over Items batch size 1 |
| 5 | Maps 1 - Adresse actuelle | Trajet domicile → locaux actuels |
| 6 | Maps 2 - Adresse candidate | Trajet domicile → adresse candidate |
| 7 | Code in JavaScript | Delta + f(delta) + risque individuel |
| 8 | Aggregate salaries | Consolide les résultats |
| 9 | Calcul risque global | Somme filtrée / nb salariés |
| 10 | Respond to Webhook | Renvoie JSON structuré |


### Modèle de scoring initial
```
risque_individuel = score_competence × f(delta)
f(delta) : delta ≤ 0 → 0 / 0 < delta ≤ 30 → delta/30 / delta > 30 → 1 + (delta-30)/10
risque_global = somme des risques individuels
```

### Setup Airtable
- Table `Salariés` — 4 champs : `id_salarie`, `adresse_domicile`, `mode_transport`, `score_competence`
- `trajet_actuel_min` intentionnellement absent — calculé dynamiquement
- 10 salariés fictifs Île-de-France importés depuis `sample-data.csv`
- Personal Access Token créé, credentials configurées dans n8n

### Setup Hoppscotch
- Premier outil de test API du parcours
- URL test : `webhook-test/...` / URL production : `webhook/...`

### Problèmes rencontrés
- Airtable écrase les champs du tuyau → référencer les nodes source avec `$('Nom').first().json.champ`
- Code node plante sur adresse vide → opérateur `?.` + vérification `!rows` avant `duration.value`
- Outputs mémorisés n8n invalident les tests → vider les outputs avant tout test production

### Tests validés
- Adresse valide Paris centre → `risque_global: 5.96` ✅
- Adresse lointaine Melun → `risque_global: 77.28`, f(delta) s'emballe ✅
- Adresse invalide → tous null, `risque_global: 0` ✅
- Adresse domicile vide → garde-fou retourne null proprement ✅
- mode_transport invalide → Maps fallback driving silencieux ✅

### Documents produits
- `workflow-n8n-documentation.docx` v1
- `base-de-connaissance-relocation.docx` v1→v2
- `codebase-carte-des-champs.docx` v1→v2
- `sample-data.csv` (10 salariés)

---

## Session 2 — Interface Lovable + refonte scoring
*24–25 mars 2026*

### Page simulation — build initial

**Bug initial :** Lovable utilisait `json.resultats` au lieu de `json.salaries` → corrigé en `json.salaries ?? []`

**Fonctionnalités livrées :**
- Texte d'introduction
- Jauge risque global colorée
- Tableau salariés trié par `risque_individuel` décroissant
- Badges score compétence colorés
- Couleurs de fond colonne risque individuel

### Refonte modèle de scoring

**Problème :** avec `score × f(delta)`, un profil score 2 à +45 min obtenait le même risque qu'un profil score 5 à +30 min. Incohérent métier.

**Décision :** remplacer `score` par `score^1.5` pour amplifier le poids des profils rares.

```
risque_individuel = score^1.5 × f(delta)
risque_global = somme / nombre total de salariés (moyenne simple)
```

| Score | score^1.5 |
|---|---|
| 1 | 1 |
| 2 | 2.83 |
| 3 | 5.20 |
| 4 | 8 |
| 5 | 11.18 |

**Seuils calibrés** sur simulations réelles Paris 9e (risque 1.01) et Melun (risque 8.58) :

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0–3 | 3–12 | > 12 |
| Risque global | 0–2 | 2–6 | > 6 |

### Page import CSV — build initial

**Bug parser CSV :** `split(',')` naïf coupait sur les virgules dans les adresses quotées → remplacé par un parser caractère par caractère gérant les guillemets.

**Bug noms de champs :** Lovable traduisait `inserts` → `insertions` → corrigé. **Règle établie :** ne jamais traduire les noms de champs JSON dans les prompts Lovable.

**Fonctionnalités livrées :**
- Upload CSV drag & drop ou clic
- Rapport d'import basique (insérés / mis à jour)
- Gestion erreur structure et erreur réseau

---

## Session 3 — Workflow import CSV : build + fix performance + refactoring
*25–26 mars 2026*

### Architecture workflow import

**Choix structurant :** CSV envoyé comme string brute dans `{ "csv": "..." }` — simplicité, pas de contrainte taille pour une base RH.

**Architecture finale :**
```
Webhook → Parse CSV → Validation lignes → Eclate lignes valides
→ IF Structure valide ?
    [false] → Respond to Webhook
    [true]  → Execute Sub-workflow (Résolution ids)
            → Boucle par ligne
                → Garde-fou search → IF id trouvé ?
                    [true]  → Airtable Update → Tag updated → retour boucle
                    [false] → Airtable Create → Tag inserted → retour boucle
            → Rapport import → Respond to Webhook
```

**URL production :** `POST /webhook/import-salaries`

### Fix performance — pre-fetch ids
**Problème :** 1 appel Airtable Search par ligne = N appels séquentiels → timeout à 280 lignes (~5 min).

**Solution :** sub-workflow dédié "Résolution ids — Import CSV" — 1 seul appel Airtable filtré sur les ids du CSV, construit une map `id_salarie → record_id`, retransmet les lignes enrichies.

**Limite MVP confirmée :** 50 lignes max par import (~1min30). Timeout Lovable à 2 min.

### Sub-workflow "Résolution ids — Import CSV"

| # | Node | Rôle |
|---|---|---|
| 1 | Execute Workflow Trigger | Reçoit `{ ids: [...], lignes: [...] }` |
| 2 | Code — Construit formule | Formule OR() filterByFormula |
| 3 | HTTP Request — Search filtré | 1 appel Airtable filtré |
| 4 | Code — Construit map | Map ids_existants + lignes enrichies |

### Static data — état final
| Variable | Rôle |
|---|---|
| `store.inserts` | Compteur inserts |
| `store.updates` | Compteur updates |
| `store.lignes_rejetees` | Lignes rejetées à la validation |

---

## Session 4 — Fix import + nouveaux workflows vider-base et stats-base
*26 mars 2026*

### Fix 4 — Diagnostic précis erreur_structure
Parser CSV amélioré avec diagnostic différencié en 4 cas : colonnes manquantes, en trop, mal orthographiées, mauvais ordre.

### Fix 3 — JSON unifié

Contrat de sortie normalisé sur tous les cas :
```json
{
  "statut": "ok" | "erreur_structure" | "import_rejete",
  "message": "...",
  "inserts": 0,
  "updates": 0,
  "rejetes": 0,
  "lignes_rejetees": [
    { "ligne_numero": 2, "id_salarie": "EMP003", "erreurs": ["adresse_domicile vide"] }
  ]
}
```

### Fix 5 — JSON invalide Respond to Webhook
Guillemets non échappés dans `erreur_structure` cassaient le JSON. Solution : Code node "Formate erreur structure" + `JSON.stringify($json)` dans le Respond to Webhook.

### Workflow vider-base — `POST /webhook/vider-base`
URL DELETE Airtable construite manuellement (n8n ne supporte pas les query params répétés `records[]`) :
```javascript
const params = batch.map(id => `records[]=${id}`).join('&');
```
Réponse : `{ statut: "ok", deleted: N }`

### Fix 6 — Vider-base base vide
IF node après lecture Airtable : si `records.length === 0` → Respond direct `{ statut: "ok", deleted: 0 }`.

### Workflow stats-base — `GET /webhook/stats-base`
Node Airtable natif "Return All" — gère la pagination automatiquement (contrairement au HTTP Request limité à 100 records). Remplacé en session suivante par HTTP Request REST pour gérer le cas base vide (le node natif coupait le flux sur 0 résultats).

### Fix 2 — Supprimé
Mode `full_replace` hors scope définitif — le flux vider + réimporter couvre le besoin.

---

## Session 5 — Interface Lovable : page import + finitions simulation
*27–28 mars 2026*

### Page import CSV — construction complète
- Compteur "Salariés en base" — GET stats-base au chargement, rafraîchi après import et après vider
- Validation côté front — blocage si > 50 lignes avant envoi
- Rapport d'import enrichi : `inserts ✅` / `updates 🔄` / `rejetes ⚠️` + détail ligne par ligne
- Bouton "Vider la base" avec confirmation
- Template XLS téléchargeable via SheetJS (4 colonnes, 2 lignes exemple)

**Incident régression :** tentative de passage `application/json` → `multipart/form-data` → cascade d'erreurs sur plusieurs nodes → rollback complet vers le format d'origine.

**Bug pagination stats-base :** node Airtable natif coupait le flux sur base vide → remplacé par HTTP Request REST (`{ records: [] }` préserve le flux). Correction définitive appliquée en session 7.

### Page simulation — finitions
- Suppression barre de progression sous le score global
- Légende seuils individuel + global
- Bouton export XLS post-simulation (2 onglets via SheetJS)

### Tests et fixes au fil de l'eau
Tests fonctionnels sur les workflows import, vider-base et stats-base. Fixes légers sur le rapport d'import et le compteur salariés.

---

## Session 6 — Nouveau modèle de scoring + découplage adresse actuelle
*30 mars 2026*

### Introduction de la tolérance

**Problème identifié :** la formule `score^1.5 × f(delta)` ignorait qu'un salarié déjà à 1h de trajet est plus fragile face à un allongement supplémentaire.

**Nouvelle formule :**
```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  ≤ 10 min        → 0
  10–20 min       → 0.5
  20–30 min       → 1
  > 30 min        → 1 + (trajet - 30) / 15
```

Champ `tolerance` ajouté dans le JSON de sortie par salarié.

### Recalibration seuils
Calibrés sur 27 profils — scénario banlieue → banlieue ~15km max :

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0–2 | 2–8 | > 8 |
| Risque global | 0–1.5 | 1.5–4 | > 4 |
| Tolérance | 0–1 | 1–2 | > 2 |

### Découplage adresse actuelle
Adresse actuelle retirée du Set node hardcodé → envoyée par Lovable dans le body POST :
```json
{ "adresse_candidate": "...", "adresse_actuelle": "..." }
```
Persistée en localStorage côté front (clé : `adresse_actuelle`). Autocomplete Google Places activé sur les deux champs.

---

## Session 7 — Lovable simulation + corrections + export XLS
*31 mars 2026*

### Stats-base — fix base vide
Node Airtable natif remplacé par HTTP Request REST — retourne `{ records: [] }` sur base vide au lieu de couper le flux.

### Page simulation — mises à jour
- Champ adresse actuelle ajouté (persisté en localStorage)
- Colonne `tolerance` ajoutée en 3e position du tableau
- Seuils recalibrés sur badges et jauge
- Légende compacte 3 pastilles sous la jauge
- Tooltips "?" sur colonnes Tolérance et Risque individuel
- Sous-titre jauge mis à jour
- Export XLS mis à jour : colonne `tolerance` dans l'onglet détail, nommage `simulation_YYYY-MM-DD.xlsx`
- Nombre de salariés en base affiché au chargement

### Page import — corrections
- Rapport d'import : mapping complet 3 cas JSON (`ok` / `import_rejete` / `erreur_structure`)
- "Base vide" avec bandeau orange si `nb_salaries = 0`
- "Aucune donnée à supprimer" si `deleted = 0`
- Template CSV mis à jour

---

## Session 8 — Tests complets + fixes + clôture V1
*31 mars – 2 avril 2026*

### Plan de tests exécuté
79 cas couvrant : workflow simulation (nominal + cas limites + scoring), import CSV (structure + validation + merge + compteurs), webhooks vider-base + stats-base, pages Lovable import et simulation.

### Bugs identifiés et corrigés
- **n8n Fix 5** — guillemets erreur_structure cassant le JSON Respond to Webhook → Code node "Formate erreur structure"
- **n8n Fix 6** — vider-base plantage base vide → IF node + Respond direct `deleted: 0`
- **Lovable** — mapping messages d'erreur import, compteur "Base vide", "Aucune donnée à supprimer", template CSV colonnes

### Tests scoring sim10–sim19
✅ Validés en session de clôture — nouvelle formule + tolérance confirmées.

---

## État final V1

### Workflows en production

| Workflow | URL | Statut |
|---|---|---|
| Simulation | `POST /webhook/relocation-simulation` | ✅ |
| Import CSV | `POST /webhook/import-salaries` | ✅ |
| Vider la base | `POST /webhook/vider-base` | ✅ |
| Stats base | `GET /webhook/stats-base` | ✅ |

### Formule de scoring finale
```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)
risque_global = somme des risques individuels / nombre total de salariés
```

### Limites MVP assumées
- Mono-client — base Airtable partagée
- Import limité à 50 lignes — timeout Lovable 2 min
- Adresse actuelle en localStorage — liée au navigateur

### Backlog V2
- Agent IA recommandation zones optimales
- Page paramétrage des seuils
- Simulation avec horaires réels (`departure_time` = `heure_debut_service` - 40 min)
- Multi-tenant
- Import > 50 lignes (bulk insert)
