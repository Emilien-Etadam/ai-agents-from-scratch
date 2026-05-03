# Explication du Code : OpenAI Intro

Ce guide parcourt chaque exemple dans `openai-intro.js`, en expliquant comment travailler avec l'API d'OpenAI depuis zéro.

## Prérequis

Avant d'exécuter cet exemple, vous aurez besoin d'un compte OpenAI, d'une clé API et d'un moyen de paiement valide.

### Obtenir une Clé API

https://platform.openai.com/api-keys

### Ajouter un Moyen de Paiement

https://platform.openai.com/settings/organization/billing/overview

### Configurer les variables d'environnement

```bash
   cp .env.example .env
```
Puis éditez `.env` et ajoutez votre clé API réelle.

## Configuration et Initialisation

```javascript
import OpenAI from 'openai';
import 'dotenv/config';

const client = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
});
```

**Ce qui se passe :**
- `import OpenAI from 'openai'` - Importe le SDK officiel OpenAI pour Node.js
- `import 'dotenv/config'` - Charge les variables d'environnement depuis le fichier `.env`
- `new OpenAI({...})` - Crée une instance client qui gère l'authentification API et les requêtes
- `process.env.OPENAI_API_KEY` - Votre clé API depuis platform.openai.com (ne la mettez jamais en dur !)

**Pourquoi c'est important :** L'objet client est votre interface vers les models d'OpenAI. Tous les appels API passent par ce client.

---

## Exemple 1 : Chat Completion Basique

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'What is node-llama-cpp?' }
    ],
});

console.log(response.choices[0].message.content);
```

**Ce qui se passe :**
- `chat.completions.create()` - La méthode principale pour envoyer des messages aux models ChatGPT
- `model: 'gpt-4o'` - Spécifie quel model utiliser (gpt-4o est le plus récent et le plus performant)
- Tableau `messages` - Contient l'historique de la conversation
- `role: 'user'` - Indique que ce message vient de l'utilisateur (vous)
- `response.choices[0]` - L'API retourne un tableau de réponses possibles ; on prend la première
- `message.content` - Le texte réel de la réponse de l'IA

**Structure de la réponse :**
```javascript
{
  id: 'chatcmpl-...',
  object: 'chat.completion',
  created: 1234567890,
  model: 'gpt-4o',
  choices: [
    {
      index: 0,
      message: {
        role: 'assistant',
        content: 'node-llama-cpp is a...'
      },
      finish_reason: 'stop'
    }
  ],
  usage: {
    prompt_tokens: 10,
    completion_tokens: 50,
    total_tokens: 60
  }
}
```

---

## Exemple 2 : System Prompts

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'system', content: 'You are a coding assistant that talks like a pirate.' },
        { role: 'user', content: 'Explain what async/await does in JavaScript.' }
    ],
});
```

**Ce qui se passe :**
- `role: 'system'` - Type de message spécial qui définit le comportement et la personnalité de l'IA
- Les messages système sont traités en premier et influencent toutes les réponses suivantes
- Le model maintiendra ce comportement tout au long de la conversation

**Pourquoi c'est important :** Les system prompts sont le moyen de spécialiser le comportement de l'IA. Ils sont la base pour créer des agents focalisés avec des rôles spécifiques (traducteur, codeur, analyste, etc.).

**Insight clé :** Même model + system prompts différents = agents complètement différents !

---

## Exemple 3 : Contrôle de la Temperature

```javascript
// Réponse focalisée
const focusedResponse = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.2,
});

// Réponse créative
const creativeResponse = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
    temperature: 1.5,
});
```

**Ce qui se passe :**
- `temperature` - Contrôle le niveau d'aléatoire dans la sortie (plage : 0.0 à 2.0)
- **Temperature basse (0.0 - 0.3) :**
    - Plus focalisée et déterministe
    - Même entrée → sortie similaire
    - Idéale pour : réponses factuelles, génération de code, extraction de données
- **Temperature moyenne (0.7 - 1.0) :**
    - Équilibre entre créativité et cohérence
    - Valeur par défaut pour la plupart des cas d'usage
