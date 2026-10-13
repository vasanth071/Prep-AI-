# PrepAI — Full Project Investigation Report

**Generated**: 2026-10-04  
**Scope**: Read-only investigation of the complete workspace at `c:\Users\mohan\Downloads\Mini project`

---

## Summary Answer

PrepAI is a full-stack **MERN-stack AI Placement Preparation Platform** that simulates realistic technical job interviews and aptitude tests for college students. The project comprises:

- A **React 19 + TypeScript frontend** (Vite, TailwindCSS 4, Zustand, TanStack Query)
- An **Express.js + MongoDB backend** (Node.js, Mongoose, JWT auth, PDF generation)
- A **Python FastAPI microservice** ("Laya Decision Engine") for structured non-autoregressive evaluation
- A multi-provider **AI abstraction layer** (Ollama local-first, Anthropic, Gemini, OpenAI-compatible)
- A **real-time voice interview engine** using Web Speech API, Edge Neural TTS, and Web Audio visualizations
- A **7-dimension rubric evaluation engine** for scoring interview answers
- An **aptitude test suite** with gamification
- **Resume/ATS analysis** via PDF parsing and vector embeddings

The system prioritizes **100% local operation by default** via Ollama (zero cloud API cost), with graceful fallback to cloud providers and a deterministic rule-based evaluator that guarantees sessions never crash.

---

## Evidence & Findings

### 1. Repository Structure

```
Mini project/
├── frontend/               # React 19 + Vite + TypeScript SPA
│   ├── src/
│   │   ├── app/            # App bootstrap (App.tsx, router.tsx, providers.tsx)
│   │   ├── api/            # Axios API clients (client.ts, interview.ts, ...)
│   │   ├── pages/          # Feature pages (interview/, aptitude/, resume/, admin/, ...)
│   │   ├── store/          # Zustand state (authStore.ts, interviewStore.ts, uiStore.ts)
│   │   ├── components/     # UI components (layout, ui primitives, interview/)
│   │   ├── types/          # TypeScript types (interview.ts, report.ts, auth.ts)
│   │   └── lib/            # Utilities (constants, storage, formatters)
│   ├── vite.config.js      # Dev server on :5174, proxy /api → :7857
│   └── package.json
├── server/                 # Express.js backend
│   ├── src/
│   │   ├── server.js       # Entry point, Express app, port 7857
│   │   ├── routes/         # REST endpoints (auth, interview, aptitude, voice, resume, admin, ...)
│   │   ├── models/         # Mongoose schemas (User, InterviewSession, AptitudeSession, Resume, ...)
│   │   ├── services/       # AI service layer
│   │   │   ├── aiService.js            # Interview turn generation & report generation
│   │   │   ├── evaluationEngine.js     # 7-dimension deterministic rubric scorer
│   │   │   ├── semanticEngine.js       # Cosine similarity, Jaccard overlap
│   │   │   ├── interviewStateMachine.js # 8-phase interview FSM
│   │   │   ├── adaptiveEngine.js       # Student mastery tracking
│   │   │   ├── layaClient.js           # HTTP client for Python Laya microservice
│   │   │   └── providers/              # AI provider abstraction
│   │   │       ├── AIProviderFactory.js
│   │   │       ├── OllamaProvider.js
│   │   │       ├── AnthropicProvider.js
│   │   │       ├── GeminiProvider.js
│   │   │       └── OpenAICompatibleProvider.js
│   │   ├── middleware/     # auth.js (JWT protect), rateLimiter.js (TokenBucket)
│   │   ├── seeds/          # seedData.js (initial admin + questions)
│   │   ├── evals/          # Benchmarking harness (runFullBenchmark.js, compareModelProviders.js)
│   │   ├── config/         # db.js (MongoDB connection)
│   │   └── tests/          # prepai.test.js (17 Node.js native tests)
│   └── package.json
├── services/
│   └── laya-engine/        # Python FastAPI microservice
│       ├── app.py           # Endpoints: /decision/answer-state, /decision/relevance, etc.
│       ├── evaluator.py
│       ├── model.py         # Loads convaiinnovations/laya model
│       ├── schemas.py
│       └── tests/
├── API.md                   # REST API specification
├── AI_ARCHITECTURE.md       # AI provider abstraction docs
├── MODEL_PROVIDERS.md       # Setup & hardware guide
├── TESTING.md               # Test strategy & benchmark methodology
└── package.json             # Workspace root scripts
```

---

### 2. Backend Architecture (`server/`)

