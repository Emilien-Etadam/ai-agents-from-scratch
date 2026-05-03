## Chain of Thought : Décision de retour (fraud vs légitime)

**Idée :** Un client demande un retour. Au lieu de laisser le model sauter directement vers "approve/reject", il doit écrire une chaîne de raisonnement explicite comme un agent de support qui documente un cas pour un superviseur.

---

### Chaîne visuelle

```text
[Requête de retour : "Claim de défaut sur des écouteurs"]
         |
[Phase 1 : Facts uniquement]
         |
[Phase 2 : Red flags]
         |
[Phase 3 : Legitimacy]
         |
[Phase 4 : Policy check]
         |
[Phase 5 : Décision finale]
```

---

### Ce que l'Exemple 14 démontre

1. **Facts :** Extraire les données uniquement, pas de jugement précoce.
2. **Red Flags :** Exécuter un screening fraud explicite checkpoint par checkpoint.
3. **Legitimacy :** Construire l'argument côté client comme force d'équilibrage.
4. **Policy Check :** Appliquer les règles (window, valeur, history) avant de décider.
5. **Decision :** Décider uniquement après que toutes les phases précédentes sont complètes.

Cette structure rend la décision auditable et debuggable.

---

### Pourquoi Cela Compte dans les Cas Borderline

Sans Chain of Thought, un cas borderline devient souvent un guess.

Exemple :
- score fraud : **6/10**
- score legitimacy : **7/10**

Un classifieur one-shot direct peut flipper aléatoirement selon le phrasing du prompt.
Avec CoT, on peut inspecter chaque phase, identifier le step faible, et corriger la chaîne (par exemple l'interprétation policy ou le traitement d'évidence manquante).

---

### Les cinq phases CoT et leur rôle

| Phase | Pourquoi elle existe |
|---|---|
| Facts | Prévenir le biais précoce en séparant l'extraction de l'évaluation |
| Red Flags | Forcer la couverture explicite de la checklist de risque fraud |
| Legitimacy | Préserver l'équité client et éviter la suspicion unilatérale |
| Policy Check | Contraindre le comportement du model avec des règles business |
| Decision | Produire un outcome traçable avec un rationale clair |

---

### Takeaway Fondamental

Le Chain of Thought n'améliore pas juste la qualité de réponse.
Il améliore la **gouvernance** :

- les superviseurs peuvent auditer pourquoi un cas a été approved/rejected,
- les équipes peuvent repérer où le drift de raisonnement a eu lieu,
- et les changements de policy peuvent être reflétés en update une seule phase au lieu de réécrire tout le prompt.

---

### CoT avec les LLMs de Reasoning vs Non-Reasoning

Une confusion courante : "Si les models de reasoning comme o3, DeepSeek-R1 ou Qwen3 avec thought mode activé raisonnent déjà internally, ai-je encore besoin d'un Chain of Thought explicite ?"

La réponse est oui, mais le rôle du CoT change.

#### Le mental model

- **LLM non-reasoning** (base GPT-4o, Llama-3 chat, Qwen3 avec `thoughts: "discourage"`, Phi) :
  - prompt -> réponse.
  - Il n'y a pas de raisonnement intermédiaire à moins de le construire.
  - Le scaffolding CoT **crée** le raisonnement qui n'existerait autrement pas.
- **LLM de reasoning** (o3, DeepSeek-R1, Claude Extended Thinking, Qwen3 avec `thoughts: "auto"`) :
  - prompt -> chaîne cachée -> réponse.
  - Le model produit déjà des tokens de raisonnement interne.
  - Le scaffolding CoT **canalise** ce raisonnement dans une forme fixe et inspectable.

#### Pourquoi le CoT explicite est critique sur un model non-reasoning

- Sans scaffolding, les cas borderline (fraud 6/10 vs legitimacy 7/10) s'effondrent en comportement de coin-flip.
- Le model n'a pas de "place" pour raisonner, donc il choisit une forme de réponse et back-fills la justification.
- Chacune des 5 phases force une couverture que le model sauterait autrement — surtout la phase legitimacy, qui contre la suspicion unilatérale.
- La schema grammar compte plus ici, car le model a moins de défenses contre le drift hors du contrat.

#### Pourquoi le CoT explicite reste valorisable sur un model de reasoning

