# Explication du Code : coding.js

Ce fichier démontre les **réponses en streaming** avec des limites de tokens et une sortie en temps réel, montrant comment obtenir un retour immédiat du LLM pendant la génération de texte.

## Décomposition du Code étape par étape

### 1. Import et Configuration (lignes 1-8)
```javascript
import {
    getLlama,
    HarmonyChatWrapper,
    LlamaChatSession,
} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
- Configuration standard pour l'interaction avec le LLM
- **HarmonyChatWrapper** : Un wrapper de format de chat pour les models utilisant le format Harmony (voir plus bas)

### 2. Comprendre le Format de Chat Harmony

#### Qu'est-ce que Harmony ?
Harmony est un format de message structuré utilisé pour les interactions de chat multi-rôles, conçu par OpenAI pour ses models gpt-oss. Ce n'est pas juste un format de prompt — c'est une repensée complète de la manière dont les models doivent structurer leurs sorties, surtout pour le raisonnement complexe et l'utilisation d'outils.

#### Structure du Format Harmony

Le format utilise des tokens et une syntaxe spéciaux pour définir des rôles tels que `system`, `developer`, `user`, `assistant` et `tool`, ainsi que des "canaux" de sortie (`analysis`, `commentary`, `final`) qui permettent au model de raisonner en interne, d'appeler des outils et de produire des réponses propres pour l'utilisateur.

**Structure de message de base :**
```
<|start|>ROLE<|message|>CONTENU<|end|>
<|start|>assistant<|channel|>CANAL<|message|>CONTENU<|end|>
```

**Les cinq rôles par ordre hiérarchique** (system > developer > user > assistant > tool) :

1. **system** : Identité globale, garde-fous et configuration du model
2. **developer** : Politique produit et instructions de style (ce qu'on appelle généralement "system prompt")
3. **user** : Messages et requêtes utilisateur
4. **assistant** : Réponses du model
5. **tool** : Résultats d'exécution d'outils

**Les trois canaux de sortie :**

1. **analysis** : Raisonnement chain-of-thought privé non montré aux utilisateurs
2. **commentary** : Préambules de tool calling et mises à jour de processus
3. **final** : Réponses propres visibles par l'utilisateur

**Exemple de Harmony en action :**
```
<|start|>system<|message|>You are a helpful assistant.<|end|>
<|start|>developer<|message|>Always be concise.<|end|>
<|start|>user<|message|>What time is it?<|end|>
<|start|>assistant<|channel|>commentary<|message|>{"tool_use": {"name": "get_current_time", "arguments": {}}}<|end|>
<|start|>tool<|message|>{"time": "2025-10-25T13:47:00Z"}<|end|>
<|start|>assistant<|channel|>final<|message|>The current time is 1:47 PM UTC.<|end|>
```

#### Pourquoi Utiliser Harmony ?

Harmony sépare la façon dont le model pense, les actions qu'il entreprend et ce qui arrive finalement à l'utilisateur, ce qui résulte en un tool calling plus propre, des valeurs par défaut plus sûres pour l'UI et une meilleure observabilité. Pour notre exemple de traduction :

- Le canal `final` garantit qu'on obtient uniquement la traduction, pas d'explications
- Le format structuré aide le model à suivre les instructions de manière plus fiable
- La hiérarchie de rôles évite les conflits d'instructions

**Note Importante** : Les models doivent être spécifiquement entraînés ou fine-tuned pour produire une sortie Harmony correcte. On ne peut pas simplement appliquer ce format à n'importe quel model. Apertus et d'autres models pas explicitement entraînés sur Harmony peuvent être confus par cette structure, mais le HarmonyChatWrapper dans node-llama-cpp gère le formatage nécessaire automatiquement.

### 3. Charger le Model (lignes 10-18)
```javascript
const llama = await getLlama();
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "hf_giladgd_gpt-oss-20b.MXFP4.gguf"
    )
});
```
- Utilise **gpt-oss-20b** : Un model de 20 milliards de paramètres
- **MXFP4** : Quantization mixed precision 4-bit pour une taille réduite
- Model plus grand = meilleures explications de code

### 4. Créer le Context et la Session (lignes 19-22)
```javascript
const context = await model.createContext();
const session = new LlamaChatSession({
    chatWrapper: new HarmonyChatWrapper(),
    contextSequence: context.getSequence(),
});
```
Configuration de session de base sans system prompt.

### 5. Définir la Question (ligne 24)
```javascript
const q1 = `What is hoisting in JavaScript? Explain with examples.`;
```
Une question technique de programmation qui nécessite une explication détaillée.

### 6. Afficher la Taille du Context (ligne 26)
```javascript
console.log('context.contextSize', context.contextSize)
```
- Affiche la taille maximale de la fenêtre de context
- Aide à comprendre les limites mémoire
- Utile pour le debugging

### 7. Exécution du Prompt en Streaming (lignes 28-36)
```javascript
const a1 = await session.prompt(q1, {
    // Tip: let the lib choose or cap reasonably; using the whole context size can be wasteful
    maxTokens: 2000,

    // Fires as soon as the first characters arrive
    onTextChunk: (text) => {
        process.stdout.write(text); // optionnel : affichage live
    },
});
```

**Paramètres clés :**

**maxTokens: 2000**
- Limite la longueur de la réponse à 2000 tokens (~1500 mots)
- Empêche la génération sans fin
- Économise du temps et du compute
- Sans limite : le model utilise tout le context

**Callback onTextChunk**
- Se déclenche **à chaque token généré**
- Reçoit le texte au fur et à mesure
- `process.stdout.write()` : Affiche sans sauts de ligne
- Crée un effet de "frappe" en temps réel

### Comment le Streaming Fonctionne

```
Sans streaming :
Utilisateur → [Attendre 10 secondes...] → Réponse complète apparaît