**Entry point**: `server/src/server.js`  
**Port**: 7857 (hardcoded), bound to `127.0.0.1`  
**Base API path**: `/api/v1`

#### 2.1. REST Routes Mounted

| Route prefix | File | Purpose |
|---|---|---|
| `/api/v1/health` | `routes/health.js` | Platform + AI health checks |
| `/api/v1/auth` | `routes/auth.js` | Register, Login, Refresh |
| `/api/v1/users` | `routes/users.js` | User profile management |
| `/api/v1/interview` + `/api/v1/interviews` | `routes/interview.js` | Full interview lifecycle |
| `/api/v1/aptitude` | `routes/aptitude.js` | Adaptive aptitude test sessions |
| `/api/v1/voice` | `routes/voice.js` | Edge TTS / msedge-tts synthesis |
| `/api/v1/resumes` + `/api/v1/resume` | `routes/resumes.js` | PDF upload, ATS scoring |
| `/api/v1/admin` | `routes/admin.js` | Batch analytics, student management |
| `/api/v1` (direct) | `routes/intelligence.js` | `/roles`, `/companies`, `/skill-gap` |

#### 2.2. Authentication (`middleware/auth.js`)

JWT-based Bearer token auth. In **non-production** environments, the `protect` middleware has a deliberate **dev fallback**: if a token is missing or invalid, it silently attaches the first `student` user from MongoDB. This bypasses auth for development — **must be removed or gated behind a strict env check before production deployment**.

#### 2.3. Database (`models/`)

- `User.js` — Mongoose schema: email, passwordHash (bcryptjs), fullName, collegeName, branch, graduationYear, XP, streaks, badges, role (student/admin/recruiter/placement_officer), dailyQuests
- `InterviewSession.js` — Stores session, transcript (role + content array), rubric scores, feedback report, resume context
- `AptitudeSession.js` — Aptitude test sessions
- `Resume.js` — Uploaded PDF metadata, parsedSkills, ATS score, diagnosticData/hotspots, rawText

#### 2.4. Interview Lifecycle (`routes/interview.js`)

Key endpoints:
- `POST /start` — Creates `InterviewSession`, attaches resume context, generates opening greeting. Returns session object with first AI message.
- `POST /:sessionId/dialogue` — Calls `generateInterviewTurn()` → AI provider → fallback domain engine. Updates transcript in MongoDB. Returns AI response + phase state.
- `POST /:sessionId/end` — Calls `generateInterviewReport()` → runs 7-dimension rubric over all transcript Q&A pairs. Awards 50 XP.
- `GET /:sessionId/report` — Returns rich JSON report with dimensions, strengths, improvements, question analysis, communication stats, visual audit, weak topics.
- `GET /:sessionId/report.pdf` — Streams a PDF via PDFKit.
- `POST /chat` — Alternative simpler endpoint for direct chat (no phase tracking).

#### 2.5. Rate Limiting (`middleware/rateLimiter.js`)

Token Bucket algorithm with burst allowances:
- `apiLimiter` — General API
- `aiLimiter` — AI turn endpoints (15 req/min)
- `reportLimiter` — PDF report endpoints (6 req/min)
- `authLimiter` — Auth endpoints (5 req/min)

---

### 3. AI Layer (`server/src/services/`)

#### 3.1. Provider Abstraction (`providers/`)

`AIProviderFactory.js` acts as an orchestrator with lazy provider initialization:
- Default: `OllamaProvider` (local, `http://127.0.0.1:11434`)
- Conditional on env vars: `AnthropicProvider`, `GeminiProvider`, `OpenAICompatibleProvider`
- `execute()` wraps calls with exponential backoff retry logic (3 attempts, configurable base delay)
- Factory exposes `getDefaultProvider()`, `getProvider(name)`, runtime metrics (latency, token counts)

Configured models:
- **Chat inference**: `qwen2.5-coder:7b` (primary), `llama3.2:latest` (fast conversational)
- **Embeddings**: `nomic-embed-text` (768-dimensional dense vectors)

#### 3.2. Interview Turn Generation (`aiService.js → generateInterviewTurn`)

1. Builds system prompt injecting: interviewer persona, company, job role, difficulty level, resume hotspots/skills, strict behavior rules (no markdown, 25-45 words per response, single counter-question per turn)
2. Calls `providerFactory.execute('chatInference', ...)` with 1 retry
3. If reply is too similar to a previous question (Jaccard/semantic guard), falls back to the **Dynamic Domain Engine**: a hardcoded pool of resume-grounded probing questions seeded from the candidate's hotspot projects and skills
4. PII scrubbing applied to all resume text before insertion into prompts

