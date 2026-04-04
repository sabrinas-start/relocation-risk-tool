# Case Study — Relocation Risk Tool

> MVP client — Build no-code/IA de A à Z : logique métier, architecture, orchestration API, interface.

---

## Le problème métier

Une entreprise industrielle doit déménager ses locaux. Elle emploie des profils techniques rares, non remplaçables à court terme, pour lesquels l'expérience prime sur la disponibilité du marché.

Le risque : un allongement des temps de trajet pousse ces profils clés à refuser de suivre. Sans outil de simulation, la décision de localisation se prend à l'aveugle — ou avec des données partielles collectées manuellement.

**Le besoin** : pouvoir simuler l'impact de plusieurs adresses candidates sur la rétention des salariés, avant de prendre une décision.

---

## La solution

Un outil de simulation et d'aide à la décision en deux pages.

Chaque salarié dispose d'un **score de compétence** renseigné par les RH sur une grille de 1 à 5 (décimales acceptées). Ce score reflète la valeur du profil pour l'entreprise — il pondère directement le calcul de risque individuel. C'est la seule donnée subjective du modèle ; toutes les autres sont calculées automatiquement.

**Page simulation** — L'utilisateur saisit une adresse candidate. L'outil recalcule automatiquement les trajets domicile/travail de chaque salarié via Google Maps, calcule un score de risque individuel et global, et affiche les résultats triés par criticité avec export XLS.

**Page import** — L'équipe RH alimente et maintient la base salariés via import CSV. Merge sur identifiant, rapport détaillé ligne par ligne, compteur en temps réel, bouton de remise à zéro.

L'outil est utilisable par un profil RH non technique, sans formation.

---

## Architecture

```
Lovable (front React)
    │
    ├── POST /webhook/relocation-simulation  ──→  n8n
    │       { adresse_candidate, adresse_actuelle }    │
    │                                                  ├── Airtable (lecture salariés)
    │                                                  ├── Google Maps API × 2 / salarié
    │                                                  └── Calcul scoring
    │       ← { risque_global, salaries: [...] }
    │
    ├── POST /webhook/import-salaries        ──→  n8n
    │       { csv: "string brute" }                    │
    │                                                  ├── Parse + validation
    │                                                  ├── Sub-workflow résolution ids
    │                                                  └── Airtable (insert / update)
    │       ← { statut, inserts, updates, rejetes, lignes_rejetees }
    │
    ├── POST /webhook/vider-base             ──→  n8n
    │       ← { statut, deleted: N }
    │
    └── GET  /webhook/stats-base             ──→  n8n
            ← { nb_salaries: N }
```

| Rôle | Outil | Raison |
|---|---|---|
| Base de données | Airtable | Connecteur n8n natif, plug-and-play |
| Orchestration + API | n8n | Boucle sur salariés, appels Maps, calcul scoring |
| Calcul trajets | Google Maps Distance Matrix API | Précis, multi-modes de transport |
| Interface | Lovable | Front React généré en prompt |
| Versionning | GitHub | Export workflows + docs markdown |

---

## Les décisions de conception

### Modèle de scoring — deux dimensions indépendantes

La première version du scoring était `risque_individuel = score^1.5 × f(delta)`. Elle capturait bien la valeur du profil, mais ignorait un signal métier important : un salarié qui fait déjà 1h de trajet n'a pas la même tolérance à un allongement supplémentaire qu'un salarié qui fait 15 min.

La formule finale introduit une dimension de tolérance indépendante du score de compétence :

```
risque_individuel = (score^1.5 + tolérance(trajet_actuel)) × f(delta)

tolérance(trajet_actuel) :
  ≤ 10 min        → 0     (marge confortable)
  10 – 20 min     → 0.5   (zone intermédiaire)
  20 – 30 min     → 1     (seuil critique)
  > 30 min        → 1 + (trajet - 30) / 15  (s'emballe)

f(delta) :
  delta ≤ 0        → 0
  0 < delta ≤ 30   → delta / 30
  delta > 30       → 1 + (delta - 30) / 10
```

