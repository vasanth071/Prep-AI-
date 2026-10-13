# PrepAI — 120-Case Evaluation Benchmark

All numbers on this page were measured on 2026-10-07 by running the real eval-v4 pipeline
(`server/src/services/evaluationEngine.js`) on the full dataset, on one machine with an
RTX 3050 6 GB GPU, Ollama, and the Laya decision service on CPU. Raw results are stored in the
`benchmarkresults` MongoDB collection and exported to
[`server/src/evals/modelComparisonReport.json`](server/src/evals/modelComparisonReport.json) and
[`server/src/evals/benchmarkReport.json`](server/src/evals/benchmarkReport.json).

> Earlier versions of this page reported MAE 5.66 / Pearson 0.907 / 92.5 % accuracy. Those figures
> came from the old keyword/word-count scorer whose rules copied phrases from this very dataset,
> so they did not measure real grading quality. They have been removed.

## 1. Dataset

`server/src/evals/benchmarkDataset100.json` (id `prepai-eval-120-v1`): 120 candidate answers, each
labelled by an expert with a hiring label (`Pass` / `Moderate` / `Fail`), a 0–100 score and an
acceptable score range.

| Category | Cases | Expected class |
|---|---:|---|
| excellent_expert | 10 | CORRECT |
| correct | 20 | CORRECT |
| short_correct | 10 | CORRECT (tests length bias) |
| partially_correct | 20 | PARTIALLY_CORRECT |
| incorrect | 20 | INCORRECT |
| hallucinated_claims | 10 | INCORRECT |
| long_incorrect | 10 | INCORRECT (tests verbosity bias) |
| irrelevant | 10 | IRRELEVANT |
| vague_evasive | 10 | INSUFFICIENT_EVIDENCE |

**Limitation:** the 120 answers are written against only **6 distinct questions**. The set is good
for comparing configurations, but it is too narrow to prove generalisation, so no weights or caps
were tuned to it.

## 2. Systems compared

Every case is run once through the full pipeline; the same run yields three systems:

- **Fusion** — LLM rubric evaluation + Laya decisions (answer state, completeness, concept presence,
  unsupported claims), conflict policy and deterministic weighted aggregation (what the product
  uses). Embedding similarity is recorded with every evaluation as supporting evidence but does
  not change the score.
- **LLM only** — the LLM rubric result before fusion.
- **Laya only** — Laya's `answer_state` decision alone (no scores).

3-class metrics map predictions to the expert label (CORRECT → Pass, PARTIALLY_CORRECT → Moderate,
INCORRECT / IRRELEVANT / INSUFFICIENT_EVIDENCE → Fail). A **false positive** is an expert-Fail
answer predicted as Pass.

## 3. Results (120 cases)

| Grading model | System | Accuracy (3-class) | Macro-F1 | MAE | Pearson r | Score in expert range | False-positive rate | p50 latency |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| **qwen3:8b** | **Fusion** | **0.858** | **0.816** | 16.05 | **0.945** | 25.8 % | **0 %** | 36.1 s |
| qwen3:8b | LLM only | 0.858 | 0.816 | 17.59 | 0.944 | 15.0 % | 0 % | 36.1 s |
| qwen2.5-coder:7b | Fusion | 0.800 | 0.742 | **14.73** | 0.916 | **33.3 %** | 0 % | 25.4 s |
| qwen2.5-coder:7b | LLM only | 0.800 | 0.742 | 16.07 | 0.916 | 24.2 % | 0 % | 25.4 s |
| — | Laya only | 0.43 | 0.40 | n/a | n/a | n/a | 45 % | 12.6 s |

Other measured properties of the fusion system:

| | qwen3:8b | qwen2.5-coder:7b |
|---|---:|---:|
| Structured-output failures | 0 % | 0 % |
| False-negative rate (expert Pass → Fail) | 0 % | 0 % |
| Signal conflicts detected and resolved | 17.5 % | 14.2 % |
| Re-run consistency (10 cases): mean score change | **0.1 pts** | 2.1 pts |
| Re-run consistency: same classification | 100 % | 100 % |
| Mean signed error vs. expert score | −12.2 pts | −8.3 pts |