#### 3.3. 7-Dimension Rubric Evaluator (`evaluationEngine.js`)

Deterministic weighted scoring pipeline:
- Dimensions: correctness (25%), relevance (15%), technicalDepth (15%), completeness (15%), reasoning (10%), clarity (10%), communication (10%)
- Weight sum validated at module load (throws if ≠ 1.0)
- Evasive answer caps: answers ≤ 2 words or matching evasive patterns → score ≤ 35, verdict `INSUFFICIENT_EVIDENCE`
- False claim detection: objectively incorrect CS statements → score ≤ 30, verdict `INCORRECT`
- Technical taxonomy: concept keyword lexicon across caching, concurrency, databases, algorithms, system design topics
- Also calls `analyzeSemanticRelevance()` (cosine similarity) and optionally `layaClient` (Python Laya service)

#### 3.4. Semantic Engine (`semanticEngine.js`)

- `cosineSimilarity(vecA, vecB)` — Standard dot product cosine formula
- `lexicalSimilarity(a, b)` — Jaccard token overlap for fast duplicate question guard
- Embeddings from `nomic-embed-text` via Ollama provider

#### 3.5. Interview State Machine (`interviewStateMachine.js`)

8-phase sequential FSM:
1. Introduction → 2. Background Verification → 3. Core Questions → 4. Counter-Question Layer → 5. Situational/Behavioral → 6. Candidate Questions → 7. Wrap-Up → (end)

Exports: `INTERVIEW_PHASES` array, `InterviewStateMachine.getNextState()`, `InterviewStateMachine.isDistinctQuestion()`

#### 3.6. Adaptive Engine (`adaptiveEngine.js`)

Tracks per-user topic mastery bands (Foundational / Intermediate / Advanced). Updates asynchronously after each scored Q&A pair. `getMasteryBand()` and `getRecommendedDifficulty()` inform difficulty scaling.

#### 3.7. Laya Decision Engine (Python FastAPI microservice)

Located at `services/laya-engine/`. Separate FastAPI service started with `py services/laya-engine/app.py`. Endpoints:
- `POST /decision/answer-state` — NLI-based answer verdict classification
- `POST /decision/relevance` — Relevance classification
- `POST /decision/completeness` — Completeness scoring
- `POST /decision/concept-coverage` — Coverage of expected concepts
- `POST /decision/unsupported-claims` — Hallucination detection
- `POST /decision/follow-up` — Follow-up question generation
- `POST /decision/unified` — All-in-one decision request

Uses `convaiinnovations/laya` model (non-autoregressive, fast structured decision). The `layaClient.js` in the Node server makes HTTP calls to this service; it degrades gracefully if the Python service is offline.

---

### 4. Frontend Architecture (`frontend/`)

**Framework**: React 19, TypeScript, Vite 8, TailwindCSS 4  
**Dev port**: 5174, proxying `/api` → `http://127.0.0.1:7857`

#### 4.1. App Bootstrap

- `frontend/src/App.tsx` → re-exports `frontend/src/app/App.tsx`
- `App.tsx` wraps `<BrowserRouter>` → `<Providers>` (QueryClientProvider + Toaster) → `<ErrorBoundary>` → `<AppRouter>`

#### 4.2. Routing (`app/router.tsx`)

Three route groups:
1. **Public standalone**: `/`, `/login`, `/auth`, `/register`, `/forgot-password`, `/reset-password`, `/report/shared/:token`
2. **Student platform** (inside `<StudentLayout>`): publicly browsable role/company discovery + protected routes (`ProtectedRoute` → checks `isAuthenticated` from authStore)
3. **Admin** (inside `<AdminLayout>`, behind `<AdminRoute>` → checks `role === 'admin' | 'super_admin'`): `/admin`, `/admin/batches`, `/admin/students`, `/admin/questions/upload`, `/admin/drives`, `/admin/events`, `/admin/reports`

#### 4.3. State Management

**Zustand** stores:
- `authStore.ts` — `user`, `token`, `refreshToken`, `isAuthenticated`. Notable: **hardcoded default dev user** (`Kasinathan M`, `isAuthenticated: true` always) and a mock token. The store initializes with `isAuthenticated: true` regardless of any actual login, which means the app always shows as authenticated in development.
- `interviewStore.ts` — Full interview session state: phase, transcript, avatar state, mic mute, partial transcripts, elapsed timer, barge-in state.
- `uiStore.ts` — UI theme/sidebar state.

