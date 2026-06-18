# Model Tiers — Cost Reference

## Haiku 4.5
- Input: $0.80/MTok | Output: $4.00/MTok
- Best for: fan-out search, fetch, extract, verify voters, classification, structured extraction
- Rule: N > 3 parallel instances → Haiku

## Sonnet 4.6 (default)
- Input: ~$3/MTok | Output: ~$15/MTok
- Best for: scope/decompose, synthesis, design, final output, complex multi-file analysis

## Opus 4.8
- Input: ~$15/MTok | Output: ~$75/MTok
- Best for: only when Sonnet repeatedly fails on complex audit/legal/financial reasoning

## Cost example — deep research (97 agents)

| Config | Cost |
|---|---|
| All Sonnet | ~$2–4 |
| Haiku on 95 fan-out, Sonnet on scope+synthesize | ~$0.20–0.40 |
| **Saving** | **~10×** |

## Cost example — analytics audit (27 modules × Haiku)

| Input | Output | Total |
|---|---|---|
| ~344k tokens | ~54k tokens | ~$0.49 |

Without Haiku (all Sonnet): ~$6–7 for the same run.
