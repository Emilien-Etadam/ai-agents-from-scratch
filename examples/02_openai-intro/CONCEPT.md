# Concepts : Comprendre les APIs OpenAI

Ce guide explique les concepts fondamentaux du travail avec les models de langage d'OpenAI, qui constituent la base pour construire des AI agents.

## Qu'est-ce que l'API OpenAI ?

L'API OpenAI donne un accès programmatique à des models de langage puissants comme GPT-4o et GPT-3.5-turbo. Au lieu d'exécuter les models localement, vous envoyez des requêtes aux serveurs d'OpenAI et recevez des réponses.

**Caractéristiques principales :**
- **Cloud-based :** Les models tournent sur l'infrastructure d'OpenAI
- **Pay-per-use :** Facturation par consommation de tokens
- **Production-ready :** Fiabilité et performance de niveau entreprise
- **Derniers models :** Accès immédiat aux dernières sorties de models

**Comparaison avec les LLMs Locaux (comme node-llama-cpp) :**

| Aspect | API OpenAI | LLMs Locaux |
|--------|------------|------------|
| **Configuration** | Clé API seulement | Télécharger les models, besoin GPU/RAM |
| **Coût** | Payez par token | Gratuit après configuration initiale |
| **Performance** | Cohérente, haute qualité | Dépend de votre matériel |
| **Confidentialité** | Données envoyées à OpenAI | Entièrement local/privé |
| **Scalabilité** | Illimitée (avec paiement) | Limitée par votre matériel |

---

## L'API Chat Completions

### Cycle Requête-Réponse

```
Vous (Client)                    OpenAI (Serveur)
     |                                |
     |  POST /v1/chat/completions    |
     |  {                             |
     |    model: "gpt-4o",            |
     |    messages: [...]             |
     |  }                             |
     |------------------------------->|
     |                                |
     |        [Traitement...]         |
     |        [Inférence du model]    |
     |        [Génération réponse]    |
     |                                |
     |  Réponse                        |
     |  {                             |
     |    choices: [{                 |
     |      message: {                |
     |        content: "..."          |
     |      }                         |
     |    }]                          |
     |  }                             |
     |<-------------------------------|
     |                                |
```

**Point clé :** Chaque requête est indépendante. L'API ne stocke pas l'historique de conversation.

---

## Rôles des Messages : La Structure de la Conversation

Chaque message a un `role` qui détermine son but :

### 1. Messages Système

```javascript
{ role: 'system', content: 'You are a helpful Python tutor.' }
```

**But :** Définir le comportement, la personnalité et les capacités de l'IA

**Pensez-y comme :**
- La « fiche de poste » de l'IA
- Invisible pour l'utilisateur final
- Définit les contraintes et les directives

**Exemples :**
```javascript
// Agent spécialiste
"You are an expert SQL database administrator."

// Ton et style
"You are a friendly customer support agent. Be warm and empathetic."

// Contrôle du format de sortie
"You are a JSON API. Always respond with valid JSON, never plain text."

// Contraintes comportementales
"You are a code reviewer. Be constructive and focus on best practices."
```

**Bonnes pratiques :**
- Restez concis mais précis
- Placez au début du tableau messages
- Mettez à jour pour changer le comportement de l'agent
- Utilisez pour les directives éthiques et le formatage de sortie

### 2. Messages Utilisateur

```javascript
{ role: 'user', content: 'How do I use async/await?' }
```

**But :** Représenter l'entrée ou les questions de l'humain

**Pensez-y comme :**
- Ce que vous demandez à l'IA
- Le prompt ou la requête
- L'instruction à suivre

### 3. Messages Assistant

```javascript
{ role: 'assistant', content: 'Async/await is a way to handle promises...' }
```

**But :** Représenter les réponses précédentes de l'IA

**Pensez-y comme :**
- L'historique de conversation de l'IA
- Contexte pour les questions de suivi
- Ce que l'IA a déjà dit

### Exemple de Flux de Conversation

```javascript
[
  { role: 'system', content: 'You are a math tutor.' },

  // Premier échange
  { role: 'user', content: 'What is 15 * 24?' },
  { role: 'assistant', content: '15 * 24 = 360' },

  // Suite (connaît le contexte)
  { role: 'user', content: 'What about dividing that by 3?' },
  { role: 'assistant', content: '360 ÷ 3 = 120' },
]
```

