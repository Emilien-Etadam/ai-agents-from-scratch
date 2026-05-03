# Prompt Engineering

Le prompt engineering offre la méthode la plus rapide et la plus simple pour façonner le comportement d'un agent — en définissant
sa personnalité, sa fonction et ses choix (comme savoir quand il doit utiliser des outils). Les agents fonctionnent avec deux catégories de prompts :
les prompts système et les prompts utilisateur.

Les prompts utilisateur correspondent aux messages que les personnes saisissent pendant la conversation. Ils varient à chaque interaction
et restent hors du contrôle du développeur.

Les prompts système contiennent des instructions établies par les développeurs et qui restent constantes tout au long du dialogue.
Ils définissent le ton de l'agent, ses capacités, ses limites et les règles d'utilisation des outils.

Consultez les system prompts d'Anthropic

https://docs.claude.com/en/release-notes/system-prompts#september-29-2025

## Conception de Prompts

Lorsque vous créez des prompts pour des agents, vous devez atteindre deux objectifs :

1. Faire en sorte que l'agent résolve bien les problèmes

- L'aider à accomplir correctement des tâches complexes
- Permettre une pensée claire et logique
- Réduire les erreurs

2. Garder la personnalité de l'agent cohérente

- Définir qui est l'agent et comment il parle
- Correspondre à la voix de votre marque
- Répondre avec l'émotion appropriée pour chaque situation

Les deux objectifs sont tout aussi importants. Une réponse précise mais formulée avec rudesse nuit à l'expérience utilisateur. Une réponse sympathique
qui n'aide pas réellement est inutile.

## Stratégies de Prompting

### Rôle de l'Agent

Donner un rôle spécifique au LLM améliore ses réponses — il adopte naturellement le vocabulaire et l'expertise de ce rôle.
Exemples :

"Vous êtes un pédiatre" → Utilise des termes médicaux, parle du développement de l'enfant, recommande des traitements adaptés à l'âge
"Vous êtes un chef" → Explique les techniques culinaires, suggère des substitutions d'ingrédients, aborde les profils de saveurs
"Vous êtes un professeur de maths du lycée" → Décompose les problèmes étape par étape, utilise un langage simple, fournit des exemples d'entraînement
"Vous êtes un fondateur de startup" → Se concentre sur la croissance, utilise des indicateurs business, réfléchit en termes de scalabilité

Rendez les rôles précis :
Au lieu de : "Vous êtes un rédacteur"
Mieux : "Vous êtes un blogueur tech qui simplifie les concepts complexes de l'IA pour les débutants"

Les rôles fonctionnent mieux pour les questions spécialisées et doivent être définis dans les system prompts.

### Soyez Précis, Pas Vague

Les LLMs interprètent les instructions littéralement. Des prompts vagues produisent des résultats aléatoires. Des prompts précis produisent des sorties cohérentes.
Exemples vague vs précis :

❌ Vague : "Écris quelque chose sur les chiens"
✅ Précis : "Écris un guide en 3 paragraphes pour entraîner un chiot à s'asseoir"

❌ Vague : "Améliore ça"
✅ Précis : "Corrige les erreurs grammaticales et réduit à moins de 100 mots"

❌ Vague : "Sois professionnel"
✅ Précis : "Utilise un langage formel, évite les contractions, adresse le lecteur avec "vous""

❌ Vague : "Analyse ces données"
✅ Précis : "Identifie les 3 tendances principales et explique ce qui a provoqué chacune"

Pourquoi c'est important : Le LLM a des milliers de façons d'interpréter des instructions vagues. Il va deviner ce que vous voulez — et souvent
se tromper. Des instructions claires éliminent les devinettes et vous donnent le contrôle sur la sortie.

Règle empirique : Si un assistant humain devrait poser des questions de clarification, votre prompt est trop vague.

### Structurer les Entrées du LLM avec JSON
Utiliser JSON pour structurer votre entrée aide les LLMs à comprendre les tâches plus clairement et facilite l'intégration. Au lieu
d'envoyer un bloc de texte, décomposez votre requête en parties étiquetées comme task, input, constraints et output_format.

Avantages
- Clarté : Les clés JSON montrent au model ce que chaque partie signifie.
- Fiabilité : Plus facile de parser et valider les réponses.
- Cohérence : Réduit les réponses aléatoires ou narratives.
- Intégration : Fonctionne bien avec les APIs et les schemas.

Bonnes Pratiques
- Gardez-le simple et peu profond — évitez l'imbrication profonde.
- Utilisez des clés descriptives ("task", "context", "constraints").
- Indiquez au model le format de sortie exact (par ex. "Répondez uniquement avec du JSON valide").
- Définissez éventuellement un JSON Schema pour imposer la structure.
- Validez toujours la réponse dans votre code.

Exemple
````
{
  "task": "summarize",
  "input_text": " - Texte de l'article ici. - ",
  "constraints": {
    "max_words": 100,
    "audience": "non-technical"
  },
  "output_format": {
    "type": "JSON",
    "schema": {
      "summary": "string",
      "key_points": ["string"]
    }
  }
}
````

Ce format structuré aide le model à séparer ce qu'il doit faire, les données à utiliser et comment répondre, ce qui donne des sorties
plus cohérentes et lisibles par machine.

### Few-Shot Prompting

Le few-shot prompting consiste à donner au LLM quelques exemples de ce que vous voulez avant de lui demander de faire une nouvelle tâche.
C'est comme montrer à un étudiant deux ou trois problèmes résolus pour qu'il comprenne le pattern.

Exemple
```
Exemple 1 :
Feedback : "La chambre était propre et calme."
Catégorie : Positive

Exemple 2 :
Feedback : "Le personnel était grossier et peu serviable."
Catégorie : Négative

Exemple 3 :
Feedback : "Le petit-déjeuner était correct, mais le café était froid."
Catégorie : Neutre

Maintenant, catégorisez ceci :
Feedback : "La vue depuis le balcon était incroyable !"
Catégorie :
```

Le model apprend à partir des exemples et continue dans le même style — ici, il répondrait :
"Positive"

Les few-shot prompts sont utiles lorsque vous voulez un ton, un format ou une logique cohérente sans retraîner le model.

### Chain of Thought

Le Chain of Thought consiste à demander au LLM de réfléchir étape par étape au lieu de sauter directement à la réponse.
Cela aide le model à mieux raisonner, surtout pour la logique, les maths ou les problèmes en plusieurs étapes.

Exemple

Question : Si 3 pommes coûtent $6, combien coûtent 5 pommes ?
Réfléchissons étape par étape.

Raisonnement du model :
3 pommes → $6 → chaque pomme coûte $2.
5 pommes × $2 = $10.

Réponse : $10

En encourageant une réflexion étape par étape, vous aidez le model à faire moins d'erreurs et à expliquer son raisonnement clairement.
