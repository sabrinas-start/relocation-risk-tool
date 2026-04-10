# Stack — Relocation Risk Tool

## Vue d'ensemble

| Rôle | Outil | Raison |
|---|---|---|
| Base de données | **Airtable** | Connecteur n8n natif, plug-and-play, alimentée via import CSV |
| Orchestration + API | **n8n** | 12 workflows : simulation, batch, import CSV, carte, quotas, persistance |
| Calcul trajets | **Google Maps Distance Matrix API** | Précis, supporte voiture / transports en commun / vélo / pieds |
| Géocodage adresses | **Google Maps Geocoding API** | Coordonnées GPS salariés pour clusters et simulation batch |
| Autocomplétion adresses | **Google Places Autocomplete** | Saisie adresse normalisée côté front |
| Interface | **Lovable** | Front React généré en prompt, carte Leaflet interactive |
| Export tableur | **SheetJS** | Export XLS côté front, sans serveur |
| Versionning | **GitHub** | Export workflows n8n + docs markdown |

---

## Détail par brique

### Airtable — tables

#### Table `Salariés`

`trajet_actuel_min` intentionnellement absent — calculé dynamiquement à chaque simulation via Maps API.

| Champ | Type | Description |
|---|---|---|
| `id_salarie` | Text | Identifiant anonymisé (ex. EMP001) — clé de merge à l'import |
| `adresse_domicile` | Text | Adresse complète — utilisée directement par Google Maps |
| `mode_transport` | Single select | `driving` / `transit` / `bicycling` / `walking` — valeurs exactes Maps API |
| `score_competence` | Number (1–5) | Score défini par les RH — décimales acceptées (ex. 3.5) |
| `lat` / `lng` | Number | Coordonnées GPS — calculées par géocodage |

> Note RGPD : pas de nom, prénom, ou données identifiantes. L'`id_salarie` anonymisé est la seule référence. Les clusters carte exposent uniquement `lat`, `lng` et `score_competence`.

#### Table `config` (1 enregistrement)

| Champ | Type | Description |
|---|---|---|
| `quota_simulation_restant` | Number | Quota simulations adresse précise |
| `quota_batch_restant` | Number | Quota simulations batch |
| `bypass` | Boolean | `true` = mode admin, pas de décrémentation |

#### Table `simulations_historique`

| Champ | Type | Description |
|---|---|---|
| `simulation_id` | Number | Numéro incrémental (1 → 25) |
| `adresse_actuelle` | Text | Adresse des locaux — clé de filtrage |
| `lat` / `lng` | Number | Coordonnées du curseur |
| `timestamp` | DateTime | ISO 8601, Europe/Paris |
| `niveau` | Text | `Faible` / `Moyen` / `Élevé` |
| `couleur` | Text | `vert` / `orange` / `rouge` |
| `resultats_json` | Long text | JSON stringifié des 20 points simulés (~4kb) |

> Règle fondamentale : Airtable est la mémoire permanente. Le client ne peut jamais supprimer un record depuis l'interface.

---

### n8n — workflows

#### `relocation-simulation` — `POST /webhook/relocation-simulation`

**Body :**
```json
{
  "adresse_candidate": "12 rue de la Paix, Paris",
  "adresse_actuelle": "24 avenue Kléber, 75016 Paris",
  "admin": false
}
```

**Flux :**
1. Vérification quota (bypass si `admin: true`)
2. Airtable : lecture tous salariés
3. Loop par salarié : Maps #1 (trajet actuel) + Maps #2 (trajet candidat) + calcul scoring
4. Aggregate + calcul `risque_global`
5. Décrémentation quota + Respond to Webhook

---

#### `simulate-batch` — `POST /webhook/simulate-batch`

**Body :**
```json
{
  "adresse_actuelle": "24 avenue Kléber, 75016 Paris",
  "lat_actuelle": 48.8648,
  "lng_actuelle": 2.2967,
  "lat_curseur": 48.8500,
  "lng_curseur": 2.3200,
  "rayon_override": 2,
  "admin": false
}
```

