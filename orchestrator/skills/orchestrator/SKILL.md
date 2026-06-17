---
name: orchestrator
description: Use when a task benefits from multiple coordinated subagents — "orchestrate", "run a workflow", fan out across files/modules, verify findings adversarially, generate-and-filter candidates, run a tournament for the best output, or loop until done. Routes the task to the right pattern and authors a native dynamic workflow.
model: claude-opus-4-8
effort: xhigh
---

# Orchestrator

Route a task to the right multi-agent pattern, then **author and run a native Claude Code dynamic workflow** — a small JS script the `/workflows` runtime executes in the background (watch it live in `/workflows`). You don't call the `Agent` tool by hand; you write a script using `agent()` / `parallel()` / `pipeline()`.

## Step 0 — Always run the full process (no shortcut)

`/orchestrator:orchestrate` ALWAYS runs the complete process — there is **no bailing out to a direct answer**, even for a small or trivial task. Every run goes through Step 1 (pick a pattern) and Step 2 (author the workflow with all its phases + final synthesis).

Do NOT shortcut, do NOT "just do it directly". Announce the choice in one line — it is always a workflow:
`Triage: workflow → <pattern> — <reason>`

Cost is accepted by design: even a one-line task spins up several Opus 4.8 agents. Full process and capability over token economy — that is the intended behavior here.

## Step 1 — Pick the pattern

| Pick | When the task is… |
|---|---|
| **Classify-And-Act** | "handle X" where X has distinct types needing different treatment; or you must triage before acting |
| **Fanout-And-Synthesize** | N independent units to process then merge (audit each module, summarize each file, review each section) |
| **Adversarial Verification** | producing findings/claims whose correctness must be checked (security review, fact-check, "is this real?") |
| **Generate-And-Filter** | you want many candidates then the best few (names, ideas, refactor options) |
| **Tournament** | you want the single best output and quality is subjective / hard to score absolutely (best copy, design, implementation) |
| **Loop-Until-Done** | open-ended discovery, unknown number of rounds (find ALL the bugs, exhaustive search) |

**Compose** when useful — e.g. Fanout to gather findings → Adversarial to verify them → return survivors. That is ONE workflow with two phases, not two workflows.

Announce the choice, then load and follow the matching recipe:

- [Classify-And-Act](./patterns/classify-and-act.md)
- [Fanout-And-Synthesize](./patterns/fanout-and-synthesize.md)
- [Adversarial Verification](./patterns/adversarial-verification.md)
- [Generate-And-Filter](./patterns/generate-and-filter.md)
- [Tournament](./patterns/tournament.md)
- [Loop-Until-Done](./patterns/loop-until-done.md)

## Step 2 — Author & run the workflow

Write a JS script and let the native runtime execute it (visible in `/workflows`). Save under `.claude/workflows/` (project) or `~/.claude/workflows/` (personal).

**Skeleton — the real, current runtime API** (mirrors working scripts in `~/.claude/.../workflows/scripts/`):

```js
export const meta = {
  name: 'orchestrate-<slug>',
  description: '<one line>',
  phases: [{ title: 'Fanout' }, { title: 'Synthesize' }], // every phase() title must be declared here
}

const CTX = `<shared context every agent needs: repo paths, intent, scope, hard rules>`

const ITEM = {                                  // JSON Schema → agents return a validated object
  type: 'object', additionalProperties: false,
  required: ['unit', 'findings'],
  properties: {
    unit: { type: 'string' },
    findings: { type: 'array', items: { type: 'string' } },
  },
}

phase('Fanout')
const parts = await parallel(UNITS.map(u => () =>
  agent(`${CTX}\n\nProcess ONLY ${u}. Return findings.`,
    { label: u, phase: 'Fanout', schema: ITEM, agentType: 'Explore' })
))

phase('Synthesize')
const merged = parts.filter(Boolean)
return merged   // ONE consolidated result
```

**Contract (every workflow):**
- **Primitives.** `agent(prompt, opts)` → one isolated agent; returns the `schema`-validated object (or text). `parallel(thunks)` → runs all, **barrier**, failures come back as `null` → always `.filter(Boolean)`. `pipeline(items, ...stages)` → **streamed** stages, `stageN(prevResult, originalItem, index)`. `phase(title)` and `log(msg)` drive the `/workflows` view.
- **opts** = `{ label, phase, schema, agentType }`. Use `agentType: 'Explore'` for read-only analysis/verification. Pass a JSON Schema as `schema` (`additionalProperties: false` + `required`) to get a clean object back — no parsing, auto-retry on mismatch.
- **Phases.** Every `phase('X')` title MUST appear in `meta.phases`.
- **Opus 4.8 / xhigh.** Agents inherit the launching session. Launch from `/orchestrator:orchestrate` (frontmatter pins `claude-opus-4-8` / `effort: xhigh`) or run `/effort ultracode` first. Effort is not a per-agent script field.
- **Concurrency.** The runtime caps ~16 concurrent agents. For larger fan-outs, split into waves (two `parallel`/`pipeline` calls).
- **Deterministic control flow.** Keep the orchestration script deterministic — no `Date.now()` / `Math.random()` / I/O in the control flow (the runtime may reject them). Put randomness, file reads and shell work inside the agents, not the script.
- **One synthesis out.** The top-level `return` is the single consolidated result — dedupe and rank first; never dump raw agent outputs.
