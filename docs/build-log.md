# Build Log — Relocation Risk Tool

## Format d'entrée

```
### [Date] — [Étape]
**Fait :** ce qui a été produit / décidé
**Apprentissage :** ce qui a été compris / découvert
**Blocage éventuel :** ce qui a posé problème et comment c'est résolu
**Prochaine étape :**
```

---

## Sessions

### 2025-XX-XX — Cadrage et stack

**Fait :**
- Faisabilité validée
- Stack choisie : Airtable + n8n + Google Maps API + Lovable
- Modèle de scoring défini
- Brief rédigé
- Contrainte RGPD identifiée → build sur données fictives géographiquement réalistes
- Fichiers produits : `brief.md`, `stack.md`, `sample-data.csv`

**Apprentissage :**
- La Distance Matrix API retourne des secondes → conversion en minutes nécessaire dans n8n
- La fonction de risque doit s'emballer au-delà de +30 min pour être réaliste (seuil de tolérance trajet)

**Prochaine étape :** Connexion Airtable dans n8n + premier appel Google Maps API

---

<!-- Ajouter les sessions suivantes ici -->
