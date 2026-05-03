## Concept : Gestion d'erreurs complète pour les agents

Les agents échouent de plus de façons que les apps régulières car ils orchestrent **plusieurs étapes non fiables** :

- **Appels LLM** (timeouts, contraintes de ressources, sorties malformées, exceptions runtime)
- **Exécution d'outils** (échecs réseau, inputs invalides, services indisponibles)
- **Logique de workflow** (policy guards, complétion partielle, chaînes de dépendances entre outils)

Cet exemple utilise trois idées pour rendre les échecs sûrs et compréhensibles :

### 1) Taxonomie d'erreurs standardisée

Utiliser un petit ensemble de classes d'erreurs avec des **codes stables** et des champs cohérents :

- **`ValidationError`** : l'input utilisateur est manquant/invalid (fail fast ; généralement pas retryable)
- **`LLMCallError`** : l'appel LLM provider/model a échoué ou retourné une sortie inutilisable (souvent retryable)
- **`ToolExecutionError`** : un outil a échoué (parfois retryable, parfois non)
- **`AgentWorkflowError`** : échec au niveau **orchestration** — l'exécution multi-étapes ne peut pas se compléter comme conçu.

En production, on pourrait splitter **`AgentWorkflowError`** en types plus fins (policy, chaîne workflow, panne système complète). Cette leçon garde **une seule classe** avec un champ **`step`** et un court commentaire source pour que la **même forme** puisse représenter ces idées dans une petite démo.

Exemples dans ce repo :

- **`policy_guard`** — après validation, un guard bloque la requête (démo : l'utilisateur mentionne **`u_demo_workflow`**).
- **`resolve_user_profile`** — en mode dégradé, **les retries primary sont épuisées**, **le fallback est essayé**, et **le fallback échoue aussi** ; l'erreur surfaçée est au niveau workflow avec l'erreur outil interne comme **`cause`**.

Chaque erreur inclut :

- **`code`** : identifiant stable, lisible par machine (bon pour les metrics et alerting)
- **`userMessage`** : message sûr et non technique affiché aux utilisateurs
- **`retryable`** : si le retry automatique est approprié
- **`details`** : données structurées pour les logs (nom d'outil, nom de step, ids, etc.)
- **`cause`** : erreur originale (pour préserver les chaînes de root cause)

**`AgentWorkflowError`** transporte additionally **`step`**. Ses `details` fusionnent `step` avec toute métadonnée extra qu'on passe.

### 2) Stratégies de Classification et de Récupération

**Normaliser, puis classifier.** **`normalizeUnknownError`** transforme des valeurs thrown arbitraires en **`AppError`**. **`classifyError`** ajoute **`retryable`** et **`type`** (`error.code`) pour que les retries, logs et messages utilisateur partagent un seul pipeline au lieu de répéter des arbres `instanceof`.

Stratégies de récupération (escalier typique), comme montré avec un vrai LLM local via `node-llama-cpp` :

- **Timeout** : borner combien de temps n'importe quelle étape peut staller
- **Retry** : uniquement quand **`classifyError`** dit **`retryable`** (avec backoff et jitter)
- **Fallback** : si l'outil primary échoue de manière **transitoire**, exécuter une alternative plus sûre qui retourne une réponse dégradée mais utile
- **Mode dégradé** : si le chemin LLM échoue, déléguer à **`runDegradedProfileResolution`** — extraction deterministe d'un id **`u_<digits>`** et le même model d'outil **primary → fallback**, sans embarquer cette logique inline dans un énorme `catch`
- **Échec gracieux** : si la récupération n'est pas possible, retourner **`formatUserFacingError`** plus un correlation id

Quand **les deux** primary (après retries) et fallback échouent en mode dégradé, l'exemple promeut ce résultat en **`AgentWorkflowError`** : l'utilisateur voit quand même un message clair, tandis que **`cause`** conserve l'échec outil sous-jacent pour le debugging.

### 3) Séparer les Messages Utilisateur de l'Information de Debugging

Les utilisateurs devraient voir :

- des prochaines étapes claires (réessayer, reformuler, raccourcir l'input)
- pas de stack traces ou d'internals provider
- un **reference id** qu'ils peuvent partager avec le support

Les développeurs/opérateurs devraient voir :

- le `code` d'erreur stable
- `details` structurés
- correlation id
- **`cause`** originale

Pour **`AgentWorkflowError`**, l'exemple affiche aussi un **banner console** (step, code, correlation id, messages, details, résumé cause) pour que les démos live et le debugging local restent lisibles à côté d'une ligne de log compacte `[agent_error]`.

### Démos Deterministes et `SIMULATION`

Pour garder les runs pédagogiques prévisibles, les user ids **`u_999`** et **`u_777`** sont pilotés par une petite map **`SIMULATION`** (voir `error-handling.js`) :

- **`u_demo_workflow`** dans le texte déclenche **`policy_guard`**.
- **`SKIP_LLM_DEGRADED u_777`** skip le LLM, entre en mode dégradé, et utilise **`u_777`** pour que primary et fallback échouent tous les deux de manière reproductible.

### Pourquoi ce pattern scale

- **Cohérence** : chaque échec est façonné de la même manière ; les erreurs inconnues sont normalisées avant traitement
- **Observabilité** : les metrics/alertes groupent par `code` et par `step` de workflow
- **Sécurité** : les détails sensibles/spécifiques au provider restent hors des messages utilisateur
- **Résilience** : les problèmes transitoires se récupèrent automatiquement ; les échecs hard dégradent ou surfacent une seule erreur au niveau workflow avec **`cause`** préservée
