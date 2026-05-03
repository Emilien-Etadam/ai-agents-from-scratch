# Explication du Code : aot-agent.js

Cet exemple démontre le pattern de prompting **Atom of Thought** en utilisant une calculatrice mathématique comme domaine.

## Architecture en Trois Phases

### Phase 1 : Planification (LLM)
```javascript
async function generatePlan(userPrompt) {
    const grammar = await llama.createGrammarForJsonSchema(planSchema);
    const planText = await session.prompt(userPrompt, { grammar });
    return grammar.parse(planText);
}
```

**Points clés :**
- Le LLM sort du **JSON structuré** (contraint par la grammar)
- Le LLM n'exécute PAS les calculs
- Chaque atom représente une opération
- Les dépendances sont explicites (tableau `dependsOn`)

**Exemple de sortie :**
```json
{
  "atoms": [
    {"id": 1, "kind": "tool", "name": "add", "input": {"a": 15, "b": 7}},
    {"id": 2, "kind": "tool", "name": "multiply", "input": {"a": "<result_of_1>", "b": 3}},
    {"id": 3, "kind": "tool", "name": "subtract", "input": {"a": "<result_of_2>", "b": 10}},
    {"id": 4, "kind": "final", "name": "report", "dependsOn": [3]}
  ]
}
```

### Phase 2 : Validation (Système)
```javascript
function validatePlan(plan) {
    const allowedTools = new Set(Object.keys(tools));

    for (const atom of plan.atoms) {
        if (ids.has(atom.id)) throw new Error(`Duplicate ID`);
        if (atom.kind === "tool" && !allowedTools.has(atom.name)) {
            throw new Error(`Unknown tool: ${atom.name}`);
        }
    }
}
```

**Valide :**
- Pas d'IDs d'atome dupliqués
- Seuls les outils autorisés sont référencés
- Les dépendances ont du sens
- La structure JSON est correcte

### Phase 3 : Exécution (Système)
```javascript
function executePlan(plan) {
    const state = {};

    for (const atom of sortedAtoms) {
        // Résoudre les dépendances
        let resolvedInput = {};
        for (const [key, value] of Object.entries(atom.input)) {
            if (value.startsWith('<result_of_')) {
                const refId = parseInt(value.match(/\d+/)[0]);
                resolvedInput[key] = state[refId];
            }
        }

        // Exécuter
        state[atom.id] = tools[atom.name](resolvedInput.a, resolvedInput.b);
    }
}
```

**Comportements clés :**
- Exécute les atomes dans l'ordre (triés par ID)
- Résout les références `<result_of_N>` depuis le state
- Chaque atome stocke son résultat dans `state[atom.id]`
- L'exécution est **deterministe** (même plan + même state = même résultat)

## Pourquoi Cela Compte

### Comparaison avec ReAct

| Aspect | ReAct | Atom of Thought |
|--------|-------|-----------------|
| **Planification** | Implicite (dans le raisonnement LLM) | Explicite (structure JSON) |
| **Exécution** | Le LLM décide la prochaine étape | Le système suit le plan |
| **Validation** | Aucune | Avant l'exécution |
| **Debugging** | Difficile (tracer dans le texte) | Facile (inspecter les atomes) |
| **Tests** | Difficiles (mock le LLM) | Faciles (tester l'exécuteur) |
| **Échecs** | Peut halluciner | Échec à un atome spécifique |

### Bénéfices

1. **Pas de raisonnement caché** : Chaque opération est un atome explicite
2. **Testable** : Exécuter le plan sans implication du LLM
3. **Debuggable** : Savoir exactement quel atome a échoué
4. **Auditable** : Le plan est une structure de données, pas du texte
5. **Deterministe** : Même input = même output (avec le même plan)

## Implémentation des Outils

Les outils sont des **fonctions pures** sans effets de bord :
```javascript
const tools = {
    add: (a, b) => {
        const result = a + b;
        console.log(`EXECUTING: add(${a}, ${b}) = ${result}`);
        return result;
    },
    // ... plus d'outils
};
```

**Pourquoi des fonctions pures ?**
- Faciles à tester
- Faciles à rejouer
- Pas de state caché
- Composables

## Flux de State
```
Question Utilisateur
      ↓
[LLM génère un plan]
      ↓
{atoms: [...]} ← Plan JSON
      ↓
[Le système valide]
      ↓
Plan valide
      ↓
[Le système exécute l'atome 1] → state[1] = résultat
      ↓
[Le système exécute l'atome 2] → state[2] = résultat (utilise state[1])
      ↓
[Le système exécute l'atome 3] → state[3] = résultat (utilise state[2])
      ↓
Réponse Finale
```

## Gestion d'Erreurs
```javascript
// La validation de l'atome échoue → re-prompter le LLM
validatePlan(plan); // lève une erreur si invalide

// L'exécution de l'outil échoue → s'arrêter à cet atome
if (b === 0) throw new Error("Division par zéro");

// Dépendance manquante → message d'erreur clair
if (!(depId in state)) {
    throw new Error(`Atom ${atom.id} depends on incomplete atom ${depId}`);
}
```

## Quand Utiliser AoT

✅ **Utiliser AoT quand :**
- L'exécution doit être auditable
- Les échecs doivent être récupérables
- Étapes multiples avec dépendances
- Les tests sont importants
- La conformité compte

❌ **Ne pas utiliser AoT quand :**
- Tâches single-step
- Tâches créatives/exploratoires
- Brainstorming
- Conversation naturelle

## Idées d'Extension

1. **Ajouter des atomes de compensation** pour le rollback
2. **Ajouter une logique de retry** par atome
3. **Paralleliser les atomes indépendants** (atomes sans dépendances partagées)
4. **Persister le plan** pour le debugging
5. **Visualiser le graphe d'atomes** (arbre de dépendances)
