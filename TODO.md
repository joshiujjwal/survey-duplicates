# SurveyDuplicates — Task Breakdown

## How to Use This File

> One task at a time. Evidence gates are not optional.

```
Workflow per task:
1. Pull latest main
2. Run full test suite — all green before you start
3. Write failing tests for the feature (red phase)
4. Implement until tests pass (green phase)
5. Manually test the feature end-to-end
6. Diff review: read every line you changed
7. Commit with format: "feat|fix|test|chore(scope): message"
8. If you learned something non-obvious → update CLAUDE.md or AGENTS.md
```

---

## Phase 0: Foundation ⬜

> Goal: working CI, reproducible dev env, first green test

- [ ] Initialize `package.json` with workspaces (`client`, `server`, `shared`)
- [ ] Configure `tsconfig.json` (strict mode, path aliases `@server/*`, `@client/*`, `@shared/*`)
- [ ] Set up ESLint + Prettier with TypeScript rules
- [ ] Configure Vitest for unit tests (`tests/unit/`)
- [ ] Configure Supertest for integration tests (`tests/integration/`)
- [ ] Configure Playwright for E2E tests (`tests/e2e/`)
- [ ] Write first smoke test: server starts and `GET /health` returns `200`
- [ ] Set up Prisma: `prisma init`, connect to PostgreSQL
- [ ] Create `.env.example` with all required variables documented
- [ ] Docker Compose for local PostgreSQL (`docker compose up -d db`)
- [ ] GitHub Actions CI: install → lint → test → build on every PR
- [ ] Review all AI config files (`CLAUDE.md`, `AGENTS.md`, copilot instructions)

**Evidence gate**: CI green, `npm test` passes locally, `GET /health` returns `200`

---

## Phase 1: User Auth + Persona Foundation ⬜

> Goal: users can register, log in, and create a basic persona profile

- [ ] **Prisma schema**: `User`, `Persona`, `PersonaValue` models
  - `User`: id, email, passwordHash, createdAt
  - `Persona`: id, userId (FK), name, bio, demographicData (jsonb), createdAt, updatedAt
  - `PersonaValue`: id, personaId (FK), category (enum), question, answer, embedding (vector)
- [ ] Run `prisma migrate dev --name init`
- [ ] Write integration tests for auth routes (red)
  - `POST /api/auth/register` — happy path, duplicate email, weak password
  - `POST /api/auth/login` — correct creds, wrong password, unknown email
  - Protected route returns `401` without valid JWT
- [ ] Implement `AuthController` + `AuthService` (bcrypt + JWT)
- [ ] JWT middleware: extract + verify token, attach `req.user`
- [ ] Write integration tests for persona CRUD (red)
  - `POST /api/personas` — create persona, validates required fields
  - `GET /api/personas/:id` — returns persona for authenticated owner
  - `PUT /api/personas/:id` — updates bio/demographic fields
- [ ] Implement `PersonaController` + `PersonaService`
- [ ] React pages: `RegisterPage`, `LoginPage`, `PersonaDashboardPage`
- [ ] `AuthContext` with `useAuth()` hook, JWT stored in `httpOnly` cookie
- [ ] Protected route wrapper component `<RequireAuth />`

**Evidence gate**: auth integration tests green, persona CRUD tests green, manual login + persona creation flow works in browser

---

## Phase 2: Persona Training (Value Elicitation) ⬜

> Goal: users can train their persona by answering curated questions; answers are embedded and stored

- [ ] **Prisma migration**: add `TrainingSession`, `TrainingQuestion` models
  - `TrainingSession`: id, personaId, completedAt, questionCount
  - `TrainingQuestion`: id, category (enum: `politics`, `ethics`, `lifestyle`, `consumer`, `social`), questionText, tags (string[])
- [ ] Seed database with 50 curated training questions across all categories
- [ ] Write unit tests for `EmbeddingService` (mock OpenAI) (red)
  - Embeds a string → returns `number[]` of length 1536
  - Handles OpenAI API errors gracefully
- [ ] Implement `EmbeddingService`: calls `text-embedding-3-small`, stores vector
- [ ] Write integration tests for training routes (red)
  - `POST /api/personas/:id/training/start` — creates session, returns question batch
  - `POST /api/personas/:id/training/answer` — saves answer + embedding
  - `GET /api/personas/:id/training/progress` — returns completion % per category
- [ ] Implement `TrainingController` + `TrainingService`
- [ ] React page: `PersonaTrainingPage`
  - Question cards with category label
  - Free-text + optional 1-5 scale input
  - Progress bar per category
  - "I don't know / skip" option
- [ ] React hook: `useTrainingSession(personaId)`

**Evidence gate**: embedding unit tests green (mocked), training integration tests green, full training flow completes in browser, persona has ≥ 10 stored values in DB

---

## Phase 3: Survey Ingestion + AI Response Engine ⬜

> Goal: users can paste/import a survey and the AI avatar answers it

- [ ] **Prisma migration**: `Survey`, `SurveyQuestion`, `SurveyResponse`, `QuestionAnswer` models
  - `Survey`: id, userId, title, sourceUrl, rawText, parsedAt
  - `SurveyQuestion`: id, surveyId, questionText, questionType (enum: `multiple_choice`, `likert`, `open_ended`, `ranking`), options (jsonb)
  - `SurveyResponse`: id, surveyId, personaId, status (enum: `pending`, `in_progress`, `completed`), completedAt
  - `QuestionAnswer`: id, responseId, questionId, answer, reasoning, confidenceScore
