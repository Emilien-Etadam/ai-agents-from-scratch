# Explication du Code : `tree-of-thought.js`

Cette walkthrough suit la structure réelle du code, pour que vous puissiez mapper chaque concept ToT à des fonctions concrètes.

## Run

```bash
node examples/12_tree-of-thought/tree-of-thought.js
```

---

## 1) Setup : model, schemas, constantes

En haut du fichier :

- `HYPOTHESIS_TYPES` définit les quatre branches concurrentes.
- `BEHAVIOR_INPUT` est la description du cas.
- `hypothesisSchema`, `scoreSchema`, `rankingSchema`, `analysisSchema` définissent les contrats JSON pour chaque phase.
- `promptJson(schema, userText)` est la utility partagée qui :
  - reset l'historique de chat,
  - impose la grammar de schema,
  - parse/répare le JSON.

Cela garde chaque fonction de phase focalisée sur la logique, pas sur le boilerplate de parser.

---

## 2) Phase 1 (Branch) : `developHypothesis()`

`developHypothesis(behavior, hypothesisType)` fait une seule chose :

- prompt le model pour raisonner à travers exactement un lens,
- retourne un objet structuré :
  - `name`
  - `argument`
  - `signals`
  - `counter_evidence`

Dans `runTreeOfThoughtMotivationAnalysis()`, ça tourne dans une boucle sur `HYPOTHESIS_TYPES`, créant quatre branches concurrentes.

---

## 3) Phase 2 (Score) : `scoreHypothesis()` + `rerankHypotheses()`

### Scoring raw par branche

`scoreHypothesis(behavior, hypothesis)` retourne :

- `score` (score numérique raw de la formule),
- `details` (`explanatory_power`, `plausibility`, `falsifiability`),
- `blindSpot`,
- `reasoning`.

### Pass de calibration anti-égalité

`rerankHypotheses(behavior, scoredHypotheses)` force un ranking strict sans égalités puis map les rangs en scores calibrés :

- rank1 -> `8.8`
- rank2 -> `8.1`
- rank3 -> `7.4`
- rank4 -> `6.7`

C'est pourquoi la console affiche :

- les évaluations raw capturées
- puis les scores calibrés utilisés pour le pruning

Les apprenants voient ainsi ce que le système utilise *vraiment* pour la sélection de branche.

---

## 4) Phase 3 (Prune) : `pruneHypotheses()`

`pruneHypotheses(scoredHypotheses)` :

- trie par score décroissant,
- garde le winner,
- retourne les branches `discarded`.

C'est le cœur structurel du ToT dans cet exemple : un winner continue, les alternatives sont éliminées.

---

## 5) Phase 4 (Conclusion) : `createConclusion()`

`createConclusion(behavior, winner)` construit l'analyse finale en utilisant uniquement :

- le nom du winner
- l'argument du winner
- les signals du winner

Les branches éliminées ne nourrissent pas la réponse finale.
Cette limitation intentionnelle est montrée dans le bloc console : `WHAT TOT LOST IN THIS RUN`.

---

## 6) Flux d'Orchestration : `runTreeOfThoughtMotivationAnalysis()`

Cette fonction est le contrôleur end-to-end :

1. branch (collecter les hypothèses)
2. score (raw + calibré)
3. prune (winner + discarded)
4. conclude (winner uniquement)
5. afficher la sortie + appeler le helper de visualisation

La visualisation est intentionnellement déléguée à :

- `writeToTMotivationVisualization(...)`

pour que le fichier exemple reste focalisé sur le flow de contrôle ToT.

---

## Ordre Suggéré de Lecture du Code

Lire les fonctions dans cet ordre :

1. `promptJson`
2. `developHypothesis`
3. `scoreHypothesis`
4. `rerankHypotheses`
5. `pruneHypotheses`
6. `createConclusion`
7. `runTreeOfThoughtMotivationAnalysis`

Cet ordre mirror le flow runtime et rend le fichier beaucoup plus facile à comprendre.
