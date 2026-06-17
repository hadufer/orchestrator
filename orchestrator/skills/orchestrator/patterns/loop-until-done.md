# Loop-Until-Done

**Use when:** open-ended discovery, unknown number of rounds — "find ALL the bugs", exhaustive sweep. Stop on results, not a fixed count.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton. Declare one phase, e.g. `phases: [{ title: 'Discover' }]`, and `log` each round — the runtime allows no mid-run input, so the stop logic must be self-contained.)*

```js
const NEW = {
  type: 'object', additionalProperties: false,
  required: ['items'],
  properties: { items: { type: 'array', items: { type: 'string' } } },
}

const found = []
let empty = 0
for (let round = 1; round <= 6 && empty < 2; round++) {
  phase('Discover')
  const r = await agent(`${CTX}\n\nFind NEW items NOT already in this list (do not repeat):\n${found.join('\n') || '(none yet)'}`,
    { label: `round:${round}`, phase: 'Discover', schema: NEW, agentType: 'orchestrator-analyst' })
  const fresh = (r?.items ?? []).filter(x => !found.includes(x))
  found.push(...fresh)
  empty = fresh.length === 0 ? empty + 1 : 0
  log(`Round ${round}: +${fresh.length} (total ${found.length})`)
}
return { found, stoppedBy: empty >= 2 ? 'two empty rounds' : 'safety cap (6)' }
```

- Stop after **2 consecutive empty rounds** OR the safety cap, whichever first — and say which fired.
- For very long sweeps, each round can itself be a `parallel` fan-out instead of a single agent.
