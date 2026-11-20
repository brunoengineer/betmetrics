# BetMetrics

BetMetrics is a hub that compares online casinos on **neutral, measured technical data**:

- **Performance:** page load times, uptime and game load speed, measured continuously by scheduled probes in Europe and South America.
- **Game availability:** which games and providers each casino offers, per market.
- **Offers:** a secondary, factual listing shown only where regulations permit it.

Target markets are **Europe** (Spain first) and **Brazil / Argentina**. The site is available in **English** (`en`), **Portuguese** (`pt-BR`) and **Spanish** (`es-ES`, `es-AR`). The language is the visitor's choice. The market (which casinos are shown) comes from the visitor's location and is independent of the language.

> **Compliance first.** Gambling advertising is strictly regulated in our markets (e.g. Spain's Royal Decree 958/2020). BetMetrics leads with technical data and does not promote bonuses. Rankings never depend on offers or commercial relationships, and only operators licensed in a market are listed there. The full rules are in [CLAUDE.md](CLAUDE.md#compliance-and-content-rules-product-level-non-negotiable). For 18+ only.

## Architecture

```
┌───────────────┐   GET /v1/…    ┌──────────────────────────────┐
│  frontend/    │ ─────────────▶ │  backend/ (Firebase)         │
│  Next.js      │                │  Cloud Functions: HTTP API   │
│  4 locales    │                │  Scheduled collectors ──┐    │
└───────────────┘                │  Firestore ◀────────────┘    │
                                 └──────────────────────────────┘
        tests/ — Playwright E2E (frontend) + API tests (backend)
```

| Folder                             | What's inside                                                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| [backend/](backend/)               | Firebase Cloud Functions (read-only API and scheduled performance collectors), plus Firestore. Region `europe-west1`. |
| [frontend/](frontend/)             | Next.js (App Router) site: mobile-first, i18n `en` / `pt-BR` / `es-ES` / `es-AR`, geolocated market.                  |
| [tests/](tests/)                   | Playwright end-to-end and API tests.                                                                                  |
| [docs/](docs/)                     | [API contract](docs/api-contract.md) (data model + endpoints) and design docs.                                        |
| [.claude/agents/](.claude/agents/) | Claude Code subagents: `backend`, `frontend`, `qa`.                                                                   |

The [API contract](docs/api-contract.md) is the source of truth that connects all three.

## Getting started

### Prerequisites

- **Node.js 24** (see [.nvmrc](.nvmrc); `nvm use` picks it up)
- **npm 11+**
- **Firebase CLI** (`npm i -g firebase-tools`) and **Java 21+** for the local emulators, once the backend exists

### Install

```sh
npm ci
cp .env.example .env   # then fill in local values. Never commit .env
```

### Common commands (run from the repo root)

| Command                       | What it does                                                   |
| ----------------------------- | -------------------------------------------------------------- |
| `npm run lint`                | ESLint + Prettier check across the repo                        |
| `npm run format`              | Auto-format with Prettier                                      |
| `npm run typecheck`           | TypeScript check in every workspace                            |
| `npm test`                    | Run tests in every workspace                                   |
| `npm run <script> -w backend` | Run a script in one workspace (`backend`, `frontend`, `tests`) |

> The workspaces are currently empty scaffolds. The run instructions for the API (Firebase emulators), the website (`next dev`) and the Playwright suites will be added here as each part is built.

## Contributing

- Use **TypeScript** everywhere.
- Work on a **feature branch** (`feat/…`, `fix/…`, `docs/…`) and open a **pull request** into `main`. Never push to `main` directly.
- **CI must pass before merge.** [.github/workflows/ci.yml](.github/workflows/ci.yml) runs lint, typecheck and tests on every PR.
- **Secrets go in environment variables only.** Keep them in `.env` locally (git-ignored), GitHub Actions secrets in CI, and Firebase secrets in production.
- Changes to data shapes or endpoints start with a PR to [docs/api-contract.md](docs/api-contract.md).

### Working with Claude Code

This repo defines three subagents in [.claude/agents/](.claude/agents/). Each one is limited to its own folder:

- **backend:** Firebase, Firestore, the API and the collectors (`/backend`)
- **frontend:** the Next.js site and its design (`/frontend`)
- **qa:** Playwright E2E and API tests, plus test-coverage reviews of PRs (`/tests`)

Shared rules for humans and agents are in [CLAUDE.md](CLAUDE.md).