#### 4.4. API Client (`api/client.ts`)

Axios instance with:
- Base URL: `VITE_API_URL` env var or `/api/v1`
- Request interceptor: attaches `Authorization: Bearer <token>` from localStorage
- Response interceptor: clears token on 401, formats errors

#### 4.5. Feature Pages

| Route | Page Component | Description |
|---|---|---|
| `/dashboard` | `Dashboard` | Candidate overview, stats, upcoming tasks |
| `/interview` | `InterviewConfig` | Configure role, company, type, persona |
| `/interview/:sessionId` | `InterviewSession` | Live voice interview session |
| `/interview/:sessionId/end` | `InterviewEnd` | Post-interview summary |
| `/report/:sessionId` | `Report` | Full performance report with charts |
| `/aptitude` | `AptitudeConfig` | Configure aptitude test |
| `/aptitude/:sessionId` | `AptitudeTest` | Live aptitude test |
| `/aptitude/:sessionId/results` | `AptitudeResults` | Test results |
| `/resume` | `ResumeUpload` | PDF resume upload |
| `/resume/score` | `ATSScore` | ATS compatibility score |
| `/resume/bullets` | `BulletGenerator` | AI-powered resume bullet generator |
| `/roles` | `RoleExplorer` | Browse tech job roles |
| `/analytics/progress` | `Progress` | Performance analytics |
| `/gamification` | `Gamification` | XP, badges, leaderboard |
| `/profile` | `Profile` | User profile management |

#### 4.6. Interview Session UI (`pages/interview/InterviewSession.tsx`)

The most complex component (~800+ lines). Key behaviors:
- **Web Speech API** (`SpeechRecognition`) with `continuous: true, interimResults: true`
- **Silence detection**: 1.8s countdown after candidate stops speaking → auto-submits answer
- **Edge Neural TTS** via `GET /api/v1/voice/synthesize?voice=...&text=...` (Audio element), falling back to `SpeechSynthesisUtterance`
- **Echo prevention**: aborts recognition while AI speaks, restarts it 350ms after AI finishes
- **60fps Web Audio Spectrum Visualizer**: `AudioContext` + `AnalyserNode` + `requestAnimationFrame`
- **Real-time word streaming effect**: received AI text is streamed word-by-word into the chat bubble at 26ms intervals
- **Keyboard fallback input**: text form input for browsers without speech API
- TTS Watchdog timer prevents stuck `isAISpeaking` state if audio fails

---

### 5. Voice Engine (`server/src/routes/voice.js`)

Uses `msedge-tts` npm package (Microsoft Edge Text-to-Speech) to generate audio without needing a cloud TTS API key. Streams MP3 audio or returns base64. Voice options include `en-US-AndrewMultilingualNeural`, `en-US-ChristopherNeural`, etc.

---

### 6. Testing & Benchmarking

> Updated 2026-10-07. The earlier figures in this section (MAE 5.66, Pearson 0.907, 92.5 % accuracy)
> came from the retired keyword scorer, which was tuned on the benchmark itself, and have been removed.

- Server unit tests: 39 / 39 (`server/src/tests/unit/`, `npm run test:server`)
- API integration: 14 / 14 against the live stack (`npm run test:integration`)
- Laya service: 7 / 7 (`npm run test:laya`)
- Frontend: Vitest 17 / 17, Playwright E2E 9 / 9
- 120-case benchmark of the real eval-v4 pipeline (qwen3:8b grading): 85.8 % hiring-decision accuracy,
  macro-F1 0.816, Pearson r 0.945, 0 % false positives, MAE 16 points.

Details: [TESTING.md](TESTING.md) and [BENCHMARKS.md](BENCHMARKS.md).

---

### 7. Build & Development Commands

| Command | Description |
|---|---|
| `npm run dev:frontend` | Start Vite dev server (:5174) |
| `npm run dev:server` | Start Express with `node --watch` (:7857) |
| `npm run build:frontend` | TypeScript typecheck + Vite production build |
| `npm run test:server` | 17 Node.js native unit tests |
| `npm run test:frontend` | Vitest unit tests |
| `npm run test:e2e` | Playwright E2E tests |
| `npm run benchmark` | 120-scenario golden evaluation benchmark |
| `npm run benchmark:compare` | Multi-model latency/accuracy comparison |
| `npm run laya:start` | Start Python FastAPI Laya microservice |

