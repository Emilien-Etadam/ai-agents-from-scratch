# Explication du Code : error-handling.js

Ce fichier démontre la **gestion d'erreurs complète pour les programmes de type agent** : une taxonomie d'erreurs typées, des timeouts et retries avec backoff, des échecs d'outils simulés, un **mode dégradé** quand le chemin LLM échoue, et **`AgentWorkflowError`** quand l'orchestration casse. Il s'exécute localement avec **`node-llama-cpp`** et un model GGUF (même stack que les autres exemples d'agents).

**Run :** `node examples/11_error-handling/error-handling.js`

---

## Décomposition du Code étape par étape

### 1. Imports

```javascript
import crypto from "node:crypto";
import { defineChatSessionFunction, getLlama, LlamaChatSession } from "node-llama-cpp";
import { fileURLToPath } from "url";
import path from "path";
```

**Ce qui se passe :**
- **`crypto`** - UUIDs pour les correlation ids (`randomUUID`) et jitter pour les délais de retry (`randomInt`).
- **`node-llama-cpp`** - Charger le model, chat session, et **`defineChatSessionFunction`** pour les outils.
- **`url` / `path`** - Résoudre **`__dirname`** dans un ES module et joindre les chemins vers le fichier **`.gguf`**.

---

### 2. Taxonomie d'erreurs

```javascript
class AppError extends Error { /* code, userMessage, retryable, details, cause */ }
class ValidationError extends AppError { /* VALIDATION_ERROR */ }
class LLMCallError extends AppError { /* LLM_CALL_FAILED; optional model */ }
class ToolExecutionError extends AppError { /* TOOL_EXECUTION_FAILED; toolName */ }
class AgentWorkflowError extends AppError { /* AGENT_WORKFLOW_FAILED; step */ }
```

