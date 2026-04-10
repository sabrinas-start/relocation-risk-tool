# Case Study — Relocation Risk Tool

> MVP client — Build no-code/IA de A à Z : logique métier, architecture, orchestration API, interface.

---

## Le problème métier

Une entreprise industrielle doit déménager ses locaux. Elle emploie des profils techniques rares, non remplaçables à court terme, pour lesquels l'expérience prime sur la disponibilité du marché.

Le risque : un allongement des temps de trajet pousse ces profils clés à refuser de suivre. Sans outil de simulation, la décision de localisation se prend à l'aveugle — ou avec des données partielles collectées manuellement.

**Le besoin** : pouvoir simuler l'impact de plusieurs adresses candidates sur la rétention des salariés, et explorer des zones géographiques pour orienter la recherche immobilière — avant de prendre une décision.

---

## La solution

Un outil de simulation et d'aide à la décision en trois pages, couvrant deux cas d'usage distincts.

Chaque salarié dispose d'un **score de compétence** renseigné par les RH sur une grille de 1 à 5 (décimales acceptées). Ce score reflète la valeur du profil pour l'entreprise — il pondère directement le calcul de risque individuel. C'est la seule donnée subjective du modèle ; toutes les autres sont calculées automatiquement.

**Page simulation** — L'entreprise a une adresse candidate identifiée. L'outil recalcule automatiquement les trajets domicile/travail de chaque salarié via Google Maps, calcule un score de risque individuel et global, et affiche les résultats triés par criticité avec export XLS.

**Page carte** — L'entreprise ne sait pas encore vers où chercher. Une carte interactive affiche les clusters de domiciles salariés pondérés par score de compétence. L'utilisateur pose un curseur sur une zone d'intérêt et lance une simulation batch sur une grille de 20 points autour de ce curseur — classés par risque global croissant. Chaque point de résultat est cliquable pour lancer une simulation précise sur cette adresse, sans ressaisie.

Les deux pages forment un flux de décision continu : exploration de zones → identification d'une adresse candidate → simulation détaillée.

**Page import** — L'équipe RH alimente et maintient la base salariés via import CSV. Merge sur identifiant, rapport détaillé ligne par ligne, compteur en temps réel, bouton de remise à zéro.

L'outil est utilisable par un profil RH non technique, sans formation.

---

## Architecture

```
Lovable (front React / Leaflet)
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

| Rôle | Outil | Raison |
|---|---|---|
| Base de données | Airtable | Connecteur n8n natif, plug-and-play |
| Orchestration + API | n8n (12 workflows) | Boucles, appels Maps, calcul scoring, sous-workflows |
| Calcul trajets | Google Maps Distance Matrix API | Précis, multi-modes de transport |
| Géocodage / Autocomplete | Google Maps Geocoding + Places API | Coordonnées GPS salariés + saisie adresse |
| Interface | Lovable (React / Leaflet) | Front généré en prompt, carte interactive |
| Export tableur | SheetJS | Export XLS côté front, sans serveur |
| Versionning | GitHub | Export workflows + docs markdown |

---

## Les décisions de conception

### Modèle de scoring — deux dimensions indépendantes

La première version du scoring était `risque_individuel = score^1.5 × f(delta)`. Elle capturait bien la valeur du profil, mais ignorait un signal métier important : un salarié qui fait déjà 1h de trajet n'a pas la même tolérance à un allongement supplémentaire qu'un salarié qui fait 15 min.

La formule finale introduit une dimension de tolérance indépendante du score de compétence :

```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  ≤ 20 min        → 0     (marge confortable)
  20 – 45 min     → 1     (zone intermédiaire)
  > 45 min        → 2     (habitué aux longs trajets, proche de la limite)

f(delta) :
  delta ≤ 0        → 0
  0 < delta ≤ 30   → delta / 30
  delta > 30       → 1 + (delta - 30) / 10