**Pourquoi c'est important :** La structure des rôles permet :
1. **Conscience du contexte :** L'IA comprend l'historique de la conversation
2. **Contrôle du comportement :** Les system prompts façonnent les réponses
3. **Conversations multi-tours :** Dialogue naturel aller-retour

---

## Stateless : Un Concept Critique

**Principe le plus important :** L'API d'OpenAI est stateless.

### Que signifie stateless ?

Chaque appel API est indépendant. Le model ne se souvient pas des requêtes précédentes.

```
Requête 1 : "Mon nom est Alice"
Réponse 1 : "Bonjour Alice !"

Requête 2 : "Comment m'appelle-je ?"
Réponse 2 : "Je ne connais pas votre nom."  ← Pas de mémoire !
```

### Comment maintenir le contexte

**Vous devez envoyer l'historique complet de la conversation :**

```javascript
const messages = [];

// Premier tour
messages.push({ role: 'user', content: 'My name is Alice' });
const response1 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages  // ["My name is Alice"]
});
messages.push(response1.choices[0].message);

// Deuxième tour - inclure tout l'historique
messages.push({ role: 'user', content: "What's my name?" });
const response2 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages  // Conversation complète !
});
```

### Implications

**Avantages :**
- ✅ Architecture simple (pas d'état côté serveur)
- ✅ Facile à scaler (n'importe quel serveur peut gérer n'importe quelle requête)
- ✅ Contrôle total du contexte (vous décidez ce qu'on inclut)

**Défis :**
- ❌ Vous gérez l'historique de la conversation
- ❌ Les coûts en tokens augmentent avec la longueur de la conversation
- ❌ Doit implémenter sa propre mémoire/persistance
- ❌ Les limites de fenêtre de context finissent par être atteintes

**Solutions en production :**
```javascript
// Réduire les anciens messages quand c'est trop long
if (messages.length > 20) {
    messages = [messages[0], ...messages.slice(-10)];  // Garder system + 10 derniers
}

// Résumer l'ancien contexte
if (totalTokens > 10000) {
    const summary = await summarizeConversation(messages);
    messages = [systemMessage, summary, ...recentMessages];
}
```

---

## Temperature : Contrôler l'Aléatoire

La temperature contrôle à quel point le modèle est « créatif » ou « aléatoire » dans sa sortie.

### Comment ça marche techniquement

Lors de la génération de chaque token, le model attribue des probabilités aux prochains tokens possibles :

```
Entrée : "The sky is"
Prochains tokens possibles :
  - "blue"     → 70% de probabilité
  - "clear"    → 15% de probabilité
  - "dark"     → 10% de probabilité
  - "purple"   → 5% de probabilité
```

**La temperature modifie ces probabilités :**

**Temperature = 0.0 (Déterministe)**
```
Toujours choisir le token de plus haute probabilité
"The sky is blue"  ← Même sortie à chaque fois
```

**Temperature = 0.7 (Équilibré)**
```
Échantillonner probabilistiquement avec un peu d'aléatoire
"The sky is blue" ou "The sky is clear"
```

**Temperature = 1.5 (Créatif)**
```
Aplatir les probabilités, autoriser les choix improbables
"The sky is purple" ou "The sky is dancing"  ← Plus surprenant !
```

### Guides Pratiques

**Temperature 0.0 - 0.3 : Tâches Focalisées**
- Génération de code
- Extraction de données
- Q&R factuel
- Classification
- Traduction

Exemple :
```javascript
// Extraire du JSON d'un texte - besoin de cohérence
temperature: 0.1
```

**Temperature 0.5 - 0.9 : Tâches Équilibrées**
- Conversation générale
- Support client
- Résumation de contenu
- Contenu éducatif

Exemple :
```javascript
// Chatbot amical
temperature: 0.7
```

**Temperature 1.0 - 2.0 : Tâches Créatives**
- Écriture de stories
- Brainstorming
- Poésie/contenu créatif
- Génération de variations

Exemple :
```javascript
// Générer 10 slogans marketing différents
temperature: 1.3
```

---

## Streaming : Réponses en Temps Réel

### Non-Streaming (Par Défaut)

```
Utilisateur : "Raconte-moi une story"
[Attente...]
[Attente...]
[Attente...]
Réponse : "Il était une fois..." (tout d'un coup)
```

**Avantages :**
- Simple à implémenter
- Gestion des erreurs facile
- Réponse complète obtenue avant traitement

**Inconvénients :**
- Paraît lent pour les réponses longues
- Pas de feedback pendant la génération
- Mauvaise expérience utilisateur pour le chat

### Streaming

```
Utilisateur : "Raconte-moi une story"
"Once"
"Once upon"
"Once upon a"
"Once upon a time"
"Once upon a time there"
...
```

**Avantages :**
- Feedback immédiat
- Paraît plus rapide
- Meilleure expérience utilisateur
- Peut traiter les tokens au fur et à mesure

**Inconvénients :**
- Code plus complexe
- Gestion des erreurs plus difficile
- Impossible de voir la réponse complète avant affichage

### Quand Utiliser Chacun

**Utiliser Non-Streaming :**
- Scripts de batch processing
- Quand vous devez analyser la réponse complète
- Outils en ligne de commande simples
- Endpoints API qui retournent des résultats complets

**Utiliser Streaming :**
- Interfaces de chat
- Applications interactives
- Génération de contenu long
- Toute application grand public où l'UX compte

---

## Tokens : La Monnaie des LLMs

### Qu'est-ce que les tokens ?

Les tokens sont les unités fondamentales que les models de langage traitent. Ce ne sont pas exactement des mots, mais des morceaux de texte.

**Exemples de tokenization :**
```
"Hello world"        → ["Hello", " world"]           = 2 tokens
"coding"             → ["coding"]                    = 1 token
"uncoded"            → ["un", "coded"]               = 2 tokens
```

### Pourquoi les tokens comptent

**1. Coût**
Vous payez par token (entrée + sortie) :
```
Requête : 100 tokens
Réponse : 150 tokens
Total facturé : 250 tokens
```

**2. Limites de Context**
Chaque model a une limite maximale de tokens :
```
gpt-4o:        128 000 tokens  (≈96 000 mots)
gpt-3.5-turbo: 16 384 tokens   (≈12 000 mots)
```

**3. Performance**
Plus de tokens = temps de traitement plus long et coût plus élevé

### Gérer l'Usage des Tokens

**Surveiller l'usage :**
```javascript
console.log(response.usage.total_tokens);
// Suivre l'usage cumulatif pour le budget
```

**Limiter la longueur de réponse :**
```javascript
max_tokens: 150  // Caper la réponse
```

**Réduire l'historique de conversation :**
```javascript
// Garder uniquement les messages récents
if (messages.length > 20) {
    messages = messages.slice(-20);
}
```

**Estimer avant d'envoyer :**
```javascript
import { encode } from 'gpt-tokenizer';

const text = "Your message here";
const tokens = encode(text).length;
console.log(`Estimated tokens: ${tokens}`);
```

---

## Sélection de Model : Choisir le Bon Outil

### GPT-4o : Le Plus Puissant

**Idéal pour :**
- Tâches de raisonnement complexes
- Génération et débogage de code
- Contenu technique
- Tâches nécessitant une haute précision
- Travail avec des données structurées

**Caractéristiques :**
- Model le plus performant
- Coût plus élevé
- Plus lent que GPT-3.5
- Idéal pour les applications critiques en qualité

**Exemples de cas d'usage :**
- Analyse de documents juridiques
- Refactoring de code complexe
- Recherche et analyse
- Tutorat éducatif

### GPT-4o-mini : Le Choix Équilibré

**Idéal pour :**
- Applications généralistes
- Bon équilibre coût/performance
- La plupart des tâches quotidiennes

**Caractéristiques :**
- Bonnes performances
- Coût modéré
- Temps de réponse rapides
- Sweet spot pour de nombreuses applications

**Exemples de cas d'usage :**
- Chatbots de support client
- Résumation de contenu
- Q&R généraliste
- Tâches de complexité modérée

### GPT-3.5-turbo : La Bête à Vitesse

**Idéal pour :**
- Tâches simples en haut volume
- Applications critiques en vitesse
- Projets à budget serré
- Classification et extraction

**Caractéristiques :**
- Très rapide
- Coût le plus bas
- Bon pour les tâches simples
- Raisonnement moins performant

**Exemples de cas d'usage :**
- Analyse de sentiment
- Classification de texte
- Formatage simple
- Traitement à haut débit

### Cadre de Décision

```
La tâche est critique et complexe ?
├─ OUI → GPT-4o
└─ NON
   └─ La vitesse est importante et la tâche simple ?
      ├─ OUI → GPT-3.5-turbo
      └─ NON → GPT-4o-mini
```

---

## Gestion des Erreurs et Résilience

### Scénarios d'Erreurs Courantes

**1. Erreurs d'Authentification (401)**
```javascript
// Clé API invalide
Error: Incorrect API key provided
```

**2. Rate Limiting (429)**
```javascript
// Trop de requêtes
Error: Rate limit exceeded
```

**3. Limites de Tokens (400)**
```javascript
// Context trop long
Error: This model's maximum context length is 16385 tokens
```

**4. Erreurs de Service (500)**
```javascript
// Problème de service OpenAI
Error: The server had an error processing your request
```

### Bonnes Pratiques

**1. Toujours utiliser try-catch :**
```javascript
try {
    const response = await client.chat.completions.create({...});
} catch (error) {
    if (error.status === 429) {
        // Implémenter backoff et retry
    } else if (error.status === 500) {
        // Retry avec backoff exponentiel
    } else {
        // Logger et gérer de manière appropriée
    }
}
```

**2. Implémenter une logique de retry :**
```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            if (i === maxRetries - 1) throw error;
            await sleep(Math.pow(2, i) * 1000);  // Backoff exponentiel
        }
    }
}
```

**3. Surveiller l'usage des tokens :**
```javascript
let totalTokens = 0;
totalTokens += response.usage.total_tokens;

if (totalTokens > MONTHLY_BUDGET_TOKENS) {
    throw new Error('Monthly token budget exceeded');
}
```

---

## Patterns Architecturaux

### Pattern 1 : Requête-Réponse Simple

**Cas d'usage :** Requêtes ponctuelles, automatisation simple

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: query }]
});
```

**Avantages :** Simple, facile à comprendre
**Inconvénients :** Pas de contexte, pas de mémoire

### Pattern 2 : Conversation Stateful

**Cas d'usage :** Applications de chat, tutorat, support client

```javascript
class Conversation {
    constructor() {
        this.messages = [
            { role: 'system', content: 'Your behavior' }
        ];
    }