- Le raisonnement interne caché n'est **pas auditable**. La compliance, le QA support et les reviews d'incident ont besoin d'une trace écrite, pas d'un opaque "on trust le model".
- Le raisonnement interne ne suit pas **votre** taxonomie. Votre checklist fraud, vos règles policy, votre workflow refund — tout ça est domain-specific et absent de n'importe quel corpus de pretraining.
- On ne peut pas fixer un step qu'on ne voit pas. Si un cas borderline continue de mal aller, les phases structurées permettent de localiser le maillon faible (par exemple : le raisonnement legitimacy est trop soft) et d'améliorer uniquement ce prompt.
- Le raisonnement interne varie entre runs. Le CoT structuré produit un contrat stable pour le tooling downstream (logging, analytics, escalation routing).
- Les traces de raisonnement publiques des models de reasoning peuvent être des **rationalisations post-hoc** plutôt que le vrai chemin de décision. Les traiter comme une feature UX, pas comme une preuve d'audit.

#### Recommandations Pratiques

- Model reasoning + CoT light : garder les 5 phases, raccourcir les phase prompts, laisser le model raisonner à l'intérieur de chaque call. Moins de verbosité, même auditabilité.
- Model non-reasoning + CoT heavy : garder les 5 phases, expand les phase prompts avec des checklists et exemples, resserrer les schemas, baisser la temperature.
- Model hybrid comme Qwen3 : choisir un mode thought par phase. Utiliser `thoughts: "auto"` sur Phase 5 (Decision) où les trade-offs comptent, et `thoughts: "discourage"` sur Phase 1 (Facts) où l'extraction est mécanique.

#### Anti-Patterns

- Dire à un model de reasoning de "think step by step" dans le prompt — dépense de tokens redondante, et ça peut dérailer la chaîne interne du model.
- Utiliser un model non-reasoning pour des décisions borderline sans CoT — les résultats ne sont ni reproductibles, ni défendables, ni sûrs en production.
- Trust les traces de raisonnement raw des models de reasoning comme preuve d'audit — elles ont l'air convaincantes mais ne sont pas policy-compliant par construction.
- Comparer les valeurs de `confidence` entre classes de models — la calibration diffère fortement ; traiter la confidence comme model-internal uniquement.

#### Bottom Line

Le CoT n'est pas un substitute pour un model de reasoning, et un model de reasoning n'est pas un substitute pour le CoT. Ils résolvent des problèmes différents :

- **Les models de reasoning** améliorent la qualité brute de réponse.
- **Le Chain of Thought** transforme tout raisonnement en un workflow gouvernable.

Pour les décisions à fort impact, on veut généralement les deux.

---

### Quand utiliser le CoT dans le travail réel

Utiliser le Chain of Thought quand les décisions sont à fort impact et nécessitent de la reviewability.

#### Mental model d'un System Admin

Une requête de déploiement semble risquée, mais pas obviously wrong.

- **Facts :** load actuelle, incidents récents, readiness de rollback.
- **Risk flags :** steps de runbook manquants, escalation de privilège, risques de timing.
- **Legitimacy :** urgence business, window de maintenance, controls de mitigation.
- **Policy :** règles de change management et portes d'approbation.
- **Decision :** approve, reject ou escalader vers review manuel.

Pourquoi le CoT fit : les décisions opérationnelles ont besoin d'un audit trail, pas de gut feeling.

#### Mental model d'un Développeur

Un pull request est controversé et peut introduire des régressions.

- **Facts :** modules changés, résultats de tests, deltas de performance.
- **Risk flags :** pas de plan de migration, dépendances fragiles, couverture faible.
- **Legitimacy :** impact utilisateur, sévérité du bug, urgence de release.
- **Policy :** exigences de review, branch protection, critères de release.
- **Decision :** merge, block ou demander des checks additionnels.

Pourquoi le CoT fit : la qualité de code review s'améliore quand le rationale est structuré et inspectable.

#### Mental model d'un Créateur d'Agent IA

Un agent de support autonome doit décider des refunds en sécurité.

- **Facts :** timeline de commande, history de compte, evidence fournie.
- **Risk flags :** patterns d'abus et incohérence d'identité.
- **Legitimacy :** indicateurs de défaut plausibles et contexte client.
- **Policy :** contraintes hard des règles business.
- **Decision :** output de workflow deterministe avec confidence et notes.

Pourquoi le CoT fit : il donne des traces de raisonnement transparentes qui sont plus faciles à monitorer et corriger.
