# Explication du Code : `chain-of-thought.js`

Cette walkthrough mappe chaque phase CoT aux fonctions réelles du fichier.

## Run

```bash
node examples/14_chain-of-thought/chain-of-thought.js
```

---

## 1) Setup : model, cas d'input et schemas

En haut du fichier :

- `RETURN_CASE` définit la requête client.
- `RETURN_POLICY` définit les contraintes business hard.
- `factsSchema`, `redFlagsSchema`, `legitimacySchema`, `policySchema`, `decisionSchema` définissent le contrat JSON pour chaque phase.
- `promptJson(schema, userText)` est la utility partagée qui :
  - reset l'historique de chat,
  - impose la grammar de schema,
  - parse et répare le JSON en sécurité.

Cela donne à chaque fonction de phase une forme de sortie stricte.

```js
const RETURN_CASE = {
    request_id: "RET-2026-0414",
    claimed_reason: "Right ear cup has intermittent sound dropouts",
    claim_timing_days_after_delivery: 23,
    order_value_eur: 189.0
    // ...
};

const RETURN_POLICY = {
    return_window_days: 30,
    max_high_value_returns_12m_before_manual_review: 2,
    mandatory_manual_review_amount_eur: 250
};

async function promptJson(schema, userText) {
    session.resetChatHistory();
    const grammar = await llama.createGrammarForJsonSchema(schema);
    const raw = await session.prompt(userText, { grammar, maxTokens: 1400, temperature: 0.2 });
    return JsonParser.parse(raw, { debug, expectObject: true, repairAttempts: true });
}
```

---

## 2) Phase 1 (Facts) : `extractFacts()`

`extractFacts(returnCase)` demande :

- uniquement les faits explicites,
- pas de scoring,
- pas de jugement.

Il retourne :

- `extracted_facts`
- `missing_information`

Cela protège contre le biais précoce avant que le raisonnement de risque ne commence.

```js
async function extractFacts(returnCase) {
    return promptJson(
        factsSchema,
        `Phase 1 of 5: FACTS ONLY.
Extract facts from the return request without evaluation, suspicion, or judgment.
Do not infer intent. Do not score. Just capture what is explicitly known.

Return request JSON:
${JSON.stringify(returnCase, null, 2)}`
    );
}
```

---

## 3) Phase 2 (Red Flags) : `screenRedFlags()`

`screenRedFlags(returnCase, facts)` effectue un screening fraud explicite avec des checkpoints fixes.

Sortie :

- `checkpoints[]` avec `present/not_present/unclear`
- `fraud_score`
- `fraud_rationale`

La partie importante est la couverture checklist, pas juste un score final.

```js
async function screenRedFlags(returnCase, facts) {
    return promptJson(
        redFlagsSchema,
        `Phase 2 of 5: RED FLAG SCREENING.
Evaluate potential fraud indicators one by one.

Use these checkpoints:
1) Frequent recent return behavior
2) High-value return pattern
3) Inconsistent payment/shipping identity
4) Weak or missing defect evidence
5) Timing pattern that looks strategic
6) Account behavior anomaly`
    );
}
```

---

## 4) Phase 3 (Legitimacy) : `assessLegitimacy()`

`assessLegitimacy(returnCase, facts)` construit l'argument côté client :

- indicateurs de défaut plausibles,
- facteurs d'équité/contexte,
- qualité des preuves de soutien.

Sortie :

- `customer_supporting_points[]`
- `legitimacy_score`
- `legitimacy_rationale`

Sans cette phase, la logique de risque tend à dominer chaque cas borderline.

```js
async function assessLegitimacy(returnCase, facts) {
    return promptJson(
        legitimacySchema,
        `Phase 3 of 5: LEGITIMACY VIEW.
Now build the customer-side case.
List reasons why this may be a legitimate return.
Do not reference fraud score. Focus on fairness and plausible product failure.`
    );
}
```

---

## 5) Phase 4 (Policy) : `checkPolicy()`

