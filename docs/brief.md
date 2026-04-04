# Brief — Relocation Risk Tool

## Contexte

Une entreprise doit déménager ses locaux. Elle emploie des profils techniques rares pour lesquels l'expérience prime. Le risque principal : que l'allongement des temps de trajet pousse des salariés clés à ne pas suivre, sans possibilité de remplacement à court terme.

## Besoin

Disposer d'un outil permettant de simuler l'impact d'un déménagement sur la rétention des salariés, en tenant compte de la valeur individuelle de chaque profil et de l'évolution de son temps de trajet.

## Approche solution

**Mode simulation** — L'utilisateur saisit une adresse candidate pour les futurs locaux. L'outil recalcule le temps de trajet domicile/travail pour chaque salarié selon son mode de transport, compare avec le trajet actuel, et produit un score de risque individuel et global.

## Modèle de scoring

- Chaque salarié dispose d'un **score de compétence** (1 à 5, décimales acceptées) défini par les RH
- Le **delta de trajet** = temps de trajet futur − temps de trajet actuel
- Le **risque individuel** = `(score^1.5 + tolérance(trajet_actuel)) × f(delta)`
- Le **risque global** = somme des risques individuels / nombre total de salariés

### Fonction de tolérance

La tolérance capture la fragilité d'un salarié déjà proche de sa limite de trajet acceptable. Elle s'additionne au score de compétence — les deux dimensions sont indépendantes.

```
trajet_actuel ≤ 10 min        → t = 0
10 < trajet_actuel ≤ 20 min   → t = 0.5
20 < trajet_actuel ≤ 30 min   → t = 1
trajet_actuel > 30 min        → t = 1 + (trajet_actuel - 30) / 15
```

### Fonction f(delta)

```
delta ≤ 0        → f = 0
0 < delta ≤ 30   → f = delta / 30
delta > 30       → f = 1 + (delta - 30) / 10   ← s'emballe
```

### Seuils de lecture

| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 2 | 2 – 8 | > 8 |
| Risque global | 0 – 1.5 | 1.5 – 4 | > 4 |
| Tolérance | 0 – 1 | 1 – 2 | > 2 |

> Seuils calibrés sur un scénario banlieue → banlieue (~15 km max). À revoir sur données réelles.

> ⚠️ Limite connue : un delta très élevé sur un profil faible peut dépasser un profil critique modérément impacté. Lire le tableau en croisant `risque_individuel` et `score_competence`.

## Périmètre MVP

- Import de la base salariés via fichier CSV depuis l'interface — merge sur `id_salarie` (insert + update, sans suppression), limite 50 lignes par import
- Base salariés : identifiant anonymisé, adresse domicile complète, mode de transport, score de compétence
- Simulation adresse par adresse
- Résultat : tableau trié par niveau de risque + score global avec jauge colorée
- Export XLS des résultats (résumé global + détail par salarié)
- Interface simple, utilisable par un profil RH non technique

## Décisions d'architecture

**Trajets calculés dynamiquement** — Le temps de trajet actuel n'est pas stocké en base. n8n appelle Google Maps deux fois par salarié à chaque simulation (domicile → locaux actuels, domicile → adresse candidate). La donnée est toujours fraîche et cohérente.

**Import CSV avec merge** — L'`id_salarie` est la clé de merge. Une ligne existante est mise à jour, une ligne nouvelle est insérée, une ligne absente du fichier est conservée. L'`id_salarie` est immuable — un changement d'id crée un doublon.

**Adresse actuelle en localStorage** — L'adresse des locaux actuels est saisie une fois dans l'interface et persistée en localStorage. Elle est envoyée à chaque simulation depuis le front.

**Usage mono-client** — La base Airtable est partagée et non isolée par client. Le multi-tenant est identifié comme évolution V2.

**Données RGPD** — Aucun nom, prénom ou donnée identifiante. L'`id_salarie` anonymisé est la seule référence. Le build s'effectue sur données fictives géographiquement réalistes.

**Score^1.5** — Le score de compétence est élevé à la puissance 1.5 pour amplifier l'écart entre profils rares et profils courants sans l'écraser.

## Hors périmètre MVP

- Authentification et gestion des droits
- Multi-tenant — isolation des données par client
- Synchronisation RH automatique
- Simulation avec horaires réels (`departure_time`)
- Import > 50 lignes en une seule passe
- Suppression de lignes à l'import

## Évolutions prévues (V2)

- **Simulation avec horaires réels** — `departure_time` = `heure_debut_service` - 40 min, deux nouveaux champs Airtable
- **Agent IA recommandation zones** — suggestions cliquables basées sur la distribution géographique des domiciles + scores, clic déclenche une simulation directe
- **Multi-tenant** — isolation des données par client, adresse actuelle persistée côté serveur
- **Import > 50 lignes** — bulk insert en une seule passe