**Objectif :**
- **`AppError`** - Une forme unique pour les logs, retries et texte utilisateur : **`code`** stable, **`userMessage`** sûr, **`details`** structurés, **`cause`** optionnel.
- Les sous-classes définissent des valeurs par défaut sensées (ex. validation n'est pas retryable ; les outils ne le sont souvent pas, sauf si on passe **`retryable: true`**).
- **`AgentWorkflowError`** ajoute **`step`** (ex. `policy_guard`, `resolve_user_profile`) pour les échecs au niveau orchestration. Le commentaire dans le source explique comment cette classe peut remplacer les erreurs policy / workflow / system dans une démo.

---

### 3. `sleep`

Simple délai basé sur `Promise`. Utilisé par **`withRetries`** entre les tentatives et dans les outils "réseau" simulés.

---

### 4. `withTimeout`

**`Promise.race`** entre le travail réel et un timer. Au timeout, reject avec un **`AppError`** avec code **`TIMEOUT`**, **`retryable: true`**, et **`details: { label, ms }`**. Le timer est cleared dans **`finally`**.

**Pourquoi c'est important :** Chaque appel LLM ou outil qui pourrait bloquer doit être borné pour que l'agent puisse se récupérer au lieu de staller indéfiniment.

---

### 5. `normalizeUnknownError`

Si la valeur thrown est déjà un **`AppError`**, la retourner. Sinon wrapper comme **`UNKNOWN_ERROR`** (non-retryable), cacher le nom/message original dans **`details`**, définir **`cause`** sur l'erreur originale.

**Pourquoi c'est important :** Les blocs catch reçoivent souvent des **`Error`**, strings ou types spécifiques à une librairie ; la normalisation rend **`retryOn`** et **`formatUserFacingError`** prévisibles.

---

### 6. `classifyError`

Appelle **`normalizeUnknownError`**, puis retourne **`{ error, retryable, type }`** où **`type`** est **`error.code`**.

**Pourquoi c'est important :** Un seul endroit pour décider "retry ?" au lieu de répéter des checks **`instanceof`** à travers **`promptLLM`**, **`runAgent`** et les prédicats **`withRetries`**.

---

### 7. `isRetryable`

Retourne **`classifyError(err).retryable`**. Utilisé comme **`retryOn`** par défaut pour **`withRetries`**.

---

### 8. `jitteredBackoffDelay`

Délai exponentiel capped à **`maxDelayMs`**, plus **random jitter** via **`crypto.randomInt`**, pour que plusieurs clients ne retryent pas en lockstep.

---

### 9. `withRetries`

Exécute **`fn`** jusqu'à **`retries + 1`** fois. Après un échec, s'il reste des tentatives et que **`retryOn(err)`** est true, attend (**`sleep`** + **`jitteredBackoffDelay`**), loggue **`[retry]`** et retry. Sinon lance **`lastErr`**.

---

### 10. `formatUserFacingError`

Construit la string affichée à l'"utilisateur" dans la démo : **`userMessage`** plus **`(Reference: <correlationId>)`**, ou un fallback générique si l'erreur n'était pas un **`AppError`**.

---

### 11. `printAgentWorkflowErrorBanner`

Quand un **`AgentWorkflowError`** est catché, affiche un bloc bordé sur **stderr** : step, code, correlation id, messages, **`details`** et un résumé court de **`cause`**. Complète le log JSON **`[agent_error]`** en une ligne.

---

### 12. `SIMULATION` et outils simulés

```javascript
const SIMULATION = {
  forceNotFound: new Set(["u_999"]),
  forcePrimaryAndFallbackFail: new Set(["u_777"]),
};
```

**`fetchUserFromPrimary`** - Simule la latence ; **`u_999`** > non-retryable "not found" ; **`u_777`** > toujours retryable primary failure (démo) ; sinon ~20% d'échec aléatoire transitoire ; le succès retourne un profile avec **`source: "primary"`**.

**`fetchUserFromFallback`** - Profile de moindre fidélité ; pour **`u_777`** lance une erreur pour que la chaîne **primary > fallback** puisse surfer **`AgentWorkflowError`** de manière deterministe.

---

### 13. Initialiser le model et la session

Même pattern que **`simple-agent.js`** : **`getLlama`**, **`loadModel`** (chemin vers **`models/Qwen3-1.7B-Q8_0.gguf`**), **`createContext`**, **`LlamaChatSession`** avec un **system prompt** qui dit au model qu'il peut fetcher des utilisateurs via des outils.

---

### 14. Enregistrer les outils

Deux wrappers **`defineChatSessionFunction`** appellent **`fetchUserFromPrimary`** et **`fetchUserFromFallback`** avec un **`userId`** en JSON Schema. **`functions`** est passé dans **`session.prompt`** pour que le LLM puisse invoquer les outils par nom.

---

### 15. `promptLLM`

Wrappe **`session.prompt`** avec **`withTimeout`**, **`withRetries`** et des erreurs sensibles à la corrélation :

- Réponse vide après trim > **`LLMCallError`** (retryable).
- **`catch`** : **`classifyError`** ; relance **`ToolExecutionError`** / **`LLMCallError`** inchangés ; tout le reste devient **`LLMCallError`** avec **`retryable`** uniquement si l'échec normalisé était **`TIMEOUT`** (**`cause`** préservée).
- **`retryOn`** : **`(err) => classifyError(err).retryable`**.

---

### 16. `runDegradedProfileResolution`

S'exécute **sans** le LLM après que le chemin LLM a échoué avec **`LLMCallError`** :

1. Extraire **`u_<digits>`** du match **`SKIP_LLM_DEGRADED`** ou du texte libre ; sinon **`ValidationError`**.
2. **`withRetries`** + **`withTimeout`** sur le primary ; **`retryOn`** uniquement pour les **`ToolExecutionError`** **retryable**.
3. Si le primary échoue encore avec une erreur outil retryable > essayer **`fetchUserFromFallback`**. Si le fallback lance > **`AgentWorkflowError`** (**`resolve_user_profile`**, **`cause`** = erreur fallback).
4. Retourne une réponse courte en puces préfixée par **"Model unavailable; answered via deterministic fallback."**

---

### 17. `runAgent`

**Flux :**

1. **`correlationId = crypto.randomUUID()`**.
2. Input vide > **`ValidationError`**.
3. Texte contient **`u_demo_workflow`** > **`AgentWorkflowError`** (**`policy_guard`**) - guard de démo après validation.
4. **`SKIP_LLM_DEGRADED u_<digits>`** > force **`LLMCallError`** sans appeler le model (démo dégradée deterministe).
5. Sinon **`promptLLM`**. Succès > **`{ ok: true, output }`**.
6. **`catch`** uniquement **`LLMCallError`** > logguer **`[degraded_mode]`**, appeler **`runDegradedProfileResolution`**, retourner **`ok: true`** avec sortie dégradée.
7. Toute autre erreur se propage au **`catch`** extérieur : **`classifyError`**, **`printAgentWorkflowErrorBanner`** optionnel pour **`AgentWorkflowError`**, **`console.error("[agent_error]", …)`**, retourner **`{ ok: false, output: formatUserFacingError(...) }`**.

---

### 18. Boucle de démo et nettoyage

**`inputs`** exécute un ensemble fixe de strings (happy path, **`u_999`**, **`u_demo_workflow`**, **`SKIP_LLM_DEGRADED u_777`**, vide). Chaque itération affiche **`USER:`**, **`runAgent`**, puis le texte de l'assistant ou le texte d'erreur.

**Dispose :** **`session`**, **`context`**, **`model`**, **`llama`** - important pour les bindings locaux/natifs.

---

## Concepts Clés Démontrés

### 1. Erreurs typées + codes stables

Les dashboards et alertes peuvent grouper par **`code`**. Les utilisateurs ne voient jamais **`details`** ou des stacks — uniquement **`userMessage`** et un **reference id**.

### 2. Classifier, puis retry

**`normalizeUnknownError` > `classifyError` > `retryable`** garde **`withRetries`** et **`promptLLM`** alignés sur ce qui compte comme transitoire.

### 3. Timeout > retry > fallback > mode dégradé

**`withTimeout`** borne le temps d'attente. **`withRetries`** gère les LLMs ou outils instables. **`runDegradedProfileResolution`** est le chemin **deterministe** quand le chemin LLM est inutilisable mais qu'on peut toujours compléter le travail avec des outils.

### 4. `AgentWorkflowError` vs erreurs outil

Un **outil** lance **`ToolExecutionError`**. Quand la **policy** bloque ou **primary + fallback** échouent tous les deux dans le flow dégradé orchestré, l'erreur surfaçée est **`AgentWorkflowError`** avec **`cause`** pointant vers l'échec interne.

---

## Sortie Attendue (représentative)

Quand vous exécutez le script, vous verrez des lignes de séparation, des lignes **`USER:`**, et soit **`ASSISTANT:`** soit **`ASSISTANT (error):`**. Pour **`u_demo_workflow`** et **`SKIP_LLM_DEGRADED u_777`** (quand le fallback échoue), **stderr** affiche le banner **AGENT WORKFLOW FAILED** plus le JSON **`[agent_error]`**. Les lignes **`[retry]`** et **`[degraded_mode]`** apparaissent quand les retries ou le chemin dégradé s'activent.

Le wording exact varie légèrement (ex. la sortie LLM sur le premier prompt dépend du model).

---

## Bonnes Pratiques

1. **Borner le temps** sur les appels LLM et outil (**`withTimeout`**).
2. **Retry uniquement les échecs transitoires** ; utiliser **`classifyError`** (ou équivalent) pour que les erreurs de validation ne soient jamais retryées aveuglément.
3. **Jitter** le backoff pour éviter les retries synchronisés.
4. **Correlation ids** sur chaque erreur visible par l'utilisateur et sur les logs structurés.
5. **Séparer** les logs opérateur (**`[agent_error]`**, banners) de ce qu'on montre aux utilisateurs finaux (**`formatUserFacingError`**).
6. **Dispose** les ressources natives/model quand le script se termine.

---

## Pourquoi Cela Compte pour les Agents IA

Les agents empilent **LLM + outils + orchestration**. Les échecs peuvent provenir de n'importe quelle couche ; sans taxonomie et classification, on soit **retry tout** (gaspilleur) soit **retry rien** (fragile). Cet exemple montre un chemin minimal mais complet depuis les **erreurs single-call** jusqu'au **`AgentWorkflowError`** au niveau **workflow**, avec un chemin d'upgrade clair vers les circuit breakers, la télémétrie réelle et les types de policy de qualité production.
