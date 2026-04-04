# Stack — Relocation Risk Tool

## Vue d'ensemble

| Rôle | Outil | Raison |
|---|---|---|
| Base de données | **Airtable** | Connecteur n8n natif, plug-and-play, alimentée via import CSV |
| Orchestration + API | **n8n** | 4 workflows : simulation, import CSV, vider la base, stats |
| Calcul trajets | **Google Maps Distance Matrix API** | Précis, supporte voiture / transports en commun / vélo / pieds |
| Interface | **Lovable** | Front généré en prompt, pas de code front à écrire |
| Doc + base de connaissance | **Notion** | Centralisation projet + build log + apprentissages |
| Versionning | **GitHub** | Export workflows n8n + docs markdown |

---

## Détail par brique

### Airtable — base salariés

Table unique `Salariés` avec 4 colonnes. `trajet_actuel_min` intentionnellement absent — calculé dynamiquement à chaque simulation via Maps API.

| Champ | Type | Description |
|---|---|---|
| `id_salarie` | Text | Identifiant anonymisé (ex. EMP001) — clé de merge à l'import |
| `adresse_domicile` | Text | Adresse complète — utilisée directement par Google Maps |
| `mode_transport` | Single select | `driving` / `transit` / `bicycling` / `walking` — valeurs exactes Maps API |
| `score_competence` | Number (1–5) | Score défini par les RH — décimales acceptées (ex. 3.5) |

> Note RGPD : pas de nom, prénom, ou données identifiantes. L'`id_salarie` anonymisé est la seule référence.

---

### n8n — workflow simulation

**URL production :** `POST /webhook/relocation-simulation`

**Body attendu :**
```json
{
  "adresse_candidate": "12 rue de la Paix, Paris",
  "adresse_actuelle": "24 avenue Kléber, 75016 Paris"
}
```

> `adresse_actuelle` est saisie et persistée côté front (localStorage). Elle n'est plus hardcodée dans n8n.

**Flux :**

1. **Webhook** : reçoit `adresse_candidate` + `adresse_actuelle` depuis Lovable
2. **Airtable** : récupère tous les salariés
3. **Loop over items** : pour chaque salarié
   - Appel Maps API : `adresse_domicile` → `adresse_actuelle`
   - Appel Maps API : `adresse_domicile` → `adresse_candidate`
   - Calcul `delta` = trajet futur − trajet actuel
   - Calcul `tolérance(trajet_actuel)`
   - Calcul `risque_individuel` = `(score^1.5 + tolérance) × f(delta)`
4. **Aggregate** : consolide les résultats
5. **Code node** : calcul `risque_global` = somme des risques / nombre de salariés
6. **Respond to webhook** : renvoie le JSON à Lovable

**Modèle de scoring :**

```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  trajet_actuel ≤ 10        → t = 0
  10 < trajet_actuel ≤ 20   → t = 0.5
  20 < trajet_actuel ≤ 30   → t = 1
  trajet_actuel > 30         → t = 1 + (trajet_actuel - 30) / 15

f(delta) :
  delta ≤ 0        → f = 0
  0 < delta ≤ 30   → f = delta / 30
  delta > 30       → f = 1 + (delta - 30) / 10   ← s'emballe

risque_global = somme des risques individuels / nombre total de salariés
```

> La tolérance capture la fragilité d'un salarié déjà à la limite de son temps de trajet acceptable. Elle s'additionne au score de compétence — les deux dimensions sont indépendantes.

> Seuils calibrés sur un scénario banlieue → banlieue (~15km max).

**Seuils de lecture :**

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 2 | 2 – 8 | > 8 |
| Risque global | 0 – 1.5 | 1.5 – 4 | > 4 |
| Tolérance | 0 – 1 | 1 – 2 | > 2 |

---

### n8n — workflow import CSV

**URL production :** `POST /webhook/import-salaries`

**Flux :** Webhook → Parse CSV → Validation lignes → Eclate lignes → Sub-workflow résolution ids → Boucle (Search + Garde-fou + IF → Update ou Create) → Rapport → Respond

**Règles d'import :**
- Template CSV strict — 4 colonnes exactes : `id_salarie`, `adresse_domicile`, `mode_transport`, `score_competence`
- Merge sur `id_salarie` : insert si nouvel id, update si id existant, conservation si id absent du fichier
- Limite MVP : 50 lignes par import (~1min30 mesuré)
- `id_salarie` immuable — un changement d'id crée un doublon

**JSON de sortie :**
```json
{
  "statut": "ok" | "erreur_structure" | "import_rejete",
  "message": "...",
  "inserts": 2,
  "updates": 1,
  "rejetes": 0,
  "lignes_rejetees": []
}
```

---

### n8n — workflow vider la base

**URL production :** `POST /webhook/vider-base`

Supprime tous les records de la table Salariés en batches de 10.

Réponse : `{ statut: "ok", deleted: N }` — retourne `deleted: 0` sans erreur si la base est déjà vide.

---

### n8n — workflow stats base

**URL production :** `GET /webhook/stats-base`

Compte le nombre de records via node Airtable natif (Return All — gère la pagination automatiquement). Réponse : `{ nb_salaries: N }`

---

### Google Maps Distance Matrix API

- Endpoint : `https://maps.googleapis.com/maps/api/distancematrix/json`
- Paramètres : `origins`, `destinations`, `mode`, `key`
- Unité retournée : secondes → convertir en minutes dans n8n
- Modes supportés : `driving`, `transit`, `bicycling`, `walking`
- Tarification : gratuit jusqu'à 40 000 éléments/mois

---

### Lovable — interface

**Page simulation :**
- Champ adresse actuelle des locaux (persistée en localStorage, clé : `adresse_actuelle`)
- Champ adresse candidate — autocomplete Google Places sur les deux champs
- Jauge risque global colorée (vert < 1.5 / orange 1.5–4 / rouge > 4) + légende compacte 3 pastilles
- Tableau salariés trié par risque individuel décroissant
- Colonnes affichées : `id_salarie` · `score_competence` · `tolerance` · `delta_min` · `risque_individuel`
- Badges colorés sur score compétence, tolérance et risque individuel selon seuils
- Tooltips "?" en tête de colonne Tolérance et Risque individuel
- Export XLS post-simulation (2 onglets : résumé global + détail salariés, nommé `simulation_YYYY-MM-DD.xlsx`)

**Page import CSV :**
- Compteur salariés en base (GET stats-base au chargement) — affiche "Base vide" si `nb_salaries = 0`
- Upload CSV avec validation 50 lignes côté front
- Instructions imports successifs
- Rapport d'import enrichi avec détail des lignes rejetées
- Bouton "Vider la base" avec confirmation — affiche "Aucune donnée à supprimer" si `deleted: 0`
- Export template CSV (SheetJS)

---

### GitHub — structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md                              ← ce fichier
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
  /screenshots
```

---

## Ce qui est hors scope MVP

- Authentification / gestion des droits
- Multi-tenant — isolation des données par client, adresse actuelle persistée côté serveur → Phase 2
- Synchronisation RH automatique
- Simulation avec horaires réels — `departure_time` = `heure_debut_service` - 40min, deux nouveaux champs Airtable → Phase 2
- Import > 50 lignes en une seule passe (bulk insert) → Phase 2
- Agent IA recommandation zones optimales → Phase 2
