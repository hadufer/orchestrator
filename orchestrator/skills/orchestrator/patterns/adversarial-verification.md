# Adversarial Verification

**Use when:** you have findings/claims and must keep only those that survive a blind refuter. Separating producer from checker removes self-preferential bias.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton. `findings` come from a prior phase — e.g. a Fanout — or are generated here first.)*

```js
const VERDICT = {
  type: 'object', additionalProperties: false,
  required: ['verdict', 'evidence'],
  properties: {
    verdict: { type: 'string', enum: ['holds', 'refuted'] },
    evidence: { type: 'string', description: 'quote the line that proves it' },
  },
}

phase('Verify')
const checked = await parallel(findings.map(f => () =>
  agent(`${CTX}\n\nTry to PROVE this is FALSE/invalid. Read the actual source. Judge ONLY the finding below — ignore how it was produced.\n${JSON.stringify(f)}`,
    { label: `v:${f.id ?? f.title}`, phase: 'Verify', schema: VERDICT, agentType: 'orchestrator-critic' })
    .then(v => (v && v.verdict !== 'refuted') ? { ...f, evidence: v.evidence } : null)
))
const survivors = checked.filter(Boolean)
return { survivors, refuted: findings.length - survivors.length }
```

- One refuter **per finding**, in parallel — each blind to the others and to the producer's reasoning.
- **Compose:** this is exactly the verify phase of a code-review workflow; feed it `findings` from a Fanout.
