---
description: Run a task as a self-updating living plan driven by a background planner agent.
argument-hint: <task to orchestrate>
model: claude-opus-4-8
effort: xhigh
---

Invoke the `orchestrator:orchestrator` skill to handle this task end-to-end:

$ARGUMENTS

Gate first: if the task seems ill-posed, built on a false premise, or trivially doable without orchestration, say so and ask for confirmation before spawning the planner — don't run 30 Opus agents at the wrong target. Otherwise proceed.

Once past the gate, ALWAYS run the orchestrator's living-plan process for this task — no shortcut to a direct answer, even for small tasks: hand it to the planner-manager (spawned in the background), which drives the plan to completion and returns one consolidated result. The skill explains the mechanism.
