# PrepAI — Database Architecture & Schema Specification

## 1. Database Overview

PrepAI uses **MongoDB (v6.0+)** via **Mongoose ORM** as its primary persistent document store. The schema design balances strict relational referential integrity with flexible document embedding for high-throughput interview turn capture and multidimensional rubric scoring.

```
                    ┌─────────────────────────┐
                    │      User Document      │
                    └────────────┬────────────┘
                                 │ 1:N
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│  Interview    │        │ TopicMastery  │        │ AptitudeTest  │
│  Sessions     │        │  (Per Topic)  │        │   Sessions    │
└───────┬───────┘        └───────────────┘        └───────────────┘
        │ 1:1
        ▼
┌───────────────┐
│ Comprehensive │
│ Performance   │
│ Report        │
└───────────────┘
```

---

## 2. Core Collections & Schema Definitions

### 2.1. `User` Schema (`server/src/models/User.js`)
Represents the authenticated candidate profile, credential hashes, target roles, and aggregate statistics.

```javascript
const userSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true, index: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ['student', 'admin', 'recruiter'], default: 'student' },
  targetRole: { type: String, default: 'Full Stack Engineer' },
  experienceLevel: { type: String, enum: ['entry', 'intermediate', 'senior'], default: 'entry' },
  skills: [{ type: String, trim: true }],
  stats: {
    totalInterviews: { type: Number, default: 0 },
    avgScore: { type: Number, default: 0 },
    aptitudeTestsTaken: { type: Number, default: 0 },
    resumesAnalyzed: { type: Number, default: 0 }
  },
  createdAt: { type: Date, default: Date.now }
});
```

### 2.2. `Interview` Schema (`server/src/models/Interview.js`)
Captures full multi-turn conversational interview sessions, 7-dimension rubric assessments, anti-hallucination verdict states, and candidate evidence quotes.

```javascript
const turnSchema = new mongoose.Schema({
  turnNumber: { type: Number, required: true },
  phase: { 
    type: String, 
    enum: [
      'INTRODUCTION', 'RESUME_BACKGROUND', 'TECHNICAL_BASICS', 
      'TECHNICAL_DEPTH', 'PROBLEM_SOLVING', 'SYSTEM_SCENARIO', 
      'BEHAVIORAL_HR', 'FINAL_WRAPUP'
    ],
    default: 'TECHNICAL_BASICS'
  },
  question: { type: String, required: true },
  expectedConcepts: [{ type: String }],
  candidateAnswer: { type: String, default: '' },
  audioUrl: { type: String },
  durationSeconds: { type: Number, default: 0 },
  
  // 7-Dimension Rubric Evaluation
  evaluation: {
    scores: {
      correctness: { type: Number, min: 0, max: 100, default: 0 },
      relevance: { type: Number, min: 0, max: 100, default: 0 },
      technicalDepth: { type: Number, min: 0, max: 100, default: 0 },
      completeness: { type: Number, min: 0, max: 100, default: 0 },
      reasoning: { type: Number, min: 0, max: 100, default: 0 },
      clarity: { type: Number, min: 0, max: 100, default: 0 },
      communication: { type: Number, min: 0, max: 100, default: 0 }
    },
    compositeScore: { type: Number, min: 0, max: 100, default: 0 },
    verdict: {
      type: String,
      enum: ['CORRECT', 'PARTIALLY_CORRECT', 'INCORRECT', 'IRRELEVANT', 'INSUFFICIENT_EVIDENCE'],
      default: 'INSUFFICIENT_EVIDENCE'
    },
    candidateEvidenceQuote: { type: String, default: '' },
    missingConcepts: [{ type: String }],
    feedback: { type: String, default: '' },
    sampleGoodAnswer: { type: String, default: '' },
    improvementTask: { type: String, default: '' }
  },
  
  isFollowUpProbe: { type: Boolean, default: false },
  probeReason: { type: String },
  createdAt: { type: Date, default: Date.now }
});

const interviewSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  role: { type: String, required: true },
  targetCompany: { type: String, default: 'General Tech' },
  difficulty: { type: String, enum: ['entry', 'intermediate', 'senior', 'staff'], default: 'intermediate' },
  status: { type: String, enum: ['in_progress', 'completed', 'abandoned'], default: 'in_progress', index: true },
  currentTurn: { type: Number, default: 0 },
  turns: [turnSchema],
  overallScore: { type: Number, default: 0 },
  startedAt: { type: Date, default: Date.now },
  completedAt: { type: Date }
});
```

### 2.3. `TopicMastery` Schema (`server/src/models/TopicMastery.js`)
Tracks the student's longitudinal progression and Exponential Moving Average across distinct engineering topics.

```javascript
const topicMasterySchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  topic: { type: String, required: true },
  category: { type: String, enum: ['Technical', 'Algorithms', 'System Design', 'Behavioral'], default: 'Technical' },
  attempts: { type: Number, default: 0 },
  successAttempts: { type: Number, default: 0 },
  averageScore: { type: Number, default: 0 },
  emaScore: { type: Number, default: 50 }, // Exponential Moving Average (alpha = 0.35)
  masteryBand: { 
    type: String, 
    enum: ['Beginner', 'Developing', 'Intermediate', 'Strong', 'Advanced'], 
    default: 'Intermediate' 
  },
  lastAssessedAt: { type: Date, default: Date.now }
});

// Compound unique index ensuring one record per user-topic combination
topicMasterySchema.index({ userId: 1, topic: 1 }, { unique: true });
```

### 2.4. `Report` Schema (`server/src/models/Report.js`)
Aggregated executive performance report generated upon session completion.

```javascript
const reportSchema = new mongoose.Schema({
  interviewId: { type: mongoose.Schema.Types.ObjectId, ref: 'Interview', required: true, unique: true, index: true },
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true, index: true },
  role: { type: String, required: true },
  overallScore: { type: Number, required: true },
  technicalReadinessIndex: { type: Number, required: true }, // 0-100%
  categoryBreakdown: {
    correctness: Number,
    relevance: Number,
    technicalDepth: Number,
    completeness: Number,
    reasoning: Number,
    clarity: Number,
    communication: Number
  },
  keyStrengths: [String],
  growthAreas: [String],
  actionPlan: [{
    topic: String,
    task: String,
    estimatedMinutes: Number
  }],
  generatedAt: { type: Date, default: Date.now }
});
```

---

## 3. Database Indexing Strategy

To guarantee sub-10ms query execution in production, the following compound and secondary indexes are applied:

| Collection | Index Keys | Type | Purpose |
| :--- | :--- | :--- | :--- |
| `users` | `{ email: 1 }` | Unique B-Tree | High-speed authentication lookup |
| `interviews` | `{ userId: 1, createdAt: -1 }` | Compound | Fast candidate dashboard interview history retrieval |
| `interviews` | `{ status: 1 }` | Secondary | Filter active vs. completed sessions |
| `topicmasteries` | `{ userId: 1, topic: 1 }` | Compound Unique | Atomic upsert during adaptive engine score update |
| `topicmasteries` | `{ userId: 1, masteryBand: 1 }` | Compound | Topic strength/weakness dashboard aggregation |
| `reports` | `{ interviewId: 1 }` | Unique | One-to-one report resolution |
| `reports` | `{ userId: 1, generatedAt: -1 }` | Compound | Fetch candidate's latest placement readiness report |

---

## 4. Seeding & Data Migration

A complete database seed script is provided at `server/src/seeds/seedData.js`. To populate the database with default aptitude questions, interview rubrics, and demo user accounts:

```bash
cd server
npm run seed
```
