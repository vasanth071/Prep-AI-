# PrepAI — REST API Specification

## 1. Global API Standards

- **Base URL**: `http://127.0.0.1:7857/api`
- **Content-Type**: `application/json` (except file uploads which use `multipart/form-data`)
- **Authentication**: JWT Bearer Token passed via HTTP Header:
  `Authorization: Bearer <token>`
- **Response Structure**:
  ```json
  {
    "success": true,
    "data": { ... },
    "message": "Optional status message"
  }
  ```
- **Error Structure**:
  ```json
  {
    "success": false,
    "error": "Error identifier",
    "message": "Human-readable explanation of error"
  }
  ```

---

## 2. Authentication Endpoints

### 2.1. Register Candidate
- **Method & Path**: `POST /api/auth/register`
- **Rate Limit**: 5 req / min (`authLimiter`)
- **Request Body**:
  ```json
  {
    "name": "Alex Mercer",
    "email": "alex.mercer@example.com",
    "password": "Password123!",
    "targetRole": "Backend Systems Engineer"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "success": true,
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "6701a2f4b931980012e84120",
      "name": "Alex Mercer",
      "email": "alex.mercer@example.com",
      "role": "student"
    }
  }
  ```

### 2.2. Login Candidate
- **Method & Path**: `POST /api/auth/login`
- **Rate Limit**: 5 req / min (`authLimiter`)
- **Request Body**:
  ```json
  {
    "email": "alex.mercer@example.com",
    "password": "Password123!"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "success": true,
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": { ... }
  }
  ```

### 2.3. Get Authenticated Profile
- **Method & Path**: `GET /api/auth/me`
- **Headers**: `Authorization: Bearer <token>`
- **Response (200 OK)**: User profile including aggregated interview and aptitude statistics.

---

## 3. Interview Engine Endpoints

### 3.1. Initialize Interview Session
- **Method & Path**: `POST /api/interview/start`
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "role": "Full Stack Engineer",
    "difficulty": "intermediate",
    "targetCompany": "FAANG / Tier-1"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "success": true,
    "interviewId": "6701a345b931980012e84125",
    "phase": "INTRODUCTION",
    "firstQuestion": "Hello Alex! Welcome to your technical interview for Full Stack Engineer. Could you introduce yourself and describe a challenging distributed systems problem you recently resolved?"
  }
  ```

### 3.2. Submit Answer & Receive 7-Dimension Rubric Evaluation
- **Method & Path**: `POST /api/interview/:id/answer`
- **Headers**: `Authorization: Bearer <token>`
- **Rate Limit**: 15 req / min (`aiLimiter`)
- **Request Body**:
  ```json
  {
    "answer": "In our microservices architecture, we utilized Redis as a write-through cache backed by Postgres. To prevent cache stampede during cold restarts, we implemented probabilistic early expiration with mutex locks on key regeneration.",
    "durationSeconds": 45
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "success": true,
    "turnNumber": 3,
    "evaluation": {
      "scores": {
        "correctness": 92,
        "relevance": 95,
        "technicalDepth": 88,
        "completeness": 85,
        "reasoning": 90,
        "clarity": 88,
        "communication": 90
      },
      "compositeScore": 89,
      "verdict": "CORRECT",
      "candidateEvidenceQuote": "we implemented probabilistic early expiration with mutex locks on key regeneration",
      "missingConcepts": ["Redis Sentinel high availability", "Memory eviction policy config (volatile-lru)"],
      "feedback": "Strong technical articulation of cache stampede mitigations using mutex locking.",
      "sampleGoodAnswer": "At scale, cache stampedes are addressed using distributed locks (Redlock) or probabilistic early expiration (XFetch algorithm)...",
      "improvementTask": "Review Redis Sentinel failover quorum mechanisms and volatile-lru memory policies."
    },
    "nextQuestion": "Explain how you ensure consistency between your Redis cache and Postgres primary database when handling network partition events.",
    "nextPhase": "TECHNICAL_DEPTH",
    "isFollowUpProbe": false,
    "isCompleted": false
  }
  ```

### 3.3. Complete Interview & Finalize Report
- **Method & Path**: `POST /api/interview/:id/complete`
- **Headers**: `Authorization: Bearer <token>`
- **Response (200 OK)**: Triggers report generation and returns summary report ID.

---

## 4. Reports & Performance Analytics

### 4.1. Get Latest Performance Report
- **Method & Path**: `GET /api/report/latest`
- **Headers**: `Authorization: Bearer <token>`
- **Response (200 OK)**: Returns the latest performance report for the logged-in candidate, including 7-dimension rubric scores, verbatim evidence quotes, missing concepts, exemplar answers, and actionable tasks.

### 4.2. Download Official PDF Report
- **Method & Path**: `GET /api/report/:id/pdf`
- **Headers**: `Authorization: Bearer <token>`
- **Rate Limit**: 6 req / min (`reportLimiter`)
- **Response**: Binary PDF stream (`Content-Type: application/pdf`).

---

## 5. Voice & Speech Synthesis (Edge-TTS)

### 5.1. Synthesize Human Speech
- **Method & Path**: `POST /api/voice/synthesize`
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "text": "Could you explain the difference between optimistic and pessimistic locking in database transactions?",
    "voice": "en-US-AndrewMultilingualNeural",
    "rate": "+0%"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "success": true,
    "audioBase64": "data:audio/mp3;base64,//uQxAAAAANIAAAAAExBTUUzLjE...",
    "durationSeconds": 4.2
  }
  ```

---

## 6. Health & Diagnostic Endpoints

### 6.1. Platform Health Check
- **Method & Path**: `GET /api/health`
- **Response (200 OK)**:
  ```json
  {
    "status": "healthy",
    "timestamp": "2026-10-04T10:15:30.000Z",
    "uptime": 14205.2,
    "database": "connected"
  }
  ```

### 6.2. AI Provider Diagnostics
- **Method & Path**: `GET /api/health/ai`
- **Response (200 OK)**:
  ```json
  {
    "activeProvider": "ollama",
    "model": "qwen2.5-coder:7b",
    "embeddingModel": "nomic-embed-text",
    "daemonReachable": true,
    "availableLocalModels": ["qwen2.5-coder:7b", "llama3.2:latest", "nomic-embed-text:latest"]
  }
  ```
