# SurveyDuplicates — Feature Specification

**Version**: 0.1.0-draft  
**Status**: Living document — update as requirements evolve  
**Owner**: joshiujjwal

---

## 1. Overview

### Problem Statement

Survey fatigue is real. Professionals, researchers, and frequent survey-takers waste hours answering the same types of questions repeatedly — political views, consumer preferences, lifestyle habits — with consistent, predictable answers. There is no mechanism to delegate this cognitive labor to an AI that actually represents the individual's authentic voice.

### Solution

SurveyDuplicates enables users to build a **persistent AI persona** trained on their own values, opinions, and preferences. Once trained, the persona can autonomously complete surveys with answers that are:
- **Consistent** with the user's documented values
- **Explainable** — every answer cites the specific value that drove it
- **Correctable** — users review and fix wrong answers, which refines the persona

### Success Criteria

- [ ] A user can complete a 20-question survey via their AI avatar in under 2 minutes
- [ ] AI answers match user's manual answers ≥ 80% of the time after Phase 2 training
- [ ] Answer accuracy improves measurably after 5+ corrections (persona refinement loop)
- [ ] System handles concurrent survey responses for 50 users without degradation

---

## 2. User Stories

| ID  | As a…  | I want to…                                           | So that…                                          |
|-----|--------|------------------------------------------------------|---------------------------------------------------|
| U01 | User   | Register and create a personal AI persona            | My survey responses are tied to my identity       |
| U02 | User   | Answer curated training questions by category        | My AI learns my authentic values                  |
| U03 | User   | Paste raw survey text into the app                   | The AI can parse and understand any survey format |
| U04 | User   | Trigger AI auto-complete for a survey                | I don't have to manually fill it out              |
| U05 | User   | See every AI answer with a reason                    | I can verify it represents me accurately          |
| U06 | User   | Correct any AI answer inline                         | The persona learns and improves over time         |
| U07 | User   | Track my persona's consistency score                 | I know how well my AI represents me               |
| U08 | Admin  | View per-user OpenAI token usage                     | I can enforce cost budgets                        |

---

## 3. Functional Requirements

### 3.1 Authentication

- [ ] Users register with email + password (min 8 chars, 1 uppercase, 1 number)
- [ ] Passwords hashed with bcrypt (rounds ≥ 12)
- [ ] JWT access token (15min expiry) + refresh token (7 days) stored in `httpOnly` cookies
- [ ] `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/auth/logout`

### 3.2 Persona Management

- [ ] Each user has exactly one active persona (expandable to multiple in future)
- [ ] Persona has: name, short bio, demographic data (age range, country, occupation — optional)
- [ ] Persona stores a collection of `PersonaValue` entries (question + answer + embedding)
- [ ] `PersonaValue` categories: `politics`, `ethics`, `lifestyle`, `consumer`, `social`
- [ ] Persona has a `consistencyScore` (0–100) computed from correction history

### 3.3 Persona Training

- [ ] System provides 50+ curated training questions seeded in DB
- [ ] Questions grouped by category; user must answer ≥ 5 per category to "activate" that category
- [ ] Answers stored as plain text + OpenAI embedding vector
- [ ] Users can retrain (update an existing answer) at any time
- [ ] Training progress visible per category (e.g., "Politics: 7/10")
- [ ] "Skip" is allowed — skipped questions don't count toward activation

### 3.4 Survey Ingestion

- [ ] User pastes raw survey text (max 10,000 characters)
- [ ] System calls GPT-4o to parse into structured `SurveyQuestion[]`
- [ ] Supported question types: `multiple_choice`, `likert_scale`, `open_ended`, `ranking`
- [ ] Parsed questions shown in preview UI before user triggers response generation
- [ ] User can manually edit parsed questions before generation

### 3.5 AI Survey Response

