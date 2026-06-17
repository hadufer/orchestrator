# Orchestrator Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A local, markdown-only Claude Code plugin that routes any task to the right multi-agent orchestration pattern and runs it with Claude Code's native `Agent` tool.

**Architecture:** A thin slash command (`/orchestrator:orchestrate`) hands the task to a router skill. The router runs a triage gate (is a workflow even warranted?), picks one of six patterns (or composes them), then loads that pattern's recipe file on demand and drives native subagents. No runtime, no dependencies, ephemeral results.

**Tech Stack:** Claude Code plugin (manifest + skill + command), pure markdown. Native `Agent` tool for subagents; multiple `Agent` calls in one message for parallelism + barrier; `isolation: "worktree"` for parallel edits; `run_in_background`/`/loop` for loops.

> **Révision 2 (2026-06-17):** l'exécution a été pivotée vers le **RUNTIME DE WORKFLOWS NATIF** (Claude écrit un script JS visible dans `/workflows`), au lieu d'appels `Agent` en conversation — pour obtenir le visuel `/workflows`. L'API réelle a été confirmée depuis les scripts existants de l'utilisateur. Les recettes `patterns/*.md` contiennent maintenant des squelettes JS réels et `SKILL.md` a une étape « Author & run the workflow ». Opus 4.8/xhigh = via le frontmatter de la commande ou `/effort ultracode`. La structure des tâches 1–5 ci-dessous est inchangée ; seul le CONTENU de `SKILL.md` et des 6 recettes reflète le runtime natif.

## Global Constraints

