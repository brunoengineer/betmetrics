# BetMetrics — shared rules

BetMetrics compares online casinos on **neutral technical data**: performance (page load, uptime, game load speed), game availability, and offers. Target markets are Europe (Spain first) and Brazil/Argentina. Supported locales are `en`, `pt-BR`, `es-ES` and `es-AR` (see [Locales and markets](#locales-and-markets)).

These rules apply to every human and every agent working in this repo.

## Engineering rules

1. **TypeScript everywhere.** All source, tests, scripts and config that can be TypeScript must be TypeScript, with `strict: true`. No new `.js` files except tool configs that require them (e.g. `eslint.config.mjs`). Every workspace `tsconfig.json` extends [tsconfig.base.json](tsconfig.base.json).
2. **Feature branches + pull requests only.** Branch from `main` as `feat/…`, `fix/…`, `chore/…`, `docs/…` or `test/…`. One logical change per PR.
3. **Never push to `main`.** No direct pushes, no force pushes, no history rewrites on `main`. All changes go through a PR.
4. **CI must pass before merge.** The `CI` workflow ([.github/workflows/ci.yml](.github/workflows/ci.yml)) runs lint, typecheck and tests on every PR. Don't merge on red, and don't disable or skip checks to get green.
5. **Secrets only in environment variables.** Never commit API keys, service-account JSON, tokens or `.env` files. Document every variable (name only, no value) in [.env.example](.env.example). In CI, use GitHub Actions secrets. In Cloud Functions, use Firebase secret params (`defineSecret`).

## Development workflow (working with Claude Code)

Every piece of work follows these steps. The developer runs every commit, push and PR action. Claude proposes them but never runs them.

1. **Branch.** Start from an up-to-date `main` and create a feature branch (`git switch main`, `git pull`, `git switch -c feat/…`). Claude may create the local branch, and confirms which branch the work is on before changing files.
2. **Work in small steps.** After each logical step, Claude runs `npm run lint` (and `typecheck` / `test` once they exist) and then **proposes the git commands**:
   - `git add <explicit paths>`. Use explicit paths, not `-A`, so stray files don't slip in.
   - `git commit -m "<type>: <summary>"`, using Conventional Commits types (`feat`, `fix`, `chore`, `docs`, `test`, `style`, `refactor`).
   - `git push`, or `git push -u origin <branch>` the first time.
3. **PR-ready checkpoint.** When Claude believes the branch is a complete, single logical change, it marks its message **PR-ready**, lists what the PR contains, and proposes the final git commands. This is the only point where Claude brings up a review. The developer confirms the scope or asks for more work, which takes the branch back to step 2.
4. **Review: only on an explicit trigger.** Claude runs the review **only** when the developer says **"review"** (e.g. "pushed, review"). A normal "done", "committed" or "pushed" during step 2 never starts one, and Claude never starts a review on its own.
   - Claude starts a separate reviewer agent with no conversation history, so the review isn't biased by having written the code. That agent follows [.claude/commands/pr-review.md](.claude/commands/pr-review.md) against the current branch, and the report is saved in `qa-agent-hub/response/pr-review/`.
   - If the branch changes `/backend` or `/frontend`, the same trigger also runs the `qa` agent's coverage review, which reports test gaps by priority.
   - Each trigger runs exactly one review. Claude relays the findings, the developer decides which to fix, and the fixes go back through step 2.
   - After review fixes are pushed, Claude asks whether to review again. It doesn't re-run until the developer says "review" again.
5. **Describe: only on an explicit trigger.** When the developer says **"describe"** (meaning the review is closed), Claude runs [.claude/commands/pr-description.md](.claude/commands/pr-description.md) on the final branch and proposes the PR title and body. The description comes after the review so it reflects the final code, including the review fixes. It's saved in `qa-agent-hub/response/pr-description/`.
6. **Open the PR.** The developer opens the PR into `main` with that title and body.
7. **Address PR feedback.** The developer pastes reviewer comments or CI failures into the session. Claude fixes them on the same branch, and the fixes go back through step 2.
8. **Merge.** The developer merges once CI is green and reviews are resolved.

Rules for every step:

- No AI attribution in commits, PR titles or PR bodies: no `Co-Authored-By` trailers and no "Generated with …" lines.
- `qa-agent-hub/` holds local generated reports. It is git-ignored and never committed.
- Anything outside the branch's scope gets flagged, with a suggestion to move it to its own branch. It isn't silently added.

## The API contract is the source of truth

- [docs/api-contract.md](docs/api-contract.md) defines the data model and endpoints shared by backend, frontend and tests.
- A contract change must come in its own PR, or a PR whose first commit is the contract change, _before_ any code depends on it. Bump the contract version in the doc's changelog.
- The backend implements the contract, the frontend consumes it, and the tests verify it. Nobody invents fields that aren't in the contract.

## Locales and markets

**Supported locales:** `en` (default), `pt-BR`, `es-ES` and `es-AR`. Every user-facing feature must work in all four.

- **Spanish shares one base.** Spanish copy is written once as `es` (a base translation, never a selectable locale). `es-ES` and `es-AR` contain only the overrides where wording, currency formatting or legal notices differ. Fallback chains: `es-ES` → `es` → `en`, `es-AR` → `es` → `en`, `pt-BR` → `en`.
- **No hardcoded user-facing strings** anywhere: UI copy lives in message files, and notices and offer text come from the API.

**Market and language are separate things:**

| Concept                   | Values                          | Decides                                                                                                   | Chosen by                                                         |
| ------------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| **Market** (jurisdiction) | `es`, `br`, `ar`                | Which casinos are listed (licensing), offers, games, currency, 18+ notice, responsible-gambling resources | **Geolocation**, server-side. Locked: the visitor can't change it |
| **Locale** (language)     | `en`, `pt-BR`, `es-ES`, `es-AR` | Language and date/number formatting **only**                                                              | The visitor (switcher, `Accept-Language`, saved preference)       |

- A visitor in Spain sees **Spanish-licensed casinos only**, in EUR, with Spain's notices, whatever language they read in. Switching to English or Portuguese changes the words, not the data.
- Never infer a market from a locale, or a locale from a market. `es-AR` doesn't mean "show Argentine casinos"; the visitor's location decides that. A market's `defaultLocale` is only a first-visit suggestion.
- Currency always comes from the market's data (`EUR` for `es`, `BRL` for `br`, `ARS` for `ar`). The locale only decides how the amount is formatted.
- Visitors outside the supported markets get a neutral information page with no casino listings, measurements or offers.

## Compliance and content rules (product-level, non-negotiable)

Gambling advertising is heavily regulated in our markets (e.g. Spain's Royal Decree 958/2020, Brazil's Law 14.790/2023 and SPA/MF ordinances, Argentine provincial rules).

- **Lead with neutral technical data.** Rankings and default sorting are based on measured performance and availability, never on offers or commercial relationships.
- **Do not promote bonuses.** No bonus amounts in headlines, hero sections, CTAs, banners, push-style UI or page titles. Offers are shown only on a casino's detail page, in a factual, de-emphasised section, and only where the contract's `displayAllowed` flag permits it for that market.
- **Licensed operators only.** A casino is listed in a market only if it has a verified licence for that market.
- **Responsible gambling.** Every page shows the visitor's market's 18+ notice and links to that market's responsible-gambling resources, in the visitor's locale. Both come from the API's `Market` data, never from hardcoded copy.
- **Neutral wording.** Use no persuasive or urgency language ("claim now", "don't miss", "best bonus"). Copy describes; it does not sell.
- These rules are product guidance, not legal advice. Anything market-specific needs legal review before launch.

## Repository layout

| Path              | Owner agent | Contents                                                                                       |
| ----------------- | ----------- | ---------------------------------------------------------------------------------------------- |
| `/backend`        | `backend`   | Firebase: Cloud Functions (read-only HTTP API + scheduled collectors), Firestore rules/indexes |
| `/frontend`       | `frontend`  | Next.js (App Router) site, mobile-first, i18n `en` / `pt-BR` / `es-ES` / `es-AR`               |
| `/tests`          | `qa`        | Playwright end-to-end tests and Playwright API tests                                           |
| `/docs`           | shared      | API contract and other design docs                                                             |
| `/.claude/agents` | shared      | Subagent definitions                                                                           |

Each agent edits only its own folder. A change another folder needs is a request to its owner (or a contract PR), not an edit by you.

**Agents never decide these on their own.** Adding, removing or upgrading a dependency, or changing `docs/api-contract.md`, always needs the developer's approval. Agents stop and return a proposal, and the main session asks the developer.

**Parallel agents:** when more than one agent works at the same time, the main session gives each one its own git worktree (or the agent creates one from `origin/main`). Each worktree has its own branch and becomes its own PR. Agents never commit, push or open PRs. Each one ends with a report giving its worktree, branch, check results, the git commands for the developer, and a draft PR title.

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
