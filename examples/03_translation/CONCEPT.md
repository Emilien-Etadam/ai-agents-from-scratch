# Concept : System Prompts et Spécialisation d'Agents

## Vue d'Ensemble

Cet exemple démontre comment transformer un LLM généraliste en un **agent spécialisé** à l'aide de **system prompts**. L'insight clé : vous n'avez pas besoin de models différents pour des tâches différentes — vous avez besoin d'instructions différentes.

## Qu'est-ce qu'un System Prompt ?

Un **system prompt** est une instruction persistante qui façonne le comportement de l'IA pour toute une session de conversation.

### Analogie
Pensez à recruer quelqu'un pour un poste :

```
Sans System Prompt             Avec System Prompt
─────────────────────         ──────────────────────
"Salut, je suis une IA."      "Je suis un traducteur professionnel
"Qu'est-ce que je peux faire ?" spécialisé en allemand scientifique.
                               Je respecte des directives et un format
                               de sortie stricts."
```

## Comment les System Prompts Fonctionnent

### Structure du Context

```
┌─────────────────────────────────────────────┐
│           FENÊTRE DE CONTEXT                │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  SYSTEM PROMPT (Toujours présent)     │ │
│  │  "You are a professional translator..." │
│  │  "Follow these rules..."              │ │
│  └───────────────────────────────────────┘ │
│                    ↓                        │
│  ┌───────────────────────────────────────┐ │
│  │  MESSAGES UTILISATEUR                 │ │
│  │  "Translate this text..."             │ │
│  └───────────────────────────────────────┘ │
│                    ↓                        │
│  ┌───────────────────────────────────────┐ │
│  │  RÉPONSES IA                          │ │
│  │  (Façonnées par le system prompt)     │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

Le system prompt est au sommet du contexte et influence **chaque** réponse.

## Pattern de Spécialisation d'Agent

### Flux de Transformation

```
┌──────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  Model Général   │ +  │ System Prompt   │ =  │ Agent Spécialisé │
│                  │    │                 │    │                  │
│ • Connaît plein  │    │ • Définir rôle  │    │ • Agent          │
│   de choses      │    │ • Poser règles  │    │   Traduction     │
│ • Pas de rôle    │    │ • Contraindre   │    │ • Agent Codeur   │
│   spécifique     │    │   la sortie     │    │ • Agent Analyse  │
└──────────────────┘    └─────────────────┘    └──────────────────┘
```

### Exemples de Spécialisation

**Agent Traduction (cet exemple) :**
```
System Prompt = Rôle + Règles + Format de Sortie
```

**Assistant Code :**
```javascript
systemPrompt: "You are an expert programmer.
Always provide working code with comments.
Explain complex logic."
```

**Analyste de Données :**
```javascript
systemPrompt: "You are a data analyst.
Always show your calculations step-by-step.
Cite data sources when available."
```

## Anatomie d'un System Prompt Efficace

### Les 5 Composants

```
┌─────────────────────────────────────────┐
│  1. DÉFINITION DU RÔLE                   │
│  "You are a [rôle spécifique]..."       │
├─────────────────────────────────────────┤
│  2. DESCRIPTION DE LA TÂCHE              │
│  "Your goal is to..."                   │
├─────────────────────────────────────────┤
│  3. RÈGES COMPORTIMENTALES               │
│  "Always do X, Never do Y..."           │
├─────────────────────────────────────────┤
│  4. FORMAT DE SORTIE                    │
│  "Format your response as..."           │
├─────────────────────────────────────────┤
│  5. CONTRAINTES                         │
│  "Do NOT include..."                    │
└─────────────────────────────────────────┘
```

### Structure de cet Exemple

```
Rôle :        "Traducteur scientifique professionnel"
Tâche :       "Traduire l'anglais vers l'allemand avec précision"
Règles :      8 directives de traduction spécifiques
Format :      Allemand idiomatique, style scientifique
Contraintes : "UNIQUEMENT le texte traduit, aucune explication"
```

## Pourquoi les System Prompts Détaillés Comptent

### Étude Comparative

**System Prompt Minimal :**
```javascript
systemPrompt: "Translate to German"
```

**Résultat :**
- Peut ajouter des explications inutiles
- Terminologie incohérente
- Niveaux de formalité mélangés
- Texte conversationnel supplémentaire

**System Prompt Détaillé (cet exemple) :**
```javascript
systemPrompt: `You are a professional translator...
- Rule 1: Preserve technical accuracy
- Rule 2: Use idiomatic German
- Rule 3: Follow scientific conventions
...
DO NOT add any explanations`
```

**Résultat :**
- ✅ Qualité cohérente
- ✅ Terminologie correcte
- ✅ Formatage approprié
- ✅ Uniquement la traduction en sortie

### Impact sur la Qualité

```
Niveau de Détail           Qualité de Sortie
───────────               ─────────────────
Très minimal   →         Imprévisible
Rôle de base   →         Assez cohérent
Détaillé       →         Très cohérent ⭐
Trop détaillé  →         Peut confondre le model
```

## Patterns de Conception de System Prompt

### Pattern 1 : Role-Playing
```
"You are a [profession] with expertise in [domain]..."
```
Amène le model à adopter cette perspective.

### Pattern 2 : Rule-Based
```
"Follow these rules:
1. Always...
2. Never...
3. When X, do Y..."
```
Des contraintes explicites conduisent à un comportement prévisible.

### Pattern 3 : Formatage de Sortie
```
"Format your response as:
- JSON
- Markdown
- Plain text only
- Step-by-step list"
```
Contrôle la structure des réponses.

### Pattern 4 : Conscience Contextuelle
```
"You remember: [previous facts]
You know that: [domain knowledge]
Current situation: [context]"
```
Prépare le model avec des informations pertinentes.

## Comment Cela se Relie aux AI Agents

### Agent = Model + System Prompt + Outils

```
┌────────────────────────────────────────────┐
│             AI Agent                       │
│                                            │
│  ┌──────────────────────────────────────┐ │
│  │  System Prompt ("Identité" Agent)    │ │
│  └──────────────────────────────────────┘ │
│                  ↓                         │
│  ┌──────────────────────────────────────┐ │
│  │  LLM ("Cerveau" Agent)               │ │
│  └──────────────────────────────────────┘ │
│                  ↓                         │
│  ┌──────────────────────────────────────┐ │
│  │  Outils ("Mains" Agent) [Optionnel]  │ │
│  └──────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

