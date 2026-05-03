# Explication du Code : batch.js

Ce fichier démontre **l'exécution parallèle** de prompts LLM multiples à l'aide de séquences de context indépendantes, permettant un traitement concurrent pour de meilleures performances.

## Décomposition du Code étape par étape

### 1. Import et Configuration (lignes 1-10)
```javascript
import {getLlama, LlamaChatSession} from "node-llama-cpp";
import path from "path";
import {fileURLToPath} from "url";

/**
 * L'exécution asynchrone améliore les performances dans les benchmarks GAIA,
 * les applications multi-agents et autres scénarios à haut débit.
 */

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
- Imports standard pour l'interaction avec le LLM
- Le commentaire explique le bénéfice en performance
- **Benchmark GAIA** : Un standard pour tester les performances des agents IA
- Utile pour les systèmes multi-agents qui doivent gérer de nombreuses requêtes

### 2. Configuration du Chemin du Model (lignes 11-16)
```javascript
const modelPath = path.join(
    __dirname,
    "../",
    "models",
    "DeepSeek-R1-0528-Qwen3-8B-Q6_K.gguf"
)
```
- Utilise **DeepSeek-R1** : Un model de 8B paramètres optimisé pour le raisonnement
- **Quantization Q6_K** : Équilibre entre qualité et taille
- Le model est chargé une fois et partagé entre les séquences

### 3. Initialiser Llama et Charger le Model (lignes 18-19)
```javascript
const llama = await getLlama();
const model = await llama.loadModel({modelPath});
```
- Initialisation standard
- Le model est chargé en mémoire une seule fois
- Sera utilisé par plusieurs séquences simultanément

### 4. Créer le Context avec Plusieurs Séquences (lignes 20-23)
```javascript
const context = await model.createContext({
    sequences: 2,
    batchSize: 1024 // Le nombre de tokens pouvant être traités en une fois par le GPU.
});
```

**Paramètres clés :**

- **sequences: 2** : Crée 2 séquences de conversation indépendantes
  - Chaque séquence a son propre historique de conversation
  - Les deux partagent le même model et pool de mémoire de context
  - Peuvent être traitées en parallèle

- **batchSize: 1024** : Nombre maximal de tokens traités par batch GPU
  - Plus grand = meilleure utilisation du GPU
  - Plus petit = usage mémoire réduit
  - 1024 est un bon équilibre pour la plupart des GPU

### Pourquoi Plusieurs Séquences ?

```
Séquence Unique (Séquentiel)     Séquences Multiples (Parallèle)
─────────────────────────       ──────────────────────────────
Traiter Prompt 1 → Réponse 1     Traiter Prompt 1 ──┐
Attendre...                                                  ├→ Les deux réponses
Traiter Prompt 2 → Réponse 2     Traiter Prompt 2 ──┘   en parallèle !

Temps total : T1 + T2              Temps total : max(T1, T2)
```

### 5. Récupérer les Séquences Individuelles (lignes 25-26)
```javascript
const sequence1 = context.getSequence();
const sequence2 = context.getSequence();
```
- Récupère deux objets séquence séparés depuis le context
- Chaque séquence maintient son propre état
- Elles peuvent être utilisées indépendamment pour des conversations différentes

### 6. Créer des Sessions Séparées (lignes 28-33)
```javascript
const session1 = new LlamaChatSession({
    contextSequence: sequence1
});
const session2 = new LlamaChatSession({
    contextSequence: sequence2
});
```
- Crée une session de chat pour chaque séquence
- Chaque session a son propre historique de conversation
- Les sessions sont complètement indépendantes
- Pas de system prompts dans cet exemple (pourrait être ajouté)

### 7. Définir les Questions (lignes 35-36)
```javascript
const q1 = "Hi there, how are you?";
const q2 = "How much is 6+6?";
```
- Deux questions complètement différentes
- Seront traitées simultanément
- Types différents : conversationnel vs. computationnel

### 8. Exécution Parallèle avec Promise.all (lignes 38-44)
```javascript
const [
    a1,
    a2
] = await Promise.all([
    session1.prompt(q1),
    session2.prompt(q2)
]);
```

**Comment ça fonctionne :**

1. `session1.prompt(q1)` démarre de manière asynchrone
2. `session2.prompt(q2)` démarre de manière asynchrone (n'attend pas #1)
3. `Promise.all()` attend que LES DEUX soient terminés
4. Retourne les résultats dans un tableau : [réponse1, réponse2]
5. Déstructure en `a1` et `a2`

**Bénéfice clé** : Les deux prompts sont traités en même temps, pas l'un après l'autre !

### 9. Afficher les Résultats (lignes 46-50)
```javascript
console.log("User: " + q1);
console.log("AI: " + a1);