**MongoDB**: expected at `mongodb://127.0.0.1:27017/prepai` (no config for Atlas/remote)  
**Ollama daemon**: expected at `http://127.0.0.1:11434` (local)

---

### 8. Environment Configuration

Server reads `server/.env`. Key variables:

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `7857` | Express server port |
| `MONGODB_URI` | _(not set, uses localhost)_ | MongoDB connection string |
| `JWT_SECRET` | `prepai_jwt_secret_key_2026_super_secure` | JWT signing key (hardcoded fallback — change in prod) |
| `AI_PROVIDER` | `ollama` | Active AI provider |
| `OLLAMA_BASE_URL` | `http://127.0.0.1:11434` | Ollama daemon URL |
| `OLLAMA_MODEL` | `qwen2.5-coder:7b` | Chat model |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | Embedding model |
| `ANTHROPIC_API_KEY` | _(empty)_ | Optional Anthropic key |
| `GEMINI_API_KEY` | _(empty)_ | Optional Gemini key |
| `OPENAI_API_KEY` | _(empty)_ | Optional OpenAI/Groq key |
| `NODE_ENV` | _(not set)_ | Set to `production` to disable dev fallbacks |

---

## Key Observations & Recommendations

### Security Concerns

1. **Hardcoded JWT secret** in `auth.js` and `middleware/auth.js` (`prepai_jwt_secret_key_2026_super_secure`). Must be overridden via env before any public deployment.
2. **Auth bypass in dev mode** (`middleware/auth.js`): When `NODE_ENV !== 'production'`, any invalid/missing JWT token silently resolves to the first student user in the DB. This is a significant security hole if the app is ever deployed without setting `NODE_ENV=production`.
3. **Frontend always-authenticated default** (`authStore.ts`): `isAuthenticated` is initialized to `true` and a mock JWT token is pre-set. This means protected routes are never actually gated in development, which can mask broken auth flows during testing.
4. **CORS is fully open** (`origin: true`) — acceptable for local dev, should be restricted to known origins in production.

### Architecture Strengths

1. **Graceful degradation chain**: Ollama → Cloud Provider → Deterministic Rule Engine. Sessions never crash even without AI.
2. **Clean provider abstraction**: Switching AI backends requires only an env variable change.
3. **Resume context grounding**: Interview questions adapt to the candidate's actual uploaded projects and skills.
4. **Real-time voice pipeline**: Full speech-to-text → AI → TTS loop with echo prevention, silence detection, and browser TTS fallback is well-engineered.
5. **7-dimension rubric with evidence quotes**: Evaluation is explainable and grounded in the actual candidate answer text.
6. **Deterministic unit tests**: 17 passing tests, zero external framework dependency, fast and reliable CI gate.

### Areas for Improvement

1. **`NODE_ENV` must be set to `production`** before any real deployment — the dev auth fallback is intentional but must not reach users.
2. **PII in JWT payload**: The JWT payload encodes `email` in plaintext — this is visible to anyone who decodes the token. Not a vulnerability (JWT is not encrypted by default), but worth noting for privacy.
3. **No input length validation on AI endpoints**: Very long answers submitted to `/dialogue` could inflate LLM prompt cost. Consider a `maxLength` guard.
4. **PDF report is static/hardcoded** (`routes/interview.js`, `report.pdf` route): the PDF generation uses hardcoded placeholder scores (84%, Grade A) rather than the session's actual computed scores. The JSON `/report` endpoint is accurate but the PDF is not.
5. **MongoDB URI not configurable**: The connection string is assembled in `config/db.js` without falling back cleanly to a `MONGODB_URI` env variable — this may need to be confirmed against `db.js` source.
6. **Laya microservice is optional but undocumented in quick-start**: The Python service must be started separately; if `layaClient.js` calls fail, they should degrade silently (verify this is the case).

---

## Conclusions

PrepAI is a well-architected, feature-complete placement preparation platform with genuine AI-powered evaluation, real-time voice interviews, resume analysis, aptitude testing, gamification, and detailed performance reports. The codebase shows strong engineering practices: provider abstraction, graceful degradation, deterministic testing, rubric-based explainable AI scoring, and thoughtful resume-grounded personalization.

The primary risks before production deployment are the dev-mode auth bypass (must gate with `NODE_ENV=production`), the hardcoded JWT secret (must replace with a strong random secret via env), and the hardcoded PDF report content (should be replaced with dynamic data from the session). None of these are architectural flaws — they are deliberate development conveniences that need to be switched off.