    async ask(userMessage) {
        this.messages.push({ role: 'user', content: userMessage });

        const response = await client.chat.completions.create({
            model: 'gpt-4o',
            messages: this.messages
        });

        this.messages.push(response.choices[0].message);
        return response.choices[0].message.content;
    }
}
```

**Avantages :** Maintient le contexte, conversation naturelle
**Inconvénients :** Les coûts en tokens augmentent, besoin de gestion

### Pattern 3 : Agents Spécialisés

**Cas d'usage :** Applications spécifiques à un domaine

```javascript
class PythonTutor {
    async help(question) {
        return await client.chat.completions.create({
            model: 'gpt-4o',
            messages: [
                {
                    role: 'system',
                    content: 'You are an expert Python tutor. Explain concepts clearly with code examples.'
                },
                { role: 'user', content: question }
            ],
            temperature: 0.3  // Réponses focalisées
        });
    }
}
```

**Avantages :** Comportement cohérent, optimisé pour le domaine
**Inconvénients :** Moins flexible

---

## Approche Hybride : Combiner Models Propriétaires et Open Source

Dans les projets réels, la meilleure solution n'est souvent pas de choisir entre OpenAI et les LLMs locaux — c'est d'utiliser **les deux de manière stratégique**.

### Pourquoi Utiliser une Approche Hybride ?

**Optimisation des coûts :** Utiliser des models chers uniquement quand nécessaire
**Conformité confidentialité :** Garder les données sensibles localement tout en exploitant le cloud pour les tâches générales
**Équilibre performance :** Models locaux rapides pour les tâches simples, models cloud puissants pour les tâches complexes
**Fiabilité :** Options de fallback quand un service est en panne
**Flexibilité :** Associer le bon outil à chaque tâche spécifique

### Architectures Hybrides Courantes

#### Pattern 1 : Traitement par Niveaux

```
Tâches simples → LLM Local (rapide, gratuit, privé)
    ↓ Si complexe
