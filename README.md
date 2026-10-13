# PrepAI — Enterprise-Grade AI-Powered Placement Preparation Platform

[![Server tests](https://img.shields.io/badge/Server%20unit%20tests-39%20passing-brightgreen.svg)](#testing--quality-verification)
[![Integration](https://img.shields.io/badge/API%20integration-14%20passing-brightgreen.svg)](#testing--quality-verification)
[![Benchmark](https://img.shields.io/badge/Grading%20accuracy-85.8%25%20(120%20cases)-blue.svg)](BENCHMARKS.md)
[![Local Inference](https://img.shields.io/badge/Local%20Inference-Ollama%20%2B%20Laya-purple.svg)](#model-providers)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

PrepAI is a privacy-first technical interview and placement preparation platform for engineering students. It runs **locally by default** with four AI layers:
1. **Generative layer**: Qwen via local Ollama. `qwen2.5-coder:7b` drives the live interviewer; `qwen3:8b` grades answers, builds reference packs and writes report prose (both configurable).
2. **Decision layer**: `convaiinnovations/laya` in a Python microservice (:7860) returns typed decisions with class probabilities (answer state, relevance, completeness, unsupported claims, follow-up need, response state).
3. **Embedding layer**: local `nomic-embed-text` via Ollama for semantic similarity to reference answers.
4. **Deterministic layer**: a 7-dimension weighted rubric ($\sum W_i = 1.0$), evidence verification, a conflict policy, a 10-state interview state machine and per-topic mastery with deterministic difficulty rules.

No score is ever invented: if the grading model is unavailable an answer is marked *evaluation failed* and can be retried, instead of receiving a fallback number.

---

## Key System Highlights

- **Local-first privacy**: inference runs on local Ollama models and the local Laya service; resumes, answers and audio stay on your machine. PII (emails, phone numbers, LinkedIn URLs) is scrubbed before any prompt.
- **Provider abstraction**: swappable generation providers (Ollama, Anthropic, Gemini, any OpenAI-compatible endpoint) with retries, an explicitly configured fallback provider and per-call observability (`model_executions`: latency, retries, fallback, structured-output validity).
- **Multi-signal evaluation (eval-v4)**: LLM rubric evaluation + Laya decisions, with embedding similarity recorded as supporting evidence. Evidence quotes must appear verbatim in the student's answer; concepts claimed without evidence become *UNVERIFIED*; disagreements between signals are recorded and resolved by evidence or flagged *review required*.
- **7-dimension rubric**: Correctness 25 %, Relevance 15 %, Technical Depth 15 %, Completeness 15 %, Reasoning 10 %, Clarity 10 %, Communication 10 %, aggregated deterministically in code. Caps can only lower a score (e.g. an INCORRECT answer cannot score above 40 on correctness).
- **10-state interviewer**: Introduction → Background → Resume → Technical Basics → Technical Depth → Problem Solving → Scenario → Behavioral/HR → Final → Feedback, with counter-question strategies (clarification, correction, depth, challenge) chosen in code from Laya signals, duplicate-question suppression and prompt-markup sanitising.
- **Interview memory (RAG)**: every finished interview is indexed per answer with `nomic-embed-text` embeddings (`interviewmemories`). Later interviews retrieve the most relevant history for the current topic, so the interviewer revisits concepts you missed before, avoids repeating questions and welcomes you back with your weakest area. View or clear it with `GET` / `DELETE /api/v1/memory`; backfill old interviews with `node server/scripts/backfill_interview_memory.js`.
- **Improvement analysis**: every report shows the gap to the placement-ready score (70), the points each dimension would add, the fastest path to the target, the topics and concepts to learn (flagging ones missed in earlier interviews) and progress versus previous interviews, with a written analysis grounded in those numbers.
- **Adaptive mastery**: per-topic recency-weighted, smoothed mastery in five bands (Beginner → Advanced) with Easy/Medium/Hard transitions that need stable evidence (no oscillation on one lucky answer).
- **Deterministic aptitude engine**: server-side timing, numeric tolerance, per-question attempts, per-topic mastery, adaptive selection and versioned questions with balanced answer positions.
- **Verified resume analysis**: PyMuPDF text extraction, structured extraction checked against the source text (nothing that is not in the resume is reported), six ATS dimensions and interview hot spots.
- **Security**: 15-minute access tokens with rotating refresh tokens and logout revocation, CORS allowlist, request ids, security headers, rate limits, ownership checks on every resource, audit log, anonymised leaderboard.
- **Measured, not claimed**: on the 120-case benchmark the grading pipeline reaches 85.8 % hiring-decision accuracy with **0 % false positives** (no failing answer passed). See [BENCHMARKS.md](BENCHMARKS.md) for every metric, including the weak ones.

---

## High-Level Architecture Flow

```mermaid
flowchart TD
    Candidate([Candidate Browser / React + Vite]) <-->|HTTPS / REST / Audio| Express[Express API Gateway :7857]
    
    subgraph Security Layer
        Express --> Limiter[Token Bucket Rate Limiter]
        Limiter --> Auth[JWT Auth & IDOR Guard]
        Auth --> PII[PII Sanitizer & Input Scrubber]
    end

    subgraph Core Orchestration
        PII --> SM[Interview State Machine\n8-Phase Adaptive Flow]
        SM --> AE[Adaptive Topic Engine\nMastery Tracking & EMA]
        SM --> EE[Evaluation Fusion Engine\n7-Dimension Weighted Rubric]
    end

    subgraph Decision AI Layer
        EE <--> LayaMicro[Laya Decision Microservice :7860\nconvaiinnovations/laya]
    end

    subgraph AI Provider Abstraction
        EE <--> Factory[AI Provider Factory]
        Factory -->|Priority 1: Local| Ollama[Ollama Local Daemon :11434\nqwen3:8b / qwen2.5-coder / nomic-embed]
        Factory -.->|Fallback 1| Anthropic[Anthropic Claude API]
        Factory -.->|Fallback 2| Gemini[Google Gemini API]
        Factory -.->|Fallback 3| OpenAI[OpenAI Compatible / Groq / vLLM]
    end

    subgraph Persistence Layer
        EE --> MongoDB[(MongoDB 8.0+ / Mongoose)]
        AE --> MongoDB
    end

    subgraph TTS & Audio Pipeline
        Express <--> EdgeTTS[Edge-TTS Audio Synthesis Pipeline]
    end
```

---

## Repository Structure

```
Mini project/
├── frontend/                     # React 18 + TypeScript + Vite + Tailwind UI
│   ├── src/
│   │   ├── components/           # Reusable UI components & layouts
│   │   ├── pages/                # Interview, Report, Aptitude, Resume, Dashboard
│   │   ├── types/                # TypeScript interfaces (report.ts, interview.ts)
│   │   └── api/                  # Axios HTTP client configuration
│   └── package.json
├── server/                       # Node.js + Express + MongoDB Backend
│   ├── src/
│   │   ├── config/               # Database and environment initialization
│   │   ├── evals/                # 120-test golden benchmark & multi-model harness
│   │   │   ├── benchmarkDataset100.json
│   │   │   ├── runFullBenchmark.js
│   │   │   └── compareModelProviders.js
│   │   ├── middleware/           # Rate limiter, auth, PII scrubbing, error handler
│   │   ├── models/               # User, Interview, TopicMastery, Report, Aptitude
│   │   ├── routes/               # REST API endpoints (auth, interview, report, etc.)
│   │   ├── services/             # Core engines & abstraction layer
│   │   │   ├── providers/        # AIProvider, Ollama, Anthropic, Gemini, OpenAI, Factory
│   │   │   ├── adaptiveEngine.js         # Topic mastery & difficulty progression
│   │   │   ├── aiService.js              # High-level AI coordination
│   │   │   ├── evaluationEngine.js       # 7-dimension rubric evaluator & Laya fusion
│   │   │   ├── interviewStateMachine.js  # 10-state interview machine & counter-question strategies
│   │   │   ├── layaClient.js             # HTTP client for Laya decision microservice
│   │   │   └── semanticEngine.js         # Cosine similarity & vector math
│   │   └── tests/                # Native Node.js unit & integration tests
│   │       └── prepai.test.js
│   └── package.json
├── services/
│   └── laya-engine/              # Non-autoregressive decision model microservice
│       ├── app.py                # FastAPI HTTP service (:7860)
│       ├── evaluator.py          # Laya question definitions & inference orchestration
│       ├── model.py              # Singleton model loader (convaiinnovations/laya)
│       ├── schemas.py            # Pydantic typed request & response models
│       ├── health.py             # Health check & resource telemetry endpoint
│       ├── requirements.txt      # Python dependencies
│       └── tests/
│           └── test_laya_engine.py # Laya model test suite
├── package.json                  # Root monorepo workspace scripts
└── [DOCUMENTATION FILES]          # Comprehensive architectural & operational guides
```

---

## Quickstart & Installation

### 1. Prerequisites
- **Node.js** 20+ (developed on Node 26)
- **Python** 3.10+ (64-bit) with PyTorch for the Laya service
- **MongoDB** 6+ running locally (default `mongodb://127.0.0.1:27017/prepai`)
- **Ollama** from [ollama.com](https://ollama.com)
- A GPU is optional but strongly recommended; the reference machine has an RTX 3050 6 GB.

### 2. Pull the Ollama models
```bash
ollama pull qwen2.5-coder:7b   # live interviewer (AI_MODEL)
ollama pull qwen3:8b           # answer grading (EVALUATION_MODEL)
ollama pull nomic-embed-text   # embeddings
```
With less than ~10 GB of VRAM, use one of the two Qwen models for both settings, or keep them
split: grading then waits until an interview ends so the dialogue model is never swapped out
mid-interview (see [BENCHMARKS.md](BENCHMARKS.md#5-configuration-chosen)).

### 3. Install dependencies
```bash
npm --prefix server install
npm --prefix frontend install
pip install -r services/laya-engine/requirements.txt
```

### 4. Configure the API
```bash
cp server/.env.example server/.env   # then set JWT_SECRET and review the model names
```
[`server/.env.example`](server/.env.example) lists every variable the code reads. The important ones:

| Variable | Default | Purpose |
|---|---|---|
| `MONGODB_URI` | `mongodb://127.0.0.1:27017/prepai` | Database |
| `JWT_SECRET` | — | **Required** in production (the API refuses to start with the dev key) |
| `JWT_ACCESS_TTL_MINUTES` / `JWT_REFRESH_TTL_DAYS` | `15` / `30` | Access / refresh token lifetime |
| `CORS_ORIGINS` | `http://localhost:5174,…` | Allowed browser origins |
| `AI_PROVIDER` | `ollama` | `ollama`, `anthropic`, `gemini` or `openai` |
| `AI_MODEL` | `qwen2.5-coder:7b` | Live interviewer dialogue |
| `EVALUATION_MODEL` | `qwen3:8b` | Grading, reference packs, report prose |
| `AI_FALLBACK_PROVIDER` | — | Optional second provider |
| `EMBEDDING_MODEL` | `nomic-embed-text` | Embeddings |
| `LAYA_SERVICE_URL` | `http://127.0.0.1:7860` | Laya decision service |
| `LAYA_TIMEOUT_MS` / `LAYA_TURN_TIMEOUT_MS` | `30000` / `7000` | Background / live-turn budgets |

### 5. Prepare the database
```bash
npm --prefix server run seed       # question bank and the placement-officer account (never deletes data)
npm --prefix server run migrate    # idempotent schema/data migration to the v4 engine
node server/scripts/balance_option_positions.js   # spread correct answers over A–D (idempotent)
```
There are no demo students: students create their own accounts. The placement-officer login is `admin@prep.ai` /
`admin123`. Change it before any shared deployment.

### 6. Run the application
One command starts everything (and is safe to run again — anything already running is left alone):
```bash
npm run dev            # same as: npm run start:all
```
It starts Ollama, the Laya service, the API and the web app in order, waits until each one is healthy,
checks that the AI models in `server/.env` are installed, and prints a ✔/✘ summary. Open
http://localhost:5174 when it says "Everything is running". Ctrl+C stops what it started.

To run a single part instead: `npm run laya:start`, `npm run dev:server`, `npm run dev:web`.
If Ollama is not running, the app refuses to start an interview and tells you how to fix it,
instead of failing on your first answer.
`GET /api/v1/health` reports the API and database status; signed-in staff can check the models,
embeddings and Laya at `GET /api/v1/internal/ai/health` and see per-feature latency at
`GET /api/v1/internal/ai/metrics`.

---

## Testing & Quality Verification

| Suite | Command | Result (2026-10-07) |
|---|---|---|
| Server unit tests (evaluation, mastery, state machine, aptitude, resume, benchmark metrics, security helpers, job deferral) | `npm run test:server` | 39 / 39 passing |
| API integration (needs MongoDB, API, Laya and Ollama running) | `npm run test:integration` | 14 / 14 passing (~4 min) |
| Laya service | `npm run test:laya` | 7 / 7 passing |
| Frontend unit tests (Vitest) | `npm run test:frontend` | 17 / 17 passing |
| End-to-end (Playwright; app must be running) | `npm run test:e2e` | 9 / 9 passing |
| Type check + lint + production build | `npm --prefix frontend run build` and `npm run lint:frontend` | clean |

The integration suite creates its own `itest_*` accounts and deletes everything they own afterwards.

### Evaluation benchmark (120 cases)
```bash
npm --prefix server run benchmark                                   # configured grading model
npm --prefix server run benchmark:compare -- qwen2.5-coder:7b qwen3:8b
```
Measured results, per-category errors and the reasoning behind the model choice are in
[BENCHMARKS.md](BENCHMARKS.md). Headline (qwen3:8b, fusion): 85.8 % hiring-decision accuracy,
macro-F1 0.816, Pearson r 0.945 with expert scores, **0 % false positives**, MAE 16 points (the
pipeline grades below the experts on wrong answers).

---

## Documentation Index

For deep dives into the platform architecture, algorithms, and operations, refer to the following dedicated manuals:

1. [Architecture Overview](ARCHITECTURE.md) — System topology, 4-layer hybrid AI architecture, component boundaries, and request lifecycle.
2. [Laya Decision Engine](LAYA_INTEGRATION.md) — Non-autoregressive decision modeling with `convaiinnovations/laya`, microservice schemas, and evaluation fusion.
3. [Adaptive Progression Engine](ADAPTIVE_ENGINE.md) — Per-topic Bayesian EMA skill tracking, 5 normalized mastery bands, and difficulty transitions.
4. [120-Scenario Golden Benchmark](BENCHMARKS.md) — Full evaluation report, statistical metrics (MAE, Pearson r, F1), and model comparisons.
5. [AI Architecture & Abstraction](AI_ARCHITECTURE.md) — Provider abstraction layer, prompt engineering, vector embeddings, and fallback mechanics.
6. [Evaluation Engine](EVALUATION_ENGINE.md) — 7-dimension weighted rubric, scoring caps, evidence extraction, and conflict resolution.
7. [Model Providers Guide](MODEL_PROVIDERS.md) — Setting up Ollama, Claude, Gemini, OpenAI, Groq, and vLLM.
8. [Security & Hardening](SECURITY.md) — Token bucket rate limiting, prompt injection defense, PII scrubbing, and IDOR protection.
9. [Database Schema & Migrations](DATABASE.md) — MongoDB collections, schemas, indexes, and performance optimizations.
10. [REST API Reference](API.md) — Complete endpoint reference with payload and response schemas.
11. [Testing & Benchmarking](TESTING.md) — Test suites, quality gates, benchmark methodology, and synthetic test harness.
12. [Deployment & Operations](DEPLOYMENT.md) — Production deployment topologies (Docker, On-Prem, Cloud) and telemetry.
