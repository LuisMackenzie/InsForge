# AGENTS.md

## Commands

```bash
# Full dev environment (postgres, postgrest, backend, frontend, deno)
cp .env.example .env && docker compose up

# Backend only (requires running postgres)
npm run dev:backend

# Frontend only
npm run dev:frontend

# Verification pipeline (run in this order)
npm run lint
npm run typecheck
npm run test:backend          # vitest unit tests
npm run test:e2e              # shell scripts against live Docker stack

# Single backend test file
cd backend && npx vitest run tests/path/to/test.test.ts

# Build everything (turbo handles order)
npm run build
```

## Monorepo Structure

| Directory | Package | Purpose |
|---|---|---|
| `backend/` | `insforge-backend` | Express.js API, migrations, vitest tests |
| `frontend/` | `insforge-dashboard` | Thin Vite shell that imports `@insforge/dashboard` |
| `packages/dashboard` | `@insforge/dashboard` | Shared dashboard (the real frontend code) |
| `packages/shared-schemas` | `@insforge/shared-schemas` | Zod schemas consumed by backend + frontend |
| `packages/ui` | `@insforge/ui` | Radix + Tailwind component library |
| `functions/` | — | Deno edge-function runtime |

**Build output**: backend compiles to `dist/server.js` (root, not `backend/dist`). Frontend compiles to `dist/frontend/`. Turbo orchestrates build order via `dependsOn: ["^build"]`.

**Build order when running manually**: `shared-schemas` → `ui` → `dashboard` → backend/frontend.

## Backend

- **Entry**: `backend/src/server.ts`, built by tsup to ESM (`dist/server.js`)
- **Path alias**: `@/` → `backend/src/` (configured in tsconfig and tsup)
- **Tests**: vitest with `maxWorkers: 1` (sequential — required, not optional). Setup in `backend/tests/setup.ts`. The `@insforge/shared-schemas` import resolves to source (`../packages/shared-schemas/src`) in tests.
- **Migrations**: SQL files in `backend/src/infra/database/migrations/` with format `NNN_description.sql`. Numbers must be unique — CI checks for duplicates via `scripts/check-migration-duplicates.js`. Run locally with `npm run migrate:up:local` (needs `dotenv -e ../.env`).

## Dashboard Package Conventions

- `packages/dashboard` uses `#`-prefixed import aliases: `#components/*`, `#lib/*`, `#features/*`, `#app/*`, etc. (defined in its `package.json` `imports` field)
- ESLint **forbids `../` imports** within `packages/dashboard` — use `#` aliases for cross-folder imports
- `frontend/` is just a thin host shell; the real dashboard lives in `packages/dashboard`

## E2E Tests

E2E tests are **bash scripts** (not vitest) in `backend/tests/local/` and `backend/tests/cloud/`. They require a running Docker stack (`docker compose up`) and hit `http://localhost:7130/api`. Run all with `npm run test:e2e` from root.

## Style

- Prettier: single quotes, semicolons, trailing comma `es5`, printWidth 100, 2-space indent
- ESLint enforces `no-floating-promises`, `no-misused-promises`, `await-thenable` — unhandled promises are errors
- `no-console` is warn (allows `console.warn` and `console.error`)
- Naming: camelCase for variables/functions, PascalCase for types/components, UPPER_CASE for enums/constants
- Files: PascalCase for components, camelCase for services (`auth.service.ts`), kebab-case for utils

## CI Gates

PRs must pass: lint + prettier + typecheck, unit tests (`test:backend`), migration duplicate check, and Docker build. E2E tests run separately against a Docker Compose stack.
