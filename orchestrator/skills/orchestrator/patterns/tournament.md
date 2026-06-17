# Tournament

**Use when:** you want THE single best output and quality is subjective (copy, design, naming, an implementation). Pairwise comparison beats absolute scoring.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton. `ANGLES` = 3–5 distinct approaches.)*

```js
phase('Attempts')
const cands = await parallel(ANGLES.map((a, i) => () =>
  agent(`${CTX}\n\nProduce the best <artifact>. Approach: ${a}`, { label: `cand:${i}`, phase: 'Attempts' })
)).then(r => r.filter(Boolean))

phase('Judge')
const PICK = {
  type: 'object', additionalProperties: false,
  required: ['winner', 'reason'],
  properties: { winner: { type: 'string', enum: ['A', 'B'] }, reason: { type: 'string' } },
}
let pool = cands
while (pool.length > 1) {
  const pairs = []
  for (let i = 0; i < pool.length; i += 2) { pairs.push(pool.slice(i, i + 2)) }
  pool = await parallel(pairs.map(p => () =>
    p.length < 2
      ? Promise.resolve(p[0])
      : agent(`${CTX}\n\nWhich is better against this rubric: <rubric>?\n\n[A]\n${p[0]}\n\n[B]\n${p[1]}`,
          { label: 'judge', phase: 'Judge', schema: PICK }).then(v => (v && v.winner === 'B') ? p[1] : p[0])
  )).then(r => r.filter(Boolean))
}
return pool[0]
```

- Vary the **angle** per attempt (the model is fixed = Opus 4.8); pairwise "A vs B" is more reliable than self-scoring.