Latency is per evaluated answer and includes the LLM evaluation, the Laya batch on CPU and the
embedding call (run in parallel). Answers are graded in the background, so it does not affect the live interview.

### Per category (fusion, 3-class accuracy / MAE)

| Category | qwen3:8b | qwen2.5-coder:7b |
|---|---|---|
| excellent_expert | 0.80 / 8.0 | 0.60 / 9.5 |
| correct | 0.65 / 10.6 | 0.55 / 11.7 |
| short_correct | 0.70 / 10.6 | 0.40 / 9.9 |
| partially_correct | 0.85 / 12.4 | 0.85 / 16.6 |
| incorrect | 1.00 / 24.7 | 1.00 / 13.9 |
| hallucinated_claims | 1.00 / 22.1 | 1.00 / 22.1 |
| long_incorrect | 1.00 / 20.5 | 1.00 / 13.8 |
| irrelevant | 1.00 / 13.0 | 1.00 / 13.6 |
| vague_evasive | 0.80 / 23.1 | 0.80 / 23.7 |

### Confusion (expert label → predicted class, fusion)

| Expert | qwen3:8b | qwen2.5-coder:7b |
|---|---|---|
| Pass (40) | 28 CORRECT, 12 PARTIALLY_CORRECT | 21 CORRECT, 19 PARTIALLY_CORRECT |
| Moderate (20) | 17 PARTIALLY_CORRECT, 3 INCORRECT | 17 PARTIALLY_CORRECT, 3 CORRECT |
| Fail (60) | 58 INCORRECT/IRRELEVANT/INSUFFICIENT, 2 PARTIALLY_CORRECT | 58 INCORRECT/IRRELEVANT/INSUFFICIENT, 2 PARTIALLY_CORRECT |

## 4. What the numbers say

1. **No grade inflation.** Neither model passed a single expert-Fail answer (0 % false positives),
   including all hallucinated, verbose-but-wrong and evasive answers.
2. **The main error is under-crediting correct answers.** Both models often call a fully correct
   answer "partially correct"; qwen3:8b does this less (28/40 vs 21/40 correct answers recognised),
   especially for short answers (70 % vs 40 %), so it is less biased towards length.
3. **Scores run low.** The pipeline scores about 8–12 points below the experts on average and
   much lower on wrong answers (experts average 27 there, the pipeline 4). This raises the
   MAE but does not change the hiring decision. Correlation with expert scores is high (0.92–0.95).
4. **Fusion improves the scores, not the classification.** Laya's completeness signal moves scores closer
   to the expert range (+9–11 points of in-range rate, −1.3 to −1.5 MAE) while the classification
   stays the LLM's. Laya alone is not a usable grader (43 % accuracy, 45 % false positives), which is
   why it is used only as a supporting signal and why conflicts are resolved by evidence.
5. **qwen3:8b is far more repeatable** (0.1 vs 2.1 points between runs).

## 5. Configuration chosen

`server/.env`:

```env
AI_MODEL=qwen2.5-coder:7b     # live interviewer dialogue
EVALUATION_MODEL=qwen3:8b     # answer grading, reference packs, report prose
```

- qwen3:8b grades better (accuracy +5.8 points, macro-F1 +7.4, short-answer accuracy +30) and is
  more consistent, so it is the grading model.
- qwen2.5-coder:7b stays the dialogue model: on the 6 GB GPU it stays resident and a live turn
  takes ~9–10 s.
- Both models do not fit in 6 GB together and switching costs ~12 s per swap (measured). Grading
  for an interview therefore waits until that interview ends (or has been idle for 20 minutes);
  the report for a 4-answer interview is ready ~3 minutes after the end. On a machine with more
  VRAM the deferral switches itself off when both settings name the same model.

## 6. Reproduce

```bash
# One model through the full pipeline (writes benchmarkresults + benchmarkReport.json)
npm --prefix server run benchmark

# Several models side by side (writes modelComparisonReport.json)
npm --prefix server run benchmark:compare -- qwen2.5-coder:7b qwen3:8b
```

Admins can also queue a run from the API: `POST /api/v1/internal/benchmarks/run`
(`{ "model": "qwen3:8b", "limit": 120 }`), and read results at `GET /api/v1/internal/benchmarks`.
The full two-model comparison took about 2 h 15 min on this machine.
