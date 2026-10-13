# PrepAI — Implementation Plan (Spec v4.0 → current repository)

Source of truth: `PrepAI_Desktop_Web_Master_Project_Specification.docx` (v4.0, desktop-web only, Laya decision layer, local-first AI).
Constraint from the product owner: **the UI must not change** — layouts, styling and screens stay as they are; only the data behind them and API wiring change.

## 1. Repository audit (2026-10-07)

| Area | Spec requirement | Current state | Gap |
|---|---|---|---|
| Stack | FastAPI + PostgreSQL/pgvector + Redis + Celery | Express + MongoDB, no Docker/Postgres/Redis installed on this machine | Infra not available; keep Express/Mongo and map each spec component (see §2) |
| Provider abstraction | `AIProvider{healthCheck,generate,stream,getModelInfo}`, `DecisionProvider`, `EmbeddingProvider` | Ollama/Anthropic/Gemini/OpenAI providers exist (`generateChat`, `checkHealth`) | Missing `stream`, decision/embedding interfaces, central config, per-call observability |
| Laya | Generic `POST /internal/laya/decision`, 8 typed decisions, probabilities, `decision_version`, no fabricated fallback | Laya service loads once (CPU, ~4 s/call) and returns real probabilities; `/evaluate` only | Fallback invents decisions from word count; client timeout 4 s < real latency so it silently falls back; no generic endpoint / `increase_difficulty` / `decision_version` |
| Evaluation | Multi-signal fusion, conflict policy, `reviewRequired`, no length/keyword scoring, no silent fallback, audit metadata | Fusion function exists but **reports use a keyword/word-count rule scorer** (fixed 78/84 scores, fixed confidence 0.94, canned "sample answers") | Rewrite evaluator; persist evaluations + signals |
| Benchmark | 100+ labelled cases, accuracy, macro-F1, MAE, correlation, FP/FN, consistency, latency, structured-output failure | 120-case dataset with the right category mix; metrics computed on the rule scorer whose regexes copy dataset phrases (overfit) | Benchmark the real pipeline per system (Laya, LLM, fusion), store results |
| Mastery | Bands Beginner→Advanced, α-smoothing + adjustments, Easy/Medium/Hard per topic, deterministic transitions | EMA mastery, 3 named levels, untouched topics shown as 60/70 | Implement spec formula/bands/rules; aptitude topics too; no invented mastery |
| State machine | 10 states incl. BACKGROUND, RESUME, FEEDBACK; code validates phase/duplicates; counter-question strategies | 8 phases by turn count; probe logic never fed real evaluations | Implement spec states + strategy selection from Laya signals |
| Aptitude | Deterministic, numeric tolerance, server timing, per-topic mastery, adaptive difficulty, question versioning | Deterministic MCQ, server time used for results | Add tolerance, per-question attempts/durations, topic mastery, adaptive selection, versions, company-pattern metadata |
| Resume | PyMuPDF → structured extraction → schema validation → "never invent" → 6 spec ATS dims → hot spots | `pdf-parse` + regex, ATS dims differ from spec | PyMuPDF extraction in the Python service, LLM structured extraction verified against source text, spec dims, CRUD endpoints |
| Voice | Browser mic → VAD → STT → turn → LLM → TTS, barge-in, observable signals only, providers configurable | Browser STT + Edge TTS, barge-in in UI; report shows a "Confidence" score | Record per-turn speech signals (duration, latency, fillers, interruptions); drop psychological "confidence" dimension; STT/TTS provider config |
| Reports | Async job, question-by-question evidence, concepts, mastery changes, next difficulty, study plan | Synchronous rule-scorer evaluation at `/end` | In-process job queue with persisted status; full report schema |
| API | `/api/v1` spec routes, envelope, request ids | Most features exist under different paths | Add spec routes (aliases + new), `X-Request-Id`, envelope on new routes |
| Security | Short-lived JWT + refresh, CORS allowlist, audit logs, upload validation, IDOR, prompt-injection boundary, rate limits | IDOR/upload/rate limits fixed last round; 7-day access tokens; CORS reflects any origin; no audit log | 15-min access + refresh rotation (client auto-refresh), CORS allowlist, security headers, audit log |
| Observability | provider/model/feature/latency/tokens/retries/fallback/Laya confidence/conflicts | In-memory counters only | `model_executions` collection, p50/p95, `/internal/ai/health`, `/internal/ai/models` |
| Gamification | Leaderboard anonymised by default | Shows other students' names and readiness | Anonymise peers |
| Tests | Unit, integration, E2E, Laya, security | 20 server unit tests, 17 vitest, stale Playwright specs (12 failing) | Expand unit/integration tests, rewrite E2E specs for the current UI |

