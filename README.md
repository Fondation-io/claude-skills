# claude-skills

Skills Claude Code personnelles. Une seule pour l'instant : **codex-review**.

## codex-review — la boucle de review adversariale

Deux modèles, un document, une dispute bornée. Claude écrit (spec, plan, puis code) et arbitre ; OpenAI Codex critique en lecture seule, round après round, jusqu'à `VERDICT: APPROVED` ou le plafond de rounds. La conversation avec le critique survit d'une invocation à l'autre : on corrige le document demain, on relance, le reviewer reprend avec toutes ses critiques en tête au lieu de tout redécouvrir.

La skill s'utilise après un brainstorming/plan superpowers (`/codex-review` sans argument trouve le document tout seul) et couvre trois étages d'une même feature : la **spec**, le **plan**, puis l'**implémentation** (le diff relu contre le plan, idéalement avant commit).

Quatre schémas racontent le fonctionnement, du survol au détail.

### Niveau 1 — l'architecture

![Architecture de la skill](codex-review/architecture.png)

Tout y est, de gauche à droite :

- **La colonne des trois étages.** Spec, plan, implémentation (le diff) — une feature descend ces trois marches, et l'annotation le dit : *une feature, une session*. Le même fil Codex accompagne les trois étages, si bien que le reviewer qui a attaqué la spec puis le plan finit par juger le code en connaissance de cause.
- **La boucle centrale.** Claude (auteur et arbitre) envoie *doc + prompt* ; Codex (un terminal, critique en lecture seule) renvoie un *VERDICT* : `APPROVED` ou `REVISE`. Le compteur *MAX ROUNDS* borne la dispute — elle se termine toujours.
- **Le cliquet de simplicité** sur le chemin du retour : chaque correction acceptée passe ce filtre avant de toucher le document (détaillé planche 4).
- **L'état persistant**, à droite : `sessions.json` (le fil Codex de chaque feature, relié par son *thread_id*), `review-log.md` (tout le débat, verbatim) et `stream.jsonl` (la progression en direct pendant qu'un round tourne).
- **La signature humaine**, en bas : rien ne s'implémente sans le oui final de l'utilisateur.

### Niveau 2a — le kickoff : quel doc, quelle session

![Kickoff — choix du doc et de la session](codex-review/architecture-kickoff.png)

Ce qui se passe avant le premier round.

À gauche, **l'entonnoir à quatre plateaux** : invoquée sans argument, la skill devine le document en essayant quatre sources dans l'ordre — ① le document qu'on vient d'écrire dans la conversation, ② un argument en cours dans `sessions.json`, ③ le disque (`git status`, le doc le plus récent), ④ sinon une seule question. Le résultat (*doc + type + feature*) est toujours annoncé, jamais silencieux : l'utilisateur peut corriger avant de brûler un round.

À droite, **les trois rails vers la bobine** (le fil Codex) : même doc → on reprend le fil ; spec → plan → impl d'une même feature → toujours le même fil ; autre feature → fil neuf. Le fil cassé se traite avec prudence : seul un fil définitivement mort justifie de repartir (une erreur passagère d'authentification ne détruit jamais une session valide — nuance ajoutée après la review, le schéma simplifie).

En bas, **le triangle de couplage** : le document, `sessions.json` et le *thread_id* se tiennent — une entrée par feature, pas de doublon.

### Niveau 2b — un round, concrètement

![Un round, concrètement](codex-review/architecture-round.png)

L'anatomie d'un aller-retour.

1. **La consigne.** Claude écrit le prompt de review, qui impose la discipline des critiques : chaque trouvaille est étiquetée *ça casse* (avec le scénario de casse) ou *trop compliqué* — chaque critique doit prouver son coût.
2. **L'enclos en lecture seule.** Codex travaille enfermé : son shell et ses accès fichiers ne peuvent rien écrire (*jamais d'écriture* — garantie qui couvre les outils intégrés de Codex ; les intégrations externes type MCP sont vérifiées à part au kickoff).
3. **Le ticker.** L'activité sort en continu sur `stream.jsonl` — démarré, il lit les fichiers, terminé — qu'on regarde toutes les 30-60 s, avec un plafond de 10 minutes : un round qui traîne échoue bruyamment au lieu de pendre en silence.
4. **Le verdict.** `APPROVED (validé)` ou `REVISE (à revoir)`. Sur REVISE, Claude corrige et repart — même conversation. Tout le débat s'accumule dans `review-log.md`, en un fichier par feature.

### Niveau 2c — le cliquet de simplicité

![Le cliquet de simplicité](codex-review/architecture-cliquet.png)

Le garde-fou contre la sur-ingénierie, appliqué par Claude à chaque REVISE.

Les critiques arrivent sur le convoyeur et chaque correction candidate passe **deux portes** : *changement minimal ?* (est-ce la plus petite modification qui résout la casse ?) et *système le plus simple ?* (la correction laisse-t-elle la machine avec autant ou moins de rouages ?). La balance en dessous fixe la règle d'or : *ajouter un rouage exige une preuve* — pas de mécanisme spéculatif.

La pancarte borne le filtre lui-même : *une vraie casse est toujours corrigée* — le cliquet gouverne la forme de la correction, jamais l'existence d'un vrai bug. (Avec une nuance apprise en conditions réelles : la prémisse se vérifie d'abord — une critique formulée concrètement peut être fausse, et se rejette alors preuve à l'appui.)

La roue à cliquet ne tourne que dans un sens : *une correction validée ne se rejuge pas*. Et les trois sorties tuent la boucle infinie : *rejeté une fois = rejeté* (sauf preuve nouvelle), *couper les cheveux en quatre : noté, pas bloquant*, et *plus rien ne casse → on s'arrête* — on présente ce qui reste plutôt que de polir sans fin.

## Fidélité des schémas

Les planches sont dessinées par `gpt-image-2` (chaque `architecture-*-body.txt` contient le scénario exact pour regénérer). Elles datent d'avant le durcissement issu de la self-review adversariale (5 rounds, voir `.codex-review/codex-review-skill-review-log.md`) ; trois simplifications à connaître :

- « fil mort → repartir » : seul un fil **définitivement** mort repart de zéro ; une erreur transitoire arrête sans toucher l'état.
- « jamais d'écriture » : garantie du bac à sable shell/fichiers de Codex ; les intégrations externes (MCP) sont signalées à part au kickoff.
- L'état (`sessions.json`) ne s'écrit qu'après un round réussi, jamais dès le démarrage du fil.

## Installation

```bash
git clone git@github.com:darksip/claude-skills.git
ln -s "$PWD/claude-skills/codex-review" ~/.claude/skills/codex-review
```

Usage : `/codex-review` (inférence automatique), ou avec arguments — `feature=<slug>`, `type=spec|plan|impl`, `rounds=<n>`, `reasoning=low|medium|high|xhigh|max`, `fresh=1`. Prérequis : `codex` CLI ≥ 0.130, authentifié (`codex login`). Le contrat complet est dans [codex-review/SKILL.md](codex-review/SKILL.md).
