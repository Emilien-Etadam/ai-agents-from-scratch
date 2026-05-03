# Concept : Function Calling et Utilisation d'Outils

## Vue d'Ensemble

Le function calling transforme les LLMs de générateurs de texte en agents capables d'entreprendre des actions et d'interagir avec le monde.

## Qu'est-ce qui fait un Agent ?

```
Générateur de Texte           Agent
──────────────               ──────
LLM → Texte seul             LLM + Outils → Peut agir
```

Le **function calling** permet au LLM d'invoquer des fonctions prédéfinies pour accéder à des données ou effectuer des actions qu'il ne peut pas faire seul.

## l'Idée Fondamentale

```
Utilisateur : "Quelle heure est-il ?"
       ↓
LLM réfléchit : "J'ai besoin de l'heure actuelle"
       ↓
LLM appelle : getCurrentTime()
       ↓
Outil retourne : "1:46:36 PM"
       ↓
LLM répond : "Il est 13:46"
```

C'est l'agency — la capacité d'AGIR, pas juste de DIRE.

## Comment Ça Fonctionne

### 1. Définition de Fonction
```javascript
getCurrentTime = {
  description: "Get the current time",
  handler: () => new Date().toLocaleTimeString()
}
```

### 2. Le LLM Voit les Outils Disponibles
```
Fonctions disponibles :
- getCurrentTime: "Get the current time"
- getWeather: "Get weather for a city"
- calculate: "Perform math"
```

### 3. Le LLM Décident Quand Utiliser
```
"Quelle heure ?" → getCurrentTime() ✓
"Combien font 5+5 ?" → calculate() ✓
"Raconte une blague" → Pas d'outil nécessaire
```

## Applications Réelles

**Assistant Personnel** : Calendrier, email, rappels
**Agent de Recherche** : Recherche web, lecture de documents
**Assistant Codeur** : Opérations fichiers, exécution de code
**Analyste de Données** : Requêtes base de données, calculs

## Point Clé

Le function calling est LA fonctionnalité qui permet les agents IA. Sans ça, les LLMs peuvent seulement parler. Avec ça, ils peuvent agir.

C'est la fondation de tous les systèmes d'agents modernes.
