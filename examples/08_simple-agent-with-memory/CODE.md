# Explication du Code : simple-agent-with-memory.js

Cet exemple étend l'agent simple avec une **mémoire persistante**, lui permettant de retenir des informations à travers les sessions tout en évitant intelligemment les sauvegardes dupliquées.

## Composants Clés

### 1. Import de MemoryManager
```javascript
import {MemoryManager} from "./memory-manager.js";
```
Classe personnalisée pour persister les mémoires de l'agent dans des fichiers JSON avec un stockage de mémoire unifié.

### 2. Initialiser le Memory Manager
```javascript
const memoryManager = new MemoryManager('./agent-memory.json');
const memorySummary = await memoryManager.getMemorySummary();
```
- Charge les mémoires existantes depuis le fichier
- Génère un résumé formaté pour le system prompt
- Gère la migration depuis les anciens schemas de mémoire

### 3. System Prompt Sensible à la Mémoire avec Raisonnement
```javascript
const systemPrompt = `
You are a helpful assistant with long-term memory.

Before calling any function, always follow this reasoning process:

1. **Compare** new user statements against existing memories below.
2. **If the same key and value already exist**, do NOT call saveMemory again.
   - Instead, simply acknowledge the known information.
   - Example: if the user says "My name is Malua" and memory already says "user_name: Malua", reply "Yes, I remember your name is Malua."
3. **If the user provides an updated value** (e.g., "I actually prefer sushi now"),
   then call saveMemory once to update the value.
4. **Only call saveMemory for genuinely new information.**

When saving new data, call saveMemory with structured fields:
- type: "fact" or "preference"
- key: short descriptive identifier (e.g., "user_name", "favorite_food")
- value: the specific information (e.g., "Malua", "chinua")

Examples:
saveMemory({ type: "fact", key: "user_name", value: "Malua" })
saveMemory({ type: "preference", key: "favorite_food", value: "chinua" })

${memorySummary}
`;
```

**Ce que ça fait :**
- Inclut les mémoires existantes dans le prompt
- Fournit des directives de raisonnement explicites pour prévenir les sauvegardes dupliquées
- Enseigne à l'agent de comparer avant de sauvegarder
- Indique quand mettre à jour vs. reconnaître les données existantes

### 4. Fonction saveMemory
```javascript
const saveMemory = defineChatSessionFunction({
    description: "Save important information to long-term memory (user preferences, facts, personal details)",
    params: {
        type: "object",
        properties: {
            type: {
                type: "string",
                enum: ["fact", "preference"]
            },
            key: { type: "string" },
            value: { type: "string" }
        },
        required: ["type", "key", "value"]
    },
    async handler({ type, key, value }) {
        await memoryManager.addMemory({ type, key, value });
        return `Memory saved: ${key} = ${value}`;
    }
});
```

**Ce qu'elle fait :**
- Utilise un format structuré clé-valeur pour toutes les mémoires
- Sauvegarde à la fois les faits et les préférences avec la même méthode
- Gère automatiquement les doublons (met à jour si la valeur change)
- Persiste dans un fichier JSON
- Retourne un message de confirmation

**Structure des Paramètres :**
- `type` : Soit "fact" soit "preference"
- `key` : Identifiant court (ex. "user_name", "favorite_food")
- `value` : L'information réelle (ex. "Alex", "pizza")

### 5. Exemple de Conversation
```javascript
const prompt1 = "Hi! My name is Alex and I love pizza.";
const response1 = await session.prompt(prompt1, {functions});
// L'agent appelle saveMemory deux fois :
// - saveMemory({ type: "fact", key: "user_name", value: "Alex" })
// - saveMemory({ type: "preference", key: "favorite_food", value: "pizza" })

const prompt2 = "What's my favorite food?";
const response2 = await session.prompt(prompt2, {functions});
// L'agent rappelle depuis la mémoire : "Pizza"
```

## Comment la Mémoire Fonctionne

### Diagramme de Flux
```
Session 1 :
Utilisateur : "My name is Alex and I love pizza"
  ↓
Agent appelle : saveMemory({ type: "fact", key: "user_name", value: "Alex" })
Agent appelle : saveMemory({ type: "preference", key: "favorite_food", value: "pizza" })
  ↓
Sauvegardé dans : agent-memory.json

Session 2 (après redémarrage) :
1. Charger les mémoires depuis agent-memory.json
2. Ajouter au system prompt
3. L'agent voit : "user_name: Alex" et "favorite_food: pizza"
4. Peut utiliser cette information dans les réponses

Session 3 :
Utilisateur : "My name is Alex"
  ↓
Agent compare : user_name déjà = "Alex"
  ↓
Pas d'appel de fonction ! Reconnaît simplement : "Yes, I remember your name is Alex."
```