- [ ] User selects persona + survey, triggers `POST /api/surveys/:id/respond`
- [ ] For each question:
  1. Compute embedding of question text
  2. Retrieve top-5 `PersonaValue` by cosine similarity
  3. Build prompt: system role as persona, user message as question + context
  4. Call GPT-4o with JSON mode output: `{ answer: string, reasoning: string, confidenceScore: number }`
  5. Store `QuestionAnswer` record
- [ ] Generation is async; UI polls for completion status
- [ ] Confidence score 0–1; scores < 0.6 flagged for mandatory review

### 3.6 Answer Review & Correction

- [ ] All answers and reasoning displayed per-question after generation
- [ ] User can accept (no action) or correct (inline text edit) each answer
- [ ] Correction creates `CorrectionLog` and upserts `PersonaValue` with new embedding
- [ ] Consistency score recalculated after each correction batch

### 3.7 Cost Management

- [ ] Per-user monthly token budget: default 100,000 tokens (configurable via env)
- [ ] Token usage tracked per OpenAI call in `TokenUsage` table
- [ ] Requests rejected with `429` when monthly budget exceeded
- [ ] Usage visible to user in dashboard

---

## 4. Non-Functional Requirements

- [ ] API response time P95 < 500ms for non-AI endpoints
- [ ] AI response generation: full 20-question survey < 30 seconds
- [ ] Database: all queries use indexes; no full-table scans in hot paths
- [ ] Zero secrets in source code (all via environment variables)
- [ ] OWASP Top 10 addressed before Phase 5 complete

---

## 5. Data Model

```
User
  id            UUID PK
  email         VARCHAR(255) UNIQUE NOT NULL
  passwordHash  TEXT NOT NULL
  createdAt     TIMESTAMPTZ

Persona
  id            UUID PK
  userId        UUID FK → User
  name          VARCHAR(100)
  bio           TEXT
  demographic   JSONB           -- { ageRange, country, occupation }
  consistencyScore  INT DEFAULT 0
  createdAt     TIMESTAMPTZ
  updatedAt     TIMESTAMPTZ

PersonaValue
  id            UUID PK
  personaId     UUID FK → Persona
  category      ENUM(politics, ethics, lifestyle, consumer, social)
  question      TEXT NOT NULL
  answer        TEXT NOT NULL
  embedding     VECTOR(1536)    -- pgvector extension
  createdAt     TIMESTAMPTZ
  updatedAt     TIMESTAMPTZ

TrainingQuestion
  id            UUID PK
  category      ENUM
  questionText  TEXT NOT NULL
  tags          TEXT[]

Survey
  id            UUID PK
  userId        UUID FK → User
  title         VARCHAR(255)
  rawText       TEXT
  parsedAt      TIMESTAMPTZ
  createdAt     TIMESTAMPTZ

SurveyQuestion
  id            UUID PK
  surveyId      UUID FK → Survey
  questionText  TEXT NOT NULL
  questionType  ENUM(multiple_choice, likert_scale, open_ended, ranking)
  options       JSONB           -- null for open_ended

SurveyResponse
  id            UUID PK
  surveyId      UUID FK → Survey
  personaId     UUID FK → Persona
  status        ENUM(pending, in_progress, completed)
  completedAt   TIMESTAMPTZ

QuestionAnswer
  id            UUID PK
  responseId    UUID FK → SurveyResponse
  questionId    UUID FK → SurveyQuestion
  answer        TEXT NOT NULL
  reasoning     TEXT NOT NULL
  confidenceScore  DECIMAL(3,2)
  reviewed      BOOLEAN DEFAULT false
  corrected     BOOLEAN DEFAULT false

CorrectionLog
  id            UUID PK
  personaId     UUID FK → Persona
  questionId    UUID FK → SurveyQuestion
  originalAnswer  TEXT
  correctedAnswer TEXT NOT NULL
  correctedAt   TIMESTAMPTZ

TokenUsage
  id            UUID PK
  userId        UUID FK → User
  endpoint      VARCHAR(100)    -- e.g. 'survey.respond', 'training.embed'
  promptTokens  INT
  completionTokens INT
  recordedAt    TIMESTAMPTZ
```

