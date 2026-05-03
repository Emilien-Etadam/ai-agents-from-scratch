# Explication du Code : simple-agent.js

Ce fichier démontre le **function calling** — la fonctionnalité centrale qui transforme un LLM d'un générateur de texte en un agent capable d'entreprendre des actions à l'aide d'outils.

## Décomposition du Code étape par étape

### 1. Import et Configuration (lignes 1-7)
```javascript
import {defineChatSessionFunction, getLlama, LlamaChatSession} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";
import {PromptDebugger} from "../helper/prompt-debugger.js";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const debug = false;
```
- **defineChatSessionFunction** : Import clé pour créer des fonctions appelables
- **PromptDebugger** : Helper pour debugger les prompts (couvert à la fin)
- **debug** : Contrôle le logging verbeux

### 2. Initialiser et Charger le Model (lignes 9-17)
```javascript
const llama = await getLlama({debug});
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "Qwen3-1.7B-Q8_0.gguf"
    )
});
const context = await model.createContext({contextSize: 2000});
```
- Utilise le model Qwen3-1.7B (bon pour le function calling)
- Définit la taille de context à 2000 tokens explicitement

### 3. System Prompt pour la Conversion de Temps (lignes 20-23)
```javascript
const systemPrompt = `You are a professional chronologist who standardizes time representations across different systems.

Always convert times from 12-hour format (e.g., "1:46:36 PM") to 24-hour format (e.g., "13:46") without seconds
before returning them.`;
```

**Objectif :**
- Définit le rôle et le comportement de l'agent
- Indique le format de sortie (24h, sans secondes)
- Garantit la cohérence de la représentation de l'heure

### 4. Créer la Session (lignes 25-28)
```javascript
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
    systemPrompt,
});
```
Session standard avec system prompt.

### 5. Définir une Fonction Outil (lignes 30-39)
```javascript
const getCurrentTime = defineChatSessionFunction({
    description: "Get the current time",
    params: {
        type: "object",
        properties: {}
    },
    async handler() {
        return new Date().toLocaleTimeString();
    }
});
```

**Décomposition :**

**description :**
- Dit au LLM ce que cette fonction fait
- Le LLM lit ça pour décider quand l'appeler

**params :**
- Définit les paramètres de la fonction (format JSON Schema)
- `properties: {}` vide signifie aucun paramètre nécessaire
- Le type doit être "object" même s'il n'y a pas de propriétés

