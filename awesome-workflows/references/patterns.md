# Workflow Patterns Reference

## Loop-until-dry

For unknown-size discovery — keep spawning until K consecutive rounds return nothing new:

```javascript
const seen = new Set()
let dry = 0
while (dry < 2) {
  const found = await parallel(FINDERS.map(f => () =>
    agent(f.prompt, { model: 'haiku', schema: BUGS_SCHEMA })
  ))
  const fresh = found.filter(Boolean).flatMap(r => r.bugs)
    .filter(b => !seen.has(key(b)))
  if (!fresh.length) { dry++; continue }
  dry = 0
  fresh.forEach(b => seen.add(key(b)))
}
// Dedup against `seen`, NOT confirmed — else rejected bugs reappear every round
```

## Adversarial verify (3-vote)

Spawn N independent skeptics per finding; kill if majority refute:

```javascript
const votes = await parallel(Array.from({ length: 3 }, () => () =>
  agent(`Try to REFUTE this claim: ${claim}. Default to refuted=true if uncertain.`,
    { model: 'haiku', schema: VERDICT_SCHEMA })
))
const survives = votes.filter(Boolean).filter(v => !v.refuted).length >= 2
```

Always keep 3 voters. Don't reduce to 1 to save cost — adversarial voting requires the quorum.

## Perspective-diverse verify

When a finding can fail in multiple ways, give each verifier a distinct lens:

```javascript
const lenses = ['correctness', 'security', 'performance']
const judgments = await parallel(lenses.map(lens => () =>
  agent(`Evaluate "${finding}" from the ${lens} perspective.`,
    { model: 'haiku', schema: VERDICT_SCHEMA })
))
// Catches failure modes that redundancy can't
```

## Judge panel

Generate N independent approaches, score, synthesize from winner:

```javascript
const approaches = await parallel(
  ['MVP-first', 'risk-first', 'user-first'].map(angle => () =>
    agent(`Design the solution from a ${angle} perspective`, { schema: DESIGN_SCHEMA })
  )
)
const scored = await agent(`Score these 3 approaches and pick the best:\n${JSON.stringify(approaches)}`,
  { schema: SCORE_SCHEMA })
const winner = approaches[scored.winner_index]
// Graft best ideas from runners-up onto winner
```

## Multi-modal sweep

Parallel agents each searching a different way — each blind to what others find:

```javascript
const STRATEGIES = [
  { label: 'by-file', prompt: 'Search by filename pattern...' },
  { label: 'by-content', prompt: 'Search by code content...' },
  { label: 'by-dependency', prompt: 'Search by import graph...' },
]
const allFindings = await parallel(STRATEGIES.map(s => () =>
  agent(s.prompt, { label: s.label, model: 'haiku', schema: FINDINGS_SCHEMA })
))
```

## Budget-aware scaling

```javascript
const MAX_PER_ROUND = budget.total ? Math.floor(budget.remaining() / 50_000) : 5
const items = candidates.slice(0, MAX_PER_ROUND)
```

## Completeness critic

Final agent asks "what did we miss?":

```javascript
const critic = await agent(
  `Here are all findings so far: ${JSON.stringify(allFindings)}.
   What sources weren't checked? What claim is unverified? What angle wasn't tried?`,
  { schema: GAPS_SCHEMA }
)
// Critic's output becomes next round of work
```

## Handling nulls from parallel/pipeline

```javascript
const results = await parallel(items.map(i => () => agent(prompt(i), { model: 'haiku' })))
// Filter nulls before using — agent returns null on terminal error or user skip
const valid = results.filter(Boolean)
```

## Phase inside pipeline — avoid race conditions

```javascript
// ❌ Race: global phase() races with pipeline stages
phase('Analyze')
const results = await pipeline(items,
  item => agent(prompt, { phase: 'Analyze' })  // ← use phase: in opts, not global phase()
)

// ✅ Correct: set phase in agent opts
const results = await pipeline(items,
  item => agent(prompt, { label: `analyze:${item}`, phase: 'Analyze', model: 'haiku', schema: SCHEMA })
)
```

## Workflow meta block (required)

```javascript
export const meta = {
  name: 'my-workflow',           // kebab-case, shown in /workflows
  description: 'One-line summary',
  phases: [
    { title: 'Find',   detail: 'Haiku finds all X' },
    { title: 'Verify', detail: 'Adversarial 3-vote per finding' },
    { title: 'Report', detail: 'Sonnet synthesizes' },
  ],
}
// meta must be a PURE LITERAL — no variables, spreads, or function calls
```
