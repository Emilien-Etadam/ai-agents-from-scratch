# Concept : Interaction de Base avec un LLM

## Vue d'ensemble

Cet exemple présente les concepts fondamentaux du travail avec un Large Language Model (LLM) exécuté localement sur votre machine. Il démontre l'interaction la plus simple possible : charger un model et lui poser une question.

## Qu'est-ce qu'un LLM Local ?

Un **LLM Local** est un model de langage IA qui s'exécute entièrement sur votre propre ordinateur, sans nécessiter de connexion internet ou d'appels API externes. Principaux avantages :

- **Confidentialité** : Vos données ne quittent jamais votre machine
- **Coût** : Pas de frais API par token
- **Contrôle** : Contrôle total sur le choix du model et les paramètres
- **Hors ligne** : Fonctionne sans connexion internet

## Composants Principaux

### 1. Fichiers de Model (Format GGUF)

```
┌─────────────────────────────┐
│   Qwen3-1.7B-Q8_0.gguf     │
│   (Fichier de Poids du Model)│
│                             │
│  • Stocke les patterns appris │
│  • Quantifié pour l'efficacité│
│  • Chargé en RAM/VRAM       │
└─────────────────────────────┘
```

- **GGUF** : Format de fichier optimisé pour llama.cpp
- **Quantization** : Réduit la taille du model (par ex. 8-bit au lieu de 16-bit)
- **Compromis** : Taille réduite et vitesse accrue vs. perte légère de qualité

### 2. Le Pipeline d'Inférence

```
Entrée Utilisateur → Model → Génération → Réponse
    ↓                ↓          ↓            ↓
 "Bonjour"        Contexte   Échantillonnage  "Salut !"
```

**Diagramme de Flux :**
```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Prompt  │ --> │ Contexte │ --> │  Model   │ --> │ Réponse  │
│          │     │ (Mémoire)│     │(Poids)   │     │  (Texte) │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

### 3. Fenêtre de Contexte

Le **contexte** est la mémoire de travail du model :

```
┌─────────────────────────────────────────┐
│           Fenêtre de Contexte            │
│  ┌─────────────────────────────────┐   │
│  │ System Prompt (le cas échéant)  │   │
│  ├─────────────────────────────────┤   │
│  │ Utilisateur : "do you know node-llama?" │
│  ├─────────────────────────────────┤   │
│  │ IA : "Yes, I'm familiar..."     │   │
│  ├─────────────────────────────────┤   │
│  │ (Espace pour plus de conversation) │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

- Taille limitée (par ex. 2048, 4096 ou 8192 tokens)
- Lorsque c'est plein, les anciens messages doivent être supprimés
- Tous les messages précédents influencent la prochaine réponse

## Comment les LLMs Génèrent des Réponses

### Génération Token par Token

Les LLMs ne génèrent pas des phrases entières d'un coup. Ils prévoient un **token** (morceau de mot) à la fois :

```
Prompt : "Qu'est-ce que l'IA ?"

Processus de génération :
"Qu'est-ce que l'IA ?" → [Model] → "L'IA"
"Qu'est-ce que l'IA ? L'IA" → [Model] → "est"
"Qu'est-ce que l'IA ? L'IA est" → [Model] → "un"
"Qu'est-ce que l'IA ? L'IA est un" → [Model] → "domaine"
... continue jusqu'à la condition d'arrêt
```

**Visualisation :**
```
Prompt en entrée
     ↓
┌────────────┐
│   Model    │ → Token 1 : "L'IA"
│  Traite    │ → Token 2 : "est"
│ & Prédit   │ → Token 3 : "un"
└────────────┘ → Token 4 : "domaine"
                → ...
```

## Concepts Clés pour les AI Agents

### 1. Traitement Stateless
- Chaque prompt est indépendant sauf si vous maintenez un contexte
- Le model n'a pas de mémoire entre les différentes exécutions de script
- Pour construire un "agent", vous devez :
  - Garder le contexte vivant entre les prompts
  - Maintenez l'historique de la conversation
  - Ajouter des outils/fonctions (couvert dans les exemples suivants)

### 2. Bases du Prompt Engineering
La façon dont vous formulez les questions affecte la réponse :

```
❌ Mauvais : "node-llama-cpp"
✅ Mieux : "do you know node-llama-cpp"
✅ Meilleur : "Explain what node-llama-cpp is and how it works"
```

### 3. Gestion des Ressources
Les LLMs consomment des ressources significatives :

```
Chargement du Model
     ↓
┌─────────────────┐
│  Usage RAM/VRAM │  ← Les models ont besoin de gigaoctets
│  Temps CPU/GPU  │  ← L'inférence prend du temps
│  Fuites mémoire ?│  ← Nettoyage obligatoire
└─────────────────┘
     ↓
Disposal Approprié
```

## Pourquoi C'est Important pour les Agents

Cet exemple basique établit les fondations des AI agents :

1. **Les agents ont besoin de LLMs pour "penser"** : Le model traite l'information et génère des réponses
2. **Les agents ont besoin de contexte** : Pour maintenir l'état à travers les interactions
3. **Les agents ont besoin de structure** : Les exemples suivants ajoutent des outils, de la mémoire et des boucles de raisonnement

## Prochaines Étapes

Après avoir compris le prompting de base, explorez :
- **System prompts** : Donner au model un rôle ou un comportement spécifique
- **Function calling** : Permettre au model d'utiliser des outils
- **Mémoire** : Persister l'information à travers les sessions
- **Patterns de raisonnement** : Comme ReAct (Reasoning + Acting)

## Diagramme : Architecture Complète

```
┌──────────────────────────────────────────────────┐
│            Votre Application                      │
│  ┌────────────────────────────────────────────┐ │
│  │         Bibliothèque node-llama-cpp        │ │
│  │  ┌──────────────────────────────────────┐  │ │
│  │  │      llama.cpp (Runtime C++)         │  │ │
│  │  │  ┌────────────────────────────────┐  │  │ │
│  │  │  │   Fichier de Model (GGUF)      │  │  │ │
│  │  │  │   • Qwen3-1.7B-Q8_0.gguf       │  │  │ │
│  │  │  └────────────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
           ↕
    ┌──────────────┐
    │  CPU / GPU   │
    └──────────────┘
```

Cette architecture en couches vous permet de construire des AI agents sophistiqués à partir d'interactions LLM de base.