- **Temperature élevée (1.2 - 2.0) :**
    - Plus créative et variée
    - Même entrée → sorties très différentes
    - Idéale pour : écriture créative, brainstorming, génération de stories

**Usage en production :**
- Complétion de code : temperature 0.2
- Support client : temperature 0.5
- Contenu créatif : temperature 1.2

---

## Exemple 4 : Contexte de Conversation

```javascript
const messages = [
    { role: 'system', content: 'You are a helpful coding tutor.' },
    { role: 'user', content: 'What is a Promise in JavaScript?' },
];

const response1 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages,
});

// Ajouter la réponse de l'IA à l'historique
messages.push(response1.choices[0].message);

// Ajouter une question de suivi
messages.push({ role: 'user', content: 'Can you show me a simple example?' });

// Deuxième requête avec le contexte complet
const response2 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages,
});
```

**Ce qui se passe :**
- Les models OpenAI sont **stateless** — ils ne se souviennent pas des conversations précédentes
- On maintient le contexte en envoyant l'historique complet de la conversation à chaque requête
- Chaque requête est indépendante ; vous devez inclure tous les messages pertinents

**Ordre des messages dans le tableau :**
1. System prompt (optionnel, mais recommandé en premier)
2. Message utilisateur précédent
3. Réponse assistant précédente
4. Message utilisateur actuel

**Pourquoi c'est important :** C'est ainsi que les chatbots se souviennent du contexte. La conversation complète est envoyée à chaque fois.

**Considération de performance :**
- Plus de messages = plus de tokens = coût plus élevé
- Les conversations longues finissent par atteindre les limites de tokens
- Les applications réelles ont besoin de stratégies de résumation ou de réduction de conversation

---

## Exemple 5 : Réponses en Streaming

```javascript
const stream = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'Write a haiku about programming.' }
    ],
    stream: true,  // Activer le streaming
});

for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content || '';
    process.stdout.write(content);
}
```

**Ce qui se passe :**
- `stream: true` - Au lieu d'attendre la réponse complète, recevez-la token par token
- `for await...of` - Itère sur le stream au fur et à mesure que les chunks arrivent
- `delta.content` - Chaque chunk contient un petit morceau de texte (souvent juste un mot ou un fragment)
- `process.stdout.write()` - Écrit sans newline pour afficher le texte progressivement

**Streaming vs Non-streaming :**

**Non-streaming (par défaut) :**
```
[Requête envoyée]
[Attendre 5 secondes...]
[Réponse complète reçue]
```

**Streaming :**
```
[Requête envoyée]
Once [chunk reçu : "Once"]
upon [chunk reçu : " upon"]
a [chunk reçu : " a"]
time [chunk reçu : " time"]
...
```

**Pourquoi c'est important :**
- Meilleure expérience utilisateur (feedback immédiat)
- Paraît plus rapide même si le temps total est similaire
- Essentiel pour les interfaces de chat en temps réel
- Permet le traitement/l'affichage anticipé des résultats partiels

**Quand utiliser le streaming :**
- Applications de chat interactives
- Génération de contenu long
- Quand l'expérience utilisateur prime sur la simplicité

**Quand NE PAS utiliser le streaming :**
- Scripts simples ou automatisations
- Quand vous avez besoin de la réponse complète avant traitement
- Traitement par batch

---

## Exemple 6 : Usage des Tokens

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'Explain recursion in 3 sentences.' }
    ],
    max_tokens: 100,
});