console.log("User: " + q2);
console.log("AI: " + a2);
```
- Affiche les deux paires question-réponse
- Les résultats apparaissent dans l'ordre malgré le traitement parallèle

## Concepts Clés Démontrés

### 1. Traitement Parallèle
Au lieu de :
```javascript
// Séquentiel (lent)
const a1 = await session1.prompt(q1);  // Attendre
const a2 = await session2.prompt(q2);  // Attendre encore
```

On utilise :
```javascript
// Parallèle (rapide)
const [a1, a2] = await Promise.all([
    session1.prompt(q1),
    session2.prompt(q2)
]);
```

### 2. Séquences de Context
Un context peut contenir plusieurs séquences indépendantes :

```
┌─────────────────────────────────────┐
│          Context (Partagé)          │
│  ┌───────────────────────────────┐  │
│  │  Poids du Model (8B params)   │  │
│  └───────────────────────────────┘  │
│                                     │
│  ┌─────────────┐  ┌─────────────┐  │
│  │ Séquence 1  │  │ Séquence 2  │  │
│  │ "Hi there"  │  │ "6+6?"      │  │
│  │ Historique..│  │ Historique..│  │
│  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────┘
```

## Comparaison de Performance

### Exécution Séquentielle
```
Requête 1 : 2 secondes
Requête 2 : 2 secondes
Total : 4 secondes
```

### Exécution Parallèle (Cet Exemple)
```
Requête 1 : 2 secondes ──┐
Requête 2 : 2 secondes ──┤ Les deux tournent
Total : ~2 secondes      └─ simultanément
```

**Gain de vitesse** : ~2x pour 2 séquences, évolue avec plus de séquences

## Cas d'Usage

### 1. Applications Multi-Utilisateurs
```javascript
// Gérer plusieurs utilisateurs simultanément
const [réponseUtil1, réponseUtil2, réponseUtil3] = await Promise.all([
    session1.prompt(requêteUtil1),
    session2.prompt(requêteUtil2),
    session3.prompt(requêteUtil3)
]);
```

### 2. Systèmes Multi-Agents
```javascript
// Plusieurs agents travaillant sur différentes tâches
const [
    réponsePlanificateur,
    réponseAnalyste,
    réponseExécuteur
] = await Promise.all([
    sessionPlanificateur.prompt("Planifier la tâche"),
    sessionAnalyste.prompt("Analyser les données"),
    sessionExécuteur.prompt("Exécuter étape 1")
]);
```

### 3. Benchmarking
```javascript
// Tester plusieurs prompts pour évaluation
const résultats = await Promise.all(
    testPrompts.map(prompt => session.prompt(prompt))
);
```

### 4. Tests A/B
```javascript
// Tester différents system prompts
const [réponseA, réponseB] = await Promise.all([
    sessionAvecPromptA.prompt(requête),
    sessionAvecPromptB.prompt(requête)
]);
```

## Considérations de Ressources

### Usage Mémoire
Chaque séquence a besoin de mémoire pour :
- L'historique de la conversation
- Les calculs intermédiaires
- Le cache KV (key-value cache pour l'attention transformer)

**Règle générale** : Plus de séquences = plus de mémoire nécessaire

### Utilisation du GPU
- **Séquence unique** : Peut sous-utiliser le GPU
- **Séquences multiples** : Meilleure utilisation du GPU
- **Trop de séquences** : Peut excéder la VRAM, causant un ralentissement

### Nombre Optimal de Séquences
Dépend de :
- VRAM disponible
- Taille du model
- Longueur du contexte
- Taille de batch

**Typique** : 2-8 séquences pour les GPU grand public

## Limites et Considérations

### 1. Limite de Context Partagé
Toutes les séquences partagent le même pool de mémoire de context :
```
Taille totale du context : 8192 tokens
Séquence 1 : 4096 tokens
Séquence 2 : 4096 tokens
Distribution maximale !
```

### 2. Pas de Vrai Parallélisme pour le CPU
Sur les systèmes CPU-only, les séquences sont entrelacées, pas vraiment parallèles. Offrent tout de même un meilleur débit global.

### 3. Surcharge de Chargement du Model
Le model est chargé une fois et partagé, ce qui est efficace. Mais le chargement initial prend tout de même du temps.

## Pourquoi Cela Compte pour les AI Agents

### Efficacité en Production
Les systèmes agents en production doivent :
- Gérer plusieurs requêtes simultanément
- Répondre rapidement aux utilisateurs
- Utiliser le matériel de manière efficace

### Architectures Multi-Agents
Les systèmes agents complexes ont souvent :
- **Agent planificateur** : Pense à la stratégie
- **Agent exécuteur** : Entreprend des actions
- **Agent critique** : Évalue les résultats

Ceux-ci peuvent tourner en parallèle en utilisant des séquences séparées.

### Scalabilité
Ce pattern est la fondation pour :
- Services web avec plusieurs utilisateurs
- Traitement par lots de données
- Systèmes agents distribués

## Bonnes Pratiques

1. **Adapter les séquences à la charge** : Ne pas en créer plus que nécessaire
2. **Surveiller l'usage mémoire** : Chaque séquence consomme de la VRAM
3. **Utiliser une taille de batch appropriée** : Équilibrer vitesse vs. mémoire
4. **Nettoyer les ressources** : Toujours disposer à la fin
5. **Gérer les erreurs** : Envelopper Promise.all dans try-catch

## Sortie Attendue

L'exécution de ce script devrait produire quelque chose comme :
```
User: Hi there, how are you?
AI: Hello! I'm doing well, thank you for asking...

User: How much is 6+6?
AI: 12
```

Les deux réponses apparaissent rapidement car elles ont été traitées simultanément !
