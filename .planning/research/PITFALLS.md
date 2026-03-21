# Pitfalls Research

**Domain:** Personal portfolio management / finance aggregation
**Researched:** 2026-03-21
**Confidence:** HIGH (core pitfalls verified across multiple sources, integration-specific items at MEDIUM)

## Critical Pitfalls

### Pitfall 1: Floating-Point Arithmetic for Financial Values

**What goes wrong:**
JavaScript uses IEEE 754 binary floating-point. `0.1 + 0.2 !== 0.3`. In a portfolio tracker, this means net worth totals drift, allocation percentages don't sum to 100%, and P&L calculations accumulate rounding errors over time. A 32-bit float cannot even represent $25,474,937.47 correctly -- it rounds to $25,474,936.32, off by $1.15. At portfolio scale with daily snapshots and currency conversions, these errors compound silently.

**Why it happens:**
JavaScript's `number` type is a 64-bit float by default. Developers use it without thinking because it "mostly works" during early development with small test amounts. The errors only surface at scale or when users compare totals that should match but are off by pennies.

**How to avoid:**
- Store all monetary values in PostgreSQL as `Decimal` (Prisma maps this to PostgreSQL NUMERIC). Never use `Float` in the Prisma schema for financial values. Never use PostgreSQL's `MONEY` type either (it has locale-dependent parsing and truncates on division instead of rounding).
- In TypeScript, use `decimal.js` for all server-side calculations. It provides arbitrary-precision decimal-aware arithmetic.
- Define a clear rounding strategy: round to 2 decimal places only at display time, never during intermediate calculations. Use 8 decimal places for crypto values.
- Store currency codes alongside all monetary values (ISO 4217 for fiat, standard symbols for crypto).
- In Prisma schema: `balance Decimal @db.Decimal(19, 8)` -- 19 total digits, 8 after the decimal point. This handles both fiat precision and crypto's sub-cent amounts.

**Warning signs:**
- Allocation percentages sum to 99.98% or 100.02% in the UI.
- P&L figures differ depending on calculation order.
- Snapshot comparisons show small unexplained drifts.
- Tests pass with round numbers but fail with realistic values like 33.33%.

**Phase to address:**
Phase 1 (database schema and core data models). This must be correct from day one -- retrofitting precision into an existing schema with historical data is extremely painful.

---

### Pitfall 2: CCXT Exchange API Inconsistencies Across Exchanges

**What goes wrong:**
CCXT promises a "unified API" but exchange implementations differ significantly. `fetchBalance()` returns spot balance by default and silently omits margin, futures, and funding wallets. Different exchanges normalize currency symbols differently (DSH vs DASH, USD vs USDT). Some exchanges require account type params (`{type: 'spot'}`), others ignore them. Rate limits vary per exchange and per endpoint -- Kraken has Cloudflare-level DDoS protection that triggers even with CCXT's built-in rate limiter enabled.

**Why it happens:**
Developers test against one exchange (usually Binance), assume the unified API handles all differences, and discover the inconsistencies only when adding a second or third exchange.

**How to avoid:**
- Build an exchange adapter abstraction layer on top of CCXT. Each exchange gets its own configuration that handles: account type params, symbol normalization, rate limit tuning, and error mapping.
- Always pass explicit `{type: 'spot'}` params to `fetchBalance()`. Log warnings when an exchange returns unexpected structure.
- Implement per-exchange retry logic with exponential backoff (3-5 retries for `NetworkError` and `ExchangeNotAvailable`, zero retries for `AuthenticationError`).
- Use Zod to validate the shape of CCXT responses before passing to business logic.
- Test against every exchange you plan to support, not just one.

**Warning signs:**
- Balance totals change when you add a new exchange.
- Intermittent "rate limit exceeded" errors in logs.
- Currency symbols appearing as duplicates.
- Silent zero balances from exchanges that require specific wallet type params.

**Phase to address:**
Phase 3 (exchange integration). Design the adapter pattern from the start.

---

### Pitfall 3: Trading 212 Pies API Deprecation and Limitations

**What goes wrong:**
The Trading 212 Pies API endpoint is officially deprecated -- "won't be further supported and updated." It still works today but could break at any time with no fix incoming. Additionally, the Pies API is missing critical features: no auto-rebalance endpoint, no pie-scoped transaction history, and no foreign exchange rate data. Rate limits are tight and endpoint-specific (e.g., account summary: 1 request per 5 seconds, instruments list: 1 request per 50 seconds). The API is beta-only.

**Why it happens:**
Trading 212 is building a replacement API ("major API upgrade coming in the coming months" -- stated since mid-2025). Developers build around the current API assuming stability, then face a forced migration.

