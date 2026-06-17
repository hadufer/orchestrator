# Orchestrator — Plugin Claude Code (workflows dynamiques, 100% local)

- **Date** : 2026-06-17
- **Statut** : design approuvé
- **Réf.** : https://claudefa.st/blog/guide/development/dynamic-workflows

## Révision 2 (2026-06-17) — pivot vers le runtime de workflows natif

Décision révisée après vérification : **l'exécution ne passe plus par des appels à l'outil `Agent` en conversation, mais par le RUNTIME DE WORKFLOWS NATIF de Claude Code** (Claude écrit un script JS exécuté en arrière-plan, visible dans `/workflows`). Raison : c'est le seul moyen d'obtenir le visuel `/workflows` — et ce runtime est natif/local (pas un produit tiers), donc « tout en local » tient toujours.

- **API réelle confirmée** depuis les scripts de workflow existants de l'utilisateur (`~/.claude/projects/.../workflows/scripts/`) : `export const meta = { name, description, phases:[{title}] }` puis corps top-level `await` + `return` ; primitives `agent(prompt, {label, phase, schema, agentType})`, `parallel(thunks)` (barrière, échecs → `null`), `pipeline(items, ...stages)` (streamé), `phase()`, `log()` ; sortie structurée via JSON Schema (`additionalProperties:false`). PAS de `model`/`effort` par agent → héritage de la session.
- **Opus 4.8 / xhigh** : garanti en lançant le workflow depuis `/orchestrator:orchestrate` (frontmatter `model: claude-opus-4-8`, `effort: xhigh`) ou via `/effort ultracode`.
- **Recettes** : les 6 fichiers `patterns/*.md` contiennent désormais des squelettes JS réels (style calqué sur les workflows de l'utilisateur), et `SKILL.md` a une étape « Step 2 — Author & run the workflow ».
- La table « Décisions verrouillées » ci-dessous est conservée pour l'historique ; la ligne « Substrat = sous-agents natifs (outil Agent) » devient « Substrat = runtime de workflows natif ».

## But

Un plugin Claude Code **100% local, markdown pur, zéro runtime / zéro dépendance** qui agit en
orchestrateur : à partir d'une tâche, il choisit puis exécute le bon pattern d'orchestration parmi
les six du blog, en pilotant l'outil `Agent` **natif** de Claude Code. On reproduit le *résultat* des
« dynamic workflows » sans le runtime JS du blog.

Les six patterns : Classify-And-Act, Fanout-And-Synthesize, Adversarial Verification,
Generate-And-Filter, Tournament, Loop-Until-Done.

## Décisions verrouillées (issues du brainstorming)

| Axe | Choix | Écarté |
|---|---|---|
| Substrat d'exécution | Sous-agents **natifs** (outil `Agent`) | SDK JS, modèles offline |
| Interface | **Routeur unique** `/orchestrate <tâche>`, auto-sélection du pattern | Commandes explicites par pattern |
| Persistance | **Éphémère** (résultats dans la conversation) | Journal, runs reprenables, dashboard |
| Forme | Skills + slash command, **markdown uniquement** | MCP, runtime, agents custom |

## Mapping primitives du blog → natif Claude Code

| Blog | Natif |
|---|---|
| `agent(prompt, opts)` | 1 appel outil `Agent` (subagent_type `general-purpose`/`Explore`/`Plan`), contexte frais |
| `parallel(thunks)` + barrière | **plusieurs appels `Agent` dans UN seul message** ; on attend tous les retours |
| `pipeline(items, …stages)` | enchaînement séquentiel d'appels `Agent` (sortie de l'un = entrée du suivant) |
| `isolation: 'worktree'` | `isolation: "worktree"` sur `Agent` |
| background / loop | `run_in_background: true`, skill `/loop`, `ScheduleWakeup` |
| structured output (JSON Schema) | **consigne de format strict** (JSON/tableau) dans le prompt du sous-agent — pas de schéma applicatif |
| token budget / concurrent cap | cap de parallélisme par défaut + choix de modèle par recette |

## Arborescence

```
C:\Users\hdufer\ultra-local\
├─ .claude-plugin/plugin.json     manifeste du plugin
├─ commands/orchestrate.md        entrée: /orchestrate <tâche> (mince → délègue au skill)
├─ skills/orchestrator/
│   ├─ SKILL.md                   LE routeur: porte de tri + table de sélection + garde-fous
│   └─ patterns/                  6 fiches chargées seulement quand le pattern est retenu
│       ├─ classify-and-act.md
│       ├─ fanout-and-synthesize.md
│       ├─ adversarial-verification.md
│       ├─ generate-and-filter.md
│       ├─ tournament.md
│       └─ loop-until-done.md
└─ README.md                      quoi / comment installer / note coût
```