Les deux dimensions s'additionnent avant d'être appliquées au delta — un expert déjà à 45 min de trajet cumule une forte valeur ET une forte fragilité.

**Pourquoi `score^1.5` et pas `score^2`** : `score^2` créait des écarts trop extrêmes entre un score 2 et un score 5. `score^1.5` amplifie suffisamment les profils rares sans écraser les profils intermédiaires.

**Note de lecture** : avec cette formule, un delta élevé sur un profil faible peut dépasser un profil critique modérément impacté. Le tableau se lit en croisant risque individuel et score de compétence — les deux colonnes ensemble.

### Deux appels Maps par salarié, à chaque simulation

Le trajet actuel n'est pas stocké en base. Il est recalculé à chaque simulation.

Raison : si l'adresse des locaux actuels change un jour, il ne faut pas mettre à jour 100 lignes manuellement. L'API fait foi — les données Maps tiennent compte des conditions réelles de transport. Le delta est toujours frais, toujours cohérent.

### Adresse actuelle en localStorage, pas hardcodée en n8n

L'adresse des locaux actuels était initialement hardcodée dans un Set node n8n. Déplacée côté front en localStorage : l'outil devient utilisable pour n'importe quelle entreprise sans toucher au workflow. L'utilisateur la saisit une fois, elle est pré-remplie à chaque session.

Limite assumée : liée au navigateur. Persistance côté serveur prévue en V2 (multi-tenant).

### Import CSV avec sub-workflow de résolution des ids

La logique de merge (insert vs update) nécessite de résoudre les ids Airtable existants avant de traiter chaque ligne. Plutôt qu'un nœud statique intermédiaire, un sub-workflow dédié "Résolution ids — Import CSV" est appelé en amont. Plus propre, plus maintenable, extensible indépendamment du workflow principal.

### Limite 50 lignes par import — assumée, pas subie

Le timeout Lovable est à 2 minutes. Un import de 50 lignes prend ~1min30 mesuré. La limite est documentée et gérée proprement : imports successifs possibles, les lignes déjà présentes sont mises à jour, les nouvelles sont insérées. C'est une contrainte MVP assumée, pas un bug.

### Stats-base en HTTP Request, pas en node Airtable natif

Le node Airtable natif coupe le flux sur 0 résultats — comportement bloquant pour afficher "Base vide" dans l'interface. Remplacé par un HTTP Request sur l'API REST Airtable qui retourne toujours `{ records: [] }`, même sur une base vide. Le flux continue dans tous les cas.

---

## Les limites connues et les choix assumés

**Mono-client** — la base Airtable est partagée. Pas d'isolation par client. Extension multi-tenant prévue en V2.

**Seuils calibrés sur un scénario banlieue → banlieue ~15km** — les seuils vert/orange/rouge sont calibrés sur un périmètre de déménagement réaliste. Ils sont à recalibrer si le contexte change (déménagement plus lointain, population très urbaine, etc.). Une page de paramétrage des seuils est prévue en V2.

**Adresses approximées** — Google Maps peut géolocaliser au centre d'un arrondissement si l'adresse est incomplète, sans remonter d'erreur. Signalé dans la documentation utilisateur.

**RGPD** — pas de nom, prénom, ou données identifiantes en base. L'`id_salarie` anonymisé est la seule référence.

---

## Évolutions prévues — V2

- **Agent IA zones optimales** — analyse la distribution géographique des domiciles et des modes de transport, suggère des zones/quartiers optimaux cliquables qui déclenchent directement une simulation
- **Page de paramétrage** — seuils risque individuel, global et tolérance configurables via interface
- **Simulation avec horaires réels** — `departure_time` = `heure_debut_service` - 40 min par salarié, deux nouveaux champs Airtable
- **Multi-tenant** — isolation des données par client, adresse actuelle persistée côté serveur
