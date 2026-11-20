# BetMetrics — shared rules

BetMetrics compares online casinos on **neutral technical data**: performance (page load, uptime, game load speed), game availability, and offers. Target markets are Europe (Spain first) and Brazil/Argentina. Launch languages are English (`en`) and Portuguese (`pt-BR`).

These rules apply to every human and every agent working in this repo.

## Engineering rules

1. **TypeScript everywhere.** All source, tests, scripts and config that can be TypeScript must be TypeScript, with `strict: true`. No new `.js` files except tool configs that require them (e.g. `eslint.config.mjs`). Every workspace `tsconfig.json` extends [tsconfig.base.json](tsconfig.base.json).
2. **Feature branches + pull requests only.** Branch from `main` as `feat/…`, `fix/…`, `chore/…`, `docs/…` or `test/…`. One logical change per PR.
3. **Never push to `main`.** No direct pushes, no force pushes, no history rewrites on `main`. All changes go through a PR.
4. **CI must pass before merge.** The `CI` workflow ([.github/workflows/ci.yml](.github/workflows/ci.yml)) runs lint, typecheck and tests on every PR. Don't merge on red, and don't disable or skip checks to get green.
5. **Secrets only in environment variables.** Never commit API keys, service-account JSON, tokens or `.env` files. Document every variable (name only, no value) in [.env.example](.env.example). In CI, use GitHub Actions secrets. In Cloud Functions, use Firebase secret params (`defineSecret`).

## The API contract is the source of truth

- [docs/api-contract.md](docs/api-contract.md) defines the data model and endpoints shared by backend, frontend and tests.
- A contract change must come in its own PR, or a PR whose first commit is the contract change, _before_ any code depends on it. Bump the contract version in the doc's changelog.
- The backend implements the contract, the frontend consumes it, and the tests verify it. Nobody invents fields that aren't in the contract.

## Compliance and content rules (product-level, non-negotiable)

Gambling advertising is heavily regulated in our markets (e.g. Spain's Royal Decree 958/2020, Brazil's Law 14.790/2023 and SPA/MF ordinances, Argentine provincial rules).

- **Lead with neutral technical data.** Rankings and default sorting are based on measured performance and availability, never on offers or commercial relationships.
- **Do not promote bonuses.** No bonus amounts in headlines, hero sections, CTAs, banners, push-style UI or page titles. Offers are shown only on a casino's detail page, in a factual, de-emphasised section, and only where the contract's `displayAllowed` flag permits it for that market.
- **Licensed operators only.** A casino is listed in a market only if it has a verified licence for that market.
- **Responsible gambling.** Every page shows an 18+ notice and links to the market's responsible-gambling resources.
- **Neutral wording.** Use no persuasive or urgency language ("claim now", "don't miss", "best bonus"). Copy describes; it does not sell.
- These rules are product guidance, not legal advice. Anything market-specific needs legal review before launch.

## Repository layout

| Path              | Owner agent | Contents                                                                                       |
| ----------------- | ----------- | ---------------------------------------------------------------------------------------------- |
| `/backend`        | `backend`   | Firebase: Cloud Functions (read-only HTTP API + scheduled collectors), Firestore rules/indexes |
| `/frontend`       | `frontend`  | Next.js (App Router) site, mobile-first, i18n `en` / `pt-BR`                                   |
| `/tests`          | `qa`        | Playwright end-to-end tests and Playwright API tests                                           |
| `/docs`           | shared      | API contract and other design docs                                                             |
| `/.claude/agents` | shared      | Subagent definitions                                                                           |

Each agent edits only its own folder. A change another folder needs is a request to its owner (or a contract PR), not an edit by you.

## Tooling

- **Node 24** ([.nvmrc](.nvmrc)), **npm workspaces** (`backend`, `frontend`, `tests`), and a single root `package-lock.json`.
- **ESLint** (flat config, `typescript-eslint`) + **Prettier**, configured at the root.
- **Playwright** for both E2E and API tests.
- **Firebase** in region `europe-west1`.

Root commands:

```sh
npm ci                 # install everything
npm run lint           # eslint + prettier --check
npm run format         # prettier --write
npm run typecheck      # tsc in every workspace that defines it
npm test               # tests in every workspace that defines them
```

Every workspace must expose `lint`-clean code and, once it has source, `typecheck` and (where relevant) `test` scripts so CI picks them up.
