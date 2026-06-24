# The living-plan process

The orchestrator has ONE mode: run the task as a **self-updating plan**. A background **planner** agent (the manager / single writer) decomposes the task, spawns waves of role sub-agents, adapts the plan as results come in, and loops until done. This is the homemade `/workflows` — a `PLAN.md` you can tail. No pattern to pick, no JS workflow to author.

## State (on disk, watchable)

`.orchestrator/PLAN.md` — current snapshot (open / refresh it):
```md
# Living Plan — round 4 · done:false · agents 11/30 · depth 2/3 · fails 0/3
- [x] 1   analyst   "map auth module"             done
- [~] 2   worker    "extract token validation"    running
- [ ] 3   worker    "patch refresh flow" (deps:2)  pending
- [x] 4   critic ⓢ  "verify findings"             done       ← ⓢ = supervised (gates its children)
      - [x] 4.1 analyst "re-check edge cases"      done
- [ ] 5   worker    "lint sweep"                  pending    (unsupervised)
```
Glyphs: `[x]` done · `[~]` running · `[ ]` pending · `[!]` failed · `[/]` blocked/skipped. Indent supervised children under their parent.

`.orchestrator/events.log` — append-only stream (tail this):
```
2026-06-24T14:03:11Z round 4  spawn  2,3,5
2026-06-24T14:05:02Z round 4  done   2 → token validator extracted
2026-06-24T14:05:02Z round 4  +1     4.1 (from 4)
```
**Watch it live:** `Get-Content .orchestrator/events.log -Wait` (PowerShell `tail -f`). `.orchestrator/` is gitignored.

## Task & delta shapes

Task fields: `id` (hierarchical, deterministic — `3`, `3.1`), `goal`, `agentType` (analyst|critic|worker), `supervised` (bool), `parent`, `deps[]`, `status` (pending|running|done|failed|blocked|skipped), `result`, `depth`.

Each spawned agent ends its report with ONE block (the planner parses + applies it):
```
### PLAN-DELTA (task: <my-id>)
status: done|failed|blocked
result: <one line>
add:                       # optional — new tasks this agent discovered
  - goal: <imperative>
    agentType: analyst|critic|worker
    supervised: true|false
    deps: [<ids>]
notes: <optional, for the planner>
```

## Supervised vs unsupervised

- **Supervised** — the parent WAITS for the child, reviews its delta, may retry/reject, and aggregates it before re-planning. A supervised task may itself spawn supervised children (the recursion, bounded by `maxDepth`). Use whenever downstream tasks depend on the result.
- **Unsupervised** — fire-and-forget: spawned, result appended, nothing gates it. Only for independent leaf tasks whose output nothing else needs. *(Cognition, "Don't Build Multi-Agents": ungated parallel agents fragment context — prefer supervised by default.)*

## Moves the planner can use

Within the plan, the planner picks whatever structure each round needs — these are tactics, not separate modes:
- **fan out** — one analyst per independent unit, then merge;
- **verify** — a critic per finding, told to refute it; keep the survivors;
- **generate + filter** — many candidates, then a critic keeps the best few;
- **tournament** — pairwise critic judging for the single best output;
- **classify** — an analyst labels the input, then route to the matching follow-up;
- **iterate** — keep discovering until two empty rounds.

Compose them across rounds as the task demands.

## The reconciliation loop (the planner runs this)

1. **Plan** — decompose the objective into the initial `PLAN.md`.
2. **Select** — runnable = `pending` ∧ all `deps` done ∧ within caps; sort by id.
3. **Spawn a wave** — one parallel batch of role agents. Mark them `running`, rewrite `PLAN.md`, append `events.log`.
4. **Reduce** — collect each agent's `PLAN-DELTA`; apply one at a time (assign child ids `parent.n`, dedupe by `agentType+goal`, reject dep-cycles, drop adds past caps). Set producing tasks `done`/`failed`. Rewrite the files.
5. **Re-plan** — revise the plan from the round's results (add/close tasks) or set `done`.
6. Repeat until **termination**.

## Caps & termination

`maxRounds` (~6) · `maxDepth` (~2) · `maxAgents` total (~30, budget) · dedupe (fingerprint = `agentType+goal`) · dep-cycle rejection · error budget (a task whose dep `failed` → `blocked`; stop if `fails > N`).
Stop on: **quiescence** (no runnable & none running) · **objective met** · any cap. Always record `terminationReason` in `PLAN.md` and the final return.

**Teardown (at termination, before synthesis):** `git worktree list` → append a `## Teardown` section to `PLAN.md` listing each worker worktree + branch + whether it has unmerged work; `git worktree remove` only the empty/merged ones, never discard unmerged work. `.orchestrator/` stays on disk (user deletes when done).

**One synthesis out:** the planner's final return is the consolidated result read from the finished plan — never the raw deltas.