Tâches complexes → API OpenAI (puissant, précis)
```

**Exemple de workflow :**
```javascript
async function processQuery(query) {
    const complexity = await assessComplexity(query);

    if (complexity < 0.5) {
        // Utiliser le model local pour les requêtes simples
        return await localLLM.generate(query);
    } else {
        // Utiliser OpenAI pour le raisonnement complexe
        return await openai.chat.completions.create({
            model: 'gpt-4o',
            messages: [{ role: 'user', content: query }]
        });
    }
}
```

**Cas d'usage :**
- Support client : Model local pour les FAQ, GPT-4 pour les problèmes complexes
- Génération de code : Local pour les scripts simples, GPT-4 pour l'architecture
- Modération de contenu : Local pour les cas évidents, cloud pour les cas limites

#### Pattern 2 : Routage Basé sur la Confidentialité

```
Données publiques → OpenAI (meilleure qualité)
Données sensibles → LLM Local (privé, sécurisé)
```

**Exemple :**
```javascript
async function handleRequest(data, containsSensitiveInfo) {
    if (containsSensitiveInfo) {
        // Traiter localement - les données ne quittent jamais votre infrastructure
        return await localLLM.generate(data, {
            systemPrompt: "You are a HIPAA-compliant assistant"
        });
    } else {
        // Utiliser le cloud pour une meilleure qualité
        return await openai.chat.completions.create({
            model: 'gpt-4o',
            messages: [{ role: 'user', content: data }]
        });
    }
}
```

**Cas d'usage :**
- Santé : Données patients → Local, Infos médicales générales → OpenAI
- Finance : Détails de transactions → Local, Analyse marché → OpenAI
- Juridique : Communications clients → Local, Recherche juridique → OpenAI

#### Pattern 3 : Écosystème d'Agents Spécialisés

```
Agent 1 (Local) : Classifieur rapide
    ↓ Route vers
