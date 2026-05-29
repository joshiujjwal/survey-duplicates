# AGENTS.md — SurveyDuplicates

> Standard agent context file (OpenAI Codex / o-series / Copilot Workspace compatible).  
> Read this before writing any code.

---

## Setup

```bash
# Install dependencies
npm install

# Start PostgreSQL (Docker)
docker compose up -d db

# Copy env and fill in secrets
cp .env.example .env

# Run database migrations
npm run db:migrate

# Seed training questions
npm run db:seed

# Start development server
npm run dev
# API → http://localhost:3001
# Client → http://localhost:5173

# Verify everything works
curl http://localhost:3001/api/health
```

---

## Running Tests

```bash
# All tests
npm test

# Unit only (fast, no DB, no OpenAI)
npm run test:unit

# Integration (requires PostgreSQL at DATABASE_URL_TEST)
npm run test:integration

# E2E (requires full dev server running)
npm run test:e2e

# Watch mode during development
npm run test:watch
```

**TDD rule**: Write failing tests BEFORE implementation. No PR merges without tests.

---

## Code Style

### TypeScript

- **Strict mode** — `strict: true` in `tsconfig.json`. No `any`. No `@ts-ignore` without explanation.
- **Path aliases**: use `@server/`, `@client/`, `@shared/` — never relative `../../..` imports across workspace boundaries
- **Shared types**: all DTO interfaces go in `src/shared/types.ts`
- **Enums**: use `const enum` for compile-time only; use string union types for runtime values that cross API boundaries
- **No barrel files** (`index.ts` re-exports) in `services/` or `controllers/` — import directly to keep dependency graphs clear

### Node.js / Express

- Controllers are thin: validate input with Zod → call one service method → return result
- Services contain all business logic; they are the only layer that touches OpenAI or complex DB queries
- Always `await` async operations — no floating promises
- Use `next(error)` in controllers, never `res.status(500).json(...)` inline
- All routes registered in `src/server/routes/index.ts` with versioned prefix `/api`

### React

- Functional components only with TypeScript props interfaces
- File naming: `PascalCase.tsx` for components, `camelCase.ts` for utilities/hooks
- Hooks prefixed with `use`: `useAuth`, `usePersona`, `useSurveyResponse`
- API state managed with **React Query** (`@tanstack/react-query`)
- Forms handled with **React Hook Form** + Zod resolvers
- No inline styles — use CSS modules or Tailwind utility classes

### OpenAI Integration

- Import OpenAI client only in `src/server/services/`
- Always set `response_format: { type: 'json_object' }` for structured outputs
- Always call `TokenBudgetService.record()` after each OpenAI API call
- Mock with `vi.mock('openai')` in all tests — zero real API calls in test suite

### Database / Prisma

- Schema in `src/server/db/prisma/schema.prisma`
- Generate client: `npm run db:generate` after any schema change
- Singleton client: always import from `src/server/db/client.ts`
- Test DB uses a separate `DATABASE_URL_TEST`; migrations run in `beforeAll()`
- Never write raw SQL — use Prisma query API

---

## PR Instructions

Every pull request MUST include:

1. **Test evidence** — paste `npm test` output showing all tests pass
2. **Manual test notes** — one sentence describing what you manually verified
3. **Type-check pass** — paste `npm run typecheck` output (exit 0)
4. **Lint pass** — paste `npm run lint` output (exit 0)
5. **Description** — what changed and why (not just what)

PRs without evidence will not be merged.

### Commit Format

```
<type>(<scope>): <short description>

Types: feat | fix | test | chore | docs | refactor | perf
Scopes: auth | persona | training | survey | ai | ui | db | ci

Examples:
  feat(survey): add SurveyParserService with GPT-4o JSON mode
  test(persona): add integration tests for training answer endpoint
  fix(ai): handle OpenAI rate limit with exponential backoff
```

---

## Architecture Constraints

- **Do not** import from `src/server/` in `src/client/` or vice versa — only `src/shared/` is cross-boundary
- **Do not** call OpenAI outside of `src/server/services/`
- **Do not** add new npm packages without updating this file with the rationale
- **Do not** disable TypeScript strict checks
- **Do not** delete or skip existing tests to make a suite pass
- **Do not** store secrets in source code — use `.env` only

---

## Key Dependencies

| Package                  | Purpose                                      |
|--------------------------|----------------------------------------------|
| `openai`                 | OpenAI SDK (embeddings + GPT-4o)             |
| `@prisma/client`         | PostgreSQL ORM                               |
| `express`                | HTTP server                                  |
| `zod`                    | Runtime schema validation                    |
| `jsonwebtoken`           | JWT sign/verify                              |
| `bcrypt`                 | Password hashing                             |
| `@tanstack/react-query`  | Server state management in React             |
| `react-hook-form`        | Form state management                        |
| `vitest`                 | Unit + integration test runner               |
| `supertest`              | HTTP integration testing                     |
| `@playwright/test`       | E2E browser testing                          |
| `pino`                   | Structured logging                           |
| `express-rate-limit`     | Rate limiting                                |
| `helmet`                 | Security headers                             |
