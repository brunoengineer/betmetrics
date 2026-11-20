# BetMetrics API contract

**Status:** Draft v0.1.0. Nothing is implemented yet, and every section is open for review.

This document is the single source of truth for the data model and the public, read-only HTTP API shared by `/backend`, `/frontend` and `/tests`. See the change process in [CLAUDE.md](../CLAUDE.md#the-api-contract-is-the-source-of-truth).

---

## 1. Conventions

| Topic       | Rule                                                                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Base URL    | `https://<api-host>/v1`, with the version in the path. Breaking changes require `/v2`.                                                               |
| Methods     | `GET` only. The public API is read-only. Data is written by scheduled collectors and an internal admin process, which are not part of this contract. |
| Format      | JSON, UTF-8, `camelCase` keys.                                                                                                                       |
| IDs         | Human-readable `slug` strings (`^[a-z0-9-]+$`), stable once published.                                                                               |
| Timestamps  | ISO 8601 in UTC, e.g. `2026-11-20T14:05:00Z`.                                                                                                        |
| Durations   | Integer **milliseconds**, with fields suffixed `Ms`.                                                                                                 |
| Percentages | Number in `0–100`, max 3 decimals, with fields suffixed `Pct`.                                                                                       |
| Money       | `{ "amount": number, "currency": "EUR" \| "BRL" \| "ARS" }`.                                                                                         |
| Markets     | `market` is a lowercase market code: `es`, `br`, `ar` (more EU markets later).                                                                       |
| Locale      | `?locale=en` or `?locale=pt-BR` (default `en`) selects the translatable text fields. Fallback is `en`.                                               |
| Pagination  | Cursor-based: `?limit=` (default 20, max 100) and `?cursor=`. Responses include `nextCursor` (`null` at the end).                                    |
| Caching     | Responses send `Cache-Control: public, max-age=300, stale-while-revalidate=600` unless an endpoint says otherwise.                                   |
| Errors      | See [§4](#4-errors).                                                                                                                                 |

### Response envelope

```jsonc
// List
{ "data": [ /* items */ ], "nextCursor": "opaque-string-or-null", "meta": { "generatedAt": "2026-11-20T14:05:00Z" } }
// Single
{ "data": { /* item */ }, "meta": { "generatedAt": "2026-11-20T14:05:00Z" } }
```

---

## 2. Data model

TypeScript is normative. Shared types should eventually live in one place, with the location to be decided by a later contract PR.

### 2.1 Market

```ts
type MarketCode = 'es' | 'br' | 'ar';

interface Market {
  code: MarketCode;
  name: LocalizedText;
  currency: 'EUR' | 'BRL' | 'ARS';
  regulators: string[]; // e.g. ["DGOJ"], ["SPA/MF"], ["LOTBA", "IPLyC"]
  responsibleGamblingUrl: string; // shown on every page for this market
  minimumAge: number; // 18
}

type LocalizedText = { en: string; 'pt-BR'?: string };
```

### 2.2 Casino

```ts
interface Casino {
  slug: string; // "example-casino"
  name: string;
  websiteUrl: string; // canonical URL measured by collectors
  logoUrl: string | null;
  markets: MarketCode[]; // markets where a verified licence exists
  licences: Licence[];
  gameProviderSlugs: string[];
  performanceSummary: PerformanceSummary | null; // null until enough data exists
  updatedAt: string;
}

interface Licence {
  market: MarketCode;
  regulator: string; // "DGOJ"
  licenceNumber: string;
  verifiedAt: string; // last manual/automated verification
  status: 'active' | 'suspended' | 'expired';
}

// Rolling aggregates over a window. They power rankings and list views.
interface PerformanceSummary {
  windowDays: 7 | 30;
  probeRegion: ProbeRegion;
  uptimePct: number; // successful checks / total checks
  pageLoad: { p50Ms: number; p75Ms: number; p95Ms: number };
  ttfb: { p50Ms: number; p75Ms: number };
  lcp: { p50Ms: number; p75Ms: number } | null;
  gameLoad: { p50Ms: number; p75Ms: number } | null; // across sampled games
  sampleCount: number;
  computedAt: string;
}
```

**Listing rule:** a casino appears in a market's results only if it has a `Licence` for that market with `status: 'active'`.

### 2.3 Measurement

One raw data point produced by a scheduled collector. It is immutable once written.

```ts
type MetricType = 'uptime' | 'page_load' | 'game_load';
type ProbeRegion = 'europe-west1' | 'southamerica-east1'; // where the probe ran

interface Measurement {
  id: string;
  casinoSlug: string;
  type: MetricType;
  probeRegion: ProbeRegion;
  measuredAt: string;
  success: boolean; // false = timeout / HTTP error / script failure
  httpStatus: number | null;
  errorCode: string | null; // e.g. "TIMEOUT", "DNS", "TLS", "HTTP_5XX"

  // page_load + uptime
  totalMs: number | null; // full response (uptime) or load event (page_load)
  ttfbMs: number | null;
  lcpMs: number | null; // page_load only

  // game_load only
  gameSlug: string | null;
  gameReadyMs: number | null; // navigation start → game canvas/iframe ready

  collectorVersion: string; // semver of the collector that produced it
}
```

### 2.4 Game

```ts
type GameType = 'slot' | 'live' | 'table' | 'crash' | 'other';

interface Game {
  slug: string;
  name: string;
  providerSlug: string;
  providerName: string;
  type: GameType;
  rtpPct: number | null; // provider-published RTP, if any
  thumbnailUrl: string | null;
}

interface GameAvailability {
  casinoSlug: string;
  gameSlug: string;
  market: MarketCode;
  available: boolean;
  lastSeenAt: string; // last time a collector confirmed it
  lastGameLoadMs: number | null;
}
```

### 2.5 Offer

Offers are secondary data. They carry compliance metadata, and the API enforces it.

```ts
interface Offer {
  id: string;
  casinoSlug: string;
  market: MarketCode;
  kind: 'deposit_match' | 'free_spins' | 'cashback' | 'other';
  title: LocalizedText; // factual, neutral wording only
  terms: {
    wageringRequirement: number | null; // e.g. 35 (= 35x)
    minDeposit: Money | null;
    maxValue: Money | null;
    eligibility: LocalizedText; // e.g. "Accounts older than 30 days"
    termsUrl: string;
  };
  validFrom: string;
  validUntil: string | null;
  displayAllowed: boolean; // set by compliance review for this market
  complianceReviewedAt: string;
}

interface Money {
  amount: number;
  currency: 'EUR' | 'BRL' | 'ARS';
}
```

**Rule:** the API **never** returns an offer with `displayAllowed: false` or an expired `validUntil`. The field is in the model for storage and admin purposes. Public clients will always see `true`.

---

## 3. Endpoints (read-only)

All endpoints accept `?locale=` where text is localised.

### Markets

| Method & path     | Description            |
| ----------------- | ---------------------- |
| `GET /v1/markets` | All supported markets. |

### Casinos

| Method & path                             | Description                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /v1/casinos?market=es`               | Casinos licensed in `market` (**required**). Query: `sort` = `uptime` \| `pageLoad` \| `gameLoad` \| `name` (default `uptime`), `window` = `7` \| `30` (default `30`), `probeRegion`, `provider` (game provider slug), `limit`, `cursor`. Returns `Casino[]`. |
| `GET /v1/casinos/{slug}`                  | A single `Casino`. Returns 404 if unknown.                                                                                                                                                                                                                    |
| `GET /v1/casinos/{slug}/measurements`     | Raw or bucketed measurements. Query: `type` (**required**), `from`, `to` (max range 90 days, default the last 7 days), `probeRegion`, `interval` = `raw` \| `hour` \| `day` (default `hour`). Bucketed responses return `MeasurementBucket[]` (below).        |
| `GET /v1/casinos/{slug}/games?market=es`  | `GameAvailability[]` joined with `Game`, filtered to `available: true` by default (`?includeUnavailable=true` to include all).                                                                                                                                |
| `GET /v1/casinos/{slug}/offers?market=es` | `Offer[]` for that market, compliance-filtered (see §2.5). `market` is **required**. Cache `max-age=900`.                                                                                                                                                     |

```ts
interface MeasurementBucket {
  start: string; // bucket start (UTC)
  type: MetricType;
  probeRegion: ProbeRegion;
  count: number;
  successPct: number;
  p50Ms: number | null;
  p75Ms: number | null;
  p95Ms: number | null;
}
```

### Games

| Method & path                            | Description                                                                                     |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `GET /v1/games`                          | Query: `provider`, `type`, `q` (name search, min 2 chars), `limit`, `cursor`. Returns `Game[]`. |
| `GET /v1/games/{slug}`                   | A single `Game`.                                                                                |
| `GET /v1/games/{slug}/casinos?market=es` | Casinos in `market` where the game is available, with `lastGameLoadMs`.                         |

### Health

| Method & path    | Description                                                                              |
| ---------------- | ---------------------------------------------------------------------------------------- |
| `GET /v1/health` | `{ "status": "ok", "version": "<api semver>", "contractVersion": "0.1.0" }`. Not cached. |

---

## 4. Errors

```jsonc
{ "error": { "code": "NOT_FOUND", "message": "Casino 'foo' not found", "details": {} } }
```

| HTTP | `code`                                    | When                                                          |
| ---- | ----------------------------------------- | ------------------------------------------------------------- |
| 400  | `INVALID_PARAMETER` / `MISSING_PARAMETER` | Bad or missing query parameter (`details.parameter` names it) |
| 404  | `NOT_FOUND`                               | Unknown slug or route                                         |
| 405  | `METHOD_NOT_ALLOWED`                      | Any non-`GET` method                                          |
| 429  | `RATE_LIMITED`                            | Rate limit exceeded (`Retry-After` header)                    |
| 500  | `INTERNAL`                                | Unexpected error (no stack traces in the body)                |

---

## 5. Storage notes (non-binding, for `/backend`)

- Collections: `markets`, `casinos`, `games`, `casinos/{slug}/availability`, `casinos/{slug}/offers`, `measurements` (top-level, indexed on `casinoSlug + type + measuredAt`), `measurementBuckets` (pre-aggregated hourly/daily).
- `performanceSummary` is recomputed by a scheduled job and denormalised onto the casino document so list views are a single query.
- Firestore security rules deny all client access. The public API goes through Cloud Functions only.
- Region: `europe-west1`. A second probe region (`southamerica-east1`) runs collectors only.

## 6. Open questions

- Collector cadence (proposal: uptime every 5 min, page load every 30 min, game load every 6 h for a sample of games).
- Whether to split `es` into Spain vs. Argentine Spanish content when a Spanish locale is added.
- Auth/rate limiting for the public API (API key vs. App Check vs. none behind CDN).
- Legal review of the offer display rules per market.

## Changelog

| Version | Date       | Change        |
| ------- | ---------- | ------------- |
| 0.1.0   | 2026-11-20 | Initial draft |