Agent 2 (OpenAI) : Analyseur approfondi
    ↓ Route vers
Agent 3 (Local) : Exécuteur d'actions
```

**Exemple :**
```javascript
class MultiModelAgent {
    async process(input) {
        // Étape 1 : Le model local classe l'intention (rapide, peu cher)
        const intent = await localLLM.classify(input);

        // Étape 2 : Router vers le gestionnaire approprié
        if (intent.requiresReasoning) {
            // Raisonnement complexe avec GPT-4
            const analysis = await openai.chat.completions.create({
                model: 'gpt-4o',
                messages: [{ role: 'user', content: input }]
            });
            return analysis.choices[0].message.content;
        } else {
            // Réponse simple avec le model local
            return await localLLM.generate(input);
        }
    }
}
```

**Cas d'usage :**
- Pipelines multi-étapes avec différents niveaux de complexité
- Systèmes d'agents où chaque agent a des capacités spécialisées
- Workflows nécessitant à la fois vitesse et intelligence

#### Pattern 4 : Développement vs Production

```
Développement → OpenAI (itération rapide, meilleurs résultats)
    ↓ Optimiser
Production → LLM Local (rentable, privé)
```

**Workflow :**
```javascript
const MODEL_PROVIDER = process.env.NODE_ENV === 'production'
    ? 'local'
    : 'openai';

async function generateResponse(prompt) {
    if (MODEL_PROVIDER === 'local') {
        return await localLLM.generate(prompt);
    } else {
        return await openai.chat.completions.create({
            model: 'gpt-4o',
            messages: [{ role: 'user', content: prompt }]
        });
    }
}
```

**Stratégie :**
1. Développer avec GPT-4 pour obtenir les meilleurs résultats rapidement
2. Ajuster les prompts et tester minutieusement
3. Passer au model local pour la production
4. Revenir à OpenAI pour les cas limites

#### Pattern 5 : Approche par Ensemble

```
Requête → [Model Local, OpenAI, Autre API]
             ↓          ↓            ↓
          Réponse   Réponse     Réponse
             ↓          ↓            ↓
          Agrégateur / Validateur
                    ↓
              Meilleure Réponse
```

**Exemple :**
```javascript
async function ensembleGenerate(prompt) {
    // Obtenir des réponses de plusieurs sources
    const [local, openai, backup] = await Promise.allSettled([
        localLLM.generate(prompt),
        openaiClient.chat.completions.create({
            model: 'gpt-4o',
            messages: [{ role: 'user', content: prompt }]
        }),
        backupAPI.generate(prompt)
    ]);

    // Utiliser un validateur pour choisir la meilleure ou combiner
    return validator.selectBest([local, openai, backup]);
}
```

**Cas d'usage :**
- Applications critiques nécessitant une haute confiance
- Vérification de faits et validation
- Réduction des hallucinations par consensus

### Analyse Coût-Bénéfice

#### Scénario : Chatbot de Support Client (10 000 requêtes/jour)

**Option A : Uniquement OpenAI**
```
10 000 requêtes × 500 tokens moy. = 5M tokens/jour
Coût : ~25-50$/jour = ~750-1500$/mois
Avantages : Qualité maximale, zéro infrastructure
Inconvénients : Cher à grande échelle, problèmes de confidentialité
```

**Option B : Uniquement LLM Local**
```
Infrastructure : 100-500$/mois (serveur/GPU)
Coût : 100-500$/mois
Avantages : Coûts prévisibles, privé, usage illimité
Inconvénients : Complexité de configuration, maintenance, qualité inférieure
```

**Option C : Hybride (80% local, 20% OpenAI)**
```
8 000 requêtes simples → LLM Local (gratuit après configuration)
2 000 requêtes complexes → OpenAI (~5-10$/jour)
Infrastructure : 100-500$/mois
Coûts API : 150-300$/mois
Total : 250-800$/mois
Avantages : Rentable, haute qualité quand besoin, flexible
Inconvénients : Architecture plus complexe
```

**Gagnant pour la plupart des projets : Approche Hybride** ✓

### Cadre de Décision

```
DÉMARRAGE : Nouvelle requête arrive
    ↓