**How to avoid:**
- Isolate all Trading 212 API calls behind a dedicated adapter service with a clean interface. When the API changes, only the adapter needs updating.
- Cache T212 responses aggressively -- with rate limits this tight, you cannot afford redundant calls.
- Respect `x-ratelimit-reset` headers.
- Do NOT depend on any undocumented behavior.
- Build a fallback path: if the T212 API dies, manual entry for those assets.

**Warning signs:**
- HTTP 429 responses from T212 even with seemingly low request volume.
- Pies endpoint returning different data structure than documented.
- Community reports of endpoint removals or changes.

**Phase to address:**
Phase 3 (T212 integration). Build with explicit awareness that this API will change.

---

### Pitfall 4: Snapshot Scheduling Silently Fails in Docker

**What goes wrong:**
NestJS `@nestjs/schedule` runs cron jobs in-process. When the Docker container restarts, any scheduled job that was supposed to run during downtime is permanently missed -- there is no job recovery, no replay, no "catch-up" mechanism. A documented bug in node-cron causes jobs to stop firing after random hours/days with no error logged. Timezone misconfiguration between the Docker container (defaults to UTC) and the user's expectations causes snapshots to run at unexpected times.

**Why it happens:**
In-process cron is the simplest approach and works in development. Developers don't notice missed jobs until they check the historical data weeks later and find gaps.

**How to avoid:**
- Set timezone explicitly in the cron decorator: `@Cron('0 2 * * *', { timeZone: 'Europe/London' })`. Make timezone configurable via environment variable.
- Add a "snapshot health check" that runs on application startup: query the last snapshot timestamp and trigger a catch-up snapshot if it is older than expected.
- Make snapshot creation idempotent -- if called twice for the same date, it updates rather than duplicates. This allows safe manual re-triggers.
- Log every snapshot attempt (start, success, failure) with timestamps.
- In historical queries, use PostgreSQL `generate_series()` to produce continuous date ranges and LEFT JOIN with actual snapshot data, filling gaps with the last known value.

**Warning signs:**
- Gaps in the historical chart with no corresponding error logs.
- Snapshot timestamps drifting or appearing at wrong times.
- Docker logs showing container restarts during scheduled snapshot windows.

**Phase to address:**
Phase 4 (snapshot/historical data). Build health check and idempotency alongside scheduling.

---

### Pitfall 5: Currency Conversion Compounding Errors and Stale Rates

**What goes wrong:**
If exchange rates are stale (free APIs update once per day at most), the net worth figure can be wrong on volatile days. Chain conversions (e.g., CHF -> USD -> EUR instead of direct CHF -> EUR) compound conversion errors. Using different rate sources for different conversions creates inconsistencies. Crypto-to-fiat rates from CCXT may differ from forex rates from a currency API.

**Why it happens:**
Developers pick one exchange rate API, assume it covers all needs, and don't think about the conversion graph.

**How to avoid:**
- Use a single, consistent rate source per asset type per snapshot. Fiat: Frankfurter API. Crypto: CCXT ticker from the exchange where the crypto is held.
- Store the exchange rates used alongside each snapshot so historical values can be reconstructed.
- Always prefer direct conversion (EUR -> USD) over chain conversion. The Frankfurter API provides cross-rates directly.
- Store rates as fetched (never round intermediate rates). Apply rounding only at display.
- Include rate fetch timestamp in the snapshot metadata so users understand data freshness.

**Warning signs:**
- Net worth jumps that don't correlate with market movements.
- Different portfolio views showing slightly different totals for the same date.
- Rate API returning HTTP errors silently (cached stale rate being reused indefinitely).

**Phase to address:**
Phase 3 (exchange integration + currency service). Rate storage strategy must be designed into the Prisma schema.

---

### Pitfall 6: Rebalancing Calculations with Edge Cases

**What goes wrong:**
Target allocations don't sum to exactly 100% due to rounding (3 categories at 33.33% each = 99.99%); zero-value categories cause division-by-zero; very small allocations generate suggestions below exchange minimums; removing or adding a category doesn't properly redistribute targets. Two-level rebalancing (macro then intra-category) can produce conflicting suggestions.

**Why it happens:**
Rebalancing math is straightforward in the happy path. Edge cases only appear with real portfolio data.

**How to avoid:**
- Normalize target allocations to sum to exactly 100% (using decimal.js). If user enters 33/33/33, store as 33.34/33.33/33.33 with clear UI indication.
- Define minimum trade thresholds. Don't suggest trades below exchange minimums.
- Handle zero and negative values explicitly.
- Use tolerance bands (e.g., +/- 5%) rather than exact targets.
- Write extensive unit tests with edge case fixtures: 0% target, equal splits, zero-value categories, below-minimum trades.

