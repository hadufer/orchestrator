# The living-plan process

The orchestrator has ONE mode: run the task as a **self-updating plan**. A background **planner** agent (the manager / single writer) decomposes the task, spawns waves of role sub-agents, adapts the plan as results come in, and loops until done. This is the homemade `/workflows` — a `PLAN.md` you can tail. No pattern to pick, no JS workflow to author.

The loop is **build → check**, and **the check decides how many rounds run** — not a fixed count. Each round does work, then a *separate* critic verifies it and reports how many checks/criteria are still `open`. You loop while that number keeps falling and stop when it hits zero (converged) or stops moving (stalled); a round/agent cap is only a backstop. *(Karpathy's verifiability thesis: an autonomous loop is only as good as its verifier — loop until verified, not for N turns.)*

## State (on disk, watchable)

`.orchestrator/runs/<run-id>/PLAN.md` — current snapshot (open / refresh it):
```md
# Living Plan — round 4 · done:false · open:3↓ · agents 11/30 · depth 2/3 · fails 0/3
Check: `npm test && npm run lint` (hook) · open 5→3 over rounds 3→4   ← the loop's control signal
- [x] 1   analyst    "map auth module"             done
- [x] 2   worker     "extract token validation"    done
- [x] 3   critic ⓢ   "run check"                   done   open:3 verdict:progressing
- [~] 4   debugger   "root-cause 3 failing tests"  running
- [ ] 5   worker     "apply fix" (deps:4)          pending
- [ ] 6   critic     "re-run check" (deps:5)       pending
```
Glyphs: `[x]` done · `[~]` running · `[ ]` pending · `[!]` failed · `[/]` blocked/skipped. `ⓢ` = supervised (gates its children); indent children under their parent. The header's `open:N` (with its ↓/↑ arrow) is the progress signal — it must trend ↓, or the loop is stalling.

`.orchestrator/runs/<run-id>/events.log` — append-only stream (tail this):
```
2026-06-24T14:03:11Z round 4  spawn  2,3,5
2026-06-24T14:05:02Z round 4  done   2 → token validator extracted
2026-06-24T14:05:02Z round 4  +1     4.1 (from 4)
```
**Watch it live:** `Get-Content .orchestrator/runs/<run-id>/events.log -Wait` (PowerShell `tail -f`). `.orchestrator/` is gitignored. Each run owns a separate `runs/<run-id>/` dir, so several orchestrations can run in the same repo without colliding.

## Task & delta shapes

Task fields: `id` (hierarchical, deterministic — `3`, `3.1`), `goal`, `agentType` (analyst|critic|debugger|worker), `supervised` (bool), `parent`, `deps[]`, `status` (pending|running|done|failed|blocked|skipped), `result`, `depth`. A `worker` task also carries `repo` + `worktree` — the `<repo>-wt-<run-id>` dir it must edit in.

Plan-level: the **`Check:`** line is the objective's success test, set at decompose time. `checkKind: hook` — a concrete command/test/lint/typecheck/build or observable criterion — when the task is verifiable; else `checkKind: critic` — an explicit written rubric an adversarial critic judges against. The check is always run by a critic, never by the agent that produced the work.

Each spawned agent ends its report with ONE block (the planner parses + applies it):
```
### PLAN-DELTA (task: <my-id>)
status: done|failed|blocked
result: <one line>
add:                       # optional — new tasks this agent discovered
  - goal: <imperative>
    agentType: analyst|critic|debugger|worker
    supervised: true|false
    deps: [<ids>]
check:                     # optional — ONLY when this agent ran THE check; this is the loop's control signal
  open: <n>                # failing checks / unmet criteria still open
  verdict: converged|progressing|stalled|regressed   # vs the prior round's open count
  items: [<one line per open issue>]
notes: <optional, for the planner>
```

## Supervised vs unsupervised

- **Supervised** — the parent WAITS for the child, reviews its delta, may retry/reject, and aggregates it before re-planning. A supervised task may itself spawn supervised children (the recursion, bounded by `maxDepth`). Use whenever downstream tasks depend on the result.
- **Unsupervised** — fire-and-forget: spawned, result appended, nothing gates it. Only for independent leaf tasks whose output nothing else needs. *(Cognition, "Don't Build Multi-Agents": ungated parallel agents fragment context — prefer supervised by default.)*

## Worktrees & multi-repo

A run is identified by a **ticket / run-id** (e.g. `MYK4-XXXX`). All code work happens in **per-repo worktrees** on a per-run branch — never the main working tree, never `master`/dev:

- **Repo landscape.** The cwd is either a single git repo or a *container of repos* — sub-dirs each with their own `.git`, often grouped (`*.back/<service>`, `*.front/<app>`, `*.nuget/<lib>`). The planner maps it at round 0.
- **Detect, then surface.** A read-only analyst wave finds which repo(s) the task touches; the chosen set goes into a `## Repos` section of `PLAN.md` **before any edit** — the user's abort gate against a wrong guess.
- **One worktree per involved repo:** `git -C <repo> worktree add -b feature/<run-id> <repo>-wt-<run-id> <base>` (base = the repo's dev branch, e.g. `feature/development`); worktrees are siblings of their repo, matching the house convention.
- **Workers are routed:** each `worker` is told its `<repo>-wt-<run-id>` dir and edits/commits ONLY there. Read-only roles (analyst/critic/debugger) need no worktree.
- **Integration = leave on the branch.** A run's deliverable is its `feature/<run-id>` branch(es); the planner NEVER merges to `master`/dev. The synthesis lists `(repo, branch, worktree)` for the user's PR flow. Two parallel runs (different tickets) use different branches + worktrees → fully isolated, state *and* git history.
- **Cross-repo checks (when repos share a contract).** If one involved repo *produces* what another *consumes* (an API field, DTO, shared type), per-repo green is not enough — add a **cross-repo check** that exercises the involved worktrees *together* (e.g. run the consumer's integration test with the producer's worktree on its path, or a critic that compares the contract across worktrees). `open` aggregates the per-repo checks **and** the cross-repo check; the run is `success` only when all are green.
- **Coordinated fixes.** A change in one repo that breaks the cross-repo check keeps the loop running: a `debugger`/`analyst` localizes which side must adapt to restore the contract, and the planner dispatches a `worker` to that repo's worktree — repeating until producer and consumer agree.

## Moves the planner can use

Within the plan, the planner picks whatever structure each round needs — these are tactics, not separate modes:
- **fan out** — one analyst per independent unit, then merge;
- **verify** — a critic per finding, told to refute it; keep the survivors;
- **check** — after a build wave, ONE critic runs the objective's check (hook or rubric) and emits `open`/`verdict`; this is what decides whether to loop again;
- **diagnose → fix** — on a failing check, a `debugger` finds the root cause + fix location, then a `worker` applies the targeted fix (don't re-build blind);
- **generate + filter** — many candidates, then a critic keeps the best few;
- **tournament** — pairwise critic judging for the single best output;
- **classify** — an analyst labels the input, then route to the matching follow-up;
- **iterate** — loop build→check while `open` keeps falling; stop when it hits 0 or stalls (see termination).

Compose them across rounds as the task demands.

## The reconciliation loop (the planner runs this)

1. **Plan** — decompose the objective into the initial `PLAN.md`.
2. **Select** — runnable = `pending` ∧ all `deps` done ∧ within caps; sort by id.
3. **Spawn a wave** — one parallel batch of role agents. Mark them `running`, rewrite `PLAN.md`, append `events.log`.
4. **Reduce** — collect each agent's `PLAN-DELTA`; apply one at a time (assign child ids `parent.n`, dedupe by `agentType+goal`, reject dep-cycles, drop adds past caps). Set producing tasks `done`/`failed`. Rewrite the files.
5. **Re-plan** — revise the plan from the round's results; read the check's `open`/`verdict` and queue the next build→check (or `diagnose→fix` on failures), or set `done`.
6. Repeat **while the check still makes progress**; stop per **termination** below.

## Termination — the check decides, not a counter

The loop runs **as long as it's productive**; the round count is an output of the checking. Stop on the FIRST of these and always record which as `terminationReason` (in `PLAN.md` and the final return):

- **success** — `open == 0` / verdict `converged`: the verify hook is green, or the critic cannot refute that the criteria are met. The goal of the loop.
- **stagnation** — `open` failed to fall for **N consecutive rounds** (default N=2). Motion without progress; stop and report what's still open.
- **regression** — `open` rose above the best round so far and didn't recover the next round → stop and keep the best round's state; don't ship the regression.
- **oscillation** — the same fix/error fingerprint is cycling (A→B→A); the `debugger` flags it → stop or switch strategy rather than feed another round.
- **budget** *(backstop only — not the intended stop)* — a HIGH safety ceiling so a runaway can't burn unbounded Opus tokens: `maxAgents` (~30) · `maxRounds` (~30 ceiling, not a target) · optional wall-clock. Hitting these means the loop failed to converge — say so.
- **escalate** — diverging with no path forward (can't localize, contradictory checks, ambiguous success) → stop and put the question to the user. Asking is a first-class outcome, not a failure.

Always-on guards: `maxDepth` (~2) · dedupe (fingerprint = `agentType+goal`) · dep-cycle rejection · a task whose dep `failed` → `blocked`.
**The check is the loop controller:** never stop on a round count while `open` is still falling, and never keep looping once it's converged or stalled.

**Teardown (at termination, before synthesis):** the `<repo>-wt-<run-id>` worktrees **hold the deliverable** (committed on `feature/<run-id>`) — keep them. Append a `## Teardown` section listing each `(repo, worktree, branch)` + its commit/unmerged state for the user to PR; `git worktree remove` only genuinely empty/failed ones. Your `runs/<run-id>/` dir stays on disk (delete with `rm -rf .orchestrator/runs/<run-id>` when done — not `rm -rf .orchestrator`, a parallel run may need it).

**One synthesis out:** the planner's final return is the consolidated result read from the finished plan — including which `terminationReason` fired and anything still `open` — never the raw deltas.
