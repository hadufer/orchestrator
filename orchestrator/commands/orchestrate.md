---
description: Run a task through the right multi-agent orchestration pattern (auto-selected).
argument-hint: <task to orchestrate>
model: claude-opus-4-8
effort: xhigh
---

Invoke the `orchestrator:orchestrator` skill to handle this task end-to-end:

$ARGUMENTS

Follow the skill's triage gate FIRST: if the task does not warrant a workflow, do it directly and say so. Otherwise author and run a native dynamic workflow (watch it in `/workflows`).
