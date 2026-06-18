---
name: awesome-workflows
description: >
  ALWAYS read this skill before writing any Workflow tool script, before calling the
  Workflow() tool, or before spawning more than 3 agents. Covers the critical patterns
  that prevent wasted tokens and wrong model choices: pipeline() vs parallel(), Haiku
  for fan-out agents, never use agents just to write files (return results instead),
  schema output, phase organization, and resume. Triggers on: writing a workflow script,
  Workflow({ script: ... }), pipeline(), parallel(), agent() in a loop, "run in parallel",
  "fan-out", "orchestrate subagents", "multi-agent", "haiku for agents",
  "how do I save results from a workflow", "workflow burning tokens", "too many agents",
  "model for subagents". If you are about to write a workflow script or call Workflow()
  — stop and read this first.
compatibility: Claude Code with Workflow tool access
license: MIT
metadata:
  author: gustavo
  version: "1.0"
---

# Awesome Workflows

Workflows orchestrate tens to hundreds of subagents. The goal: fast, cheap, correct.

**Full reference:** [patterns.md](references/patterns.md) | [model-tiers.md](references/model-tiers.md)

---

## The 3 rules before you write a single line

1. **pipeline() by default.** Only reach for parallel() when you genuinely need ALL results before the next step. If in doubt, pipeline.
2. **Never use an agent just to write a file.** Return results from the workflow; write files with Write tool in the main conversation.
3. **Haiku for fan-out, Sonnet for synthesis.** N > 3 parallel instances → `model: "haiku"`.

---

## pipeline() vs parallel()

```javascript
// ✅ DEFAULT — pipeline: item A in stage 2 while item B is still in stage 1
const results = await pipeline(
  items,
  item => agent(findPrompt(item), { model: "haiku", schema: FIND_SCHEMA }),
  found => agent(verifyPrompt(found), { model: "haiku", schema: VERIFY_SCHEMA }),
  verified => agent(fixPrompt(verified))  // sonnet (default) — final output
)

// ✅ parallel() — ONLY when stage N needs ALL stage N-1 results before it can start
const allFindings = await parallel(finders.map(f => () => agent(f.prompt, { model: "haiku" })))
const deduped = dedupe(allFindings.filter(Boolean))  // ← needs ALL to deduplicate
const verified = await parallel(deduped.map(d => () => agent(verifyPrompt(d))))
```

A barrier (parallel between stages) wastes wall-clock time: if 5 finders run and the slowest
takes 3× the fastest, you pay the idle time of the 4 fast ones. Use it only when cross-item
context is truly required (dedup, count-before-proceeding, "compare all findings").

---

## Saving results — the most common anti-pattern

```javascript
// ❌ WRONG — spawns N agents at Sonnet cost just to write files
for (const [module, data] of Object.entries(allOutputs)) {
  await agent(`Write this JSON to ${OUT}/${module}.json:\n${JSON.stringify(data)}`,
    { label: `save:${module}` })
}

// ✅ CORRECT — return everything, write in main conversation with Write tool
return { summary, outputs: allOutputs }
// Then in main conversation: for each entry, use Write tool directly
```

The only time a save agent is justified: the data is too large to return as a workflow result
(>1MB), or you need complex transformation before writing.

---

## Model routing — quick table

| Role | Model | Why |
|---|---|---|
| Fan-out (search, fetch, extract, verify) | `haiku` | High volume, low reasoning |
| Scope / decompose | default (sonnet) | Shapes the whole workflow |
| Synthesis / final report | default (sonnet) | Coherent output |
| Complex audit / legal / financial | `opus` | Only when sonnet repeatedly fails |

**10× cost reduction** is typical when routing fan-out agents to Haiku.
See [model-tiers.md](references/model-tiers.md) for full cost table.

---

## Schema output — use it for structured data

```javascript
const FINDING_SCHEMA = {
  type: 'object',
  required: ['gaps', 'coverage_pct'],
  properties: {
    coverage_pct: { type: 'number' },
    gaps: { type: 'array', items: { type: 'object', required: ['event','fix'],
      properties: { event: { type: 'string' }, fix: { type: 'string' } }
    }}
  }
}

// Agent returns validated object — no parsing needed
const result = await agent(prompt, { schema: FINDING_SCHEMA, model: "haiku" })
result.gaps.forEach(...)  // safe to use directly
```

Without schema: agent returns raw text → you parse → you handle parse errors.
With schema: agent retries until output is valid → you get the object.

---

## Phase organization

```javascript
export const meta = {
  name: 'my-workflow',
  description: 'What it does',
  phases: [
    { title: 'Discover', detail: 'Find all X' },
    { title: 'Analyze',  detail: 'Haiku classifies each X' },
    { title: 'Report',   detail: 'Synthesize findings' },
  ],
}

phase('Discover')
const items = await agent('Find all modules...', { label: 'discover' })

phase('Analyze')
const results = await pipeline(items, item =>
  agent(analyzePrompt(item), { label: `analyze:${item}`, phase: 'Analyze', model: 'haiku', schema: SCHEMA })
)

phase('Report')
const report = await agent('Synthesize: ' + JSON.stringify(results), { label: 'report' })
return report
```

Use `phase:` in agent opts (inside pipeline/parallel) to avoid races on the global phase state.

---

## Resume pattern — free checkpoint recovery

Every workflow auto-journals. If interrupted or edited:

```javascript
// Edit the script file, then:
Workflow({ scriptPath: "<path>", resumeFromRunId: "<runId>" })
// Completed agents return cached — only new/changed calls re-run
```

This means: stop a runaway workflow, fix the script, resume without paying for completed work.

---

## Budget-aware scaling

```javascript
// Scale depth to user's +500k token directive
while (budget.total && budget.remaining() > 50_000) {
  const found = await agent('Find more gaps...', { schema: GAPS_SCHEMA })
  gaps.push(...found.gaps)
  log(`${gaps.length} found, ${Math.round(budget.remaining()/1000)}k remaining`)
}
```

`budget.total` is null if no target set (no directive) — guard with `budget.total &&` to avoid infinite loops.

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Save agent per result | Return all results; Write in main conversation |
| parallel() everywhere | pipeline() by default; parallel() only for cross-item dedup |
| Sonnet for mechanical fan-out | model: "haiku" for N > 3 parallel instances |
| No schema on structured output | Add schema — agent retries until valid, you get the object |
| Forgetting `phase:` inside pipeline() | Set it explicitly — global phase() call races with pipeline stages |
| `Date.now()` in script | Not allowed (breaks resume) — stamp timestamps after workflow returns |
| `Math.random()` in script | Not allowed — vary by index instead |

---

For full patterns (loop-until-dry, adversarial verify, judge panel, multi-modal sweep):
see [patterns.md](references/patterns.md)
