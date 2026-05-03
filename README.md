# AI Agents From Scratch

Apprenez à construire des AI agents en local, sans framework. Comprenez ce qui se passe sous le capot avant d'utiliser des frameworks de production.

![Vue d'ensemble de l'architecture des agents](diagrams/agent-architecture.png)


## Objectif

Ce dépôt vous enseigne à construire des AI agents à partir des principes fondamentaux en utilisant des **LLMs locaux** et **node-llama-cpp**. En suivant ces exemples, vous comprendrez :

- Comment fonctionnent les LLMs au niveau fondamental
- Ce que sont réellement les agents (LLM + outils + patterns)
- Comment fonctionnent les différentes architectures d'agents
- Pourquoi les frameworks font certains choix de conception

> Une version Python de ce tutoriel est disponible ici :
> https://github.com/pguso/agents-from-scratch

**Philosophie** : Apprendre en construisant. Comprendre en profondeur, puis utiliser les frameworks en toute connaissance de cause.

## Site compagnon

Ce dépôt dispose désormais d'un **site compagnon correspondant** :

**https://agentsfromscratch.com**

Le site n'est **pas un substitut à ce dépôt**, mais un **compagnon conceptuel** qui :

- Explique *pourquoi* chaque exemple existe
- Visualise le chemin d'apprentissage, des appels LLM bruts aux agents complets
- Sépare le **code**, les **explications** et les **concepts clés**
- Vous aide à comprendre les architectures d'agents avant d'utiliser des frameworks

**Workflow recommandé :**
- Utilisez **GitHub** pour exécuter, modifier et étudier le code
- Utilisez le **site** pour les modèles mentaux, les explications et la progression

> Considérez le site comme la *carte* et ce dépôt comme le *terrain*.

## Fondamentaux des Agents - Des LLMs au ReAct

### Prérequis
- Node.js 18+
- Au moins 8 Go de RAM (16 Go recommandés)
- Téléchargez les models et placez-les dans le dossier `./models/`, détails dans [DOWNLOAD.md](DOWNLOAD.md)

### Installation
```bash
npm install
```

### Exécuter les exemples
```bash
node intro/intro.js
node simple-agent/simple-agent.js
node react-agent/react-agent.js
```

## Parcours d'Apprentissage

Suivez ces exemples dans l'ordre pour construire votre compréhension progressivement :

### 1. **Introduction** - Interaction de base avec un LLM
`intro/` | [Code](examples/01_intro/intro.js) | [Explication du code](examples/01_intro/CODE.md) | [Concepts](examples/01_intro/CONCEPT.md)

**Ce que vous apprendrez :**
- Charger et exécuter un LLM local
- Cycle de base prompt / response

**Concepts clés** : Chargement de model, context, pipeline d'inference, génération de tokens

---

### 2. (Optionnel) **OpenAI Intro** - Utilisation de models propriétaires
`openai-intro/` | [Code](examples/02_openai-intro/openai-intro.js) | [Explication du code](examples/02_openai-intro/CODE.md) | [Concepts](examples/02_openai-intro/CONCEPT.md)

**Ce que vous apprendrez :**
- Comment appeler des LLMs hébergés (comme GPT-4)
- Contrôle de la température
- Utilisation des tokens

**Concepts clés** : Endpoints d'inference, latence réseau, coût vs contrôle, confidentialité des données, dépendance aux fournisseurs

---

### 3. **Translation** - System Prompts & Spécialisation
`translation/` | [Code](examples/03_translation/translation.js) | [Explication du code](examples/03_translation/CODE.md) | [Concepts](examples/03_translation/CONCEPT.md)

**Ce que vous apprendrez :**
- Utiliser des system prompts pour spécialiser les agents
- Contrôle du format de sortie
- Comportement basé sur le rôle
- Wrappers de chat pour différents models

**Concepts clés** : System prompts, spécialisation des agents, contraintes comportementales, prompt engineering

---

