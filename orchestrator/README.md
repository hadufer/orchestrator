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

That's the whole interface: give it a task. It spawns the planner-manager in the background, which owns `.orchestrator/PLAN.md` + `events.log` and drives the loop, then returns one consolidated result.

**Watch it live:**
```powershell
Get-Content .orchestrator/events.log -Wait
```
and open `.orchestrator/PLAN.md` for the current task tree. (`.orchestrator/` is gitignored.)

## Agents

The plugin ships four role subagents — you do not create them, they load with the plugin:

| `subagent_type` | Role | Profile |
|---|---|---|
| `orchestrator:orchestrator-planner` | manager — decompose / re-plan / drive the loop; writes only `.orchestrator/` | Opus 4.8 · xhigh · no worktree |
| `orchestrator:orchestrator-analyst` | read-only — find / classify / research / discover / generate | Opus 4.8 · xhigh · no Edit/Write |
| `orchestrator:orchestrator-critic` | read-only adversarial — refute / filter / judge / verify | Opus 4.8 · xhigh · no Edit/Write |
| `orchestrator:orchestrator-worker` | read-write — act / build / refactor | Opus 4.8 · xhigh · isolated worktree |

**Setup: after installing or updating, run `/reload-plugins`** so the agents register as subagent types. Confirm with `/agents`.

## How it works

One mode: the planner maintains a living plan and reconciles it each round — select runnable tasks → spawn a wave → agents return `PLAN-DELTA`s → the planner (single writer) applies them → re-plan → repeat until quiescence or a cap. Sub-agents are **supervised** (gated + aggregated by a parent, may recurse) or **unsupervised** (fire-and-forget). Within the plan the planner can fan out, verify with critics, generate-and-filter, run a tournament, or iterate — as tactics, not separate modes. See `skills/orchestrator/patterns/plan-and-adapt.md`.

## Cost

Every agent runs on **Opus 4.8 at `xhigh` effort** — capability over token economy. A run spends many times the tokens of a normal turn (many Opus agents). Use it for work that genuinely needs decomposition, parallelism, isolation, or adversarial checking — not for two-line fixes.