- [ ] Write unit tests for `SurveyParserService` (red)
  - Parses plain text into structured `SurveyQuestion[]`
  - Handles multiple choice, Likert scale, open-ended
  - Returns parse errors for malformed input
- [ ] Implement `SurveyParserService` using GPT-4o with structured output (JSON mode)
- [ ] Write unit tests for `PersonaAnswerService` (mock OpenAI + DB) (red)
  - Given a question + persona values, returns `{ answer, reasoning, confidenceScore }`
  - Retrieves top-k relevant persona values via cosine similarity on embeddings
  - Formats prompt with persona context window
- [ ] Implement `PersonaAnswerService`
  - Retrieve top-5 relevant `PersonaValue` embeddings for the question
  - Build system prompt: "You are {name}. Your values: ..."
  - Call GPT-4o, parse structured answer + reasoning
  - Store `QuestionAnswer` with confidence score
- [ ] Write integration tests for survey routes (red)
  - `POST /api/surveys` — ingest raw survey text, returns parsed questions
  - `POST /api/surveys/:id/respond` — triggers AI response generation for a persona
  - `GET /api/surveys/:id/responses/:responseId` — returns answers with reasoning
- [ ] Implement `SurveyController` + orchestration
- [ ] React pages: `SurveyIngestPage`, `SurveyResponsePage`
  - Survey text paste input + parsed preview
  - Response view: each question with AI answer + "why" collapsible reasoning
  - Confidence score badge per answer

**Evidence gate**: parser unit tests green, answer service unit tests green (mocked), integration tests green, end-to-end demo: paste a 10-question survey → AI answers all with reasoning

---

## Phase 4: Review, Correction & Persona Refinement ⬜

> Goal: users review AI answers, correct them, and corrections feed back into persona

- [ ] Write unit tests for `PersonaRefinementService` (red)
  - User correction updates the relevant `PersonaValue` or creates new one
  - Re-embeds corrected answer
  - Logs correction in `CorrectionLog`
- [ ] **Prisma migration**: `CorrectionLog` model
  - `CorrectionLog`: id, personaId, questionId, originalAnswer, correctedAnswer, correctedAt
- [ ] Implement `PersonaRefinementService`
- [ ] Write integration tests for correction route (red)
  - `PUT /api/surveys/:id/responses/:responseId/answers/:answerId/correct`
- [ ] Implement correction endpoint
- [ ] React component: `AnswerReviewCard`
  - Shows question, AI answer, reasoning
  - Inline edit to correct answer
  - "Accept" / "Correct" buttons
  - Correction triggers toast + background refinement
- [ ] Persona consistency score: % of answers accepted without correction
- [ ] React page: `PersonaHealthPage` — consistency chart, top-corrected categories

**Evidence gate**: refinement unit tests green, correction integration tests green, correcting an answer updates persona values in DB and consistency score changes

---

## Phase 5: Polish, Security & Hardening ⬜

> Goal: production-ready security, rate limiting, input validation, error handling

- [ ] Input validation: `zod` schemas for all API request bodies
- [ ] Rate limiting: `express-rate-limit` on `/api/surveys/:id/respond` (prevent runaway OpenAI calls)
- [ ] OpenAI cost guard: per-user monthly token budget stored in DB, reject when exceeded
- [ ] API error handler middleware: never leak stack traces to client
- [ ] Sanitize all user text before embedding (strip PII warnings via regex heuristics)
- [ ] CORS policy: restrict to known origins in production
- [ ] Helmet.js security headers
- [ ] Playwright E2E tests: register → train → survey → review full flow
- [ ] Performance: add DB indexes on `PersonaValue.personaId`, `SurveyQuestion.surveyId`
- [ ] Audit all TODO comments in codebase and resolve or ticket them
- [ ] README: fill in any remaining TODO placeholders

**Evidence gate**: E2E tests green, `npm audit` shows no high/critical vulnerabilities, manual security review complete

---

## Phase 6: Ship ⬜

> Goal: deployed, documented, ready for users

- [ ] Dockerfile for server (multi-stage build)
- [ ] Vite production build for client, served as static files from Express
- [ ] Environment config for staging + production
- [ ] `prisma migrate deploy` in CI/CD pipeline
- [ ] Deploy to target platform (Railway / Render / Fly.io — TBD)
- [ ] Smoke test production: register, train, survey, review
- [ ] Tag `v1.0.0` release

**Evidence gate**: production URL returns `200`, full user flow works in production, no errors in server logs after 10 minutes

---

## Parking Lot 🅿️

_Ideas not in current scope but worth revisiting:_

- Browser extension to auto-detect and fill surveys on SurveyMonkey / Google Forms
- Persona sharing: allow other users to "hire" your persona for their surveys
- Differential privacy: add noise to embeddings before storage
- Multi-persona support (work persona vs. personal persona)
- Survey result analytics dashboard
- Export answers as PDF report

---

## Lessons Learned 📝

_Update this section as you build. Each entry should be a non-obvious discovery._

| Date | Lesson | Impact |
|------|--------|--------|
| —    | —      | —      |
