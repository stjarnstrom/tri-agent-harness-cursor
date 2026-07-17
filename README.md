# Tri-Agent Harness (Cursor)

Orchestration and guardrails in one harness: **Planner → Generator → Pre-QA Gate → Evaluator**, with hooks, lints, sandbox, and an anti-slop improvement loop.

This repo is a **Cursor-only harness scaffold**, not a finished application. You provide a product prompt; the harness creates `docs/` planning artifacts and application code under `app/` sprint by sprint.

## Architecture

```
Layer 3: Phase gates     pre-qa-gate.sh (lints, artifacts, secrets)
Layer 2: Orchestration   Planner → Generator ↔ Evaluator (sprint loop)
Layer 1: Environment     hooks, ESLint plugin, sandbox, review personas
```

**Environment defines the rails. Orchestration drives the train.**

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| **Git repo** | `git init` before setup — hooks install into `.git/hooks/`. Without one the harness runs but **warns that guardrails are OFF** |
| **Bun or Node ≥ 20** | `bun install` preferred; `npm install` works as fallback |
| **Cursor CLI** (`cursor`) | Required for `./cursor-harness.sh`. Must be installed and authenticated |
| **`.env.local`** | Copy from `.env.example`; never commit real secrets |

Optional:
- **gitleaks** — `brew install gitleaks` for full secret scan in pre-commit
- **Playwright** — installed by the Generator under `app/` when it scaffolds your app

## Quick start

```bash
git init
bun install && bun run setup
cp .env.example .env.local

./cursor-harness.sh "Build a project management tool with kanban boards"
```

The harness reads `docs/spec.md` and `docs/sprint-status.md` and resumes from the first sprint not in terminal `Pass` or `Skipped` state.

## Product layout

| Path | Purpose |
|------|---------|
| Repo root | Harness: `scripts/`, `harness/`, `docs/`, `agents/` |
| `app/` | **Product root** — Generator scaffolds here (`app/package.json`, `app/src/`, etc.) |

From sprint 2 onward, the pre-QA gate requires `app/package.json` and application source under `app/`.

## Before your first run

**First runs pause by default.** On a fresh project (no `docs/sprint-status.md` yet) the harness sets `HARNESS_PAUSE=sprint` automatically. Answer `a` at the first checkpoint, or set `HARNESS_PAUSE=off` / `HARNESS_YES=1`, for fully unattended runs.

**Cost is real.** Cap blast radius with `HARNESS_MAX_SPRINTS_PER_RUN=1` and re-run to continue.

**QA needs a runnable app.** The Evaluator starts the dev server from `app/` and drives it with Playwright. First-run failures are often environmental — missing Playwright browsers (`cd app && npx playwright install`) or a port already in use.

**One clone = one product.** Start each product from a fresh clone. The generated app lives under `app/` in this repo.

## Usage modes

### 1. Autonomous (recommended)

```bash
./cursor-harness.sh "your product prompt"
./cursor-harness.sh "your product prompt" 5    # max 5 QA rounds per sprint
```

Override the Cursor model with `HARNESS_MODEL` (default: `composer-2.5`).

### 2. Manual phase handoffs

Drive each phase from Cursor Agent Manager:

```bash
./scripts/cursor-plan.sh "your product prompt"
./scripts/cursor-build.sh
./scripts/cursor-qa.sh
./scripts/cursor-post-qa.sh --sprint N --qa-round M
```

Prompts live in `cursor/prompts/`. Each script writes `docs/cursor-handoff.md` with required reads and expected outputs.

## What happens during a run

1. **Planner** (once) — product spec and sprint plan
2. **Per sprint:** Generator → Pre-QA Gate → Evaluator (retry loop on failure)
3. **Anti-slop** — QA failures logged to `.gc-cache/` for weekly guardrail review

Sprint status progression: `Not started` → `In progress` → `Ready for QA` → `Pass` or `Fail` (or `Skipped` when advancing after max QA rounds).

| Phase | Who | Key outputs |
|-------|-----|-------------|
| **Planner** | Cursor agent | `docs/spec.md`, `docs/sprint-plan.md`, `docs/sprint-status.md` |
| **Generator** | Cursor agent | `docs/sprint-[N]-contract.md`, code under `app/` |
| **Pre-QA Gate** | Shell script | `docs/mechanical-checks-sprint-[N].md` |
| **Evaluator** | Cursor agent | `docs/qa-report-sprint-[N].md`, status → Pass/Fail |

### Agent watchdog

`cursor agent` sometimes finishes writing artifacts but never exits. The harness runs an **artifact watchdog** (on by default):

```bash
HARNESS_AGENT_WATCHDOG=1          # set 0 to wait for cursor agent to exit on its own
HARNESS_AGENT_POLL_SEC=15
HARNESS_AGENT_STABLE_POLLS=2
HARNESS_PHASE_TIMEOUT=7200
```

## Documentation map

| If you want to… | Read |
|-----------------|------|
| Visual introduction | [`docs/guide.html`](docs/guide.html) |
| Phase boundaries and file ownership | [`docs/runtime-contract.md`](docs/runtime-contract.md) |
| Agent sandbox, lint, anti-slop rules | [`harness/AGENT-INSTRUCTIONS.md`](harness/AGENT-INSTRUCTIONS.md) |
| Agent personas | [`agents/`](agents/) |
| Product root convention | [`app/README.md`](app/README.md) |
| Extend lints and guardrails | [`CONTRIBUTING.md`](CONTRIBUTING.md) |

## Key files

| Path | Purpose |
|------|---------|
| `cursor-harness.sh` | Autonomous loop via Cursor CLI |
| `scripts/cursor-*.sh` | Manual phase handoffs |
| `cursor/prompts/` | Handoff prompts for Agent Manager |
| `scripts/pre-qa-gate.sh` | Mechanical gate between Generator and Evaluator |
| `app/` | Product root (Generator scaffolds here) |
| `sdk-orchestrator/` | Validation/handoff helpers |
| `harness/AGENT-INSTRUCTIONS.md` | Universal agent rules |
| `agents/*.md` | Planner, Generator, Evaluator personas |

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HARNESS_MODEL` | `composer-2.5` | Cursor model for all phases |
| `HARNESS_ON_MAX_ROUNDS` | `halt` | `advance` to move on with known failures |
| `HARNESS_MAX_QA_ROUNDS` | `3` | Max Generator↔Evaluator retries per sprint |
| `HARNESS_PAUSE` | `off` (`sprint` on fresh project) | `sprint` / `phase` / `design` pause modes |
| `HARNESS_YES` | `0` | Set to `1` to skip pause prompts |
| `HARNESS_MAX_SPRINTS_PER_RUN` | unlimited | Stop after N sprints; re-run to resume |
| `HARNESS_AGENT_WATCHDOG` | `1` | Stop hung `cursor agent` when artifacts land |
| `HARNESS_SANDBOX` | git root | Pre-commit filesystem boundary |

## Guardrails

```bash
bun lint:harness          # ESLint rules with agent-prompt error messages
bun gc:weekly             # Review recurring failures → new rules
bun run setup             # Install pre-commit hook + .cursorignore
npm run test:harness      # Harness unit tests
```
