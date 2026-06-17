---
description: Run a task through the right multi-agent orchestration pattern (auto-selected).
argument-hint: <task to orchestrate>
model: claude-opus-4-8
effort: xhigh
---

Invoke the `orchestrator:orchestrator` skill to handle this task end-to-end:

$ARGUMENTS

ALWAYS run the orchestrator's full process end-to-end for this task — no shortcut to a direct answer, even for small tasks. Use the native workflow runtime if available (watch it in `/workflows`); otherwise fall back to in-conversation subagents. The skill explains both modes.