- **Plugin root:** `C:\Users\hdufer\ultra-local\orchestrator\` (refines the spec's flat layout: plugin nested so `--plugin-dir ./orchestrator` is clean and process docs stay separate).
- **Manifest path:** `.claude-plugin/plugin.json`; plugin `name` = `orchestrator`. Only `name` is required; include `description`, `version`, `author`.
- **Invocation is namespaced:** command = `/orchestrator:orchestrate`, skill = `orchestrator:orchestrator`. Bare `/orchestrate` is not possible for plugin commands.
- **Markdown only.** No code, no MCP, no `agents/` custom defs, no runtime. Orchestration is structured `Agent` calls.
- **Persistence: none** (ephemeral; results in conversation).
- **Reference files** are loaded on demand via relative markdown links from `SKILL.md` (e.g. `[x](./patterns/fanout-and-synthesize.md)`).
- **Guardrails baked into prompts:** triage gate before any spawn; fan-out cap ~6; right model per role (haiku=volume, sonnet/opus=judgment); structured returns; one final synthesis.
- **Deliverable is prompt/markdown** → "tests" are behavioral: the plugin loads, the command/skill are present, and dry-run routing picks the expected pattern. No unit-test framework (nothing executable to assert on).
- **Git:** this dir is NOT a git repo and the user has not asked to version it → commit steps are omitted. Run `git init` first if you want per-task commits.

---

### Task 1: Plugin scaffold + manifest

**Files:**
- Create: `C:\Users\hdufer\ultra-local\orchestrator\.claude-plugin\plugin.json`

**Interfaces:**
- Produces: a loadable plugin named `orchestrator`. Later tasks add `commands/orchestrate.md` and `skills/orchestrator/**`, which Claude Code auto-discovers from the plugin root.

- [ ] **Step 1: Create the manifest**

`orchestrator\.claude-plugin\plugin.json`:
```json
{
  "name": "orchestrator",
  "description": "Dynamic multi-agent workflows: routes a task to the right orchestration pattern (classify-and-act, fanout-and-synthesize, adversarial verification, generate-and-filter, tournament, loop-until-done) and drives Claude Code's native subagents. Local, markdown-only, no runtime.",
  "version": "0.1.0",
  "author": { "name": "hdufer" }
}
```

- [ ] **Step 2: Load the plugin and verify it is recognized**

Run (from `C:\Users\hdufer\ultra-local`): `claude --plugin-dir ./orchestrator`
Or, in an existing session after adding files: `/reload-plugins`
Expected: no manifest errors; plugin `orchestrator` is listed as active. (Commands/skills appear after Tasks 2–4.)

---

### Task 2: Entry command

**Files:**
- Create: `C:\Users\hdufer\ultra-local\orchestrator\commands\orchestrate.md`

**Interfaces:**
- Consumes: nothing yet (skill arrives in Task 3).
- Produces: `/orchestrator:orchestrate <task>` — invokes the `orchestrator:orchestrator` skill with `$ARGUMENTS`.

- [ ] **Step 1: Create the command file**

`orchestrator\commands\orchestrate.md`:
```markdown
---
description: Run a task through the right multi-agent orchestration pattern (auto-selected).
argument-hint: <task to orchestrate>
---

Invoke the `orchestrator:orchestrator` skill to handle this task end-to-end:

$ARGUMENTS

Follow the skill's triage gate FIRST: if the task does not warrant a multi-agent workflow, do it directly and say so instead of spawning agents.
```

- [ ] **Step 2: Verify the command is present**

Run: `/reload-plugins` then type `/` and confirm `/orchestrator:orchestrate` is listed with its argument hint.
Expected: command appears. (Invoking it fully is verified in Task 5, once the skill exists.)

---

### Task 3: Router skill (`SKILL.md`)

**Files:**
- Create: `C:\Users\hdufer\ultra-local\orchestrator\skills\orchestrator\SKILL.md`

**Interfaces:**
- Consumes: `$ARGUMENTS` task text (from the command, or natural-language auto-trigger).
- Produces: a triage decision, a pattern choice, and on-demand links to `./patterns/<name>.md` (created in Task 4). The six link targets MUST match Task 4 filenames exactly: `classify-and-act.md`, `fanout-and-synthesize.md`, `adversarial-verification.md`, `generate-and-filter.md`, `tournament.md`, `loop-until-done.md`.

- [ ] **Step 1: Create the router skill**

`orchestrator\skills\orchestrator\SKILL.md`:
```markdown
---
name: orchestrator
description: Use when a task benefits from multiple coordinated subagents — "orchestrate", "run a workflow", fan out across files/modules, verify findings adversarially, generate-and-filter candidates, run a tournament for the best output, or loop until done. Routes the task to the right pattern and drives native subagents.
---

# Orchestrator

Route a task to the right multi-agent pattern and run it with Claude Code's **native `Agent` tool**. There is no runtime and no external service — "orchestration" is just how you structure `Agent` calls.

## Step 0 — Triage gate (always first)

Workflows are powerful and EXPENSIVE: you pay for many agents instead of one. Before spawning anything, decide whether the task warrants it.

**Do NOT orchestrate** — do the task directly and say why — when it is:
- a small/local change (a few lines, one file), or
- a question with a single direct answer, or
- something faster to do yourself than to brief an agent.

**Orchestrate** only when the task has genuine parallelism, needs isolation between units, needs adversarial checking, or explores an open-ended space.

When unsure, lean toward doing it directly. State the decision in one line:
`Triage: direct — <reason>` or `Triage: workflow → <pattern> — <reason>`.

## Step 1 — Pick the pattern

| Pick | When the task is… |
|---|---|
| **Classify-And-Act** | "handle X" where X has distinct types needing different treatment; or you must triage before acting |
| **Fanout-And-Synthesize** | N independent units to process then merge (audit each module, summarize each file, review each section) |
| **Adversarial Verification** | producing findings/claims whose correctness must be checked (security review, fact-check, "is this real?") |
| **Generate-And-Filter** | you want many candidates then the best few (names, ideas, refactor options) |
| **Tournament** | you want the single best output and quality is subjective / hard to score absolutely (best copy, design, implementation) |
| **Loop-Until-Done** | open-ended discovery, unknown number of rounds (find ALL the bugs, exhaustive search) |

**Compose** when useful — e.g. Fanout to gather findings → Adversarial to verify them → present survivors.

Announce the choice, then load and follow the matching recipe:

- [Classify-And-Act](./patterns/classify-and-act.md)
- [Fanout-And-Synthesize](./patterns/fanout-and-synthesize.md)
- [Adversarial Verification](./patterns/adversarial-verification.md)
- [Generate-And-Filter](./patterns/generate-and-filter.md)
- [Tournament](./patterns/tournament.md)
- [Loop-Until-Done](./patterns/loop-until-done.md)

## Global guardrails (every pattern)

- **Parallel = one message.** To run agents concurrently, emit multiple `Agent` calls in a SINGLE message; they run in parallel and return together (the barrier). Sequential dependencies go in separate messages.
- **Cap fan-out** at ~6 concurrent agents by default; if there are more units, batch them.
- **Right model per role:** `haiku` for high-volume/simple work (per-file summaries, candidate generation); `sonnet`/`opus`/inherit for judgment (synthesis, refutation, pairwise judging).
- **Fresh & isolated:** give each agent only what it needs in its prompt. Verifiers/refuters/judges must NOT see the producer's reasoning.
- **Structured returns:** tell each agent to end with a strict block (JSON or a fixed table) so you can parse and merge reliably.
- **Isolation for edits:** if agents modify the repo in parallel, pass `isolation: "worktree"`.
- **One synthesis out.** Always finish with a single consolidated result for the user — never a dump of raw agent outputs.
```

- [ ] **Step 2: Verify the skill loads and triggers**

Run: `/reload-plugins`. Then give a workflow-shaped task (e.g. "orchestrate: review these 5 files for issues").
Expected: the skill activates, prints a `Triage:` line, and names a pattern. (Pattern files don't exist yet → it will fail to open the link; that's fixed in Task 4.)

---

### Task 4: Pattern playbook (six recipe files)

**Files:**
- Create: `orchestrator\skills\orchestrator\patterns\classify-and-act.md`
- Create: `orchestrator\skills\orchestrator\patterns\fanout-and-synthesize.md`
- Create: `orchestrator\skills\orchestrator\patterns\adversarial-verification.md`
- Create: `orchestrator\skills\orchestrator\patterns\generate-and-filter.md`
- Create: `orchestrator\skills\orchestrator\patterns\tournament.md`
- Create: `orchestrator\skills\orchestrator\patterns\loop-until-done.md`

**Interfaces:**
- Consumes: the global guardrails defined in `SKILL.md` (Task 3) — each recipe assumes them (parallel=one message, cap ~6, model-per-role, structured returns, one synthesis).
- Produces: six self-contained recipes the router dispatches to. Filenames MUST match the links in `SKILL.md` exactly.

- [ ] **Step 1: `classify-and-act.md`**

```markdown
# Classify-And-Act

**Use when:** the task is "handle X" and X comes in distinct types needing different treatment, or you must triage before acting.

**Recipe:**
1. Define a CLOSED set of labels up front.
2. **Classify:** one `Agent` (`haiku`/`sonnet`) reads the input and returns a label, ending with a strict block: `CLASS: <label>` then `REASON: <one line>`.
3. **Act on the label:** dispatch either to a specialized handler (a tailored `Agent` prompt per class) or to another pattern (e.g. class "audit" → Fanout-And-Synthesize).
4. (Optional) run the classifier again at the END to label/package the final output.

**Output:** the chosen class + the result of the matching action.
```

- [ ] **Step 2: `fanout-and-synthesize.md`**

```markdown
# Fanout-And-Synthesize

**Use when:** the task splits into N independent units that can be processed without seeing each other, then merged.

**Recipe:**
1. Decompose into units (files, modules, sections, items) and list them.
2. **Fan out (barrier):** in ONE message, emit one `Agent` per unit (cap ~6; batch if more). Each agent:
   - `subagent_type`: `Explore` (read-only analysis) or `general-purpose` (if it must run things); `model: haiku` for simple/volume work.
   - prompt: the unit's scope + exactly what to return, ending with a strict block, e.g. `## <unit>` then `- finding: …` then `- evidence: file:line`.
3. **Synthesize:** once all return, merge — yourself, or with one `general-purpose`/`sonnet` synthesizer if the volume is large: dedupe, resolve conflicts, group by theme.
4. Present ONE consolidated result (summary on top, then per-unit detail).

**Output:** a unified, deduped report keyed by unit, with a short summary header.
```

- [ ] **Step 3: `adversarial-verification.md`**

```markdown
# Adversarial Verification

**Use when:** you have findings/claims and must keep only those that hold up. Separating producer from checker removes self-preferential bias.

**Recipe:**
1. Get the findings (from a prior Fanout, a review, or generate them now). Number them.
2. **Refute (barrier):** in ONE message, one `Agent` per finding (cap ~6; batch if more). Each refuter:
   - `subagent_type`: `general-purpose` (or `Explore` if read-only); `model`: `sonnet`/inherit (judgment).
   - prompt: ONLY the single finding + the context needed to check it. Do NOT include the producer's reasoning or the other findings. Instruction: "Try to prove this is FALSE/invalid. Cite evidence. End with `VERDICT: holds` or `VERDICT: refuted — <reason>`."
3. Keep findings whose refuter returns `holds`; drop/flag the rest.
4. Present survivors with evidence; list refuted/uncertain ones briefly.

**Output:** verified findings + a short "refuted/uncertain" appendix.
```

- [ ] **Step 4: `generate-and-filter.md`**

```markdown
# Generate-And-Filter

**Use when:** you want many candidates (noisy is fine) then the best few — naming, brainstorming, refactor proposals.

**Recipe:**
1. **Generate:** either one `Agent` (`haiku`) asked for many candidates (e.g. 15), or several agents in one message each producing a batch. Quantity over polish; fixed format, one per line.
2. **Filter:** one `Agent` (`sonnet`, judgment) receives ALL candidates + an explicit rubric. It dedupes near-duplicates, scores against the rubric, and returns the top-k each with a one-line justification.
3. Present the top-k; optionally summarize what was discarded and why.

**Output:** top-k candidates with justifications; discarded set summarized.
```

- [ ] **Step 5: `tournament.md`**

```markdown
# Tournament

**Use when:** you want THE single best output and quality is subjective (copy, design, naming, an implementation). Pairwise comparison beats absolute scoring.

**Recipe:**
1. **Attempts (barrier):** in ONE message, spawn N solver agents (3–5) on the SAME task with DIFFERENT angles (vary the prompt: "optimize for X" vs "for Y"; or vary framing/model). Each returns one candidate in a fixed format.
2. **Pairwise judging:** compare candidates two at a time. For each pair, one `Agent` (`sonnet`/`opus`, judgment) sees both (anonymized A/B) + the rubric and returns `WINNER: A|B` + a one-line reason. Run a simple bracket: pair them, advance winners, repeat until one remains. Independent pairs can be judged in parallel (one message).
3. Present the winner + why it won, and optionally the runner-up.

**Output:** the winning candidate + a short rationale from the final comparison.
```

- [ ] **Step 6: `loop-until-done.md`**

```markdown
# Loop-Until-Done

**Use when:** open-ended discovery where you don't know how many passes — "find ALL the bugs", exhaustive search. Completion depends on results, not a fixed count.

**Recipe:**
1. Keep a running set of items found (start empty).
2. **Round:** spawn one `Agent` (fresh context) to find NEW items not already in the set — pass it the list of what's already found so it doesn't repeat. It returns new items in a fixed format, or `NONE`.
3. Add new items to the set. Repeat.
4. **Stop** after 2 consecutive rounds that add nothing new — OR at a safety cap (e.g. 6 rounds), whichever comes first. State which fired.
5. (Optional) for long runs use `run_in_background: true` or the `/loop` skill.

**Output:** the full accumulated set + which stop condition fired.
```

- [ ] **Step 7: Verify recipes load on demand**

Run: `/reload-plugins`, then `/orchestrator:orchestrate summarize these files: a.md b.md c.md`.
Expected: router announces `Fanout-And-Synthesize`, opens `./patterns/fanout-and-synthesize.md`, and structures one `Agent` per file in a single message. (Use a folder with a few small files, or stub names, to observe routing without a large run.)

---

### Task 5: README + acceptance dry-run

**Files:**
- Create: `C:\Users\hdufer\ultra-local\orchestrator\README.md`

**Interfaces:**
- Consumes: the command/skill/patterns from Tasks 1–4.
- Produces: install/usage/cost docs + a recorded acceptance check.

- [ ] **Step 1: Create the README**

`orchestrator\README.md`:
```markdown
# orchestrator

Local Claude Code plugin: turns a task into a dynamic multi-agent workflow using Claude Code's **native** subagents. No runtime, no dependencies, no external services — orchestration is just structured `Agent` calls. Reproduces the six "dynamic workflow" patterns locally.

## Install (local dev)

From the folder that contains `orchestrator/`:

    claude --plugin-dir ./orchestrator

Already in a session? Run `/reload-plugins`.

## Use

    /orchestrator:orchestrate <your task>

Or just ask Claude to "orchestrate" / "run a workflow" — the skill auto-triggers.

The router first decides whether the task even warrants a workflow (it refuses small/trivial tasks and just does them). If it does, it picks one of six patterns — Classify-And-Act, Fanout-And-Synthesize, Adversarial Verification, Generate-And-Filter, Tournament, Loop-Until-Done — or composes several.

## Cost

Workflows spend MANY times the tokens of a normal turn (many agents instead of one). Use them for work that genuinely needs parallelism, isolation, or adversarial checking — not for two-line fixes.
```

- [ ] **Step 2: Acceptance dry-run (the verification)**

Run `/reload-plugins`, then check routing on these and confirm the expected pattern is announced:
- `/orchestrator:orchestrate summarize these 8 files` → **Fanout-And-Synthesize**
- `/orchestrator:orchestrate find all the bugs in this module` → **Loop-Until-Done** (or Fanout → Adversarial)
- `/orchestrator:orchestrate pick the best name for this product` → **Generate-And-Filter** or **Tournament**
- `/orchestrator:orchestrate fix the typo in line 3 of README` → **Triage: direct** (no spawn)

Expected: first three announce the right pattern and load the matching recipe; the fourth is refused by the triage gate and done directly. This is the acceptance criterion — no automated tests, since the deliverable is prompts.
```