---

## 6. API Interface

### Auth

| Method | Path                    | Auth | Description                          |
|--------|-------------------------|------|--------------------------------------|
| POST   | `/api/auth/register`    | —    | Register user, returns tokens        |
| POST   | `/api/auth/login`       | —    | Login, returns tokens                |
| POST   | `/api/auth/refresh`     | Cookie | Refresh access token               |
| POST   | `/api/auth/logout`      | JWT  | Invalidate refresh token             |

### Persona

| Method | Path                              | Auth | Description                       |
|--------|-----------------------------------|------|-----------------------------------|
| POST   | `/api/personas`                   | JWT  | Create persona                    |
| GET    | `/api/personas/:id`               | JWT  | Get persona details               |
| PUT    | `/api/personas/:id`               | JWT  | Update bio/demographics           |
| GET    | `/api/personas/:id/training/questions` | JWT | Get training questions batch |
| POST   | `/api/personas/:id/training/answer`    | JWT | Submit answer + trigger embed |
| GET    | `/api/personas/:id/training/progress`  | JWT | Progress by category          |

### Surveys

| Method | Path                                              | Auth | Description                     |
|--------|---------------------------------------------------|------|---------------------------------|
| POST   | `/api/surveys`                                    | JWT  | Ingest + parse survey text      |
| GET    | `/api/surveys`                                    | JWT  | List user's surveys             |
| GET    | `/api/surveys/:id`                                | JWT  | Get survey + questions          |
| POST   | `/api/surveys/:id/respond`                        | JWT  | Trigger AI response generation  |
| GET    | `/api/surveys/:id/responses/:responseId`          | JWT  | Get all answers + reasoning     |
| PUT    | `/api/surveys/:id/responses/:responseId/answers/:answerId/correct` | JWT | Submit correction |

---

## 7. Test Plan

### Unit Tests (`tests/unit/`)

| Test File                         | What It Covers                                           |
|-----------------------------------|----------------------------------------------------------|
| `EmbeddingService.test.ts`        | Embed string → vector, OpenAI error handling             |
| `SurveyParserService.test.ts`     | Parse various survey formats, edge cases, malformed input|
| `PersonaAnswerService.test.ts`    | Prompt construction, cosine similarity retrieval (mocked)|
| `PersonaRefinementService.test.ts`| Correction → upsert PersonaValue, consistency recalc     |
| `authUtils.test.ts`               | JWT sign/verify, token expiry                            |

### Integration Tests (`tests/integration/`)

| Test File                  | What It Covers                                    |
|----------------------------|---------------------------------------------------|
| `auth.test.ts`             | Full register/login/refresh/logout cycle via HTTP |
| `persona.test.ts`          | CRUD + training session via HTTP + real DB        |
| `survey.test.ts`           | Ingest → parse → respond → correct via HTTP       |
| `tokenBudget.test.ts`      | Budget enforcement rejects requests at limit      |

### E2E Tests (`tests/e2e/`)

| Test File               | Flow                                                   |
|-------------------------|--------------------------------------------------------|
| `fullFlow.spec.ts`      | Register → train (10 Qs) → paste survey → AI responds → review |
| `correction.spec.ts`    | Correct answer → verify persona updated in DB          |

---

## 8. Open Questions

- [ ] **pgvector vs. external vector DB**: pgvector sufficient for MVP (< 100K vectors)?  
  _Decision needed before Phase 2_
- [ ] **Async job queue**: should survey generation use Bull/BullMQ or SSE polling?  
  _Polling sufficient for MVP, queue for Phase 6+_
- [ ] **PII handling**: should persona values be encrypted at rest?  
  _Conservative default: yes, using Postgres column-level encryption_
- [ ] **Multi-persona per user**: scoped to Phase 6 parking lot — confirm before Phase 4_
- [ ] **Survey import formats**: plain text only for MVP, or also Google Forms JSON export?
