## Tree of Thought : Analyse de motivation d'une personne

**Idée :** Une personne montre un comportement puzzle. L'agent explore multiple explications psychologiques en parallèle, les score, et expand uniquement l'hypothèse la plus forte en analyse finale.

---

### Arbre visuel

```text
[Comportement : "Personne quitte un emploi sécurisé sans plan clair"]
         |
    [Phase 1 : Branch - 4 hypothèses]
    /        |         |        \
[Évitement][Burnout][Croissance][Pression externe]
    |         |          |            |
[Phase 2 : Score]
   6         9          7            4
             |
      [Phase 3 : Prune - perdants retirés]
             |
 [Phase 4 : Conclusion depuis UNIQUEMENT Burnout]
```

---

### Ce que l'Exemple 12 démontre

1. **Branch :** Construire quatre hypothèses psychologiques concurrentes.
2. **Score :** Évaluer chaque hypothèse indépendamment.
3. **Prune :** Garder uniquement la branche au score le plus élevé.
4. **Conclusion :** Produire une analyse finale depuis le winner uniquement.

Cela met intentionnellement en lumière la force du ToT (exploration structurée) et sa faiblesse (perte d'information après pruning).

---

### Les trois principes fondamentaux dans le code

| Principe | Ce qui se passe dans le code |
|---|---|
| Branching | `developHypothesis()` crée une hypothèse par lens |
| Évaluation | `scoreHypothesis()` score chaque hypothèse |
| Pruning | `pruneHypotheses()` élimine tous les non-winners |

---

### Désavantage structurel montré explicitement

À la fin, la console affiche ce qui a été éliminé.
Ces branches peuvent contenir des insights correctifs, mais elles n'influencent plus la conclusion finale.

C'est la limitation fondamentale du pruning ToT strict.

---

### Quand utiliser le ToT dans le travail réel

Utiliser le Tree of Thought quand on a besoin d'un winner clair et d'un chemin de décision simple.

#### Mental model d'un System Admin

On reçoit une alerte production : la latence API est passée de 200 ms à 2 s après un release.

- **Branches :** Saturation DB, tempête de cache miss, ou problème réseau noisy-neighbor.
- **Score :** Chaque branche reçoit un scoring basé sur les preuves des dashboards et logs.
- **Prune :** Choisir la cause la plus confiante (par exemple cache collapse).
- **Act :** Exécuter un chemin de remediation en premier (par exemple cache warmup d'urgence + rollback TTL).

Pourquoi le ToT fit : la réponse à incident nécessite souvent un chemin de décision rapide et auditable au lieu de maintenir de nombreux tracks de remediation en parallèle.

#### Mental model d'un Développeur

On a besoin d'accélérer un endpoint lent avant un launch.

- **Branches :** Ajouter le caching Redis, réécrire la requête avec de meilleurs indexes, ou pré-computer les données de manière asynchrone.
- **Score :** Évaluer par effort d'implémentation, risque, gain attendu et testabilité.
- **Prune :** Sélectionner une stratégie à implémenter maintenant.
- **Act :** Ship le changement choisi et mesurer.

Pourquoi le ToT fit : quand les deadlines approchent, les équipes ont généralement besoin d'un winner d'implémentation, pas d'un experiment d'architecture combiné.

#### Mental model d'un Créateur d'Agent IA

On construit un agent de codage autonome qui doit choisir une stratégie de fix.

- **Branches :** Patch minimal, refactor plus profond, ou rollback + guardrail.
- **Score :** Ranger par risque d'échec, blast radius et confiance basée sur les preuves du repo.
- **Prune :** Garder un plan d'exécution.
- **Act :** Exécuter, vérifier et rapporter.

Pourquoi le ToT fit : on obtient un comportement prévisible, un coût token/outil plus bas, et des postmortems plus faciles car l'agent suit un plan explicite.

Le ToT est le plus fort quand :

- le temps est limité,
- la sortie doit être une direction actionnable,
- et le coût de maintenir de nombreuses alternatives vivantes est plus élevé que le risque de perdre de la nuance.
