# SurveyDuplicates

> **AI persona for surveys** — train your personal AI profile that takes surveys on your behalf, consistent with your values and preferences.

![Status](https://img.shields.io/badge/status-🚧%20Early%20Development-orange)
![Stack](https://img.shields.io/badge/stack-TypeScript%20%7C%20React%20%7C%20Node.js%20%7C%20PostgreSQL%20%7C%20OpenAI-blue)
![License](https://img.shields.io/badge/license-MIT-green)

---

## What It Does

SurveyDuplicates lets you build an **AI avatar of yourself** by training it on your values, opinions, demographics, and preferences. Once trained, your avatar can:

1. **Automatically complete surveys** on your behalf with responses consistent with your profile
2. **Explain its reasoning** — every answer traces back to a preference or value you defined
3. **Stay consistent** across survey sessions via a persistent persona memory
4. **Learn and refine** as you review and correct its responses

---

## Tech Stack

| Layer        | Technology                          |
|--------------|-------------------------------------|
| Frontend     | React 18 + TypeScript + Vite        |
| Backend      | Node.js + Express + TypeScript      |
| Database     | PostgreSQL 15 + Prisma ORM          |
| AI           | OpenAI API (GPT-4o + Embeddings)    |
| Auth         | JWT + bcrypt                        |
| Testing      | Vitest (unit) + Supertest (API) + Playwright (E2E) |
| CI/CD        | GitHub Actions                      |

---

## Getting Started

```bash
# 1. Clone
git clone https://github.com/joshiujjwal/survey-duplicates.git
cd survey-duplicates

# 2. Install dependencies
npm install

# 3. Set up environment
cp .env.example .env
# Edit .env — add DATABASE_URL, OPENAI_API_KEY, JWT_SECRET

# 4. Migrate database
npm run db:migrate

# 5. Start dev server (API + client with hot reload)
npm run dev

# 6. Run tests
npm test
```

### Prerequisites

- Node.js >= 20
- PostgreSQL >= 15 (or Docker: `docker compose up -d db`)
- OpenAI API key

---

## Project Structure

```
survey-duplicates/
├── src/
│   ├── server/              # Express API
│   │   ├── routes/          # Route definitions
│   │   ├── controllers/     # Request handlers
│   │   ├── services/        # Business logic (AI, persona, survey)
│   │   ├── middleware/       # Auth, validation, error handling
│   │   ├── models/          # Prisma model helpers
│   │   └── db/              # Prisma client + migrations
│   ├── client/              # React SPA
│   │   ├── components/      # Reusable UI components
│   │   ├── pages/           # Route-level page components
│   │   ├── hooks/           # Custom React hooks
│   │   ├── context/         # React context providers
│   │   └── utils/           # Shared utilities + API client
│   └── shared/              # Types/interfaces shared between client & server
├── tests/
│   ├── unit/                # Pure logic, no I/O
│   ├── integration/         # API routes + DB
│   └── e2e/                 # Playwright browser flows
├── docs/
│   ├── spec.md              # Feature specification
│   └── adr/                 # Architecture Decision Records
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/           # CI pipelines
├── CLAUDE.md                # AI agent context (Anthropic)
├── AGENTS.md                # AI agent context (OpenAI Codex)
├── TODO.md                  # Evidence-gated task breakdown
└── README.md
```

---

## Contributing

1. **Read** `TODO.md` + `docs/spec.md` before starting
2. **Run tests first**: `npm test` — all must pass before any changes
3. **TDD**: write failing tests before implementation (red → green → refactor)
4. **Small PRs**: one feature or fix per PR
5. **Evidence required**: PRs must include test output, screenshots, or logs
6. **Update context files**: if you learn something non-obvious, update `CLAUDE.md` or `AGENTS.md`

---

## License

MIT
