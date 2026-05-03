# Explication du Code : think.js

Ce fichier démontre l'utilisation des system prompts pour le **raisonnement logique** et la **résolution de problèmes quantitatifs**, montrant comment configurer un LLM en tant qu'agent de raisonnement spécialisé.

## Décomposition du Code étape par étape

### 1. Import et Configuration (lignes 1-8)
```javascript
import {
    getLlama,
    LlamaChatSession,
} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
- Imports standard pour l'interaction avec le LLM
- Configuration des chemins pour localiser le fichier du model

### 2. Initialiser et Charger le Model (lignes 10-18)
```javascript
const llama = await getLlama();
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "Qwen3-1.7B-Q6_K.gguf"
    )
});
```
- Utilise **Qwen3-1.7B-Q6_K** : Un model de 1,7B paramètres avec quantization 6-bit
- Plus petit que l'exemple de traduction (1,7B vs 8B paramètres)
- La quantization Q6_K offre un équilibre entre taille et qualité

### 3. Définir le System Prompt (lignes 19-24)
```javascript
const systemPrompt = `You are an expert logical and quantitative reasoner.
    Your goal is to analyze real-world word problems involving families, quantities, averages, and relationships
    between entities, and compute the exact numeric answer.

    Goal: Return the correct final number as a single value — no explanation, no reasoning steps, just the answer.
    `
