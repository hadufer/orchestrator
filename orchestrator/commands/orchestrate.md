---
description: Run a task through the right multi-agent orchestration pattern (auto-selected).
argument-hint: <task to orchestrate>
model: claude-opus-4-8
effort: xhigh
---

Invoke the `orchestrator:orchestrator` skill to handle this task end-to-end:

$ARGUMENTS

ALWAYS run the orchestrator's full process end-to-end and author + run a native dynamic workflow for this task (watch it in `/workflows`). No shortcut to a direct answer, even for small tasks.
