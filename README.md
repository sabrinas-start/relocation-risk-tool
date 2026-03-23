# relocation-risk-tool

Outil de simulation de risque RH lié à un déménagement de locaux.
Contexte
Quand une entreprise déménage, elle risque de perdre des profils techniques clés si l'allongement des temps de trajet dépasse leur seuil de tolérance. Cet outil permet de simuler cet impact avant de choisir une adresse.
Fonctionnement
L'utilisateur saisit une adresse candidate pour les futurs locaux. L'outil recalcule le temps de trajet domicile/travail pour chaque salarié, compare avec la situation actuelle, et produit un score de risque individuel et global.
Modèle de scoring :

Risque individuel = score de compétence (1–5) × f(delta trajet)
La fonction s'emballe au-delà de +30 min de trajet supplémentaire
Risque global = somme pondérée des risques individuels

Stack
- **Airtable** — base de données salariés
- **n8n** — orchestration + appel API
- **Google Maps Distance Matrix API** — calcul des trajets
- **Lovable** — interface utilisateur
  
Structure du repo
/relocation-risk-tool
  README.md
  /docs
    brief.md          — cahier des charges
    stack.md          — détail technique de la stack
    build-log.md      — journal de build session par session
  /n8n
    workflow-export.json
  /data
    sample-data.csv   — jeu de données fictif (RGPD)
Statut
🚧 En cours de build — MVP en développement