**Flux en 4 phases :**
1. Lecture salariés + calcul barycentre pondéré par `score_competence` + génération grille 4×5
2. Phase 1 : calcul trajets actuels pour tous les salariés (batching par mode de transport, 25 origines/appel)
3. Phase 2 : appel sub-workflow `simulate-one-point` × 20
4. Classement par `risque_global` croissant + reverse geocoding + write `simulations_historique` Airtable

> `lat_actuelle`/`lng_actuelle` → destination Phase 1 (trajets actuels). `lat_curseur`/`lng_curseur` → centre de la grille Phase 2. Ces deux paires ne doivent jamais être confondues.

---

#### `simulate-one-point` — `POST /webhook/simulate-one-point`

Sub-workflow. Reçoit les salariés avec trajets actuels pré-calculés. Calcule les trajets candidats + scoring. Retourne `risque_global` + liste salariés + `statut` (ok / partiel si > 10% d'erreurs Maps).

---

#### `clusters-carte` — `GET /webhook/clusters-carte`

K-means K=4 pondéré par `score_competence`, 3 runs, meilleure inertie. Reverse geocoding des 4 centroïdes. Retourne `salaries[]` (lat/lng/score uniquement) + `clusters[]` (lat_centre, lng_centre, nb_salaries, nb_score5, label).

---

#### `import-salaries` — `POST /webhook/import-salaries`

Parse CSV → validation structure → sub-workflow résolution ids (1 appel Airtable filtré) → boucle insert/update → rapport détaillé.

**JSON de sortie :**
```json
{
  "statut": "ok" | "erreur_structure" | "import_rejete",
  "inserts": 2,
  "updates": 1,
  "rejetes": 0,
  "lignes_rejetees": [
    { "ligne_numero": 3, "id_salarie": "EMP003", "erreurs": ["adresse_domicile vide"] }
  ]
}
```

---

#### `geocode-salaries` — `POST /webhook/geocode-salaries`

Géocode les salariés sans coordonnées GPS. Écrit `lat` et `lng` dans Airtable.

---

#### `vider-base` — `POST /webhook/vider-base`

Supprime tous les records de la table Salariés en batches. Réponse : `{ statut: "ok", deleted: N }`.

---

#### `stats-base` — `GET /webhook/stats-base`

Nœud Airtable natif Return All. Réponse : `{ nb_salaries: N }`.

---

#### `stats-agent` — `GET /webhook/stats-agent`

Réponse : `{ nb_salaries, nb_geocodes, nb_a_geocoder, derniere_maj }`.

---

#### `search-salarie` — `GET /webhook/search-salarie?id=EMP001`

Réponse : `{ statut: "ok" | "not_found", salarie: { id_salarie, score_competence, adresse_geocodee, mode_transport, lat, lng } }`.

---

#### `get-quota` — `GET /webhook/get-quota`

Réponse : `{ quota_simulation_restant, quota_batch_restant }`.

---

#### `get-simulations` — `POST /webhook/get-simulations`

Filtre `simulations_historique` Airtable par correspondance exacte sur `adresse_actuelle`. Réponse : `{ simulations: [...] }` avec `points` JSON.parse.

---

### Modèle de scoring

```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  ≤ 20 min        → 0
  20 – 45 min     → 1
  > 45 min        → 2

f(delta) :
  delta ≤ 0        → 0
  0 < delta ≤ 30   → delta / 30
  delta > 30       → 1 + (delta - 30) / 10

risque_global = somme des risques individuels / nombre total de salariés
```

**Seuils de lecture :**

| Indicateur | Faible | Moyen | Élevé |
|---|---|---|---|
| Risque individuel | 0 – 2 | 2 – 8 | > 8 |
| Risque global | 0 – 1.5 | 1.5 – 4 | > 4 |
| Tolérance | 0 – 1 | 1 – 2 | > 2 |

---

### Google Maps APIs

**Distance Matrix API**
- Endpoint : `https://maps.googleapis.com/maps/api/distancematrix/json`
- Paramètres : `origins`, `destinations`, `mode`, `key`
- Batching : jusqu'à 25 origines par appel
- Unité retournée : secondes → convertir en minutes dans n8n
- Tarification : gratuit jusqu'à 40 000 éléments/mois

**Geocoding API**
- Endpoint : `https://maps.googleapis.com/maps/api/geocode/json`
- Utilisé pour : géocodage domiciles salariés + reverse geocoding centroïdes clusters

**Places Autocomplete**
- Utilisé côté front (Lovable) pour la saisie des adresses actuelle et candidate

---

### Lovable — interface

**Page simulation (`Index.tsx`, routes `/` et `/simulation`) :**
- Champ adresse actuelle — autocomplete Places, persistée en localStorage (`adresse_actuelle`, `adresse_actuelle_lat`, `adresse_actuelle_lng`)
- Champ adresse candidate — autocomplete Places
- Jauge risque global colorée + légende compacte
- Tableau salariés trié par risque individuel décroissant — colonnes : `id_salarie`, `score_competence`, `tolerance`, `delta_min`, `risque_individuel`
- Badges colorés + tooltips "?" sur Tolérance et Risque individuel
- Affichage quota restant (masqué en mode admin)
- Export XLS : 2 onglets (résumé global + détail salariés), nommé `simulation_YYYY-MM-DD.xlsx`

**Page carte (`Carte.tsx`) :**
- Carte Leaflet vanilla (CDN, `useEffect`) — hauteur `calc(100vh - 220px)`
- 4 clusters `L.circle` proportionnels (rayon géographique) + tooltip au survol
- Curseur repositionnable au clic → bouton "Simuler cette zone"
- Tableau 20 points : rank, zone, adresse, risque, badge Faible/Moyen/Élevé/Partiel
- Lignes partielles : fond grisé + tooltip explicatif
- Clic ligne → `mapRef.current.setView` sur le point
- Bouton "Voir le détail" → pré-remplissage `adresse_candidate` en localStorage + navigate('/')
- Cercles permanents numérotés et colorés (Airtable) — cliquables pour recharger résultats
- Historique : 5 blocs par défaut, "Charger plus" (+5), compteur X/25
- Export XLS par simulation, nommé `simulation_carte_YYYY-MM-DD.xlsx`
- Bouton recentrer barycentre
- Affichage quota batch restant (masqué en mode admin)

**Page import (`Import.tsx`) :**
- Compteur salariés en base (GET stats-base au chargement)
- Upload CSV avec validation 50 lignes côté front
- Rapport d'import : insérés ✅ / mis à jour 🔄 / rejetés ⚠️ + détail ligne par ligne
- Bouton "Vider la base" avec confirmation
- Export template CSV (SheetJS)

**Accès :**
- Écran login au chargement — flag `mode` (`client` / `admin`) persistent en localStorage
- Bouton déconnexion dans le header

**localStorage — état final :**

| Clé | Valeur | Origine |
|---|---|---|
| `adresse_actuelle` | string | Page Carte / Simulation |
| `adresse_actuelle_lat` | string (float) | Page Carte |
| `adresse_actuelle_lng` | string (float) | Page Carte |
| `mode` | `"client"` / `"admin"` | Écran login |

---

### GitHub — structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    brief-v2.md
    stack.md                    ← ce fichier
    build-log.md
    build-log-v2.md
    case-study.md
    workflow-n8n-documentation.docx
    relocation-risk-v2-presentation.docx
  /n8n
    workflow-relocation-simulation.json
    workflow-simulate-batch.json
    workflow-simulate-one-point.json
    workflow-import-salaries.json
    workflow-geocode-salaries.json
    workflow-vider-base.json
    workflow-stats-base.json
    workflow-stats-agent.json
    workflow-search-salarie.json
    workflow-clusters-carte.json
    workflow-get-quota.json
    workflow-get-simulations.json
  /data
    sample-data.csv
  /screenshots
```

---

## Ce qui est hors scope V2

- Multi-tenant — isolation des données par client → V3
- Simulation avec horaires réels — `departure_time` = `heure_debut_service` - 40min → V3
- Import > 50 lignes en une seule passe — pattern asynchrone → V3
- Page paramètres — seuils configurables via UI → V3
- Authentification robuste — mots de passe hardcodés en V2, variables d'environnement si déploiement Vercel → V3