**Warning signs:**
- "Sell all" suggestions for categories that are only slightly overweight.
- NaN or Infinity values appearing in suggestion amounts.
- Allocation pie chart showing a tiny unlabeled slice (the rounding remainder).

**Phase to address:**
Phase 6 (rebalancing features). Build with extensive test coverage from the start.

---

### Pitfall 7: Docker Service Startup Order and Networking

**What goes wrong:**
Docker Compose `depends_on` only waits for the container to start, not for the service inside to be ready. NestJS starts, tries to connect to PostgreSQL, PostgreSQL is still initializing, connection fails, app crashes. Common mistake: using `localhost` in database connection strings instead of the Docker service name.

**Why it happens:**
Works fine in local development where PostgreSQL is already running. Only manifests in fresh `docker-compose up`.

**How to avoid:**
- Use `depends_on` with `condition: service_healthy`.
- Add a healthcheck to the PostgreSQL service: `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}`.
- Use the Docker service name as the database host (`postgres`, not `localhost`).
- Use the internal container port (5432), not the host-mapped port.
- Prisma supports connection retry via the connection string `connect_timeout` parameter.
- Use `.env` file for all connection strings, with Docker-specific overrides in `docker-compose.yml`.

**Warning signs:**
- App works locally but crashes on first `docker-compose up`.
- Intermittent "ECONNREFUSED" errors on startup.

**Phase to address:**
Phase 1 (project scaffolding and Docker setup). Get this right immediately.

---

### Pitfall 8: Recharts Blank Rendering with React 19

**What goes wrong:**
Recharts 3.x has a known bug where charts render as blank/empty containers when used with React 19.2.x. No errors appear in the console. The issue is caused by an internal dependency on `react-is` that does not match the installed React version.

**Why it happens:**
Recharts uses `react-is` internally for component type checking. When React 19 is installed but `react-is` is not explicitly installed at the matching version, the type checks fail silently and charts render nothing.

**How to avoid:**
- Install `react-is` explicitly at the same major version as React: `npm install react-is@^19.0.0`.
- Test chart rendering early in frontend development -- do not wait until the dashboard is fully built.
- If the issue persists despite the fix, switch to Nivo (`@nivo/pie`, `@nivo/line`, `@nivo/bar`) which has confirmed React 19 compatibility.

**Warning signs:**
- Charts render as empty white rectangles with no console errors.
- Charts work in development with React 18 but break after upgrading to React 19.