```

Les deux dimensions s'additionnent avant d'être appliquées au delta — un expert déjà à 45 min de trajet cumule une forte valeur ET une forte fragilité.

**Pourquoi `score^1.5` et pas `score^2`** : `score^2` créait des écarts trop extrêmes entre un score 2 et un score 5. `score^1.5` amplifie suffisamment les profils rares sans écraser les profils intermédiaires.

**Note de lecture** : avec cette formule, un delta élevé sur un profil faible peut dépasser un profil critique modérément impacté. Le tableau se lit en croisant risque individuel et score de compétence — les deux colonnes ensemble.

### Deux cas d'usage, un flux de décision continu

La carte et la simulation ne sont pas deux outils indépendants — elles forment une séquence. La carte permet d'identifier les zones prometteuses via une simulation batch sur grille de 20 points. Chaque point de résultat est directement cliquable pour pré-remplir la page Simulation avec cette adresse et lancer une analyse détaillée. Le flux passe naturellement de l'exploration au diagnostic sans ressaisie.

### Simulation batch — exploration guidée, pas grille automatique

La première approche V2 générait une grille automatique centrée sur le barycentre pondéré des salariés, avec un rayon calculé par la distance Haversine moyenne. En production, le rayon moyen était tiré par les outliers géographiques — la grille couvrait des zones non pertinentes.

L'approche retenue est une exploration guidée : l'utilisateur voit où vivent ses profils critiques sur la carte (clusters k-means K=4 pondérés par score de compétence), choisit une zone d'intérêt, et lance une simulation resserrée de 2 km autour de son curseur. L'interface met la décision dans les mains de l'utilisateur, pas dans un algorithme opaque.

### Trajet actuel calculé une seule fois pour 20 points

La simulation batch aurait pu recalculer le trajet domicile → bureau actuel pour chaque point — soit 20 fois le même calcul. Le trajet actuel est identique quel que soit le point candidat. Il est calculé en Phase 1 pour tous les salariés, stocké en mémoire, et réutilisé en Phase 2. Volume d'appels Maps divisé par 20 sur la composante la plus coûteuse.

### K-means pondéré pour les clusters carte

Les clusters de domiciles salariés affichés sur la carte sont calculés avec un k-means K=4 pondéré par `score_competence`. Un salarié score 5 contribue 2,5× plus au positionnement d'un centroïde qu'un salarié score 2. Le résultat : les clusters se positionnent autour des zones où vivent les profils les plus critiques — directement utiles pour orienter le curseur de simulation.

### Persistance des simulations en Airtable, pas en localStorage

L'historique des simulations batch était initialement stocké en localStorage. Limite identifiée en conditions réelles : un changement d'appareil ou de navigateur efface tout. La persistance a été migrée vers une table Airtable `simulations_historique`. L'historique survit à tout changement d'environnement, et les cercles permanents sur la carte sont reconstruits automatiquement au chargement.

Règle fondamentale retenue : Airtable est la mémoire permanente. Le client ne peut jamais supprimer un record depuis l'interface.

### Deux appels Maps par salarié, à chaque simulation

Le trajet actuel n'est pas stocké en base. Il est recalculé à chaque simulation.

Raison : si l'adresse des locaux actuels change un jour, il ne faut pas mettre à jour 100 lignes manuellement. L'API fait foi — les données Maps tiennent compte des conditions réelles de transport. Le delta est toujours frais, toujours cohérent.

### Adresse actuelle en localStorage, pas hardcodée en n8n

L'adresse des locaux actuels était initialement hardcodée dans un Set node n8n. Déplacée côté front en localStorage : l'outil devient utilisable pour n'importe quelle entreprise sans toucher au workflow. L'utilisateur la saisit une fois, elle est pré-remplie à chaque session.

### Import CSV avec sub-workflow de résolution des ids

La logique de merge (insert vs update) nécessite de résoudre les ids Airtable existants avant de traiter chaque ligne. Un sub-workflow dédié "Résolution ids — Import CSV" est appelé en amont : 1 seul appel Airtable filtré sur les ids du CSV, construit une map `id_salarie → record_id`. Plus propre, plus maintenable, extensible indépendamment du workflow principal.

### Limite 50 lignes par import — assumée, pas subie

Le timeout Lovable est à 2 minutes. Un import de 50 lignes prend ~1min30 mesuré. La limite est documentée et gérée proprement : imports successifs possibles, les lignes déjà présentes sont mises à jour, les nouvelles sont insérées. C'est une contrainte MVP assumée, pas un bug.

---

## Les limites connues et les choix assumés

**Mono-client** — la base Airtable est partagée. Pas d'isolation par client. Extension multi-tenant prévue en V3.

**Limite 300 salariés pour la simulation batch** — au-delà, le temps d'exécution dépasse le timeout synchrone (~76 secondes pour 300 salariés). Limite documentée dans l'interface. Pattern asynchrone prévu en V3.

**Seuils calibrés sur un scénario banlieue → banlieue ~15km** — à recalibrer si le contexte change (déménagement plus lointain, population très urbaine). Une page de paramétrage est prévue en V3.

**Correspondance exacte sur l'adresse actuelle pour l'historique** — le filtrage des simulations dans Airtable repose sur une correspondance exacte de l'adresse actuelle. Risque marginal si l'adresse est saisie manuellement. Mitigation : l'adresse vient toujours de Google Places Autocomplete, normalisée à chaque session.

**RGPD** — pas de nom, prénom, ou données identifiantes en base. L'`id_salarie` anonymisé est la seule référence. Les clusters carte exposent uniquement `lat`, `lng` et `score_competence` — pas d'`adresse_domicile`.

---

## Évolutions prévues — V3+

- **Multi-tenant** — isolation des données par client, adresse actuelle persistée côté serveur
- **Simulation avec horaires réels** — `departure_time` = `heure_debut_service` - 40 min par salarié, deux nouveaux champs Airtable
- **Page paramètres** — seuils risque individuel, global et tolérance configurables via interface, stockés en Airtable
- **Import > 50 lignes** — pattern asynchrone avec polling