**Dans cet exemple :**
- System Prompt : "You are a translator..."
- LLM : Model Apertus-8B
- Outils : Aucun (la traduction est faite par le model lui-même)

**Dans des agents plus complexes :**
- System Prompt : "You are a research assistant..."
- LLM : N'importe quel model
- Outils : Recherche web, calculatrice, accès fichiers, etc.

## Applications Pratiques

### 1. Spécialisation Domainale
```
Médical → "You are a medical professional..."
Juridique → "You are a legal expert..."
Technique → "You are an engineer..."
```

### 2. Contrôle de Sortie
```
API JSON → "Always respond in valid JSON"
Markdown → "Format all responses as markdown"
Code → "Only output executable code"
```

### 3. Contraintes Comportementales
```
Concis → "Use maximum 2 sentences"
Détaillé → "Explain thoroughly with examples"
Neutre → "Avoid opinions, state only facts"
```

### 4. Support Multi-Langue
```
systemPrompt: `You are a multilingual assistant.
Respond in the same language as the input.`
```

## Wrappers de Chat Expliqués

Les différents models nécessitent des formats de conversation différents :

```
Type de Model        Format Requis         Wrapper
──────────────       ───────────────────   ─────────────────
Llama 2/3           Format Llama          LlamaChatWrapper
Style GPT           Format ChatML         ChatMLWrapper
Models Harmony      Format Harmony        HarmonyChatWrapper
```

**Ce qu'ils font :**
```
Votre Message → [Chat Wrapper] → Prompt Formaté → Model
                    ↓
          Ajoute des tokens spéciaux :
          <|system|>, <|user|>, <|assistant|>
```

Le wrapper garantit que le model comprend quelle partie est le system prompt, quelle est le message utilisateur, etc.

## Points Clés

1. **Les system prompts sont puissants** : Ils changent fondamentalement comment le model se comporte
2. **Le détail est mieux** : Instructions plus spécifiques = résultats plus cohérents
3. **La structure compte** : Rôle + Règles + Format + Contraintes
4. **Pas besoin de retrain** : Même model, comportements différents
5. **Fondation des agents** : Les system prompts sont la première étape pour construire des agents spécialisés

## Parcours d'Évolution

```
1. Prompting de Base           (intro.js)
       ↓
2. System Prompts              (translation.js) ← Vous êtes ici
       ↓
3. System Prompts + Outils     (simple-agent.js)
       ↓
4. Raisonnement multi-tours    (react-agent.js)
       ↓
5. Systèmes Agents Complets
```

Cet exemple fait le pont entre l'usage basique des LLMs et le vrai comportement d'agent en montrant comment se spécialiser par des instructions.