## 2. Architecture decisions for this machine

| Spec component | Implementation here | Reason |
|---|---|---|
| FastAPI API | Existing Express API (`server/`) | Working, tested codebase; spec says preserve correct functionality; no Python web stack in place for the API |
| PostgreSQL + pgvector | MongoDB collections mirroring the spec tables (`interview_turns`, `answer_evaluations`, `evaluation_signals`, `model_executions`, `benchmark_results`, `audit_logs`, `study_plans`, `interview_reports`, `aptitude_attempts`) | No Postgres/Docker installed; Mongo is running |
| Redis rate limit / cache | Existing in-process token bucket | No Redis; single API instance |
| Celery workers | In-process persisted job queue (`jobQueue`) with restart recovery | No broker available |
| Laya service | Existing Python FastAPI service (`services/laya-engine`, port 7860 configurable) | Already loads the model once |
| PyMuPDF | Added to the Python service as `/internal/resume/extract` | Spec-mandated extractor lives with the other Python ML code |

## 3. Execution phases

1. **Foundation** — central config, provider interfaces (generate/stream/health/modelInfo, embedding, decision), model-execution observability, request ids, security headers, CORS allowlist, audit log, JWT access/refresh rotation (+ client auto-refresh, no UI change).
2. **Laya** — generic decision endpoint with spec labels + `decision_version`, remove fabricated fallbacks (503 when not ready), realistic client timeout, decision provider, Python tests.
3. **Evaluation engine eval-v4** — input validation → reference concepts → embeddings → Laya → LLM evaluator (JSON schema, one retry) → evidence → conflict policy → deterministic weighted aggregation; explicit `EVALUATION_FAILED` instead of fake scores; persisted evaluations and signals; retry endpoint.
4. **Mastery & difficulty** — spec formula, bands, per-topic Easy/Medium/Hard, deterministic transitions, aptitude + interview topics.
5. **Interview engine** — spec states, deterministic phase progression, counter-question strategies from signals, versioned prompt files, resume/difficulty context, per-turn persistence and speech signals, background per-turn evaluation.
6. **Aptitude** — numeric tolerance, attempts with durations, topic mastery, adaptive question selection, question versions, company-pattern metadata.
7. **Resume** — PyMuPDF extraction, structured extraction verified against source, spec ATS dimensions, hot spots, CRUD + analyze.
8. **Reports & study plan** — async report job, full report schema, persisted study plans + spec routes.
9. **Dashboard / admin / API surface** — spec routes, admin drill-down & AI-quality analytics, anonymised leaderboard.
10. **Benchmark** — run Laya / LLM / fusion on the 120-case set, store metrics, admin-triggered run endpoint, honest README numbers.
11. **Tests** — unit + integration + Laya + rewritten Playwright E2E.
12. **Final verification** — full Playwright MCP run across every screen, fix, re-run, final report.

## 4. Status and known limitations (final, 2026-10-07)

All 12 phases are implemented and verified (see TESTING.md and BENCHMARKS.md).

- **Infrastructure substitutions**: Postgres/pgvector, Redis, Celery, LiveKit, Deepgram and ElevenLabs are
  not installed or keyed here. MongoDB collections, an in-process token bucket and a persisted
  in-process job queue are used instead; STT is the browser Web Speech API and TTS is Edge neural
  voices. A single API instance is assumed (rate limits and the job queue are per process).
- **Hardware**: an RTX 3050 6 GB cannot hold the dialogue and grading models together (a swap costs ~12 s).
  Grading for an interview is therefore deferred until it ends; the report for a 4-answer interview is
  ready about 3 minutes later. Laya runs on CPU (2–4 s per decision batch, 7 s budget on live turns;
  when it times out the turn uses the default strategy and the signal is recorded as unavailable).
- **Grading quality**: 85.8 % hiring-decision accuracy, 0 % false positives, but the pipeline under-credits
  some fully correct answers (30 % called partially correct) and scores wrong answers below the experts
  (MAE 16). The benchmark has 120 answers to only 6 questions, so it cannot prove generalisation;
  nothing was tuned to it.
- **Voice**: live microphone, VAD and barge-in cannot be exercised by browser automation; the typed-answer
  path, the speech-metrics plumbing and the TTS audio endpoint were verified.
- **Question bank**: 21 seeded aptitude questions (balanced answer positions); admins add more by CSV.
- **Badges**: the "STAR Champion" badge reads a legacy STAR-compliance score that eval-v4 no longer
  produces, so it cannot currently unlock.
- **Public TTS endpoint**: `/api/v1/voice/synthesize` is unauthenticated (an `<audio>` element cannot send
  a bearer token); it is rate-limited and capped at 1 500 characters.