Avec streaming :
Utilisateur → [Token 1] → [Token 2] → [Token 3] → ... → Complet
             "What"      "is"        "hoisting"
             (Feedback immédiat !)
```

### 8. Afficher la Réponse Finale (ligne 38)
```javascript
console.log("\n\nFinal answer:\n", a1);
```
- Affiche la réponse complète une nouvelle fois
- Utile pour le logging ou la vérification
- Montre le texte intégral après le streaming

### 9. Nettoyage (lignes 41-44)
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
Nettoyage standard des ressources.

## Concepts Clés Démontrés

### 1. Réponses en Streaming

**Pourquoi le streaming compte :**
- **Meilleure UX** : Les utilisateurs voient la progression immédiatement
- **Arrêt anticipé** : Peut arrêter si la réponse dérive
- **Perception de vitesse** : Paraît plus rapide que d'attendre
- **Debugging** : Voir la génération en temps réel

**Comparaison :**
```
Non-streaming :           Streaming :
═══════════════         ═══════════════
Requête envoyée          Requête envoyée
[10s d'attente...]       "What" (0,1s)
Réponse complète         "is" (0,2s)
                         "hoisting" (0,3s)
                         ... continue
                         (Même temps total, meilleure expérience !)
```

### 2. Limites de Tokens

**maxTokens contrôle la longueur de génération :**

```
Sans limite :             Avec limite (2000) :
─────────                ─────────────────
Peut générer indéfiniment Arrête à 2000 tokens
Utilise tout le context  Économise du compute
Coût imprévisible        Coût prévisible
```

**Approximation en tokens :**
- 1 token ≈ 0,75 mot (anglais)
- 2000 tokens ≈ 1500 mots
- 4-5 paragraphes d'explication détaillée

### 3. Pattern de Feedback en Temps Réel

Le callback `onTextChunk` permet :
```javascript
onTextChunk: (text) => {
    // Faire quelque chose avec chaque chunk :
    process.stdout.write(text);      // Sortie console
    // socket.emit('chunk', text);   // WebSocket vers client
    // buffer += text;               // Accumuler pour traitement
    // analyzePartial(text);         // Analyse en temps réel
}
```

### 4. Awareness de la Taille de Context

```javascript
console.log('context.contextSize', context.contextSize)
```

Montre la capacité mémoire du model :
- Petits models : 2048-4096 tokens
- Models moyens : 8192-16384 tokens
- Grands models : 32768+ tokens

**Pourquoi ça compte :**
```
Taille de Context : 4096 tokens
Prompt : 100 tokens
Réponse max : 2000 tokens
Historique : Jusqu'à 1996 tokens
```

## Cas d'Usage

### 1. Explications de Code (Cet Exemple)
```javascript
prompt: "Explain hoisting in JavaScript"
→ Streams une explication détaillée avec exemples
```

### 2. Génération de Contenu Long
```javascript
prompt: "Write a blog post about AI agents"
maxTokens: 3000
→ Streams l'article pendant qu'il est écrit
```

### 3. Tutorat Interactif
```javascript
// L'utilisateur voit l'explication se construire
prompt: "Teach me about closures"
onTextChunk: (text) => displayToUser(text)
```

### 4. Applications Web
```javascript
// Server-Sent Events ou WebSocket
onTextChunk: (text) => {
    websocket.send(text);  // Envoyer au navigateur
}
```

## Considérations de Performance

### Vitesse de Génération de Tokens

Dépend de :
- **Taille du model** : Plus grand = plus lent par token
- **Matériel** : GPU > CPU
- **Quantization** : Moins de bits = plus rapide
- **Longueur de context** : Context plus long = plus lent

**Vitesses typiques :**
```
Taille Model    GPU (RTX 4090)    CPU (M2 Max)
──────────      ──────────────    ────────────
1,7B            50-80 tok/s       15-25 tok/s
8B              20-35 tok/s       5-10 tok/s
20B             10-15 tok/s       2-4 tok/s
```

### Quand Utiliser maxTokens

```
✓ Utiliser maxTokens quand :
  • La longueur de réponse est prévisible
  • Vous voulez économiser du compute
  • Testing/debugging
  • Rate limiting d'API

✗ Ne pas limiter quand :
  • Besoin d'une réponse complète
  • La longueur varie beaucoup
  • Utilisation de stop sequences plutôt
```

## Patterns Avancés de Streaming

### Pattern 1 : Amélioration Progressive
```javascript
let buffer = '';
onTextChunk: (text) => {
    buffer += text;
    if (buffer.includes('\n\n')) {
        // paragraphe complet prêt
        processParagraph(buffer);
        buffer = '';
    }
}
```

### Pattern 2 : Arrêt Anticipé
```javascript
let isRelevant = true;
onTextChunk: (text) => {
    if (text.includes('irrelevant_keyword')) {
        isRelevant = false;
        // Arrêter la génération (nécessiterait une API supplémentaire)
    }
}
```

### Pattern 3 : Multi-Consommateur
```javascript
onTextChunk: (text) => {
    console.log(text);           // Console
    logFile.write(text);         // Fichier
    websocket.send(text);        // Client
    analyzer.process(text);      // Analyse
}
```

## Sortie Attendue

Lors de l'exécution, vous verrez :
1. La taille du context logguée (ex. "context.contextSize 32768")
2. La réponse en streaming apparaissant token par token
3. La réponse finale complète affichée une nouvelle fois

Exemple de flux de sortie :
```
context.contextSize 32768
Hoisting is a JavaScript mechanism where variables and function
declarations are moved to the top of their scope before code
execution. For example:

console.log(x); // undefined (not an error!)
var x = 5;

This works because...
[continue en streaming...]

Final answer:
[Réponse complète affichée à nouveau]
```

## Pourquoi Cela Compte pour les AI Agents

### Expérience Utilisateur
- Les agents en temps réel paraissent plus réactifs
- Les utilisateurs peuvent interrompre si la direction est mauvaise
- Meilleur pour les interfaces conversationnelles

### Gestion des Ressources
- Les limites de tokens empêchent la génération sans fin
- Coûts et durées prévisibles
- Possibilité d'annuler des opérations coûteuses tôt

### Patterns d'Intégration
- Les UIs web montrent un effet de "frappe"
- Les CLIs affichent une sortie progressive
- Les APIs stream vers les clients efficacement

Ce pattern est essentiel pour les systèmes d'agents en production où l'expérience utilisateur et le contrôle des ressources comptent.