console.log("Token usage:");
console.log("- Prompt tokens: " + response.usage.prompt_tokens);
console.log("- Completion tokens: " + response.usage.completion_tokens);
console.log("- Total tokens: " + response.usage.total_tokens);
```

**Ce qui se passe :**
- `max_tokens` - Limite la longueur de la réponse de l'IA
- `response.usage` - Contient les détails de consommation des tokens
- **Prompt tokens :** Votre entrée (messages envoyés)
- **Completion tokens :** La sortie de l'IA (la réponse)
- **Total tokens :** Somme des deux (ce pour quoi vous êtes facturé)

**Comprendre les tokens :**
- Tokens ≠ mots
- 1 token ≈ 0,75 mots (en anglais)
- "hello" = 1 token
- "chatbot" = 2 tokens ("chat" + "bot")
- La ponctuation et les espaces comptent comme des tokens

**Pourquoi c'est important :**
1. **Contrôle des coûts :** Vous payez par token
2. **Limites de context :** Les models ont des limites maximales de tokens (ex. gpt-4o : 128 000 tokens)
3. **Contrôle des réponses :** Utilisez `max_tokens` pour éviter des réponses trop longues

**Limites pratiques :**
```javascript
// Empêcher les réponses hors contrôle
max_tokens: 150,  // ~100 mots

// Réponses brèves
max_tokens: 50,   // ~35 mots

// Contenu plus long
max_tokens: 1000, // ~750 mots
```

**Estimation des coûts (approximatif) :**
- GPT-4o : $5 par 1M tokens d'entrée, $15 par 1M tokens de sortie
- GPT-3.5-turbo : $0,50 par 1M tokens d'entrée, $1,50 par 1M tokens de sortie

---

## Exemple 7 : Comparaison de Models

```javascript
// GPT-4o - Le plus performant
const gpt4Response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
});

// GPT-3.5-turbo - Plus rapide et moins cher
const gpt35Response = await client.chat.completions.create({
    model: 'gpt-3.5-turbo',
    messages: [{ role: 'user', content: prompt }],
});
```

**Models disponibles :**

| Model | Idéal Pour | Vitesse | Coût | Fenêtre de Context |
|-------|----------|-------|------|----------------|
| `gpt-4o` | Tâches complexes, raisonnement, précision | Moyenne | $$$ | 128K tokens |
| `gpt-4o-mini` | Équilibre performance/coût | Rapide | $$ | 128K tokens |
| `gpt-3.5-turbo` | Tâches simples, haut volume | Très Rapide | $ | 16K tokens |

**Choisir le bon model :**
- **Utilisez GPT-4o quand :**
    - Raisonnement complexe requis
    - La haute précision est critique
    - Travail avec du code ou du contenu technique
    - Qualité > vitesse/coût

- **Utilisez GPT-4o-mini quand :**
    - Besoin de bonnes performances à moindre coût
    - La plupart des tâches généralistes

- **Utilisez GPT-3.5-turbo quand :**
    - Classification ou extraction simple
    - Tâches à haut volume et faible complexité
    - La vitesse est critique
    - Contraintes budgétaires

**Astuce :** Commencez avec gpt-4o pour le développement, puis évaluez si des models moins chers fonctionnent pour votre cas d'usage.

---

## Gestion des Erreurs

```javascript
try {
    await basicCompletion();
} catch (error) {
    console.error("Error:", error.message);
    if (error.message.includes('API key')) {
        console.error("\nMake sure to set your OPENAI_API_KEY in a .env file");
    }
}
```

**Erreurs courantes :**
- `401 Unauthorized` - Clé API invalide ou manquante
- `429 Too Many Requests` - Limite de rate dépassée
- `500 Internal Server Error` - Problème de service OpenAI
- `Context length exceeded` - Trop de tokens dans la conversation

**Bonnes pratiques :**
- Toujours utiliser try-catch avec les appels async
- Vérifier les types d'erreurs et fournir des messages utiles
- Implémenter une logique de retry pour les erreurs transitoires
- Surveiller l'usage des tokens pour éviter les erreurs de limite

---

## Points Clés

1. **Nature Stateless :** Les models ne se souviennent pas. Vous envoyez le contexte complet à chaque fois.
2. **Rôles des Messages :** `system` (comportement), `user` (entrée), `assistant` (réponse IA)
3. **Temperature :** Contrôle la créativité (0 = focalisé, 2 = créatif)
4. **Streaming :** Meilleure UX pour les applications en temps réel
5. **Gestion des Tokens :** Surveillez l'usage pour les coûts et les limites
6. **Sélection de Model :** Choisissez en fonction de la complexité de la tâche et du budget
