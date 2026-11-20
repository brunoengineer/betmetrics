# BetMetrics API contract

**Status:** Draft v0.2.0. Nothing is implemented yet, and every section is open for review.

This document is the single source of truth for the data model and the public, read-only HTTP API shared by `/backend`, `/frontend` and `/tests`. See the change process in [CLAUDE.md](../CLAUDE.md#the-api-contract-is-the-source-of-truth).

---

## 1. Conventions

| Topic       | Rule                                                                                                                                                                          |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Base URL    | `https://<api-host>/v1`, with the version in the path. Breaking changes require `/v2`.                                                                                        |
| Methods     | `GET` only. The public API is read-only. Data is written by scheduled collectors and an internal admin process, which are not part of this contract.                          |
| Format      | JSON, UTF-8, `camelCase` keys.                                                                                                                                                |
| IDs         | Human-readable `slug` strings (`^[a-z0-9-]+$`), stable once published.                                                                                                        |
| Timestamps  | ISO 8601 in UTC, e.g. `2026-11-20T14:05:00Z`.                                                                                                                                 |
| Durations   | Integer **milliseconds**, with fields suffixed `Ms`.                                                                                                                          |
| Percentages | Number in `0–100`, max 3 decimals, with fields suffixed `Pct`.                                                                                                                |
| Money       | `{ "amount": number, "currency": "EUR" \| "BRL" \| "ARS" }`. The currency comes from the market, never from the locale (see [§1.2](#12-markets-and-locales-are-independent)). |
| Markets     | `market` is a lowercase market code: `es`, `br`, `ar` (more EU markets later).                                                                                                |
| Locale      | `?locale=` is one of `en`, `pt-BR`, `es-ES` or `es-AR` (default `en`). See [§1.1](#11-locales).                                                                               |
| Pagination  | Cursor-based: `?limit=` (default 20, max 100) and `?cursor=`. Responses include `nextCursor` (`null` at the end).                                                             |
| Caching     | Responses send `Cache-Control: public, max-age=300, stale-while-revalidate=600` unless an endpoint says otherwise.                                                            |
| Errors      | See [§4](#4-errors).                                                                                                                                                          |

### 1.1 Locales

```ts
type Locale = 'en' | 'pt-BR' | 'es-ES' | 'es-AR';

// Translation keys used in storage. 'es' is the shared Spanish base and is never a public locale.
type TranslationKey = Locale | 'es';

// A value stored per translation key. 'en' is always required, because it is the final fallback.
type Localized<T> = { en: T } & Partial<Record<Exclude<TranslationKey, 'en'>, T>>;
type LocalizedText = Localized<string>;
type LocalizedUrl = Localized<string>; // for resources that have per-language pages
```

| Locale  | Language            | Fallback chain        |
| ------- | ------------------- | --------------------- |
| `en`    | English             | `en`                  |
| `pt-BR` | Portuguese (Brazil) | `pt-BR` → `en`        |
| `es-ES` | Spanish (Spain)     | `es-ES` → `es` → `en` |
| `es-AR` | Spanish (Argentina) | `es-AR` → `es` → `en` |

- **Spanish shares one base.** Spanish text is written once under `es`. `es-ES` and `es-AR` hold **overrides only**, where wording, legal notices or regional usage differ (e.g. _vos_ forms in `es-AR`). A field that has only `es` serves both Spanish locales.
- **The API resolves the locale on the server.** In public responses, every `Localized<T>` field is returned as a single resolved `T` for the requested locale, using the chain above. The data model below shows the stored (`Localized<…>`) shape. The response shape is the same with those fields resolved.
- Responses include `meta.locale` (the requested locale) and a `Content-Language` header with the same value.
- `?locale=es`, or any value not in the table, returns `400 INVALID_PARAMETER`.
- Error `message` strings are developer-facing and always in English.

### 1.2 Markets and locales are independent

- **The market** (`es`, `br`, `ar`) decides _what_ is shown: which casinos (licensing), which offers, which games, the currency, and the legal and responsible-gambling notices.
- **The locale** decides only _how_ it is shown: the language and the date and number formats.
- The two never imply each other. For example, a visitor in Spain who reads in English (`market=es`, `locale=en`) sees only DGOJ-licensed casinos, amounts in EUR, and Spain's notices translated into English. A visitor in Argentina who reads in Portuguese sees ARS amounts and Argentine notices.
- **The visitor's market is geolocated and locked.** The frontend derives it server-side from the visitor's country and does not let visitors change it. Visitors from outside the supported markets get a neutral information page with no casino listings, measurements or offers. The API does not geolocate: it serves whichever `market` it is asked for, and enforcing the lock is the frontend's job (see [§6](#6-open-questions)).
- `Market.defaultLocale` is only a starting suggestion, used when the visitor has no saved language preference and their `Accept-Language` doesn't match a supported locale.

### Response envelope

```jsonc
// List
{ "data": [ /* items */ ], "nextCursor": "opaque-string-or-null", "meta": { "generatedAt": "2026-11-20T14:05:00Z", "locale": "es-AR" } }
// Single
{ "data": { /* item */ }, "meta": { "generatedAt": "2026-11-20T14:05:00Z", "locale": "es-AR" } }
```

---

## 2. Data model

TypeScript is normative. Shared types should eventually live in one place, with the location to be decided by a later contract PR. Fields typed `Localized<…>` are stored per translation key and resolved to a single value in API responses (see [§1.1](#11-locales)).

### 2.1 Market

```ts
type MarketCode = 'es' | 'br' | 'ar';
type Currency = 'EUR' | 'BRL' | 'ARS';

interface Market {
  code: MarketCode;
  name: LocalizedText;
  currency: Currency; // es → EUR, br → BRL, ar → ARS
  defaultLocale: Locale; // es → es-ES, br → pt-BR, ar → es-AR
  regulators: string[]; // e.g. ["DGOJ"], ["SPA/MF"], ["LOTBA", "IPLyC"]
  minimumAge: number; // 18
  legalNotices: {
    ageNotice: LocalizedText; // this market's 18+ wording, shown on every page
    regulatorNotice: LocalizedText | null; // any other statement the market requires
  };
  responsibleGambling: ResponsibleGamblingResource[]; // ordered: the first is the primary resource
}

interface ResponsibleGamblingResource {
  kind: 'support' | 'self_exclusion' | 'regulator';
  name: LocalizedText;
  url: LocalizedUrl; // per-language page where the resource has one
  phone: string | null; // E.164, e.g. "+34900000000"
}
```

Notice wording and resources are **data, not UI strings**, so legal can update them without a frontend release. Every market must have `legalNotices.ageNotice` and at least one `responsibleGambling` entry with text for `en` and for its default locale's language (`pt-BR` for `br`; `es` or the regional key for `es` and `ar`).

### 2.2 Casino

```ts
interface Casino {
  slug: string; // "example-casino"
  name: string; // brand name, not translated
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

**Listing rule:** a casino appears in a market's results only if it has a `Licence` for that market with `status: 'active'`. The locale never affects which casinos are listed.

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
  name: string; // product name, not translated
  providerSlug: string;
  providerName: string;
  type: GameType; // an enum: the frontend translates its label
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
    minDeposit: Money | null; // currency always equals the market's currency
    maxValue: Money | null;
    eligibility: LocalizedText; // e.g. "Accounts older than 30 days"
    termsUrl: LocalizedUrl; // operator's terms page, per language where available
  };
  validFrom: string;
  validUntil: string | null;
  displayAllowed: boolean; // set by compliance review for this market
  complianceReviewedAt: string;
}

interface Money {
  amount: number;
  currency: Currency;
}
```

**Rule:** the API **never** returns an offer with `displayAllowed: false` or an expired `validUntil`. The field is in the model for storage and admin purposes. Public clients will always see `true`.

An offer belongs to exactly one market, and its compliance review covers every locale of that market. A Spanish offer is reviewed once, and its `en`, `es` and `es-ES` texts are all part of that review.

---

## 3. Endpoints (read-only)

All endpoints accept `?locale=` (default `en`). It changes only the language of `Localized` fields, never which items are returned.

### Markets

| Method & path            | Description                                                                                           |
| ------------------------ | ----------------------------------------------------------------------------------------------------- |
| `GET /v1/markets`        | All supported markets.                                                                                |
| `GET /v1/markets/{code}` | One `Market`, including its legal notices and responsible-gambling resources. Returns 404 if unknown. |

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
| `GET /v1/health` | `{ "status": "ok", "version": "<api semver>", "contractVersion": "0.2.0" }`. Not cached. |

---

## 4. Errors

```jsonc
{ "error": { "code": "NOT_FOUND", "message": "Casino 'foo' not found", "details": {} } }
```

| HTTP | `code`                                    | When                                                                                             |
| ---- | ----------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 400  | `INVALID_PARAMETER` / `MISSING_PARAMETER` | Bad or missing query parameter, including an unsupported `locale` (`details.parameter` names it) |
| 404  | `NOT_FOUND`                               | Unknown slug, market code or route                                                               |
| 405  | `METHOD_NOT_ALLOWED`                      | Any non-`GET` method                                                                             |
| 429  | `RATE_LIMITED`                            | Rate limit exceeded (`Retry-After` header)                                                       |
| 500  | `INTERNAL`                                | Unexpected error (no stack traces in the body)                                                   |

---

## 5. Storage notes (non-binding, for `/backend`)

- Collections: `markets`, `casinos`, `games`, `casinos/{slug}/availability`, `casinos/{slug}/offers`, `measurements` (top-level, indexed on `casinoSlug + type + measuredAt`), `measurementBuckets` (pre-aggregated hourly/daily).
- `performanceSummary` is recomputed by a scheduled job and denormalised onto the casino document so list views are a single query.
- `Localized<…>` fields are stored as maps keyed by `TranslationKey` (`en`, `pt-BR`, `es`, `es-ES`, `es-AR`) and resolved in the API layer. Don't store pre-resolved copies per locale.
- Firestore security rules deny all client access. The public API goes through Cloud Functions only.
- Region: `europe-west1`. A second probe region (`southamerica-east1`) runs collectors only.

## 6. Open questions

- Collector cadence (proposal: uptime every 5 min, page load every 30 min, game load every 6 h for a sample of games).
- Auth/rate limiting for the public API (API key vs. App Check vs. none behind CDN).
- Legal review of the offer display rules per market.
- Legal review of each market's `legalNotices` and `responsibleGambling` content in every locale.
- Where geolocation comes from (the hosting platform's country header vs. a geo-IP lookup) and how strictly it must be applied (VPNs, border regions).
- Whether the API itself must also restrict responses by the caller's location, or the frontend's market lock is enough.
- Argentina licenses gambling by province. `ar` may need to split into provincial markets (e.g. CABA, Buenos Aires Province) with separate notices.

## Changelog

| Version | Date       | Change                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0.2.0   | 2026-11-20 | Added Spanish: locales `es-ES` and `es-AR`, sharing an `es` base with regional overrides and a defined fallback chain. `Localized<T>` replaces the two-language `LocalizedText`, and the API now resolves the locale on the server (`meta.locale`, `Content-Language`). Documented that market and locale are independent, with the market geolocated and locked. `Market` gains `defaultLocale`, `legalNotices` and `responsibleGambling[]`, which replace `responsibleGamblingUrl` (**breaking**). `Offer.terms.termsUrl` is now localised. Added `GET /v1/markets/{code}`. Resolved the open question about Spanish. |
| 0.1.0   | 2026-11-20 | Initial draft                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
