# GitHub Copilot Instructions — SurveyDuplicates

## Project Context

SurveyDuplicates is a full-stack TypeScript monorepo. Users build an AI persona trained on their personal values (via OpenAI embeddings), then the persona autonomously completes surveys using GPT-4o.

**Stack**: React 18 + Vite (client) · Express + Node.js (server) · PostgreSQL + Prisma · OpenAI API · Vitest + Playwright

---

## Coding Conventions

### TypeScript
- Always use strict TypeScript — no `any`, no `// @ts-ignore` without explanation
- Shared types (DTOs, enums) live in `src/shared/types.ts` — import from there in both client and server
- Use path aliases: `@server/`, `@client/`, `@shared/` (never `../../..` across workspace boundaries)
- Use `const enum` only for compile-time constants; use string union types for values crossing API boundaries

### Server (Express)
- Controllers: validate with Zod → call service → return result. That's it.
- Services: all business logic, all OpenAI calls, all complex DB queries
- Error handling: throw `AppError(message, statusCode)` in services; global middleware catches it
- Logging: use `pino` logger, never `console.log` in server code
- OpenAI: only import in `src/server/services/`. Always log token usage to `TokenUsage` table.

### Client (React)
- Functional components only with explicit TypeScript props interfaces
- Server state: React Query (`@tanstack/react-query`) — no manual `useEffect` fetch loops
- Forms: React Hook Form + Zod resolver
- No inline styles; use Tailwind utility classes
- Hooks in `src/client/hooks/`, prefixed with `use`

### Database
- Always use Prisma query API — no raw SQL
- Import Prisma client from `src/server/db/client.ts` (singleton pattern)
- Run `npm run db:generate` after any schema change

---

## Testing Conventions

- **Write tests before implementation** (red → green → refactor)
- Unit tests in `tests/unit/` — mock all I/O with `vi.mock()`
- Integration tests in `tests/integration/` — use real test DB, mock OpenAI
- E2E tests in `tests/e2e/` — Playwright, full browser flow
- Never skip or delete tests to make a suite pass
- Mock OpenAI in ALL tests — `vi.mock('openai')` — never real API calls in tests

```typescript
// Standard OpenAI mock pattern
vi.mock('openai', () => ({
  default: vi.fn().mockImplementation(() => ({
    embeddings: {
      create: vi.fn().mockResolvedValue({
        data: [{ embedding: new Array(1536).fill(0.1) }],
        usage: { prompt_tokens: 10, total_tokens: 10 }
      })
    },
    chat: {
      completions: {
        create: vi.fn().mockResolvedValue({
          choices: [{ message: { content: JSON.stringify({ answer: 'test', reasoning: 'test', confidenceScore: 0.9 }) } }],
          usage: { prompt_tokens: 50, completion_tokens: 30, total_tokens: 80 }
        })
      }
    }
  }))
}));
```

---

## Boundaries — Do NOT Do These

- ❌ Do not refactor working code unless explicitly asked
- ❌ Do not remove, skip, or comment out existing tests
- ❌ Do not add `any` types to pass type checks
- ❌ Do not call OpenAI API outside of `src/server/services/`
- ❌ Do not hardcode secrets, API keys, or connection strings
- ❌ Do not import server-side code into client or vice versa (only `@shared/` is shared)
- ❌ Do not add new dependencies without noting the rationale in `AGENTS.md`
- ❌ Do not write raw SQL — use Prisma

---

## Key File Locations

| What                        | Where                                              |
|-----------------------------|----------------------------------------------------|
| Shared types/DTOs           | `src/shared/types.ts`                              |
| OpenAI embedding calls      | `src/server/services/EmbeddingService.ts`          |
| GPT-4o survey parsing       | `src/server/services/SurveyParserService.ts`       |
| Persona AI answer engine    | `src/server/services/PersonaAnswerService.ts`      |
| Prisma schema               | `src/server/db/prisma/schema.prisma`               |
| Prisma singleton client     | `src/server/db/client.ts`                          |
| API typed fetch wrapper     | `src/client/utils/api.ts`                          |
| JWT middleware              | `src/server/middleware/auth.ts`                    |
| Global error handler        | `src/server/middleware/error.ts`                   |
| Route definitions           | `src/server/routes/`                               |
| Training question seeds     | `src/server/db/seed.ts`                            |