**Phase to address:**
Phase 5 (dashboard and visualization). Verify chart rendering immediately when setting up the frontend charting layer.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Using JS `number` for money | No extra library, simple code | Accumulating rounding errors, unreliable P&L | Never for financial values |
| Hardcoding base currency to EUR | Skip currency conversion complexity | Complete rewrite when user wants USD or GBP view | Never -- configurable from day one |
| Single CCXT call without adapter layer | Faster initial integration | Every new exchange requires modifying core business logic | Never -- adapter pattern is minimal overhead |
| Storing snapshots without the rates used | Simpler Prisma schema | Historical values cannot be recalculated; changing rate source invalidates all history | Never |
| Using `depends_on` without healthcheck | Simpler docker-compose.yml | Flaky startups, CI failures, wasted debugging time | Never -- healthchecks are 3 lines of YAML |
| Installing nestjs-ccxt or trading212-api | Faster initial setup | Unmaintained packages break with no upstream fix | Never -- wrap directly |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| CCXT `fetchBalance` | Assuming it returns all wallet types | Always pass `{type: 'spot'}` explicitly; query each relevant wallet type separately |
| CCXT rate limiter | Trusting `enableRateLimit: true` alone | Add per-exchange rate limit config and retry with exponential backoff for 429/DDoS errors |
| CCXT currency symbols | Assuming symbols are consistent across exchanges | Use CCXT's currency code normalization but validate: log when an unknown symbol appears |
| Trading 212 Pies | Calling API in a tight loop | Respect `x-ratelimit-reset` header; batch reads; cache aggressively |
| Trading 212 Pies | Expecting rebalance or transaction endpoints | These don't exist -- build rebalancing as read-only suggestions |
| Frankfurter API | Calling for every conversion individually | Fetch all needed rates in one call, cache for snapshot duration, store rates with snapshot |
| PostgreSQL in Docker | Using `localhost:5433` in connection string | Use service name (`postgres`) and internal port (`5432`) |
| Prisma in Docker | Running `prisma migrate dev` in production container | Use `prisma migrate deploy` in production. `migrate dev` drops and recreates in some scenarios. |
| Recharts + React 19 | Installing Recharts without `react-is` | Always install `react-is@^19.0.0` alongside Recharts when using React 19 |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Fetching all exchange balances sequentially | Snapshot takes 30+ seconds with 5+ exchanges | Use `Promise.allSettled()` to fetch in parallel (respecting per-exchange rate limits) | 3+ exchanges |
| Querying raw snapshots for chart data | Chart endpoint takes seconds to respond | Add database indexes on snapshot date columns from day one; consider materialized views after 6+ months | 6+ months of daily snapshots |
| Recalculating currency conversions on every page load | Slow dashboard, redundant API calls | Convert once during snapshot creation; store converted values alongside native | Any multi-currency portfolio |
| Loading full portfolio history for "current" dashboard view | Unnecessary data transfer, slow page load | Separate "current state" query from "historical" query | 1+ year of data |
| Prisma N+1 queries on snapshot entries | Loading snapshots triggers separate query per entry | Use Prisma `include` or `select` to eager-load entries in a single query | Any snapshot with multiple entries |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Storing exchange API keys in plain text in `.env` committed to git | Full exchange account access if repo is exposed | `.env` in `.gitignore` always; use Docker secrets for production; API keys should be read-only with no withdrawal permissions |
| CCXT API keys with trading/withdrawal permissions | Compromised key can drain exchange accounts | Create API keys with read-only permissions only (balance + positions); this app only reads data |
| Exposing NestJS API without any auth on a public network | Anyone can view financial data | Bind to localhost only, or add a simple API key/token for non-local access. Use Docker network isolation. |
| Trading 212 API key in frontend code | Key visible in browser dev tools | All external API calls go through the NestJS backend only; frontend never touches exchange/broker APIs directly |
| Docker container running as root | Container escape gives root access to host | Use non-root user in Dockerfiles (`USER node`); do not mount sensitive host directories |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Floating-point arithmetic | Phase 1: Schema + data models | All Prisma Decimal fields; decimal.js in service layer; unit tests with adversarial numbers |
| CCXT inconsistencies | Phase 3: Exchange integration | Adapter pattern in place; tested against each target exchange; error handling for all CCXT exception types |
| T212 API deprecation | Phase 3: Broker integration | Clean adapter interface; no T212-specific logic leaking into business layer; cached responses |
| Snapshot scheduling failures | Phase 4: Historical data | Health check on startup; idempotent snapshot creation; gap detection in queries |
| Currency conversion errors | Phase 3: Integration | Rates stored per snapshot; single source per asset type; direct conversion preferred |
| Rebalancing edge cases | Phase 6: Rebalancing | Unit tests for: 0% target, equal splits, zero-value categories, below-minimum trades |
| Docker networking | Phase 1: Project setup | Healthchecks in docker-compose; service names in connection strings; cold-start tested |
| Recharts + React 19 | Phase 5: Dashboard | react-is installed; chart rendering verified; Nivo fallback ready |

## Sources

- [CCXT Manual - Rate Limiting](https://github.com/ccxt/ccxt/wiki/manual) - Exchange-specific rate limits and enableRateLimit behavior
- [CCXT Issue #13382 - fetchBalance Spot Default](https://github.com/ccxt/ccxt/issues/13382) - fetchBalance returns spot only by default
- [Trading 212 API Docs](https://docs.trading212.com/api) - Rate limits, beta status, endpoint restrictions
- [Trading 212 Community - API Update](https://community.trading212.com/t/trading-212-api-update/87988) - Beta status, upcoming changes
- [Modern Treasury - Floats Don't Work for Cents](https://www.moderntreasury.com/journal/floats-dont-work-for-storing-cents) - Why integers/decimals over floats
- [Crunchy Data - Working with Money in Postgres](https://www.crunchydata.com/blog/working-with-money-in-postgres) - NUMERIC vs MONEY type
- [Recharts React 19 Fix](https://www.bstefanski.com/blog/recharts-empty-chart-react-19) - react-is workaround
- [Recharts GitHub Issue #6857](https://github.com/recharts/recharts/issues/6857) - Blank rendering with React 19.2.3
- [Docker Compose Health Checks Guide](https://last9.io/blog/docker-compose-health-checks/) - depends_on with service_healthy
- [Decimal.js GitHub](https://github.com/MikeMcl/decimal.js/) - Arbitrary-precision decimal library
- [nestjs-ccxt npm](https://www.npmjs.com/package/nestjs-ccxt) - Unmaintained, last publish 2+ years ago
- [trading212-api GitHub](https://github.com/bennycode/trading212-api) - Unmaintained, last update ~1 year ago

---
*Pitfalls research for: Personal portfolio management / finance aggregation*
*Researched: 2026-03-21*
