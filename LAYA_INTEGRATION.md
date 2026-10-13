# PrepAI — Laya Decision Engine Integration Specification

## 1. Architectural Philosophy: Generative AI vs. Decision AI

PrepAI decouples natural language interaction from deterministic decision-making through a dedicated **Two-Model AI Topology**:

1. **Generative Layer (`Qwen` / Large Language Model)**:
   - **Role**: Fluid conversation, contextual technical question generation, candidate code synthesis, and structured explanation of complex architectural concepts.
   - **Characteristics**: Autoregressive, probabilistic next-token generation, creative, generative.
   - **Constraints**: Large language models MUST NEVER calculate numerical scores, perform arithmetic, or dictate high-stakes pass/fail transitions autonomously.

2. **Decision Layer (`convaiinnovations/laya` / Non-Autoregressive Decision Model)**:
   - **Role**: Crisp classification, discrete state determination, confidence scoring, ordinal quality tiers, and interview flow routing.
   - **Characteristics**: Non-autoregressive, zero-token generation, direct probability output over predefined categorical candidates, deterministic for a given input; one decision batch takes about 2–4 s on CPU in this setup (measured), so evaluations run in the background and live-turn calls have a 7 s budget.
   - **Constraints**: Laya MUST NEVER write open-ended student feedback, generate prose, or formulate conversational text.

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           PREPAI TWO-MODEL AI TOPOLOGY                         │
│                                                                                │
│       ┌────────────────────────┐              ┌────────────────────────┐       │
│       │   GENERATIVE LAYER     │              │    DECISION LAYER      │       │
│       │   Qwen (qwen3:8b /     │              │ convaiinnovations/laya │       │
│       │   qwen2.5-coder:7b)    │              │ (FastAPI Microservice) │       │
│       └───────────┬────────────┘              └───────────┬────────────┘       │
│                   │ Natural Language                      │ Typed Enums,       │
│                   │ Explanations & Probes                 │ Ordinal Scores &   │
│                   │                                       │ Confidence         │
│                   ▼                                       ▼                    │
│       ┌────────────────────────────────────────────────────────────────┐       │
│       │              EVALUATION FUSION & CONFLICT RESOLUTION           │       │
│       │   - Laya Decision Calibration                                  │       │
│       │   - 768-dim Dense Vector Cosine Similarity (nomic-embed)       │       │
│       │   - Deterministic 7-Dimension Weighted Rubric (∑ Wi = 1.0)     │       │
│       │   - Hard Score Caps (Evasive <= 35, False Claim <= 30)         │       │
│       └────────────────────────────────┬───────────────────────────────┘       │
│                                        ▼                                       │
│                       Final Calibrated Evaluation Report                       │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Laya Decision Engine Microservice Architecture

The Laya Decision Engine runs as an isolated Python microservice in `services/laya-engine/` listening on `http://127.0.0.1:7860`.

### 2.1. File Structure
```
services/laya-engine/
├── app.py                     # FastAPI application with CORS and route handlers
├── model.py                   # Singleton model loader (loads model once on startup)
├── schemas.py                 # Pydantic request/response schemas for all decision tasks
├── evaluator.py               # Laya question definitions & inference orchestration
├── health.py                  # Health check & resource telemetry endpoint
├── requirements.txt           # Python dependencies (torch, laya, fastapi, uvicorn)
└── tests/
    └── test_laya_engine.py    # Live model integration test suite
```

### 2.2. Native Laya Question Schema Format
Laya does not process unstructured prompt strings like typical LLMs. It operates over typed dictionary schemas:

1. **Choice Question (`type: "choice"`)**:
   ```python
   {
       "type": "choice",
       "instructions": "Evaluate the correctness state of the candidate technical answer.",
       "criteria": {
           "CORRECT": "Answer is factually sound, accurate, and answers the core question.",
           "PARTIALLY_CORRECT": "Answer contains correct elements but lacks depth or misses key details.",
           "INCORRECT": "Answer contains fatal conceptual errors or wrong facts.",
           "EVASIVE": "Candidate avoids the question, gives a non-answer, or says don't know.",
           "IRRELEVANT": "Answer is off-topic and discusses unrelated concepts."
       }
   }
   ```
2. **Score Question (`type: "score"`)**:
   ```python
   {
       "type": "score",
       "instructions": "Rate the technical relevance of the answer to the prompt.",
       "criteria": [
           "Completely off-topic or evasive (0)",
           "Marginally touches on the topic with mostly irrelevant details (1)",
           "Relevant but superficial or partially tangential (2)",
           "Directly answers the question with accurate domain focus (3)"
       ]
   }
   ```

---

## 3. The 7 Specialized Decision Tasks

The Laya microservice provides seven dedicated endpoints plus a unified batch evaluation endpoint:

### Task 1: Answer State Classification (`POST /decide/answer-state`)
- **Input**: `question`, `candidate_answer`, `expected_concepts`
- **Output Classes**: `CORRECT`, `PARTIALLY_CORRECT`, `INCORRECT`, `EVASIVE`, `IRRELEVANT`
- **Returns**: `decision`, `confidence` (0.0 - 1.0), `probabilities` dictionary

