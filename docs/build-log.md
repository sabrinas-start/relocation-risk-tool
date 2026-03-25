# Build log — Relocation Risk Tool

> Fichier évolutif — décisions prises, blocages rencontrés, apprentissages au fil du build.

---

## Session 1 — Fondations et documentation

**Date :** mars 2026

### Réalisé
- Cadrage complet : brief, modèle de scoring, stack validée
- Repo GitHub créé et structuré (`/docs` `/n8n` `/data`)
- `brief.md`, `stack.md`, `sample-data.csv` produits
- Base Airtable créée — 4 colonnes, données importées
- Clé Google Maps Distance Matrix API générée et restreinte

### Décisions
- Adresses domicile complètes plutôt que codes postaux — utilisables directement par Maps API sans transformation
- Valeurs `mode_transport` conservées en anglais — valeurs exactes attendues par Maps API, toute traduction = dette technique
- Champ `trajet_actuel_min` écarté d'Airtable — calculé dynamiquement à chaque simulation via Maps API
- Adresse des locaux actuels en paramètre global dans n8n (Set node) — pas dans Airtable

---

## Session 2 — Build workflow n8n simulation

**Date :** 23 mars 2026

### Réalisé
- Workflow n8n simulation complet — 10 nodes validés
- Tests de robustesse — 7 cas couverts (voir journal-tests.docx)
- Workflow exporté → `/n8n/workflow-simulation.json`

### Problèmes rencontrés
- **Adresse vide → plantage** — `rows[0]` undefined quand Maps reçoit une origine vide. Corrigé avec l'opérateur de chaînage optionnel `?.`.
- **Outputs mémorisés n8n** — quand un node est exécuté manuellement dans l'éditeur, son output est mémorisé et réutilisé par le workflow de production. Protocole obligatoire : vider tous les outputs avant tout test via URL de production.

### Décisions
- Garde-fous : opérateur `?.` sur la lecture Maps, vérification `status !== 'OK'` avant calcul
- `score_competence` aberrant (ex. 59) : validation Airtable (plage 1–5) à mettre en place avant mise en production avec vraies données RH

---

## Session 3 — Décisions d'architecture + workflow import CSV

**Date :** 24 mars 2026

### Décisions architecture
- **Import CSV via Lovable** — le client importe sa base via l'interface, pas via Airtable directement. Deux webhooks n8n distincts.
- **Merge sur `id_salarie`** — insert si nouvel id, update si id existant, conservation si id absent du fichier. L'`id_salarie` est immuable.
- **Template CSV strict** — 4 colonnes imposées. Fichier modèle téléchargeable depuis Lovable.
- **Mono-client assumé** — base partagée, pas d'isolation par client. Multi-tenant documenté en V2.

### Réalisé
- Workflow n8n import CSV complet — 12 nodes
- Pattern merge insert/update via Airtable Search + garde-fou + IF
- Static data (`$getWorkflowStaticData`) pour comptage inserts/updates entre itérations

### Problèmes rencontrés
- **Airtable Search — 0 résultats coupe le flux** — quand Search ne trouve rien, il ne produit aucun item. Le IF ne se déclenche jamais. Corrigé par un Code node garde-fou qui fabrique un item `{ found: false }`.
- **Droits Airtable** — token sans `data.records:write` → erreur 403 silencieuse sur Update et Create. Scope ajouté au token.

---

## Session 4 — Interface Lovable + refonte scoring

**Date :** 25 mars 2026

### Réalisé — page simulation
- Connexion front → webhook simulation déboguée (`json.resultats` → `json.salaries`)
- Jauge risque global colorée selon seuils
- Tableau salariés trié par risque décroissant
- Badges score compétence colorés (rouge = 5 / orange = 3–4 / vert = 1–2)
- Couleurs de fond colonne risque individuel selon seuils

### Réalisé — page import CSV
- Upload CSV (drag & drop ou clic)
- Rapport d'import : X insérés, Y mis à jour
- Gestion erreur structure CSV et erreur réseau

### Refonte modèle de scoring
- **Problème** : avec `score × f(delta)`, un profil score 2 à +45 min obtenait le même risque qu'un profil score 5 à +30 min.
- **Décision** : remplacer `score` par `Math.pow(score, 1.5)` pour amplifier l'écart entre profils.
- **Risque global** : passage de la somme brute à la moyenne simple (`somme / nombre de salariés`). La dilution est assumée — le tableau individuel porte les alertes.
- **Limite documentée** : un delta très élevé sur un profil faible peut encore dépasser un profil critique modérément impacté. À signaler aux RH.

### Seuils validés sur simulations réelles
| Indicateur | Vert | Orange | Rouge |
|---|---|---|---|
| Risque individuel | 0 – 3 | 3 – 12 | > 12 |
| Risque global | 0 – 2 | 2 – 6 | > 6 |

### Problèmes rencontrés
- **Parser CSV naif** — `split(',')` cassait sur les virgules dans les adresses entre guillemets. Remplacé par un parser caractère par caractère gérant les guillemets.
- **Mismatch noms de champs Lovable** — Lovable traduisait `inserts` en `insertions` et `updates` en `mises_a_jour`. Corrigé dans le code Lovable. Règle : ne jamais traduire les noms de champs JSON dans les prompts Lovable.
- **Désactivation/réactivation n8n** — après modification d'un node en production, désactiver puis réactiver le workflow pour que les changements prennent effet.

---

## Prochaines étapes

- [ ] Finitions UI — supprimer barre de progression + ajouter légendes seuils
- [ ] Nettoyage base Airtable (lignes de test à supprimer)
- [ ] Agent IA d'interprétation post-simulation (API Anthropic dans Lovable)
- [ ] Tests workflow import en prod + JSON → GitHub
- [ ] README.md final
- [ ] Case study Notion
