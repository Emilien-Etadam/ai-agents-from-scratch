# Explication du Code : `graph-of-thought.js`

C'est une walkthrough code-first de l'implémentation GoT utilisée dans l'Exemple 13.

## Run

```bash
node examples/13_graph-of-thought/graph-of-thought.js
```

---

## 1) Objet graphe central : `ThoughtGraph`

`ThoughtGraph` est la structure de données centrale.

### State stocké

- `nodes: Map<string, node>`
- `edges: Map<string, parentId[]>`
- `nextId` pour les IDs de nœuds séquentiels (`n1`, `n2`, ...)

### Méthodes clés

- `addNode(type, content, meta, parentIds)`
- `get(id)`
- `parents(id)`
- `byType(type)`
- `printGraph()` (trace debug de tous les nœuds + edges)

C'est ce qui rend l'exemple vraiment basé sur un graphe au lieu d'un arbre.

---

## 2) Utility d'appel JSON partagée : `promptJson()`

`promptJson(schema, userText)` est réutilisée par chaque opération :

- reset l'historique de chat,
- impose la grammar de schema,
- parse le JSON en sécurité.

Toutes les fonctions d'opération sont ensuite propres et focalisées sur la logique de graphe.

---

## 3) Fonctions de Phase (et le type de nœud que chacune crée)

### `branch(...)` -> nœuds `hypothesis`

- input : comportement root + lenses d'hypothèse
- output : un nœud par lens
- parent : toujours root

### `scoreAll(...)` -> met à jour `score` sur les nœuds hypothesis

- scoring de critères raw par hypothèse
- pass de reranking strict (pas d'égalités) avec spread calibré
- écrit le score de retour dans les nœuds du graphe

### `contrast(...)` -> nœud `contrast`

- input : deux nœuds hypothesis
- output : nœud de contradiction
- parents : les deux nœuds comparés

### `refine(...)` -> nœud `refined`

- input : nœud faible + nœud fort/contexte
- output : version améliorée de l'argument faible
- parents : les deux nœuds sources

### `aggregate(...)` -> nœud `synthesis`

- input : multiple nœuds sources
- output : synthèse intégrée
- parents : tous les nœuds sources

### `conclude(...)` -> nœud `conclusion`

- input : strands de haute valeur sélectionnés
- output : analyse finale intégrée
- parents : multiple nœuds synthesis/contrast/refined

---

## 4) Flow de Contrôleur : `runGoTMotivationAnalysis()`

Cette fonction orchestre tout :

1. créer `root`
2. brancher en 4 hypothèses
3. scorer + reranker les hypothèses
4. construire les nœuds contrast
5. raffiner les nœuds faibles/moyens
6. créer deux nœuds synthesis
7. conclure depuis multiple strands
8. afficher le graphe + narrative finale + générer la visualisation

L'étape de ranking affecte quels nœuds sont considérés `strongA`, `strongB`, `medium`, `weak`, ce qui influence ensuite la sélection contrast/refine.

---

## 5) Pourquoi c'est du GoT dans le code (pas juste dans le concept)

Regarder les tableaux parent dans les appels `addNode(...)` :

- `contrast` : deux parents
- `refine` : deux parents
- `aggregate` : plusieurs parents
- `conclusion` : plusieurs parents

Les nœuds à multiple parents sont impossibles dans un arbre strict ; c'est la signature concrète dans le code du GoT.

---

## 6) Intégration de Visualisation

La logique de visualisation est intentionnellement extraite dans du code helper :

- `writeGoTMotivationVisualization(...)`

Pour que ce fichier exemple reste focalisé sur les opérations de graphe et l'orchestration.

---

## Ordre Suggéré de Lecture du Code

1. Classe `ThoughtGraph`
2. `promptJson`
3. `branch`
4. `scoreAll`
5. `contrast`
6. `refine`
7. `aggregate`
8. `conclude`
9. `runGoTMotivationAnalysis`

Cela vous donne le même ordre que l'exécution runtime et le chemin d'apprentissage le plus clean.