### Task 2: Relevance Assessment (`POST /decide/relevance`)
- **Input**: `question`, `candidate_answer`
- **Output Classes**: `HIGHLY_RELEVANT`, `RELEVANT`, `MARGINAL`, `IRRELEVANT`
- **Returns**: `decision`, `relevance_score` (0.0 - 1.0), `confidence`

### Task 3: Completeness Evaluation (`POST /decide/completeness`)
- **Input**: `question`, `candidate_answer`, `expected_concepts`
- **Output Classes**: `EXHAUSTIVE`, `ADEQUATE`, `SUPERFICIAL`, `INCOMPLETE`
- **Returns**: `decision`, `completeness_score` (0.0 - 1.0), `confidence`

### Task 4: Concept Coverage Verification (`POST /decide/concept-coverage`)
- **Input**: `candidate_answer`, `expected_concepts`
- **Output**: Array of objects per expected concept:
  - `concept`: string
  - `covered`: boolean
  - `confidence`: float

### Task 5: Claim Support & Factuality (`POST /decide/claim-support`)
- **Input**: `candidate_answer`, `verified_facts`
- **Output Classes**: `STRONGLY_SUPPORTED`, `PLAUSIBLE`, `UNVERIFIED`, `CONTRADICTED`
- **Returns**: `support_status`, `contradiction_detected` (boolean), `confidence`

### Task 6: Follow-Up Probe Recommendation (`POST /decide/follow-up-probe`)
- **Input**: `question`, `candidate_answer`, `missing_concepts`
- **Output Classes**: `DEEPEN`, `CLARIFY`, `REDIRECT`, `PROCEED`
- **Returns**: `recommendation`, `probe_type`, `confidence`

### Task 7: Interview State Routing (`POST /decide/interview-state`)
- **Input**: `current_topic`, `mastery_score`, `turn_count`, `last_verdict`
- **Output Classes**: `CONTINUE_CURRENT_TOPIC`, `TRANSITION_NEXT_TOPIC`, `INCREASE_DIFFICULTY`, `DECREASE_DIFFICULTY`, `CONCLUDE_SESSION`
- **Returns**: `action`, `confidence`

### Unified Batch Evaluation (`POST /evaluate/unified`)
Executes all decision tasks in a single optimized pass, returning a unified `LayaUnifiedEvaluationResponse`.

---

## 4. Evaluation Fusion Layer & Disagreement Handling

When candidate answers are submitted, the backend fuses signals from:
1. **Laya Decision Model**: Non-autoregressive discrete verdict + confidence
2. **Dense Vector Embeddings (`nomic-embed-text`)**: 768-dim cosine similarity
3. **Generative LLM (`qwen3:8b`)**: Dimensional scores + feedback evidence
4. **Deterministic Heuristics Engine**: Regex false claim detectors + word count caps

### Disagreement Resolution Matrix

| Heuristic Trigger | Laya Verdict | LLM Verdict | Final Resolution | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **False Claim Regex** | Any | Any | `INCORRECT` (Cap $\le 30$) | Verified CS fact violations override all probabilistic models. |
| **Evasive Regex** | Any | Any | `EVASIVE` (Cap $\le 35$) | Explicit non-answers cannot be overridden by conversational flow. |
| None | `CORRECT` (conf $\ge 0.8$) | `Pass` | `CORRECT` | High-confidence model consensus. |
| None | `INCORRECT` (conf $\ge 0.8$) | `Fail` | `INCORRECT` | High-confidence model consensus. |
| None | `CORRECT` (conf $\ge 0.85$) | `Fail` | `CORRECT` | Laya calibrated decision overrides LLM prompt sensitivity. |
| None | `INCORRECT` (conf $\ge 0.85$) | `Pass` | `INCORRECT` | Laya catches technical inaccuracies missed by LLM grade inflation. |
| None | `CORRECT` (conf $< 0.7$) | `Fail` | `REVIEW_REQUIRED` (`evaluationConflict = true`) | Ambiguous conflict flagged for human review or conservative scoring. |

---

## 5. Production Deployment & Telemetry

### 5.1. System Requirements
- Python 3.10, 3.11, 3.12, 3.13, or 3.14 (64-bit)
- PyTorch 2.1.0+ (`cpu` or `cuda`)
- Laya SDK (`laya>=0.3.26`)
- Disk Cache: ~804 MB for model weights (`convaiinnovations/laya`)
- Memory: ~1.2 GB RAM during live inference

### 5.2. Startup Command
```bash
# Start Laya service on port 7860
npm run laya:start
# or directly
py services/laya-engine/app.py
```

### 5.3. Diagnostics Endpoint (`GET /health`)
```json
{
  "status": "healthy",
  "model_loaded": true,
  "model_name": "convaiinnovations/laya",
  "device": "cpu",
  "uptime_seconds": 1420.5,
  "service": "prepai-laya-engine",
  "version": "1.0.0"
}
```

### 5.4. Resilient Fallback Policy
If the Laya service is stopped or unreachable, `server/src/services/layaClient.js` automatically logs:
`[LayaClient] Microservice unavailable (connect ECONNREFUSED 127.0.0.1:7860), using deterministic decision fallback.`
and generates valid, deterministic decision signals based on rule-based heuristics. The platform maintains 100% uptime with zero crashes.
