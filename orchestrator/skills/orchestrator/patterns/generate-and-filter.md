# Generate-And-Filter

**Use when:** you want many candidates (noisy is fine) then the best few — naming, brainstorming, refactor proposals.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton. `GENERATORS` = a few distinct angles.)*

```js
const LIST = {
  type: 'object', additionalProperties: false,
  required: ['candidates'],
  properties: { candidates: { type: 'array', items: { type: 'string' } } },
}

phase('Generate')
const batches = await parallel(GENERATORS.map((g, i) => () =>
  agent(`${CTX}\n\nProduce ~15 candidates. Angle: ${g}. Quantity over polish.`,
    { label: `gen:${i}`, phase: 'Generate', schema: LIST, agentType: 'orchestrator-analyst' })
)).then(r => r.filter(Boolean))
const pool = [...new Set(batches.flatMap(b => b.candidates))]

phase('Filter')
const TOP = {
  type: 'object', additionalProperties: false,
  required: ['top'],
  properties: {
    top: {
      type: 'array',
      items: {
        type: 'object', additionalProperties: false,
        required: ['candidate', 'why'],
        properties: { candidate: { type: 'string' }, why: { type: 'string' } },
      },
    },
  },
}
const best = await agent(`${CTX}\n\nDedupe, then pick the top-k against this rubric: <rubric>.\nCandidates:\n${pool.join('\n')}`,
  { label: 'filter', phase: 'Filter', schema: TOP, agentType: 'orchestrator-critic' })
return best
```

- One generation pass is cheap; the **filter** is the quality gate (rubric + dedup), not the generators.
