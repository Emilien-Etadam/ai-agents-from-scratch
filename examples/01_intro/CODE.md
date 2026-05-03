# Explication du Code : intro.js

Ce fichier démontre l'interaction la plus basique avec un LLM (Large Language Model) local en utilisant node-llama-cpp.

## Décomposition du Code étape par étape

### 1. Importer les Modules Requis
```javascript
import {
    getLlama,
    LlamaChatSession,
} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";
```
- **getLlama** : Fonction principale pour initialiser le runtime llama.cpp
- **LlamaChatSession** : Classe pour gérer les conversations de chat avec le model
- **fileURLToPath** et **path** : Modules Node.js standards pour gérer les chemins de fichiers

### 2. Configurer le Chemin du Dossier
```javascript
const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
- Comme les ES modules n'ont pas `__dirname` par défaut, on le crée manuellement
- Cela nous donne le chemin du dossier du fichier courant
- Nécessaire pour localiser le fichier du model par rapport à ce script

### 3. Initialiser le Runtime Llama
```javascript
const llama = await getLlama();
```
- Crée l'instance principale llama.cpp
- Initialise le runtime C++ sous-jacent pour l'inférence du model
- Doit être fait avant de charger tout model

### 4. Charger le Model
```javascript
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "Qwen3-1.7B-Q8_0.gguf"
    )
});
```
- Charge un fichier de model quantisé (format GGUF)
- **Qwen3-1.7B-Q8_0.gguf** : Un model de 1,7 milliard de paramètres, quantisé en 8-bit
- Le model est stocké dans le dossier `models` à la racine du dépôt
- Le chargement du model en mémoire prend quelques secondes

### 5. Créer un Context
```javascript
const context = await model.createContext();
```
- Un **context** représente la mémoire de travail du model
- Il contient l'historique de la conversation et l'état actuel
- A une taille fixe (par défaut : taille maximale de context du model)
- Tous les prompts et réponses sont stockés dans ce context

### 6. Créer une Session de Chat
```javascript
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
});
```
- **LlamaChatSession** : API de haut niveau pour les interactions de type chat
- Utilise une séquence du context pour maintenir l'état de la conversation
- Gère automatiquement le formatage des prompts et le parsing des réponses

### 7. Définir le Prompt
```javascript
const prompt = `do you know node-llama-cpp`;
```
- Question simple pour tester si le model connaît la bibliothèque que nous utilisons
- Cela sera envoyé au model pour traitement

### 8. Envoyer le Prompt et Obtenir la Réponse
```javascript
const a1 = await session.prompt(prompt);
console.log("AI: " + a1);
```
- **session.prompt()** : Envoie le prompt au model et attend la complétion
- Le model génère une réponse basée sur son entraînement
- On affiche la réponse dans la console avec le préfixe "AI:"

### 9. Nettoyer les Ressources
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
- **Important** : Toujours libérer les ressources une fois terminé
- Libère la mémoire et les ressources GPU
- Empêche les fuites de mémoire dans les applications longue durée
- Doit être fait dans cet ordre (session → context → model → llama)

## Concepts Clés Démontrés

1. **Initialisation basique d'un LLM** : Charger un model et créer un context d'inférence
2. **Prompting simple** : Envoyer une question et recevoir une réponse
3. **Gestion des ressources** : Nettoyage approprié des ressources allouées

## Sortie Attendue

Lorsque vous exécutez ce script, vous devriez voir une sortie comme :
```
AI: Yes, I'm familiar with node-llama-cpp. It's a Node.js binding for llama.cpp...
```

La réponse exacte variera en fonction des données d'entraînement du model et des paramètres de génération.
