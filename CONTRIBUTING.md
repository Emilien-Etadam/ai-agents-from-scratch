# Lignes directrices pour contribuer

Merci d'envisager de contribuer à AI Agents from Scratch !

## Philosophie du Projet

Ce dépôt enseigne les fondamentaux des AI agents en construisant à partir de zéro. Chaque contribution doit soutenir cette mission pédagogique.

**Principes fondamentaux :**
- **Clarté avant ingéniosité** - Le code doit être facile à comprendre
- **Les fondamentaux d'abord** - Pas de boîtes noires ni de magie
- **Apprentissage progressif** - Chaque exemple s'appuie sur le précédent
- **Local d'abord** - Aucune dépendance API

## Types de Contributions

### Signalement de bugs
Vous avez trouvé quelque chose de cassé ? Ouvrez une issue avec :
- L'exemple concerné (`intro/`, `react-agent/`, etc.)
- Ce que vous attendiez vs ce qui s'est passé
- Votre environnement (version Node, OS, model utilisé)
- Étapes pour reproduire le problème

### Améliorations de la documentation
- Fautes de frappe et corrections grammaticales
- Explications plus claires
- Meilleurs commentaires dans le code
- Exemples supplémentaires dans la documentation
- Diagrammes et visualisations

### Nouveaux exemples
Vous souhaitez ajouter un nouveau pattern d'agent ? Parfait ! Veuillez :
1. **Ouvrez une issue d'abord** - discutons pour voir si ça s'insère bien
2. Suivez la structure existante :
   - `pattern-name/pattern-name.js` - Code fonctionnel
   - `pattern-name/CODE.md` - Explication détaillée du code
   - `pattern-name/CONCEPT.md` - Pourquoi c'est important, cas d'usage
3. Gardez-le simple et bien commenté
4. Testez soigneusement avec au moins un model

### Améliorations de code
- Optimisations de performance (avec benchmarks)
- Meilleure gestion des erreurs
- Noms de variables plus clairs
- Sorties console plus utiles

## Ce Que Nous Ne Cherchons Pas

- Intégrations de framework (LangChain, etc.) - ce dépôt enseigne ce qu'ils font
- Exemples d'API cloud - gardez-le local
- Fonctionnalités de production (monitoring, scaling) - c'est éducatif
- Abstractions complexes - gardez-le accessible aux débutants

## Processus de Contribution

1. **Fork** le dépôt
2. **Créez une branche** : `git checkout -b fix/description-du-problème`
3. **Apportez vos modifications** et testez soigneusement
4. **Commit** avec des messages clairs : `git commit -m "Fix: clarifier l'explication de la boucle ReAct"`
5. **Push** : `git push origin fix/description-du-problème`
6. **Ouvrez un Pull Request** avec :
   - Titre clair
   - Description des changements et de leur raison
   - L'issue adressée (le cas échéant)

## Standards de Code

- Utilisez des noms de variables clairs et descriptifs
- Ajoutez des commentaires expliquant *pourquoi*, pas seulement *quoi*
- Respectez le style de code existant (pas de linter, suivez simplement les patterns)
- Gardez les exemples autonomes (un fichier quand c'est possible)
- Testez avec les models Qwen ou Llama avant de soumettre

## Standards de Documentation

- Utilisez un langage clair et simple
- Expliquez les concepts avant le code
- Incluez des diagrammes quand utile (l'art ASCII est acceptable !)
- Fournissez des cas d'usage réels
- Liez vers les exemples connexes

## Structure d'un Exemple
```
new-pattern/
├── new-pattern.js # Le code fonctionnel
├── CODE.md # Explication ligne par ligne
└── CONCEPT.md # Explication de haut niveau
```

**CODE.md doit inclure :**
- Prérequis
- Analyse détaillée du code étape par étape
- Comment l'exécuter
- Sortie attendue

**CONCEPT.md doit inclure :**
- Quel problème il résout
- Pourquoi ce pattern est important
- Applications réelles
- Diagrammes simples

## Obtenir de l'Aide

- Vous ne savez pas si votre idée correspond ? **Ouvrez une issue pour discuter**
- Bloqué sur l'implémentation ? **Posez votre question dans l'issue**
- Vous voulez travailler à deux sur quelque chose ? **Contactez-nous !**

## Licence

En contribuant, vous acceptez que vos contributions seront sous la même licence que le projet (MIT).

## Reconnaissance

Tous les contributeurs seront reconnus dans le README. Merci d'aider les autres à apprendre !

---

**Des questions ?** Ouvrez une issue ou contactez-nous. Heureux d'aider guider votre contribution ! 