**Pourquoi 6 fichiers séparés et pas tout dans `SKILL.md`** : divulgation progressive. Le `SKILL.md`
(toujours chargé) reste court ; le détail d'un pattern n'entre dans le contexte que quand ce pattern
est choisi. Moins de tokens par run.

## Composants

### `.claude-plugin/plugin.json`
Manifeste standard : `name: "orchestrator"`, `description`, `version: "0.1.0"`, `author`.
`commands/` et `skills/` auto-découverts depuis la racine du plugin.

### `commands/orchestrate.md`
Entrée mince. Frontmatter : `description`, `argument-hint: "<tâche>"`. Corps : instruit Claude
d'invoquer le skill `orchestrator` sur `$ARGUMENTS`.

### `skills/orchestrator/SKILL.md` (le routeur)
Frontmatter `name`/`description` (description rédigée pour auto-trigger : « orchestre, workflow,
lance plusieurs agents, fanout, tournoi, vérifie… »). Contenu :

1. **Porte de tri (étape 0)** — la tâche mérite-t-elle un workflow ? Sinon : la faire directement et
   le dire. Refus si triviale / mono-fichier / réponse immédiate (le blog : pas de workflow pour un
   fix de 2 lignes).
2. **Table de sélection** des 6 patterns (quand choisir chacun) + composition autorisée
   (ex. Fanout → Adversarial).
3. **Dispatch** : charger `patterns/<nom>.md` et suivre la recette.
4. **Garde-fous globaux** : cap parallélisme ≤ ~6 par défaut, choix de modèle (haiku = volume,
   sonnet/opus = jugement), sorties structurées, **une synthèse finale unique** présentée à l'utilisateur.

### `skills/orchestrator/patterns/*.md`
Chaque fiche décrit : *quand l'utiliser*, la *structure exacte des appels `Agent`* (combien, quels
`subagent_type`, prompts types, parallèle vs séquentiel, barrière), le *format de sortie attendu*, et
*comment synthétiser/présenter*.

- **classify-and-act** : 1 classifieur → label structuré → dispatch vers traitement spécialisé (ou vers un autre pattern).
- **fanout-and-synthesize** : découper en N unités indépendantes → N `Agent` en **un seul message** → 1 synthétiseur fusionne (dédup, conflits). Barrière naturelle.
- **adversarial-verification** : produire des findings → pour chaque, 1 `Agent` réfuteur **frais et aveugle** (consigne : « tente de prouver que c'est faux ») → garder ceux qui résistent. Composable après un fanout.
- **generate-and-filter** : générer beaucoup (1 agent ×N idées OU N agents) → 1 agent filtre (rubrique explicite + dédup) → top-k.
- **tournament** : N solveurs en parallèle (approches/températures variées) → juges en **comparaison par paires** (bracket « A vs B ») → gagnant. Pairwise > scoring absolu.
- **loop-until-done** : tours successifs d'`Agent` de découverte ; agréger le nouveau ; **stop après 2 tours consécutifs sans nouveauté** (+ cap de tours de sécurité). Option `run_in_background` / `/loop`.

### `README.md`
Quoi, installation locale, exemple d'usage, **note coût** (workflows = chers, réservés aux tâches qui le méritent).

## Garde-fous & coût

- Anti-gâchis : porte de tri **obligatoire** avant tout spawn.
- Parallélisme plafonné par défaut.
- Modèle adapté au rôle (volume vs jugement).
- Avertissement coût dans `README` et `SKILL.md`.

## Hors périmètre v0.1 (greffable plus tard sans rien casser)

- Persistance / journal des runs ; runs reprenables (ID, cache, resume) ; dashboard `/workflows`.
- Commandes explicites par pattern.
- Agents custom prédéfinis (`agents/`).
- Modèles offline / runtime JS / SDK.

## Vérification d'acceptation

Pas de code exécutable — c'est du prompt. Check = activer le plugin et lancer `/orchestrate` sur des
tâches jouets, vérifier le routage :

- « résume ces N fichiers » → **Fanout-And-Synthesize**
- « trouve tous les bugs » → **Loop-Until-Done** (ou Fanout → Adversarial)
- « meilleur nom / design pour X » → **Generate-And-Filter** ou **Tournament**
- un « fix de 2 lignes » → la **porte de tri refuse** et fait la tâche directement, sans spawn.

## Risques / à confirmer à l'implémentation

- Mécanisme exact d'activation d'un plugin **local** sous Windows (chemin, `/plugin`, marketplace local, ou `settings.json`).
- Format précis du frontmatter `command`/`skill` dans un plugin + **namespacing** du `/orchestrate`.
- Confirmer que plusieurs `Agent` dans un même message tournent bien en parallèle ici (le system-reminder de l'environnement le confirme).
