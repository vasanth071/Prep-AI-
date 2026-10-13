# PrepAI — Answer Evaluation Engine (eval-v4)

Source: [`server/src/services/evaluationEngine.js`](server/src/services/evaluationEngine.js).
Every interview answer is graded by this pipeline in the background; the result, every signal and
its model metadata are stored in `answer_evaluations` and `evaluation_signals`.

## 1. Rubric and aggregation

| Dimension | Key | Weight | What it measures |
|---|---|---:|---|
| Correctness | `correctness` | 0.25 | Factual accuracy of the technical content |
| Relevance | `relevance` | 0.15 | Whether the answer addresses the question asked |
| Technical depth | `technicalDepth` | 0.15 | Mechanisms, internals, precise terminology |
| Completeness | `completeness` | 0.15 | Coverage of the expected concepts |
| Reasoning | `reasoning` | 0.10 | Justification and trade-offs |
| Clarity | `clarity` | 0.10 | Logical structure, no contradictions |
| Communication | `communication` | 0.10 | Precise, concise articulation |

Weights are validated at load (`validateRubricWeights`: every dimension present, non-negative,
sum = 1.0). The model only proposes dimension scores; the overall score is computed in code:

$$S = \mathrm{round}\Big(\sum_{i=1}^{7} W_i \cdot S_i\Big), \quad S_i \in [0,100]$$

Answer length and keyword counts are never scoring inputs.

## 2. Pipeline

```
answer ─► input validation ──(empty / "I don't know" / "skip" …)──► INSUFFICIENT_EVIDENCE, score 0, no model call
            │
            ├─► reference pack (expected + required concepts, common mistakes, reference answer; built once per question)
            │
            ├─► in parallel:
            │     LLM rubric evaluator (EVALUATION_MODEL, JSON schema, 1 retry)
            │     Laya batch: answer_state, relevance, completeness, unsupported_claim, followup_needed,
            │                 response_state + per-concept presence (class probabilities)
            │     Embedding similarity to the question / reference answer (recorded, not scored)
            │
            ├─► schema validation + evidence verification
            ├─► concept merge (LLM evidence × Laya presence)
            ├─► classification + conflict policy
            ├─► completeness blend (70 % LLM, 30 % Laya expected value)
            ├─► classification caps (can only lower a dimension)
            └─► deterministic weighted aggregation
```

If the LLM evaluator fails (provider down, two invalid outputs) the evaluation is stored with
`status: failed`, `overall: null` and `reviewRequired: true`. **No fallback score is produced**; the
job queue retries it, and it can be re-run through `POST /api/v1/evaluations/{id}/retry` or
`POST /api/v1/interviews/{id}/report/retry`.

## 3. Classifications

`CORRECT`, `PARTIALLY_CORRECT`, `INCORRECT`, `IRRELEVANT`, `INSUFFICIENT_EVIDENCE`.

### Caps (applied after classification; they never raise a score)

| Classification | Cap |
|---|---|
| INCORRECT | correctness ≤ 40 |
| IRRELEVANT | relevance ≤ 25, correctness ≤ 30 |
| INSUFFICIENT_EVIDENCE | correctness ≤ 25, completeness ≤ 25 |

## 4. Evidence rules

- Every `evidence` quote and every `incorrectClaims` entry must appear **verbatim** (case- and
  whitespace-insensitive) in the student's answer; anything else is discarded.
- A concept the model marks PRESENT without a verifiable quote becomes **UNVERIFIED**.
- Concept status after merging with Laya: PRESENT, MISSING, INCORRECT or **DISPUTED** (the LLM and
  Laya disagree). Missing concepts feed the report and the study plan.

## 5. Conflict policy

The LLM classification is compared with Laya's `answer_state`:

1. Same class, adjacent classes (e.g. CORRECT vs PARTIALLY_CORRECT) or Laya confidence < 0.4 →
   no conflict; confidence is the average of both.
2. Otherwise it is a conflict, resolved by evidence:
   - concept coverage ≥ 60 % and no incorrect claim → the positive classification wins;
   - an incorrect claim (verified quote, or Laya `unsupported_claim = YES` with confidence ≥ 0.6) or
     coverage < 34 % → the negative classification wins;
   - otherwise the LLM classification is kept and the evaluation is marked **review required**.
3. Every conflict is stored (`type`, both classifications and confidences, coverage, claim flag,
   rule, resolution), and resolved conflicts lower the confidence.

## 6. Output (per answer)

`status`, `classification`, `confidence`, seven dimension scores, `overall`, `conceptAnalysis`,
`strengths`, `missingConcepts`, `incorrectClaims`, `evidence`, `improvementActions`, `idealAnswer`,
`followUpNeeded`, `reviewRequired`, `conflict`, `signalsAvailable`, `evaluatorVersion` (`eval-v4`),
`promptVersion`, `provider`, `model`, `latencyMs`.

## 7. Measured quality

On the 120-case benchmark with `qwen3:8b`: 85.8 % hiring-decision accuracy, macro-F1 0.816,
Pearson r 0.945, 0 % false positives, MAE 16 points (scores run low on wrong answers), re-run
score change 0.1 points. Full results, including the weak spots, are in [BENCHMARKS.md](BENCHMARKS.md).

## 8. Tests

`server/src/tests/unit/evaluation.test.js` covers weight validation, aggregation, input
validation, verbatim-evidence filtering, UNVERIFIED concepts, schema rejection, the conflict
policy, caps, the no-fallback failure path, and a full mocked pipeline run.