**handler :**
- La fonction JavaScript réelle qui s'exécute
- Retourne l'heure actuelle sous forme de string (ex. "1:46:36 PM")
- Peut être async (utiliser await à l'intérieur)

### Comment le Function Calling Fonctionne

```
1. Utilisateur demande : "Quelle heure est-il ?"
2. LLM lit :
   - System prompt
   - Fonctions disponibles (getCurrentTime)
   - Description de la fonction
3. LLM décide : "Je devrais appeler getCurrentTime()"
4. La librairie exécute : handler()
5. Le handler retourne : "1:46:36 PM"
6. LLM reçoit le résultat comme "tool output"
7. LLM traite : Convertit en format 24h selon le system prompt
8. LLM répond : "13:46"
```

### 6. Enregistrer les Fonctions (ligne 41)
```javascript
const functions = {getCurrentTime};
```
- Crée un objet avec toutes les fonctions disponibles
- Fonctions multiples : `{getCurrentTime, getWeather, calculate, ...}`
- Le LLM peut choisir quelle(s) fonction(s) appeler

### 7. Définir le Prompt Utilisateur (ligne 42)
```javascript
const prompt = `What time is it right now?`;
```
Une question qui nécessite l'utilisation d'un outil.

### 8. Exécuter avec les Fonctions (ligne 45)
```javascript
const a1 = await session.prompt(prompt, {functions});
console.log("AI: " + a1);
```
- **{functions}** rend les outils disponibles au LLM
- Le LLM appellera automatiquement getCurrentTime si nécessaire
- La réponse inclut le résultat de l'outil traité par le LLM

### 9. Debug du Context du Prompt (lignes 49-55)
```javascript
const promptDebugger = new PromptDebugger({
    outputDir: './logs',
    filename: 'qwen_prompts.txt',
    includeTimestamp: true,
    appendMode: false
});
await promptDebugger.debugContextState({session, model});
```

**Ce que ça fait :**
- Sauvegarde l'intégralité du prompt envoyé au model
- Montre exactement ce que le LLM voit (y compris les définitions de fonctions)
- Utile pour debugger pourquoi le model appelle/ne rappelle pas les fonctions
- Écrit dans `./logs/qwen_prompts_[timestamp].txt`

### 10. Nettoyage (lignes 58-61)
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
Nettoyage standard.

## Concepts Clés Démontrés

### 1. Function Calling (Utilisation d'Outils)

C'est ce qui en fait un "agent" :
```
Sans outils :             Avec outils :
LLM → Texte seul         LLM → Peut entreprendre des actions
                              ↓
                       Appeler des fonctions
                       Accéder à des données
                       Exécuter du code
```

### 2. Pattern de Définition de Fonction

```javascript
defineChatSessionFunction({
    description: "Ce que fait la fonction",  // Le LLM lit ça
    params: {                                // Paramètres attendus
        type: "object",
        properties: {
            nomParam: {
                type: "string",
                description: "À quoi sert ce paramètre"
            }
        },
        required: ["nomParam"]
    },
    handler: async (params) => {             // Votre code
        // Faire quelque chose avec params
        return résultat;
    }
});
```

### 3. JSON Schema pour les Paramètres

Utilise le JSON Schema standard :
```javascript
// Sans paramètres
properties: {}

// Un paramètre string
properties: {
    ville: {
        type: "string",
        description: "Nom de la ville"
    }
}

// Plusieurs paramètres
properties: {
    a: { type: "number" },
    b: { type: "number" }
},
required: ["a", "b"]
```

### 4. Prise de Décision de l'Agent

```
Utilisateur : "Quelle heure est-il ?"
         ↓
    LLM réfléchit :
    "J'ai besoin de l'heure actuelle"
    "Je vois la fonction : getCurrentTime"
    "La description correspond à mon besoin"
         ↓
    LLM sort un format spécial :
    {function_call: "getCurrentTime"}
         ↓
    La librairie intercepte et exécute handler()
         ↓
    Le handler retourne : "1:46:36 PM"
         ↓
    LLM reçoit : Résultat outil
         ↓
    LLM applique le system prompt :
    Convertir en format 24h
         ↓
    Réponse finale : "13:46"
```

## Cas d'Usage

### 1. Récupération d'Information
```javascript
const getWeather = defineChatSessionFunction({
    description: "Get weather for a city",
    params: {
        type: "object",
        properties: {
            city: { type: "string" }
        }
    },
    handler: async ({city}) => {
        return await fetchWeather(city);
    }
});
```

### 2. Calculs
```javascript
const calculate = defineChatSessionFunction({
    description: "Perform arithmetic calculation",
    params: {
        type: "object",
        properties: {
            expression: { type: "string" }
        }
    },
    handler: async ({expression}) => {
        return eval(expression); // (Soyez prudent avec eval !)
    }
});
```

### 3. Accès aux Données
```javascript
const queryDatabase = defineChatSessionFunction({
    description: "Query user database",
    params: {
        type: "object",
        properties: {
            userId: { type: "string" }
        }
    },
    handler: async ({userId}) => {
        return await db.users.findById(userId);
    }
});
```

### 4. APIs Externes
```javascript
const searchWeb = defineChatSessionFunction({
    description: "Search the web",
    params: {
        type: "object",
        properties: {
            query: { type: "string" }
        }
    },
    handler: async ({query}) => {
        return await googleSearch(query);
    }
});
```

## Sortie Attendue

À l'exécution :
```
AI: 13:46
```

Le LLM :
1. A appelé getCurrentTime() en interne
2. A obtenu "1:46:36 PM"
3. A converti en format 24h
4. A supprimé les secondes
5. A retourné "13:46"

## Debugging avec PromptDebugger

La sortie de debug montre le prompt complet incluant les schemas de fonctions :
```
System: You are a professional chronologist...

Functions available:
- getCurrentTime: Get the current time
  Parameters: (none)

User: What time is it right now?
```

Cela aide à debugger :
- Le model a-t-il vu la fonction ?
- La description était-elle claire ?
- Les paramètres correspondaient-ils aux attentes ?

## Pourquoi Cela Compte pour les AI Agents

### Agents = LLMs + Outils

```
LLM seul :                    LLM + Outils :
├─ Générer du texte           ├─ Générer du texte
└─ C'est tout                 ├─ Accéder à des données réelles
                              ├─ Effectuer des calculs
                              ├─ Appeler des APIs
                              ├─ Exécuter des actions
                              └─ Interagir avec le monde
```

### Fondation pour les Agents Complexes

Cet exemple simple est la fondation pour :
- **Agents de recherche** : Chercher sur le web, lire des documents
- **Agents codeurs** : Exécuter du code, vérifier les erreurs
- **Assistants personnels** : Calendrier, email, rappels
- **Agents d'analyse** : Interroger des bases de données, calculer des statistiques

Tout commence avec le function calling de base !

## Bonnes Pratiques

1. **Descriptions claires** : Le LLM les utilise pour décider quand appeler
2. **Type safety** : Utiliser JSON Schema correctement
3. **Gestion d'erreurs** : Le handler devrait catcher les erreurs
4. **Retourner des strings** : Le LLM traite le texte le mieux
5. **Garder les fonctions focalisées** : Un objectif clair par fonction

C'est l'agent minimum viable : un LLM + un outil + configuration appropriée.