`checkPolicy(returnCase, policy, redFlags, legitimacy)` applique les règles hard :

- return window
- seuils de valeur
- return-history triggers

Sortie :

- statuts par règle dans `policy_checks[]`
- `policy_outcome` (`approve`, `reject`, `manual_review`)

C'est le portail de gouvernance entre l'analyse et l'action.

```js
async function checkPolicy(returnCase, policy, redFlags, legitimacy) {
    return promptJson(
        policySchema,
        `Phase 4 of 5: POLICY CHECK.
Apply policy strictly. Do not invent rules.

Policy JSON:
${JSON.stringify(policy, null, 2)}

Fraud score: ${redFlags.fraud_score}
Legitimacy score: ${legitimacy.legitimacy_score}`
    );
}
```

---

## 6) Phase 5 (Decision) : `makeDecision()`

`makeDecision(...)` peut décider uniquement après toutes les phases précédentes.

Sortie :

- `final_decision`
- `confidence`
- `decision_reasoning`
- `customer_message`
- `internal_note`

Le prompt référence explicitement la gestion de conflit (par exemple fraud 6/10 vs legitimacy 7/10), donc le résultat doit expliquer comment la policy résout la tension.

```js
async function makeDecision(returnCase, phase1Facts, redFlags, legitimacy, policyResult) {
    return promptJson(
        decisionSchema,
        `Phase 5 of 5: FINAL DECISION.
You can decide only now. Use all prior phases.
Explain trade-offs clearly. If conflict exists (e.g., fraud 6/10 vs legitimacy 7/10),
show how policy resolves it.`
    );
}
```

---

## 7) Flux d'Orchestration : `runChainOfThoughtReturnDecision()`

Le contrôleur principal exécute les phases dans un ordre strict :

1. facts
2. red flags
3. legitimacy
4. policy check
5. final decision

Puis il affiche un rapport compact et écrit une visualisation navigateur via :

- `writeCoTReturnVisualization(...)`

Cela garde le fichier core focalisé sur la logique CoT.

```js
async function runChainOfThoughtReturnDecision(returnCase, policy) {
    const facts = await extractFacts(returnCase);
    const redFlags = await screenRedFlags(returnCase, facts);
    const legitimacy = await assessLegitimacy(returnCase, facts);
    const policyResult = await checkPolicy(returnCase, policy, redFlags, legitimacy);
    const decision = await makeDecision(returnCase, facts, redFlags, legitimacy, policyResult);

    writeCoTReturnVisualization(__dirname, {
        returnCase, policy, facts, redFlags, legitimacy, policyResult, decision
    });
}
```

---

## 8) Adapter l'implémentation par classe de model

Le code actuel utilise `Qwen3-1.7B-Q8_0.gguf`, qui peut tourner aussi bien en model de reasoning qu'en model non-reasoning. Le scaffolding 5-phases est conçu pour fonctionner avec les deux classes — mais la façon de le tuner diffère.

Pour le côté conceptuel de cette distinction, voir la section "CoT with reasoning vs non-reasoning LLMs" dans [CONCEPT.md](CONCEPT.md).

### Ce que le code actuel assume

- Un model hybrid qui peut ou non raisonner internally.
- Des schemas JSON par phase via `promptJson(...)`.
- Une `temperature` basse (0.2) et un budget `maxTokens` généreux par phase.
- Un historique de chat isolé par phase via `session.resetChatHistory()`.

C'est intentionnellement une configuration middle-ground pour que l'exemple fonctionne sans forcer les lecteurs à télécharger un model spécifique.

### Tuning pour les models non-reasoning

Si on swappe pour un model base/chat sans raisonnement interne (Llama-3 chat, Phi, Mistral-instruct, Qwen3 avec `thoughts: "discourage"`) :

```js
const raw = await session.prompt(userText, {
    grammar,
    maxTokens: 1800,
    temperature: 0.1
});
```