### 4. **Think** - Raisonnement & Résolution de problèmes
`think/` | [Code](examples/04_think/think.js) | [Explication du code](examples/04_think/CODE.md) | [Concepts](examples/04_think/CONCEPT.md)

**Ce que vous apprendrez :**
- Configurer les LLMs pour le raisonnement logique
- Problèmes quantitatifs complexes
- Limites du raisonnement pur des LLMs
- Quand utiliser des outils externes

**Concepts clés** : Agents de raisonnement, décomposition de problèmes, tâches cognitives, limites du raisonnement

---

### 5. **Batch** - Traitement parallèle
`batch/` | [Code](examples/05_batch/batch.js) | [Explication du code](examples/05_batch/CODE.md) | [Concepts](examples/05_batch/CONCEPT.md)

**Ce que vous apprendrez :**
- Traiter plusieurs requêtes simultanément
- Séquences de contexte pour le parallélisme
- Traitement batch sur GPU
- Optimisation des performances

**Concepts clés** : Exécution parallèle, séquences, batch size, optimisation du throughput

---

### 6. **Coding** - Streaming & Contrôle des réponses
`coding/` | [Code](examples/06_coding/coding.js) | [Explication du code](examples/06_coding/CODE.md) | [Concepts](examples/06_coding/CONCEPT.md)

**Ce que vous apprendrez :**
- Réponses en streaming en temps réel
- Limites de tokens et gestion du budget
- Affichage progressif des sorties
- Optimisation de l'expérience utilisateur

**Concepts clés** : Streaming, génération token par token, contrôle des réponses, feedback en temps réel

---

### 7. **Simple Agent** - Function Calling (Outils)
`simple-agent/` | [Code](examples/07_simple-agent/simple-agent.js) | [Explication du code](examples/07_simple-agent/CODE.md) | [Concepts](examples/07_simple-agent/CONCEPT.md)

**Ce que vous apprendrez :**
- Fonctionnement du function calling / utilisation d'outils
- Définir des outils que le LLM peut utiliser
- JSON Schema pour les paramètres
- Comment les LLMs décident quand utiliser des outils

**Concepts clés** : Function calling, définitions d'outils, prise de décision par l'agent, capacité d'action

**C'est ici que la génération de texte devient de l'agence !**

---

### 8. **Simple Agent with Memory** - État persistant
`simple-agent-with-memory/` | [Code](examples/08_simple-agent-with-memory/simple-agent-with-memory.js) | [Explication du code](examples/08_simple-agent-with-memory/CODE.md) | [Concepts](examples/08_simple-agent-with-memory/CONCEPT.md)

**Ce que vous apprendrez :**
- Persister des informations entre les sessions
- Gestion de la mémoire à long terme
- Stockage de faits et de préférences
- Stratégies de récupération en mémoire

**Concepts clés** : Mémoire persistante, gestion d'état, systèmes de mémoire, augmentation du contexte

---

### 9. **ReAct Agent** - Raisonnement + Action
`react-agent/` | [Code](examples/09_react-agent/react-agent.js) | [Explication du code](examples/09_react-agent/CODE.md) | [Concepts](examples/09_react-agent/CONCEPT.md)

**Ce que vous apprendrez :**
- Pattern ReAct (Reason → Act → Observe)
- Résolution itérative de problèmes
- Utilisation d'outils étape par étape
- Boucles de correction automatique

**Concepts clés** : Pattern ReAct, raisonnement itératif, cycles observation-action, agents multi-étapes

**C'est la base des frameworks d'agents modernes !**

---

### 10. **AoT Agent** - Planification par Atom of Thought
`aot-agent/` | [Code](examples/10_aot-agent/aot-agent.js) | [Explication du code](examples/10_aot-agent/CODE.md) | [Concepts](examples/10_aot-agent/CONCEPT.md)

**Ce que vous apprendrez :**
- Méthodologie Atom of Thought
- Planification atomique pour les calculs multi-étapes
- Gestion des dépendances entre les opérations
- Sorties JSON structurées pour les plans de raisonnement
- Exécution déterministe des plans

