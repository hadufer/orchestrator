# orchestrator

Local Claude Code plugin: routes a task to the right pattern and **authors a native Claude Code dynamic workflow** — a JS script the built-in `/workflows` runtime runs in the background. No dependencies, no external services; it uses the native workflow runtime. Reproduces the six "dynamic workflow" patterns locally, with full `/workflows` monitoring.

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

Or just ask Claude to "orchestrate" / "run a workflow" — the skill auto-triggers.

The router ALWAYS runs the full process (no shortcut, even for small tasks): it picks one of six patterns — Classify-And-Act, Fanout-And-Synthesize, Adversarial Verification, Generate-And-Filter, Tournament, Loop-Until-Done — or composes several, and runs it as a workflow.

It then writes a workflow script (under `.claude/workflows/`) and runs it on the native runtime. Watch phases, agents and token usage live with `/workflows` (`p` pause, `x` stop, `s` save); the consolidated result lands back in the conversation.

## Cost

Every agent runs on **Opus 4.8 at `xhigh` effort** (the launching session, pinned by the command frontmatter; equivalently `/effort ultracode`) — capability over token economy. Workflows therefore spend MANY times the tokens of a normal turn (many Opus agents instead of one). Use them for work that genuinely needs parallelism, isolation, or adversarial checking — not for two-line fixes.
