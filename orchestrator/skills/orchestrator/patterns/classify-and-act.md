# Classify-And-Act

**Use when:** the task is "handle X" and X comes in distinct types needing different treatment, or you must triage before acting.

*(Assumes the `meta` + `CTX` setup from the skill's skeleton. Define the CLOSED label set in the schema enum.)*

```js
const LABEL = {
  type: 'object', additionalProperties: false,
  required: ['class', 'reason'],
  properties: {
    class: { type: 'string', enum: ['typeA', 'typeB', 'typeC'] },
    reason: { type: 'string' },
  },
}

phase('Classify')
const { class: cls } = await agent(`${CTX}\n\nClassify into exactly one label.`,
  { label: 'classify', phase: 'Classify', schema: LABEL, agentType: 'orchestrator:orchestrator-analyst' })

phase('Act')
const handlers = {
  typeA: `<instructions for A>`,
  typeB: `<instructions for B>`,
  typeC: `<instructions for C>`,
}
const result = await agent(`${CTX}\n\nHandle as ${cls}:\n${handlers[cls]}`, { label: `act:${cls}`, phase: 'Act', agentType: 'orchestrator:orchestrator-worker' })
return { class: cls, result }
```

- **Compose:** a class may dispatch into another pattern (e.g. `audit` → call the Fanout-And-Synthesize phases instead of a single `agent`).
