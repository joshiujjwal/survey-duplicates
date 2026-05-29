# CLAUDE.md — SurveyDuplicates

> Context file for Claude and other AI coding agents. Keep under 200 lines.  
> Update this file whenever you discover something non-obvious about this codebase.

---

## Project in One Sentence

SurveyDuplicates is a TypeScript monorepo (React + Express + PostgreSQL) that lets users build an AI persona trained on their values, which then auto-completes surveys on their behalf using OpenAI embeddings + GPT-4o.

---

## Commands

```bash
# Install all workspace deps
npm install

# Dev (starts Express API on :3001 + Vite client on :5173 with proxy)
npm run dev

# Run all tests (unit + integration)
npm test

# Run only unit tests
npm run test:unit

# Run only integration tests (requires running PostgreSQL)
npm run test:integration

# Run E2E tests (requires dev server running)
npm run test:e2e

# Lint
npm run lint

# Type-check (no emit)
npm run typecheck

# Build for production
npm run build

# Database migrations
npm run db:migrate        # dev migrations
npm run db:migrate:deploy # production migrations
npm run db:seed           # seed training questions
npm run db:studio         # Prisma Studio UI

# Generate Prisma client after schema changes
npm run db:generate
```

---

## Directory Map

```
src/server/
  routes/          → Express router files — one file per resource (auth, personas, surveys)
  controllers/     → Thin request handlers; delegate to services immediately
  services/        → All business logic lives here
    EmbeddingService.ts       → OpenAI text-embedding-3-small calls
    SurveyParserService.ts    → GPT-4o JSON-mode survey parsing
    PersonaAnswerService.ts   → Retrieves top-k values, builds prompt, calls GPT-4o
    PersonaRefinementService.ts → Correction → re-embed → upsert PersonaValue
    TokenBudgetService.ts     → Tracks + enforces monthly OpenAI token spend
  middleware/      → auth.ts (JWT verify), validate.ts (zod), error.ts (global handler)
  models/          → Typed wrapper functions around Prisma queries (no raw SQL)
  db/              → prisma/schema.prisma, Prisma client singleton

src/client/
  pages/           → Route-level components (one per page)
  components/      → Reusable UI pieces; never contain fetch logic
  hooks/           → useAuth, usePersona, useTraining, useSurveyResponse
  context/         → AuthContext (JWT state), PersonaContext
  utils/           → api.ts (typed fetch wrapper with base URL + auth header)

src/shared/
  types.ts         → Interfaces shared by client + server (DTOs, enums)
  constants.ts     → Shared constants (categories, token limits)

tests/
  unit/            → No I/O; mock all OpenAI calls with vi.mock()
  integration/     → Uses a real test DB; runs migrations before suite
  e2e/             → Playwright; starts full dev server
```

---

## Non-Obvious Conventions

### OpenAI Calls
- **All** OpenAI calls go through `services/`; controllers never import `openai` directly
- Mock OpenAI in unit tests with `vi.mock('openai')` — never make real API calls in tests
- Structured output (JSON mode): always include `response_format: { type: 'json_object' }` + a JSON schema in the system prompt
- Token usage: always log to `TokenUsage` table after every OpenAI call using `TokenBudgetService.record()`

### Embeddings
- Model: `text-embedding-3-small` (1536 dimensions, cheapest option)
- Cosine similarity computed in-app (not pgvector) for MVP; switch to `pgvector` `<=>` operator if performance degrades
- Embedding stored as `REAL[]` in Postgres until pgvector extension confirmed available

### Database
- Prisma client is a singleton: import from `src/server/db/prisma.ts`, never instantiate `new PrismaClient()` inline
- All DB operations in `models/` directory — services call model functions, not Prisma directly
- Test DB: separate `DATABASE_URL_TEST` in `.env.test`; migrations run in `beforeAll()` via `prisma migrate deploy`

### Auth
- Access token: 15min JWT in `Authorization: Bearer` header
- Refresh token: 7-day JWT in `httpOnly` cookie named `refreshToken`
- `req.user` is typed as `{ userId: string; email: string }` after JWT middleware

### Error Handling
- Services throw typed errors: `class AppError extends Error { constructor(message, statusCode) }`
- Global error middleware in `middleware/error.ts` converts to `{ error: string }` JSON
- Never `console.log` in production paths — use a structured logger (pino)

### React
- No class components
- State: React Query for server state, `useState`/`useReducer` for local UI state
- No Redux — if you find yourself reaching for it, reconsider the component hierarchy
- API calls only in hooks (`hooks/`) or React Query `queryFn`, never directly in components

### TypeScript
- Strict mode enabled — no `any` without a `// TODO: type this` comment and a ticket
- Path aliases: `@server/`, `@client/`, `@shared/` — configured in `tsconfig.json` + Vite
- All API request/response shapes defined in `src/shared/types.ts`

---

## Workflow

```
1. git pull origin main
2. npm test   ← all tests must be green before starting
3. Pick the next unchecked item from TODO.md
4. Write failing tests (red)
5. Implement until tests pass (green)
6. npm run typecheck && npm run lint
7. Manual test the feature in browser
8. git diff   ← read every changed line
9. git commit -m "feat|fix|test|chore(scope): description"
10. If you learned something non-obvious → add it to this file
```

---

## Environment Variables

```bash
# .env.example
DATABASE_URL=postgresql://user:password@localhost:5432/survey_duplicates
DATABASE_URL_TEST=postgresql://user:password@localhost:5432/survey_duplicates_test
OPENAI_API_KEY=sk-...
JWT_SECRET=change-me-in-production-min-32-chars
JWT_REFRESH_SECRET=different-secret-min-32-chars
MONTHLY_TOKEN_BUDGET=100000       # per user, default
PORT=3001
CLIENT_URL=http://localhost:5173
NODE_ENV=development
```

---

## Gotchas

- **pgvector**: requires `CREATE EXTENSION vector;` in PostgreSQL before first migration. Add to `db/seed.ts` or migration.
- **Prisma + JSON columns**: `Prisma.JsonValue` is not typed — define explicit interfaces in `shared/types.ts` and cast.
- **Vite proxy**: in dev, client requests to `/api/*` are proxied to `:3001` via `vite.config.ts`. Don't hardcode `:3001` in client code.
- **OpenAI rate limits**: `EmbeddingService` should implement exponential backoff — use `openai` SDK's built-in retry config (`maxRetries: 3`).
- **Training question seeds**: `prisma db seed` runs `src/server/db/seed.ts` — idempotent (upsert, not insert).
