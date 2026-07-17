# Application package.json Scripts (Generator Reference)

When scaffolding an application in a harness repo, **keep harness scripts at the
repo root and product scripts under `app/package.json`**. The pre-QA gate runs
application checks from `app/` only — it must not execute orchestrator unit tests
via `npm test` at the repo root.

## Harness scripts (repo root — do not remove)

| Script | Location | Purpose |
|--------|----------|---------|
| `test:harness` | root `package.json` | Orchestrator/state-machine tests in `tests/*.test.mjs` |
| `lint:harness` | root `package.json` | Agent-prompt ESLint rules |
| `pre-qa-gate` | root `package.json` | Mechanical gate (orchestrator invokes this) |

## Application scripts (Generator adds under `app/package.json`)

| Script | Purpose | Pre-QA gate |
|--------|---------|-------------|
| `test:unit` | App unit/integration tests (Vitest, node:test, etc.) | **Runs** |
| `test:e2e` | Playwright E2E (`playwright test`) | **Runs** |
| `test` | Optional convenience: `npm run test:unit && npm run test:e2e` | Runs only if it does **not** reference `test:harness` |
| `build` | Production build | **Runs** |
| `typecheck` | `tsc --noEmit` or equivalent | **Runs** if present |
| `dev` | Dev server for Evaluator | Not run by gate |

## Example after Sprint 1 scaffolds a Vite app under `app/`

`app/package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "typecheck": "tsc -b --noEmit",
    "test:unit": "vitest run",
    "test:e2e": "playwright test",
    "test": "npm run test:unit && npm run test:e2e"
  }
}
```

Root `package.json` keeps harness scripts unchanged:

```json
{
  "scripts": {
    "lint:harness": "eslint --config .eslintrc.harness.cjs",
    "test:harness": "node --test tests/*.test.mjs",
    "pre-qa-gate": "bash scripts/pre-qa-gate.sh"
  }
}
```

## Anti-patterns (pre-QA gate will fail or skip incorrectly)

```json
"test": "node --test tests/*.test.mjs && playwright test"
```

This mixes harness orchestrator tests with app E2E. Use separate `test:unit` /
`test:e2e` instead.

```json
"test": "npm run test:harness && playwright test"
```

Same problem — gate detects `test:harness` in `test` and requires `test:unit`.
