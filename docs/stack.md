# Stack — Relocation Risk Tool
 
## Vue d'ensemble
 
| Rôle | Outil | Raison |
|---|---|---|
| Base de données | **Airtable** | Connecteur n8n natif, plug-and-play, alimentée via import CSV |
| Orchestration + API | **n8n** | Deux workflows : simulation + import CSV |
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
| `score_competence` | Number (1–5) | Score défini par les RH |
 
> Note RGPD : pas de nom, prénom, ou données identifiantes. L'`id_salarie` anonymisé est la seule référence.
 
---
 
### n8n — workflow simulation
 
**Flux :**
 
1. **Webhook** : reçoit l'adresse candidate depuis Lovable
2. **Set node** : injecte l'adresse des locaux actuels (paramètre global)
3. **Airtable** : récupère tous les salariés
4. **Loop over items** : pour chaque salarié
   - Appel Maps API : `adresse_domicile` → locaux actuels
   - Appel Maps API : `adresse_domicile` → adresse candidate
   - Calcul `delta` = trajet futur − trajet actuel
   - Calcul `risque_individuel` = `score^1.5 × f(delta)`
5. **Aggregate** : consolide les résultats
6. **Code node** : calcul `risque_global` = somme des risques / nombre de salariés
7. **Respond to webhook** : renvoie le JSON à Lovable
 
**Fonction f(delta) :**
 
```
si delta <= 0       → f = 0
si 0 < delta <= 30  → f = delta / 30
si delta > 30       → f = 1 + (delta - 30) / 10   ← s'emballe
```
 
**JSON de sortie :**
 
```json
{
  "risque_global": 8.58,
  "salaries": [
    {
      "id_salarie": "EMP001",
      "score_competence": 5,
      "trajet_actuel_min": 35,
      "trajet_futur_min": 72,
      "delta_min": 37,
      "risque_individuel": 15.32
    }
  ]
}
```
 
---
 
### n8n — workflow import CSV
 
**Flux :**
 
1. **Webhook** : reçoit la string CSV brute depuis Lovable
2. **Code — Parse CSV** : validation structure (4 colonnes exactes) → rejet global si invalide
3. **Code — Validation lignes** : validation champ par champ → deux tableaux : `lignes_valides` et `lignes_rejetees`
4. **Code — Eclate lignes** : tableau → items n8n distincts + reset compteurs static data
5. **Loop over items** : pour chaque ligne valide
   - **Airtable Search** : cherche l'`id_salarie`
   - **Code garde-fou** : intercepte le cas 0 résultats → `found: true/false`
   - **IF** : route vers Update (id existant) ou Create (nouvel id)
   - **Code Tag** : incrémente `store.inserts` ou `store.updates` → retour boucle
6. **Code — Rapport** : lit les compteurs static data
7. **Respond to webhook** : renvoie `{ inserts, updates, lignes_rejetees }` à Lovable
 
**Règles d'import :**
- Template CSV strict — 4 colonnes exactes dans cet ordre : `id_salarie`, `adresse_domicile`, `mode_transport`, `score_competence`
- Merge sur `id_salarie` : insert si nouvel id, update si id existant, conservation si id absent du fichier
- `id_salarie` immuable — un changement d'id crée un doublon
 
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
- Champ texte adresse candidate + bouton "Simuler"
- Jauge risque global colorée (vert < 2 / orange 2–6 / rouge > 6)
- Tableau salariés trié par risque individuel décroissant (colonnes : id_salarie, score_competence, delta_min, risque_individuel)
- Couleurs risque individuel : gris = 0 / vert < 3 / orange 3–12 / rouge > 12
- Badges score compétence colorés : vert = 1–2 / orange = 3–4 / rouge = 5
- Légende des seuils affichée sous la jauge
 
**Page import CSV :**
- Upload fichier CSV (drag & drop ou clic)
- Rapport d'import : X insérés, Y mis à jour
- Gestion erreur structure CSV
- Gestion erreur réseau
 
Lovable appelle les webhooks n8n et affiche les réponses JSON.
 
---
 
### GitHub — structure du repo
 
```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md            ← ce fichier
    build-log.md
    workflow-n8n-documentation.docx
    workflow-import-documentation.docx
  /n8n
    workflow-simulation.json
    workflow-import.json
  /data
    sample-data.csv
```
 
---
 
## Ce qui est hors scope MVP
 
- Authentification / gestion des droits
- Mode cartographie (grille géographique + carte de chaleur) → Phase 2
- Multi-tenant — isolation des données par client → Phase 2
- Synchronisation RH automatique
- Prise en compte des horaires (heure de pointe vs hors pointe)
- Agent IA d'interprétation des résultats → backlog immédiat post-MVP