## La Classe MemoryManager

Située dans `memory-manager.js` :
```javascript
class MemoryManager {
  async loadMemories()           // Charger depuis JSON (gère la migration de schema)
  async saveMemories()           // Écrire dans JSON
  async addMemory()              // Méthode unifiée pour tous les types de mémoire
  async getMemorySummary()       // Formater les mémoires pour le system prompt
  extractKey()                   // Helper pour migration
  extractValue()                 // Helper pour migration
}
```

**Bénéfices :**
- Méthode unifiée unique pour tous les types de mémoire
- Détection et prévention automatique des doublons
- Mises à jour automatiques des valeurs quand l'information change

## Concepts Clés

### 1. Format Structuré de Mémoire
Toutes les mémoires utilisent maintenant une structure cohérente :
```javascript
{
  type: "fact" | "preference",
  key: "user_name",           // Identifiant
  value: "Alex",              // Les données réelles
  source: "user",             // D'où ça vient
  timestamp: "2025-10-29..."  // Quand ça a été sauvegardé/mis à jour
}
```

### 2. Prévention Intelligente des Doublons
L'agent est entraîné à :
- **Comparer** avant de sauvegarder
- **Sauter** si les données sont identiques
- **Mettre à jour** si la valeur a changé
- **Reconnaître** les mémoires existantes au lieu de les resauvegarder

### 3. État Persistant
- Les mémoires survivent aux redémarrages du script
- Stockées dans un fichier JSON avec métadonnées
- Chargées au démarrage et injectées dans le prompt

### 4. Intégration de la Mémoire dans le System Prompt
Les mémoires sont automatiquement formatées et injectées :
```
=== LONG-TERM MEMORY ===

Known Facts:
- user_name: Alex
- location: Paris

User Preferences:
- favorite_food: pizza
- preferred_language: French
```

## Pourquoi Cela Compte

**Sans mémoire** : L'agent recommence à zéro à chaque fois, pose les mêmes questions en permanence

**Avec mémoire de base** : L'agent se souvient, mais peut sauvegarder des doublons gaspilleusement

**Avec mémoire intelligente** : L'agent se souvient ET évite les sauvegardes redondantes en raisonnant d'abord

Cela permet :
- **Des réponses personnalisées** basées sur l'historique utilisateur
- **Un usage mémoire efficace** (pas d'entrées dupliquées)
- **Des conversations naturelles** qui paraissent continues
- **Des agents stateful** qui maintiennent le contexte
- **Des mises à jour automatiques** quand l'information change

## Sortie Attendue

**Première exécution :**
```
User: "Hi! My name is Alex and I love pizza."
AI: "Nice to meet you, Alex! I've noted that you love pizza."
[Appelle saveMemory deux fois - nouvelle information sauvegardée]
```

**Deuxième exécution (après redémarrage) :**
```
User: "What's my favorite food?"
AI: "Your favorite food is pizza! You mentioned that you love it."
[Pas d'appel de fonction - rappelle depuis la mémoire chargée]
```

**Troisième exécution (déclaration dupliquée) :**
```
User: "My name is Alex."
AI: "Yes, I remember your name is Alex!"
[Pas d'appel de fonction - reconnaît le doublon, reconnaît simplement]
```

**Quatrième exécution (information mise à jour) :**
```
User: "I actually prefer sushi now."
AI: "Got it! I've updated your favorite food to sushi."
[Appelle saveMemory une fois - met à jour la valeur existante]
```

## Processus de Raisonnement

Le system prompt guide explicitement l'agent à travers cet arbre de décision :
```
Nouvelle déclaration utilisateur
    ↓
Comparer aux mémoires existantes
    ↓
    ├─→ Correspondance exacte ? → Reconnaître uniquement (pas de sauvegarde)
    ├─→ Valeur mise à jour ? → Sauvegarder pour mettre à jour
    └─→ Nouvelle information ? → Sauvegarder comme nouveau
```

Cette approche raisonnement-d'abord rend l'agent plus intelligent et efficace avec les opérations de mémoire !
