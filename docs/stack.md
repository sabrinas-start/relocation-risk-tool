# Stack — Relocation Risk Tool

## Vue d'ensemble

| Rôle | Outil | Raison |
|---|---|---|
| Base de données | **Airtable** | Connecteur n8n natif, plug-and-play, éditable par les RH |
| Orchestration + API | **n8n** | Boucle sur salariés, appel Maps API, calcul score |
| Calcul trajets | **Google Maps Distance Matrix API** | Précis, supporte voiture / transports en commun / vélo / pieds |
| Interface | **Lovable** | Front généré en prompt, pas de code front à écrire |
| Doc + base de connaissance | **Notion** | Centralisation projet + build log + apprentissages |
| Versionning | **GitHub** | Export workflows n8n + docs markdown |

---

## Détail par brique

### Airtable — base salariés

Table unique `Salariés` avec les colonnes suivantes :

| Champ | Type | Description |
|---|---|---|
| `id_salarie` | Text | Identifiant anonymisé (ex. EMP001) |
| `code_postal_domicile` | Text | Code postal du domicile |
| `mode_transport` | Single select | `driving` / `transit` / `bicycling` / `walking` |
| `trajet_actuel_min` | Number | Temps de trajet actuel en minutes |
| `score_competence` | Number (1–5) | Score défini par les RH |
| `adresse_actuelle_locaux` | Text | Adresse des locaux actuels (pré-remplie) |

> Note RGPD : pas de nom, prénom, ou données identifiantes. L'ID anonymisé est la seule référence.

---

### n8n — workflow principal

**Flux MVP :**

1. **Trigger** : webhook appelé par Lovable avec l'adresse candidate
2. **Airtable node** : récupère tous les salariés
3. **Loop over items** : pour chaque salarié
   - Appel **Google Maps Distance Matrix API** avec origine = code postal domicile, destination = adresse candidate, mode = `mode_transport`
   - Calcul du delta trajet = `trajet_futur - trajet_actuel_min`
   - Calcul du risque individuel = `score_competence × f(delta)`
4. **Aggregate** : calcul du score de risque global (somme pondérée)
5. **Respond to webhook** : renvoie le JSON à Lovable

**Fonction de risque f(delta) :**

```
si delta <= 0       → f = 0
si 0 < delta <= 30  → f = delta / 30
si delta > 30       → f = 1 + (delta - 30) / 10   ← s'emballe
```

---

### Google Maps Distance Matrix API

- Endpoint : `https://maps.googleapis.com/maps/api/distancematrix/json`
- Paramètres : `origins`, `destinations`, `mode`, `key`
- Unité retournée : secondes → convertir en minutes dans n8n
- Modes supportés : `driving`, `transit`, `bicycling`, `walking`
- Tarification : gratuit jusqu'à 40 000 éléments/mois

---

### Lovable — interface

**Pages MVP :**

1. **Saisie adresse candidate** : champ texte + bouton "Simuler"
2. **Résultats** :
   - Score de risque global (score + jauge visuelle)
   - Tableau des salariés trié par risque individuel décroissant (colonnes : ID, delta trajet, risque individuel)

Lovable appelle le webhook n8n et affiche la réponse JSON.

---

### GitHub — structure du repo

```
/relocation-risk-tool
  README.md
  /docs
    brief.md
    stack.md          ← ce fichier
    build-log.md
  /n8n
    workflow-export.json
  /data
    sample-data.csv
```

---

## Ce qui est hors scope MVP

- Authentification / gestion des droits
- Mode cartographie (grille géographique + carte de chaleur) → Phase 2
- Synchronisation RH automatique
- Prise en compte des horaires (heure de pointe vs hors pointe)