Les données sont sensibles/réglementées ?
├─ OUI → Utiliser le model local (confidentialité d'abord)
└─ NON → Continuer
    ↓
La tâche est simple/répétitive ?
├─ OUI → Utiliser le model local (rentable)
└─ NON → Continuer
    ↓
La haute précision est critique ?
├─ OUI → Utiliser OpenAI (qualité d'abord)
└─ NON → Continuer
    ↓
Est-ce du haut volume ?
├─ OUI → Utiliser le model local (coût à l'échelle)
└─ NON → Utiliser OpenAI (simplicité)
```

### L'Avenir : Sélection Intelligente de Models

Les systèmes avancés choisiront automatiquement les models en fonction de facteurs temps réel :

```javascript
class IntelligentModelSelector {
    async selectModel(query, context) {
        const factors = {
            complexity: await this.analyzeComplexity(query),
            latency: context.userTolerance,
            budget: context.remainingBudget,
            accuracy: context.requiredConfidence,
            privacy: context.dataClassification
        };

        // Un model ML prédit le meilleur fournisseur
        const selection = await this.mlSelector.predict(factors);

        return {
            provider: selection.provider,  // 'local' | 'openai-mini' | 'openai-4'
            confidence: selection.confidence,
            reasoning: selection.reasoning
        };
    }
}
```

### Point Clé

**Vous n'êtes pas obligé de choisir.** Les applications IA modernes bénéficient d'utiliser le bon model pour chaque tâche :
- **OpenAI / Claude / Models open source auto-hébergés :** Raisonnement complexe, précision critique, développement rapide
- **Local pour le scale :** Confidentialité, contrôle des coûts, haut volume, fonctionnement hors ligne
- **Les deux pour réussir :** Systèmes de production rentables, flexibles et fiables

La meilleure architecture exploite les forces de chaque approche tout en atténuant leurs faiblesses.

---

## Se Préparer pour les Agents

Les concepts couverts ici sont **fondamentaux** pour construire des AI agents :

### Vous comprenez maintenant :

- **Comment communiquer avec les LLMs** (bases de l'API)
- **Comment façonner le comportement** (system prompts)
- **Comment maintenir le contexte** (historique des messages)
- **Comment contrôler la sortie** (temperature, tokens)
- **Comment gérer les réponses** (streaming, erreurs)

### Ce qui arrive ensuite pour les agents :

- **Function calling / Tool use** - Permettre à l'IA d'entreprendre des actions
- **Systèmes de mémoire** - État persistant entre les sessions
- **Patterns ReAct** - Raisonnement et observation itératifs

**En bref :** Vous ne pouvez pas construire de bons agents sans maîtriser ces fondamentaux. Chaque pattern d'agent se construit sur cette base.

---

## Points Clés

1. **Le stateless est une force et un fardeau :** Vous contrôlez le contexte, mais vous devez le gérer
2. **Les system prompts sont votre arme secrète :** Même model → comportements différents
3. **La temperature change tout :** Adaptez-la à votre type de tâche
4. **Les tokens sont la vraie monnaie :** Surveillez et optimisez l'usage
5. **Le choix du model compte :** N'utilisez pas une masse pour taper un clou
6. **Le streaming améliore l'UX :** Utilisez-le pour les applications grand public
7. **La gestion des erreurs n'est pas optionnelle :** Le réseau va planter, prévoyez-le

---

## Lectures Complémentaires

- [Documentation API OpenAI](https://platform.openai.com/docs/api-reference)
- [OpenAI Cookbook](https://cookbook.openai.com/)
- [Bonnes Pratiques pour le Prompt Engineering](https://platform.openai.com/docs/guides/prompt-engineering)
- [Comptage de Tokens](https://platform.openai.com/tokenizer)
