# Fanout-And-Synthesize

**Use when:** the task splits into N independent units processed without seeing each other, then merged.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton; below is the phase logic.)*

```js
const UNIT = {
  type: 'object', additionalProperties: false,
  required: ['unit', 'findings'],
  properties: {
    unit: { type: 'string' },
    findings: { type: 'array', items: { type: 'string' } },
  },
}

phase('Fanout')
const parts = await parallel(UNITS.map(u => () =>
  agent(`${CTX}\n\nProcess ONLY ${u}. Return your findings; empty array if none.`,
    { label: u, phase: 'Fanout', schema: UNIT, agentType: 'orchestrator:orchestrator-analyst' })
)).then(r => r.filter(Boolean))

phase('Synthesize')
// Small volume: merge here. Large volume: hand the JSON to one synthesizer agent.
const all = parts.flatMap(p => p.findings.map(f => ({ unit: p.unit, finding: f })))
const report = await agent(`${CTX}\n\nMerge into one deduped, grouped report:\n${JSON.stringify(all)}`,
  { label: 'synthesize', phase: 'Synthesize' })
return report
```

- **Cap:** >16 units → split `UNITS` into waves (two `parallel` calls), then concat.
- Each agent sees ONLY its unit — that isolation is the point.