**Concepts clés** : Planification AoT, opérations atomiques, résolution de dépendances, validation de plans, raisonnement structuré

---

### 11. **Error Handling** - Résilience pour LLM + Outils
`error-handling/` | [Code](examples/11_error-handling/error-handling.js) | [Explication du code](examples/11_error-handling/CODE.md) | [Concepts](examples/11_error-handling/CONCEPT.md)

**Ce que vous apprendrez :**
- Taxonomie typée des erreurs (validation, LLM, outils, workflow) avec des codes stables
- Timeouts, retries avec backoff/jitter, et classification des pannes transitoires
- Dégradation gracieuse quand le chemin LLM échoue (fallback d'outil déterministe)
- Erreurs au niveau orchestration (`AgentWorkflowError`) et correlation IDs pour le support

**Concepts clés** : Taxonomie des erreurs, politiques de retry, timeouts, fallbacks, mode dégradé, observabilité, messages sécurisés pour l'utilisateur

---

### 12. **Tree of Thought** - Recherche sur les branches de raisonnement
`tree-of-thought/` | [Code](examples/12_tree-of-thought/tree-of-thought.js) | [Explication du code](examples/12_tree-of-thought/CODE.md) | [Concepts](examples/12_tree-of-thought/CONCEPT.md)

**Ce que vous apprendrez :**
- Générer plusieurs actions candidates depuis le même plan partiel
- Classer et élaguer les branches avec un score déterministe en code
- Exécuter une boucle compacte de beam search avec des décisions conservées/élaguées inspectables
- Vérifier le chemin gagnant avec des contrôles de cohérence explicites

**Concepts clés** : Tree of Thought, beam search, élagage de branches, objectifs vérifiables, contrôleurs de recherche

---

### 13. **Graph of Thought** - Fusion DAG pour sorties multi-sources
`graph-of-thought/` | [Code](examples/13_graph-of-thought/graph-of-thought.js) | [Explication du code](examples/13_graph-of-thought/CODE.md) | [Concepts](examples/13_graph-of-thought/CONCEPT.md)

**Ce que vous apprendrez :**
- Modéliser le raisonnement comme un DAG : extractions sources parallèles → règles de fusion → version finale
- Résoudre les conflits explicitement avant la génération (`must_include`, `must_avoid`, `conflict_notes`)
- Ajouter des contrôles déterministes de fusion et de conformité du draft
- Exécuter les nœuds indépendants en parallèle pour réduire la latence

**Concepts clés** : Graph of Thought, orchestration DAG, fusion multi-sources, merge-before-generate, réconciliation de politiques

**Guide de décision** : utilisez ToT quand vous devez explorer des chemins concurrents ; utilisez GoT quand vous devez combiner plusieurs sources en une politique cohérente. Comparez les deux dans :
- [Concept ToT](examples/12_tree-of-thought/CONCEPT.md)
- [Concept GoT](examples/13_graph-of-thought/CONCEPT.md)

---

### 14. **Chain of Thought** - Prise de décision auditable par étapes
`chain-of-thought/` | [Code](examples/14_chain-of-thought/chain-of-thought.js) | [Explication du code](examples/14_chain-of-thought/CODE.md) | [Concepts](examples/14_chain-of-thought/CONCEPT.md)

**Ce que vous apprendrez :**
- Diviser une décision à risque élevé en phases de raisonnement explicites
- Prévenir le biais précoce avec une étape d'extraction de faits uniquement
- Équilibrer les signaux de fraude avec les preuves de légitimité avant l'application des politiques
- Produire une décision finale auditable avec des sorties adaptées au client et internes

**Concepts clés** : Chain of Thought, traces de raisonnement structurées, décisions contraintes par politiques, explicabilité, workflows prêts pour relecture

---

## Structure de la Documentation

Chaque dossier d'exemple contient :

- **`<name>.js`** - L'exemple de code fonctionnel
- **`CODE.md`** - Explication détaillée du code
  - Analyse ligne par ligne
  - Ce que fait chaque partie
  - Comment ça marche
- **`CONCEPT.md`** - Concepts de haut niveau
  - Pourquoi c'est important pour les agents
  - Patterns architecturaux
  - Applications réelles
  - Diagrammes simples

## Concepts Clés

### Qu'est-ce qu'un AI Agent ?

```
AI Agent = LLM + System Prompt + Tools + Memory + Reasoning Pattern
           ─┬─   ──────┬──────   ──┬──   ──┬───   ────────┬────────
            │          │           │       │              │
         Brain      Identity    Hands   State         Strategy
```

### Évolution des Capacités

```
1. intro          → Usage de base du LLM
2. translation    → Comportement spécialisé (system prompts)
3. think          → Capacité de raisonnement
4. batch          → Traitement parallèle
5. coding         → Streaming & contrôle
6. simple-agent   → Utilisation d'outils (function calling)
7. memory-agent   → État persistant
8. react-agent    → Raisonnement stratégique + utilisation d'outils
```

### Patterns Architecturaux

**Simple Agent (Étapes 1-5)**
```
User → LLM → Response
```

**Tool-Using Agent (Étape 6)**
```
User → LLM ⟷ Tools → Response
```

**Memory Agent (Étape 7)**
```
User → LLM ⟷ Tools → Response
       ↕
     Memory
```

**ReAct Agent (Étape 8)**
```
User → LLM → Think → Act → Observe
       ↑      ↓      ↓      ↓
       └──────┴──────┴──────┘
           Itérer jusqu'à résolution
```

## Utilitaires Helper

### PromptDebugger
`helper/prompt-debugger.js`

Utilitaire pour déboguer les prompts envoyés au LLM. Affiche exactement ce que le model voit, y compris :
- System prompts
- Function definitions
- Historique de conversation
- État du context

Exemple d'utilisation dans `simple-agent/simple-agent.js`

## ️ Structure du Projet - Fondamentaux

```
ai-agents/
├── README.md                          ← Vous êtes ici
├─ examples/
├── 01_intro/
│   ├── intro.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 02_openai-intro/
│   ├── openai-intro.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 03_translation/
│   ├── translation.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 04_think/
│   ├── think.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 05_batch/
│   ├── batch.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 06_coding/
│   ├── coding.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 07_simple-agent/
│   ├── simple-agent.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 08_simple-agent-with-memory/
│   ├── simple-agent-with-memory.js
│   ├── memory-manager.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 09_react-agent/
│   ├── react-agent.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 10_aot-agent/
│   ├── aot-agent.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 11_error-handling/
│   ├── error-handling.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 12_tree-of-thought/
│   ├── tree-of-thought.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 13_graph-of-thought/
│   ├── graph-of-thought.js
│   ├── CODE.md
│   └── CONCEPT.md
├── 14_chain-of-thought/
│   ├── chain-of-thought.js
│   ├── CODE.md
│   └── CONCEPT.md
├── helper/
│   └── prompt-debugger.js
├── models/                             ← Placez vos models GGUF ici
└── logs/                               ← Sorties de debug
```

## Ressources Complémentaires

- **node-llama-cpp** : [GitHub](https://github.com/withcatai/node-llama-cpp)
- **Model Hub** : [Hugging Face](https://huggingface.co/models?library=gguf)
- **Format GGUF** : Models quantifiés pour l'inference locale

## Contribuer

Il s'agit d'une ressource éducative. N'hésitez pas à :
- Suggérer des améliorations à la documentation
- Ajouter d'autres patterns d'exemples
- Corriger des bugs ou des explications floues
- Partager ce que vous avez construit !

## Licence

Ressource éducative - utilisez et modifiez selon vos besoins pour l'apprentissage.

---

**Construit avec ❤️ pour ceux qui veulent vraiment comprendre les AI agents**

Commencez par `intro/` et progressez étape par étape. Chaque exemple s'appuie sur le précédent. Lisez à la fois CODE.md et CONCEPT.md pour une compréhension complète.

Bon apprentissage ! 
