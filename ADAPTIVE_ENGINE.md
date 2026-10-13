# PrepAI — Adaptive Topic Mastery & Interview Progression Engine

## 1. Engine Architecture & Mathematical Foundations

PrepAI replaces simplistic static question sequences with a multi-signal **Adaptive Topic Mastery Engine**. Instead of relying solely on crude counts of consecutive correct or incorrect answers, the platform computes a continuous, Bayesian-smoothed **Exponential Moving Average (EMA)** of candidate competence per topic.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         ADAPTIVE ENGINE SIGNAL PIPELINE                          │
│                                                                                  │
│   Candidate Answer ──► [7-Dimension Rubric Score (0-100)]                        │
│                                │                                                 │
│                                ▼                                                 │
│   Prior Topic Mastery ──► [Bayesian EMA Smoother: M_t = α*S_t + (1-α)*M_prev]   │
│                                │                                                 │
│                                ▼                                                 │
│   Laya State Decision ──► [Fusion Router: Difficulty & Topic Transition Policy]  │
│   (INCREASE / DECREASE /       │                                                 │
│    TRANSITION / CONCLUDE)      ▼                                                 │
│                        [Selected Next Topic & Difficulty Tier]                   │
│                        [Promote / Maintain / Demote / Advance]                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Topic Mastery Calculation Formulas

### 2.1. Exponential Moving Average (EMA) Update Rule
For a candidate answering a question in topic $k$ at turn $t$:

$$M_{k, t} = \alpha \cdot \left(\frac{S_t}{100}\right) + (1 - \alpha) \cdot M_{k, t-1}$$

Where:
- $M_{k, t} \in [0.0, 1.0]$: Updated mastery score for topic $k$ after turn $t$.
- $S_t \in [0, 100]$: Composite rubric score awarded on turn $t$.
- $\alpha \in (0, 1]$: Recency weighting factor (default $\alpha = 0.35$).
  - Higher $\alpha$ rapidly adapts to recent breakthroughs or mistakes.
  - Lower $\alpha$ preserves historical evidence against transient anomalies.

### 2.2. Confidence-Adjusted Dynamic Alpha
When the Laya Decision Engine provides a high decision confidence $C_{\text{laya}} \in [0.5, 1.0]$, $\alpha$ scales dynamically:

$$\alpha_{\text{dyn}} = \alpha_{\text{base}} \times \left(0.8 + 0.4 \times C_{\text{laya}}\right)$$

This gives greater weight to unambiguous, high-confidence evaluations while dampening volatile, ambiguous turns.

---

## 3. Normalized Mastery Bands

PrepAI categorizes continuous topic mastery $M_{k}$ into five discrete operational bands:

| Band Name | Mastery Range | Operational Profile | Recommended Question Tier |
| :--- | :--- | :--- | :--- |
| **`Beginner`** | $[0.00, 0.40)$ | Fundamental conceptual gaps; struggles with core definitions and basic syntax. | `beginner` |
| **`Developing`** | $[0.40, 0.60)$ | Understands basics but lacks architectural depth and edge-case handling. | `beginner` $\to$ `intermediate` |
| **`Intermediate`** | $[0.60, 0.75)$ | Solid grasp of core concepts; capable of standard implementation and problem solving. | `intermediate` |
| **`Strong`** | $[0.75, 0.88)$ | Proficient in production-grade concerns, performance trade-offs, and failure domains. | `intermediate` $\to$ `advanced` |
| **`Advanced`** | $[0.88, 1.00]$ | Deep architectural mastery, distributed systems trade-offs, and low-level internals. | `advanced` |

---

## 4. Difficulty Adjustment & Progression Algorithms

### 4.1. Difficulty Promotion Criteria
A candidate's difficulty tier is promoted (`beginner` $\to$ `intermediate` or `intermediate` $\to$ `advanced`) when:
1. **Demonstrated Mastery**:
   - Current topic mastery $M_{k} \ge 0.75$, **AND**
   - The candidate scored $\ge 80$ points on the most recent turn.
2. **Laya Decision Alignment**:
   - Laya classifies `InterviewStateDecision` as `INCREASE_DIFFICULTY` with confidence $\ge 0.75$.

### 4.2. Difficulty Demotion Criteria
A candidate's difficulty tier is gracefully demoted (`advanced` $\to$ `intermediate` or `intermediate` $\to$ `beginner`) when:
1. **Demonstrated Struggle**:
   - Current topic mastery $M_{k} < 0.45$, **AND**
   - The candidate scored $\le 45$ points on the most recent turn.
2. **Laya Decision Alignment**:
   - Laya classifies `InterviewStateDecision` as `DECREASE_DIFFICULTY` with confidence $\ge 0.75$.

### 4.3. Anti-Thrashing Guardrails
To prevent rapid oscillating between difficulty levels:
- A difficulty adjustment incurs a **cooldown of at least 2 turns** before another adjustment in the same direction can occur.
- Severe demotion to `beginner` triggers targeted diagnostic questions to identify foundational misconceptions rather than simply repeating failed questions.

---

## 5. Topic Switching & Multi-Topic Coverage

A technical interview must evaluate candidate breadth and depth across multiple domains (e.g., Data Structures, System Design, Concurrency, Database Architecture).

### 5.1. Topic Transition Policy
The engine transitions to the next technical topic when:
1. **Mastery Confirmed**: At least 2 turns have been conducted on topic $k$, and $M_{k} \ge 0.75$.
2. **Deficit Confirmed**: At least 3 turns have been conducted on topic $k$, and the candidate shows persistent difficulty ($M_{k} < 0.40$), indicating saturation without immediate improvement.
3. **Laya Signal**: Laya recommends `TRANSITION_NEXT_TOPIC` with high confidence.

### 5.2. Session Completion Criteria
An interview session moves to the `WRAPUP` phase and concludes when:
1. All core target topics have met minimum turn quotas (minimum 2 turns per required skill).
2. The total interview turn limit (default: 8 to 12 questions) is reached.
3. Laya emits `CONCLUDE_SESSION` with confidence $\ge 0.85$ based on comprehensive evaluation.

---

## 6. Implementation References

- Adaptive Engine Logic: [`server/src/services/adaptiveEngine.js`](file:///c:/Users/mohan/Downloads/Mini%20project/server/src/services/adaptiveEngine.js)
- Interview State Machine: [`server/src/services/interviewStateMachine.js`](file:///c:/Users/mohan/Downloads/Mini%20project/server/src/services/interviewStateMachine.js)
- Laya Client Connector: [`server/src/services/layaClient.js`](file:///c:/Users/mohan/Downloads/Mini%20project/server/src/services/layaClient.js)
- Unit Tests: [`server/src/tests/prepai.test.js`](file:///c:/Users/mohan/Downloads/Mini%20project/server/src/tests/prepai.test.js) (Test Suites 3 & 4)