```

**Éléments clés :**

1. **Rôle** : "expert logical and quantitative reasoner"
   - Fixe les attentes pour la pensée mathématique/analytique

2. **Périmètre de la Tâche** : "real-world word problems involving families, quantities, averages, and relationships"
   - Indique au model quel type de problèmes attendre
   - Le prépare pour des tâches complexes de comptage et de calcul

3. **Contrainte de Sortie** : "Return the correct final number as a single value — no explanation"
   - Force une sortie concise
   - Juste la réponse, pas le travail

### Pourquoi Cette Conception de System Prompt ?

Le prompt est conçu pour le type de problème spécifique :
- Problèmes verbaux avec des relations familiales complexes
- Conditions imbriquées multiples
- Nécessite un suivi attentif des personnes et quantités
- Exige des calculs arithmétiques

### 4. Créer le Context et la Session (lignes 25-29)
```javascript
const context = await model.createContext();
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
    systemPrompt
});
```
- Crée le contexte pour la conversation
- Initialise la session avec le system prompt de raisonnement
- Pas de chat wrapper nécessaire (utilisation du format par défaut du model)

### 5. Le Problème Verbal Complexe (lignes 31-40)
```javascript
const prompt = `My family reunion is this week, and I was assigned the mashed potatoes to bring.
...
`;
```

**Ce problème est volontairement complexe pour tester le raisonnement :**

**Personnes à compter :**
- L'orateur (1)
- Mère et père (2)
- Frère jumeau + conjoint(e) (2)
- 2 enfants du frère (2)
- Tante + conjoint (2)
- 1 enfant de la tante (1)
- Grand-mère (1)
- Frère de la grand-mère + conjointe (2)
- Fille du frère + conjoint (2)
- 3 enfants (3, mais ne mangent pas de glucides)

**Calculs nécessaires :**
1. Compter le total d'adultes
2. Compter le total d'enfants
3. Soustraire les enfants qui ne mangent pas
4. Calculer les besoins en pommes de terre : (adultes × 1,5) + (enfants qui mangent × 0,5)
5. Convertir en livres : total pommes de terre × 0,5 lbs
6. Convertir en sacs : livres ÷ 5, arrondir au supérieur

**La complexité :**
- Relations familiales (qui est marié avec qui)
- Personnes décédées (à soustraire du compte)
- Besoins alimentaires spéciaux (les cousins germains ne mangent pas de glucides)
- Conversions d'unités (pommes de terre → livres → sacs)

### 6. Exécuter et Afficher (lignes 42-43)
```javascript
const answer = await session.prompt(prompt);
console.log(`AI: ${answer}`);
```
- Envoie le problème complexe au model
- Le model utilise ses capacités de raisonnement pour résoudre le problème
- Sort juste le chiffre final (basé sur le system prompt)

### 7. Nettoyage (lignes 45-48)
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
- Nettoyage standard des ressources

## Concepts Clés Démontrés

### 1. Configuration d'un Agent de Raisonnement
Cela montre comment configurer un LLM pour la pensée analytique :

```
System Prompt → Le LLM devient un "moteur de raisonnement"
```

Au lieu d'une IA conversationnelle, on obtient :
- Traitement analytique focalisé
- Calcul mathématique
- Déduction logique

### 2. Contrôle du Format de Sortie
Comparez ces approches :

**Sans contrainte :**
```
AI: Laissez-moi y aller étape par étape.
D'abord, je compte les adultes...
[explication longue]
Donc la réponse est 3 sacs.
```

**Avec contrainte (cet exemple) :**
```
AI: 3
```

### 3. Test de Complexité de Problème
Cet exemple teste la capacité du model à :
- Parser un langage naturel complexe
- Suivre plusieurs entités et relations
- Appliquer des opérations arithmétiques
- Gérer les cas limites (personnes décédées, restrictions alimentaires)
- Effectuer des conversions d'unités

### 4. Agents de Tâche Spécialisée
Cela démontre la création d'agents spécifiques à une tâche :

```
LLM Général + System Prompt "Agent de Raisonnement" = Résolveur de Problèmes Maths
```

Le même pattern fonctionne pour :
- Puzzles logiques
- Analyse de données
- Calculs scientifiques
- Raisonnement statistique

## Défis et Limites

### 1. La Taille du Model Compte
Le model de 1,7B paramètres peut avoir du mal avec :
- Les problèmes de comptage très complexes
- Le raisonnement multi-étapes nécessitant de la mémoire de travail
- Les cas limites du problème

Les models plus grands (7B, 13B+) performant généralement mieux sur les tâches de raisonnement.

### 2. Raisonnement Caché
Le system prompt demande "juste la réponse", donc on ne voit pas :
- Le processus de raisonnement du model
- Où il a pu faire des erreurs
- Son niveau de confiance

### 3. Pas d'Outils
Le model doit faire tous les calculs "dans sa tête" sans :
- Une calculatrice
- La prise de notes
- La vérification étape par étape

Les exemples ultérieurs (comme react-agent) résolvent cela en donnant des outils au model.

## Pourquoi Cela Compte pour les AI Agents

### Le Raisonnement est Fondamental
Tous les agents utiles ont besoin de capacités de raisonnement :
- **Agents de planification** : Raisonner sur des séquences d'actions
- **Agents de recherche** : Analyser et synthétiser l'information
- **Agents de décision** : Évaluer les options et conséquences

### Le System Prompt Façonne le Comportement
Cet exemple montre que le même model peut se comporter différemment selon les instructions :
- Agent traducteur (exemple précédent)
- Agent de raisonnement (cet exemple)
- Agent codeur (exemples ultérieurs)

### Fondation pour les Agents Complexes
Comprendre comment faire du prompting pour le raisonnement est essentiel avant d'ajouter :
- Des outils (donner au model une calculatrice)
- De la mémoire (se souvenir des calculs précédents)
- Des processus multi-étapes (pattern ReAct)

## Sortie Attendue

L'exécution de ce script devrait produire quelque chose comme :
```
AI: 3
```

La réponse exacte dépend de la capacité du model à :
- Compter correctement tous les membres de la famille
- Appliquer les taux de consommation
- Convertir les unités
- Arrondir au supérieur pour des sacs entiers

## Améliorer Cette Approche

Pour obtenir un meilleur raisonnement :
1. **Utiliser des models plus grands** : 7B+ paramètres
2. **Ajouter du prompting étape par étape** : "Montrez votre travail"
3. **Fournir des outils** : Donner au model une calculatrice
4. **Utiliser le chain-of-thought** : Encourager le raisonnement explicite
5. **Vérifier les réponses** : Exécuter plusieurs fois ou utiliser plusieurs models

L'exemple react-agent démontre certaines de ces améliorations.
