# orchestrator

Local Claude Code plugin: runs any task as a **self-updating living plan**. You give it a task; a background **planner** agent decomposes it, spawns waves of role subagents, adapts the plan as results come in, and loops until done — a homemade `/workflows` you can watch on disk. No dependencies, no native runtime needed.

## Install (local dev)

From the folder that contains `orchestrator/`:

```bash
claude --plugin-dir ./orchestrator
```

Already in a session? Run `/reload-plugins`.

## Use

```
/orchestrator:orchestrate <your task>
```

That's the whole interface: give it a task. It spawns the planner-manager in the background, which owns `.orchestrator/runs/<run-id>/PLAN.md` + `events.log` and drives the loop, then returns one consolidated result.

**Watch it live** (the run-id is printed when you launch):
```powershell
Get-Content .orchestrator/runs/<run-id>/events.log -Wait
```
and open `.orchestrator/runs/<run-id>/PLAN.md` for the current task tree. (`.orchestrator/` is gitignored; each run gets its own `runs/<run-id>/`, so parallel runs in one repo don't collide.)

## Agents

The plugin ships five role subagents — you do not create them, they load with the plugin:

| `subagent_type` | Role | Profile |
|---|---|---|
| `orchestrator:orchestrator-planner` | manager — decompose / re-plan / drive the loop; writes only `.orchestrator/` | Opus 4.8 · xhigh · no worktree |
| `orchestrator:orchestrator-analyst` | read-only — find / classify / research / discover / generate | Opus 4.8 · xhigh · no Edit/Write |
| `orchestrator:orchestrator-critic` | read-only adversarial — refute / filter / judge / run the check | Opus 4.8 · xhigh · no Edit/Write |
| `orchestrator:orchestrator-debugger` | read-only — root-cause a failing check, name the fix | Opus 4.8 · xhigh · no Edit/Write |
| `orchestrator:orchestrator-worker` | read-write — act / build / refactor | Opus 4.8 · xhigh · assigned `<repo>-wt-<run-id>` worktree |

**Setup: after installing or updating, run `/reload-plugins`** so the agents register as subagent types. Confirm with `/agents`.

## How it works

One mode: the planner maintains a living plan and reconciles it each round as a **build → check** loop — select runnable tasks → spawn a wave → a critic runs the objective's check and reports how many criteria are still `open` → the planner (single writer) applies the `PLAN-DELTA`s → re-plan → loop **while the check keeps making progress**. The number of rounds is decided by the checking, not a fixed cap (a high cap is only a backstop). Sub-agents are **supervised** (gated + aggregated by a parent, may recurse) or **unsupervised** (fire-and-forget). Within the plan the planner can fan out, verify with critics, **diagnose a failing check with a debugger then fix**, generate-and-filter, run a tournament, or iterate — as tactics, not separate modes. For code changes it auto-detects which repo(s) a task touches and works in per-repo `<repo>-wt-<run-id>` worktrees on `feature/<run-id>` branches (never `master`/dev) — so each run lands as ready-to-PR branches and parallel runs stay fully isolated. See `skills/orchestrator/patterns/plan-and-adapt.md`.

## Cost

Every agent runs on **Opus 4.8 at `xhigh` effort** — capability over token economy. A run spends many times the tokens of a normal turn (many Opus agents). Use it for work that genuinely needs decomposition, parallelism, isolation, or adversarial checking — not for two-line fixes.