- Baisser la `temperature` encore (0.05 - 0.15). Les cas borderline régressent fortement avec du sampling créatif.
- Augmenter `maxTokens` par phase. Le model a souvent besoin de place pour "parler seul" dans le JSON avant de s'engager sur des scores.
- Garder les schemas stricts. Éviter les champs free-form larges ; les remplacer par des enums, tableaux de longueur fixe ou strings courtes bornées.
- Ajouter des exemples explicites aux phase prompts ("Example checkpoint: { check, status, evidence }"). Les models non-reasoning lachent sur les exemples de format bien plus vite que sur des specs abstraites.

### Tuning pour les models de reasoning

Si on swappe pour un model tuning reasoning (o3, DeepSeek-R1, Qwen3 avec `thoughts: "auto"`, Claude Extended Thinking via API) :

```js
const raw = await session.prompt(userText, {
    grammar,
    maxTokens: 900,
    temperature: 0.3
});
```

- Raccourcir les phase prompts. Le model raisonne déjà internally ; les instructions verbose ajoutent du bruit.
- Baisser `maxTokens` pour les phases purement structurelles (Facts, Policy Check). Elles n'ont pas besoin de longs budgets de pensée.
- Garder les schemas comme un **contrat**, pas comme une béquille de raisonnement. Leur rôle principal ici est l'interopérabilité downstream.
- Si le runtime le supporte, logger la trace de raisonnement interne pour le debugging uniquement — jamais comme partie de l'audit trail.

### Spécificités Qwen3

Pour `node-llama-cpp`, le switch clean pour le comportement de pensée Qwen est l'option wrapper :

```js
import { QwenChatWrapper } from "node-llama-cpp";

const reasoningWrapper = new QwenChatWrapper({
    thoughts: "auto",
    keepOnlyLastThought: true
});

const nonReasoningWrapper = new QwenChatWrapper({
    thoughts: "discourage"
});
```

Puis créer la chat session avec le wrapper voulu pour cette phase/run :

```js
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
    systemPrompt,
    chatWrapper: reasoningWrapper // ou nonReasoningWrapper
});
```

Un pattern utile est de mélanger les modes wrapper par phase :

- `thoughts: "discourage"` sur Phase 1 (Facts) et Phase 4 (Policy Check) — travail mécanique.
- `thoughts: "auto"` sur Phase 2 (Red Flags), Phase 3 (Legitimacy) et Phase 5 (Decision) — travail de jugement.

Cela garde la latence totale basse tout en préservant le raisonnement où il compte.

### Callouts par phase

- **Phase 1 (Facts)** — les models non-reasoning hallucinent souvent des entrées de fait qui ont l'air plausibles mais qui n'étaient jamais dans l'input. Resserir le schema (`minItems`, champs de type enum) et instructer explicitement : "Do not infer."
- **Phase 2 (Red Flags)** — les models de reasoning ont tendance à trop suspecter quand on leur donne un framing fraud. Les ancrer avec la liste de checkpoints fixes plutôt que la génération de red flags open-ended.
- **Phase 3 (Legitimacy)** — cette phase existe exactement pour contrer le biais de la Phase 2. Ne pas la fusionner dans la Phase 2 pour économiser des tokens, peu importe la classe de model. C'est un contrepoids structurel.
- **Phase 4 (Policy Check)** — les deux classes bénéficient d'injecter la policy comme JSON inline plutôt que de la décrire en prose. Réduit le drift et l'invention silencieuse de règles.
- **Phase 5 (Decision)** — la calibration de confidence diffère fortement entre classes. Un `confidence: 0.79` d'un model de reasoning n'est pas directement comparable à `0.79` d'un model base. Traiter la confidence comme model-internal ; router sur `final_decision` et `policy_outcome` à la place.

---

## Ordre Suggéré de Lecture du Code

1. `promptJson`
2. `extractFacts`
3. `screenRedFlags`
4. `assessLegitimacy`
5. `checkPolicy`
6. `makeDecision`
7. `runChainOfThoughtReturnDecision`

Cette séquence mirror le runtime et rend l'exemple facile à raisonner.
