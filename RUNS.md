# LoCoMo Run Log

Record of benchmark runs so they can be resumed or re-evaluated later.
Resuming requires the **same** `--project-name`, `--run-id` and `--chunk-size` as the original run.

## Setup (all runs below)

- Backend: Mem0 Cloud (`MEM0_BACKEND=cloud`, v3 API)
- Answerer / Judge: `deepseek-flash` via `OPENAI_BASE_URL=https://api.deepseek.com`
- Config: `.env` (not committed)

## Runs

| Date | Project name | run_id | Mem0 user_id | Conversations | chunk-size | Status | Notes |
|---|---|---|---|---|---|---|---|
| 2026-11-25 | `locomo-smoke2` | `06e9dd60` | `locomo_0_06e9dd60` | 0 | 1 (official) | Done: all 152 Qs of conv 0 evaluated | See Results below. Ingestion ~1h50m (~17 s/chunk). |
| 2026-11-25 | `locomo-smoke` | `104f6d81` | `locomo_0_104f6d81` | 0 | 1 | Failed / abandoned | Failed with HTTP 400 (old `/v3/memories/` add endpoint). Only 1 chunk recorded; do not reuse. |

## Results

### `locomo-smoke2` — conversation 0, all 152 questions (2026-11-25)

Result file: `results/locomo/locomo_results_20261125_174704.json`
Setup: Mem0 Cloud v3, chunk-size 1, answerer + judge `deepseek-flash`, `LLM_MAX_TOKENS` = 4096 (default).

| Cutoff | Overall | single-hop (70) | multi-hop (32) | temporal (37) | open-domain (13) |
|---|---|---|---|---|---|
| top_10 | **86.8%** (132/152) | 85.7% | 71.9% | 100.0% | 92.3% |
| top_20 | **88.2%** (134/152) | 88.6% | 71.9% | 100.0% | 92.3% |
| top_50 | **90.8%** (138/152) | 88.6% | 87.5% | 97.3% | 92.3% |
| top_200 | **91.4%** (139/152) | 91.4% | 81.3% | 97.3% | 100.0% |

Reference (official, Mem0 Platform, GPT-5 answerer/judge, all 1540 Qs): 92.5% @top_200, 91.8% @top_50.

**Notes**
- 5 misses are empty answers, not memory misses: DeepSeek hit the 4096-token limit while reasoning (`finish_reason=length`) even after 5 retries.
  Affected: q32 (top_20, top_200), q51 (top_200), q70 (top_200), q116 (top_50).
  If all were answered correctly, top_200 would be at most 142/152 = 93.4%.
- 4 questions were wrong at every cutoff (likely real retrieval/reasoning misses):
  - q40 multi-hop — "How many times has Melanie gone to the beach in 2023?" GT: 2, answered 1
  - q66 multi-hop — "What does Melanie do with her family on hikes?" GT: roast marshmallows, tell stories
  - q85 single-hop — "What are Caroline's plans for the summer?" GT: researching adoption agencies
  - q144 single-hop — "How did Melanie's son handle the accident?" GT: scared but reassured by family
- More memories help: overall rises from 86.8% (top_10) to 91.4% (top_200); multi-hop benefits most.
- Mem0 stored at least 447 distinct memories for this conversation (419 dialogue turns).
- Small sample (1 of 10 conversations); not directly comparable to the official number.

**Next steps**
- Set `LLM_MAX_TOKENS=16384` in `.env`, move the checkpoints of q32/q51/q70/q116 out of `results/locomo/predicted_locomo-smoke2/`, and resume to re-answer them.
- Optionally ingest the remaining 9 conversations for a full 1540-question run.

## Commands

Resume `locomo-smoke2` (continues ingestion from the last completed chunk):

```bash
source .venv/bin/activate
python3 -m benchmarks.locomo.run --project-name locomo-smoke2 --conversations 0 --resume --run-id 06e9dd60
```

Once ingestion is complete, evaluate all questions for conversation 0 on the same memories
(drop `--max-questions`):

```bash
python3 -m benchmarks.locomo.run --project-name locomo-smoke2 --conversations 0 --resume --run-id 06e9dd60
```

## Where things are

- Progress / checkpoints: `results/locomo/predicted_<project-name>/_progress_<conv>.json`
- Logs: `logs/locomo_<timestamp>.log`
- Dataset: `datasets/locomo/locomo10.json`
- Mem0 users created by runs are **not** deleted automatically; clean up `locomo_*` users in the Mem0 dashboard when no longer needed.
