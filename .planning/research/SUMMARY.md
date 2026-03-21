# Project Research Summary

**Project:** Finance Manager
**Domain:** Personal portfolio management / multi-source finance aggregation
**Researched:** 2026-03-21
**Confidence:** HIGH

## Executive Summary

This is a self-hosted, single-user portfolio tracker that aggregates crypto exchange balances (CCXT), Trading 212 pie holdings, and manually entered assets into a unified net worth view with daily snapshots, historical charts, and two-tier rebalancing suggestions. The expert approach for this class of application is a layered monolith -- NestJS backend with feature-based modules, React SPA for visualization, PostgreSQL for persistence, all containerized via Docker Compose. The closest open-source analogue is Ghostfolio (NestJS + Angular + Prisma + PostgreSQL), which validates this architecture. The key differentiators over existing tools are two-tier rebalancing (macro-category and intra-category) and native Trading 212 integration, neither of which any open-source competitor offers.

The recommended stack is NestJS 11 + React 19 + PostgreSQL 16 + Prisma 7 + CCXT 4.x, with Recharts for charts (plus a react-is fix for React 19 compatibility), TanStack Query for server state, Zustand for client state, and shadcn/ui + Tailwind CSS for the UI layer. All external API integrations (CCXT, Trading 212, Frankfurter currency API) should be wrapped in thin custom services behind a shared adapter interface -- the existing npm wrapper packages (nestjs-ccxt, trading212-api) are unmaintained and must not be used. Financial calculations must use decimal.js with PostgreSQL NUMERIC storage from day one; this is non-negotiable and cannot be retrofitted.

The primary risks are: (1) floating-point arithmetic corrupting financial data if decimal.js is not used consistently, (2) the Trading 212 Pies API being officially deprecated with a replacement "coming soon" since mid-2025, (3) CCXT's "unified" API masking significant per-exchange inconsistencies, and (4) in-process cron scheduling silently missing snapshots during Docker container restarts. All four risks have concrete mitigation strategies documented in the pitfalls research and must be addressed in the phases where the relevant code is written.

## Key Findings

### Recommended Stack

The core stack (NestJS + React + PostgreSQL) is decided upfront and well-suited for the domain. Prisma 7 is the recommended ORM over TypeORM (fragile migrations, weaker types) and Drizzle (less mature NestJS integration). CCXT 4.x is the only viable library for multi-exchange crypto balance fetching. Trading 212 integration should use direct HTTP calls via @nestjs/axios since the API surface is small (~100 lines of code) and the third-party wrapper is stale. The Frankfurter API provides free, no-key-required ECB exchange rate data that matches the daily snapshot cadence perfectly.

**Core technologies:**
- **NestJS 11 + TypeScript 5.7**: Backend framework -- decorator-based architecture maps cleanly to feature modules, built-in DI, scheduling, and validation
- **React 19 + Vite 6**: Frontend -- standard modern React with fast HMR; paired with TanStack Query 5 for server state and Zustand 5 for client state
- **PostgreSQL 16 + Prisma 7**: Database + ORM -- schema-first with compile-time type safety; NUMERIC type for financial precision
- **CCXT 4.x**: Crypto exchange abstraction -- 100+ exchanges, unified balance fetching; wrap directly, never use nestjs-ccxt
- **Recharts 3.8 + react-is**: Charts -- requires explicit react-is@19 install to fix silent blank rendering on React 19
- **decimal.js**: Financial math -- mandatory for all monetary calculations; JavaScript floats are unacceptable
- **@nestjs/schedule**: Daily cron -- simpler than BullMQ, no Redis needed for single-user daily jobs
- **Frankfurter API**: Fiat exchange rates -- free, no API key, ECB-sourced, daily updates
- **shadcn/ui + Tailwind CSS 4**: UI components -- copy-paste model means no dependency to maintain

### Expected Features

**Must have (table stakes -- P1):**
- Unified net worth display combining all sources in base currency
- Manual asset entry (name, currency, balance) for unsupported sources
- Base currency selection + multi-currency conversion (EUR/USD/GBP)
- CCXT integration for automated crypto balance fetching
- Portfolio allocation view (pie chart by macro category)
- Basic P&L display (portfolio-level, absolute + percentage)
- Daily snapshot system (capture at 2 AM, store full portfolio state)
- Docker Compose deployment (backend + frontend + database)

**Should have (differentiators -- P2):**
- Trading 212 pies integration (unique among open-source tools)
- Historical portfolio value charts (line chart with daily/weekly/monthly)
- P&L drill-down by category and individual asset
- Macro-level rebalancing (target allocations, deviation display, suggestions)
- Allocation visualization with target overlay (bar chart)
- Historical performance by category (per-category trend lines)

**Defer (v2+ -- P3):**
- Intra-category rebalancing (requires macro rebalancing working first)
- Trading bot grouping under CCXT (specialized category logic)
- Data export (CSV/JSON)
- PWA wrapper

### Architecture Approach

The architecture is a **layered monolith** with four distinct layers: Presentation (React SPA), API (NestJS controllers), Service (business logic), and Integration (external API adapters). Feature-based NestJS modules (portfolio, assets, snapshots, rebalancing, integrations, scheduler) each own their controller, service, and DTOs. External APIs are isolated behind an `ExchangeAdapter` interface so CCXT, Trading 212, and future sources conform to a common contract. The Prisma schema is the single source of truth for the data model. Currency conversion uses a bridge pattern through a single reference currency to avoid N-squared rate pairs.

**Major components:**
1. **Portfolio Service** -- aggregates all asset sources, computes unified net worth in base currency
2. **Integration Layer (CCXT Adapter, T212 Adapter, Currency Service)** -- isolated external API access behind shared interface; each adapter normalizes to `NormalizedBalance`
3. **Snapshot Service + Scheduler** -- daily cron captures full portfolio state; stores native values + exchange rates for auditability
4. **Rebalancing Engine** -- pure calculation service comparing current allocations against user-defined targets; generates suggestions with tolerance bands
5. **React SPA with TanStack Query** -- dashboard, charts, and tables; server state cached via TanStack Query, client preferences in Zustand

### Critical Pitfalls

1. **Floating-point arithmetic for money** -- Use decimal.js for ALL calculations and PostgreSQL NUMERIC(19,8) for storage. Never use JavaScript `number` or PostgreSQL `MONEY` type. Must be correct from Phase 1; retrofitting is extremely painful.
2. **CCXT exchange inconsistencies** -- `fetchBalance()` returns only spot by default, symbols differ across exchanges, rate limits vary. Build an adapter abstraction with per-exchange config, explicit type params, Zod response validation, and retry with exponential backoff.
3. **Trading 212 Pies API deprecation** -- The API is officially deprecated with no timeline for replacement. Isolate behind a clean adapter interface so only one file changes when the new API arrives. Build a manual-entry fallback path.
4. **Snapshot scheduling fails silently in Docker** -- In-process cron has no catch-up mechanism for missed jobs. Add startup health check that triggers catch-up if last snapshot is stale; make snapshot creation idempotent; set timezone explicitly.
5. **Recharts blank rendering with React 19** -- Charts render as empty containers with zero console errors. Fix: install `react-is@^19.0.0` explicitly. Fallback: swap to Nivo if unresolvable.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Project Foundation and Data Model
**Rationale:** Everything depends on the database schema and Docker setup being correct. Floating-point precision and Docker networking pitfalls must be addressed before any feature code. The Prisma schema defines the contract for every subsequent phase.
**Delivers:** Scaffolded NestJS backend + React frontend, Prisma schema with all core models (Asset, Snapshot, SnapshotEntry, ExchangeConnection, AllocationTarget, ExchangeRate, Settings), Docker Compose with healthchecks, environment configuration.
**Addresses:** Docker Compose deployment (P1), foundational data model
**Avoids:** Floating-point arithmetic pitfall (decimal types from day one), Docker networking pitfall (healthchecks, service names)

### Phase 2: Manual Assets and Core Portfolio
**Rationale:** Manual asset entry is the simplest data source and validates the data model, CRUD operations, and currency conversion pipeline end-to-end before adding external API complexity. The unified net worth display is the core value proposition.
**Delivers:** Manual asset CRUD, base currency selection, currency conversion via Frankfurter API, unified net worth calculation, basic portfolio summary API + frontend dashboard.
**Addresses:** Manual asset entry (P1), base currency + conversion (P1), unified net worth display (P1), portfolio allocation view (P1)
**Avoids:** Currency conversion compounding errors (bridge pattern, rate storage), client-side conversion anti-pattern

### Phase 3: Exchange Integrations
**Rationale:** CCXT and T212 integrations depend on the adapter interface and currency service from Phase 2. Grouping both integrations together ensures the adapter pattern is validated with two real implementations.
**Delivers:** CCXT adapter for crypto exchanges, Trading 212 adapter for pies, normalized balance fetching, on-demand refresh capability.
**Addresses:** CCXT integration (P1), Trading 212 pies integration (P2)
**Avoids:** CCXT inconsistencies pitfall (adapter pattern, per-exchange config), T212 deprecation pitfall (isolated adapter, fallback path)

### Phase 4: Snapshots and Historical Data
**Rationale:** The daily snapshot system depends on a working portfolio aggregation pipeline (Phase 2 + Phase 3). Historical views depend on accumulated snapshots. The scheduler is the last piece before the product captures data autonomously.
**Delivers:** Daily snapshot cron job, snapshot capture with full state + exchange rates, historical portfolio value chart (LineChart), startup health check and catch-up logic.
**Addresses:** Daily snapshot system (P1), historical portfolio value charts (P2), historical performance by category (P2)
**Avoids:** Snapshot scheduling pitfall (health check, idempotency, timezone config)

### Phase 5: Dashboard and Visualization
**Rationale:** With data flowing (manual + automated + historical), the dashboard can now display meaningful visualizations. Charting depends on having real data to render.
**Delivers:** Full dashboard with allocation pie chart, historical line charts, P&L display (portfolio and category level), deviation bar charts, responsive layout.
**Addresses:** Portfolio allocation view (P1), basic P&L display (P1), P&L drill-down (P2), allocation with targets overlay (P2)
**Avoids:** Recharts React 19 blank rendering (react-is fix, Nivo fallback ready)

### Phase 6: Rebalancing Engine
**Rationale:** Rebalancing is the primary differentiator but depends on accurate allocation data (Phase 5) and correct financial math (all prior phases). It should come last among core features so the data pipeline is proven before building suggestions on top of it.
**Delivers:** Target allocation CRUD, macro-level rebalancing with deviation and suggestions, tolerance bands, rebalancing UI with suggestion table and deviation chart.
**Addresses:** Macro-level rebalancing (P2), allocation visualization with targets (P2)
**Avoids:** Rebalancing edge cases pitfall (normalization to 100%, minimum trade thresholds, zero-value handling)

### Phase 7: Polish and V2 Features
**Rationale:** Intra-category rebalancing, trading bot grouping, and data export are refinements that depend on all prior phases being stable.
**Delivers:** Intra-category rebalancing, CCXT trading bot category grouping, data export, UX refinements.
**Addresses:** Intra-category rebalancing (P3), trading bot grouping (P3), data export (P3)

### Phase Ordering Rationale

- **Foundation first (Phase 1):** Docker setup and Prisma schema are dependencies for everything. Two critical pitfalls (floating-point, Docker networking) must be prevented here.
- **Manual before automated (Phase 2 before 3):** Manual assets validate the data model and currency pipeline without external API complexity. If the conversion math is wrong, it is easier to debug with controlled manual data.
- **Integrations together (Phase 3):** Building both CCXT and T212 adapters in the same phase validates the adapter interface with two real implementations. If the interface is wrong, you discover it before building on top of it.
- **Snapshots after integrations (Phase 4):** The scheduler orchestrates the full aggregation pipeline. It cannot work without all data sources in place.
- **Dashboard after data (Phase 5):** Charts need real data. Building visualizations before data sources exist leads to rework when actual data shapes differ from assumptions.
- **Rebalancing last (Phase 6):** The differentiator feature depends on accurate aggregation, conversion, and allocation data. Building it on a proven pipeline reduces debugging surface.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3:** CCXT exchange-specific behaviors, T212 API current state (beta, deprecated endpoints). Test against each target exchange before committing to the adapter design.
- **Phase 6:** Rebalancing edge cases are numerous. Research phase should include building a test fixture library with adversarial portfolio data (zero values, extreme allocations, rounding traps).

Phases with standard patterns (skip research-phase):
- **Phase 1:** Docker Compose + NestJS scaffolding + Prisma schema are extremely well-documented. No novel decisions.
- **Phase 2:** CRUD + REST API + currency conversion are standard NestJS patterns. Frankfurter API is trivial to integrate.
- **Phase 4:** NestJS scheduling is well-documented. The health-check-on-startup pattern is straightforward.
- **Phase 5:** Recharts/Nivo chart implementation is well-documented. TanStack Query integration is standard.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified against npm/official docs. Compatibility matrix confirmed (NestJS 11.1.16, Prisma 7.4.x, React 19.2.4, CCXT 4.5.44, TanStack Query 5.91.3, Zustand 5.0.12, Recharts 3.8.0). No speculative choices. |
| Features | HIGH | Competitive analysis against 6+ tools (Ghostfolio, Rotki, Kubera, Capitally, Sharesight, Snowball). Feature gaps and differentiators clearly identified. Dependency graph validated. |
| Architecture | HIGH | Layered monolith is standard for this domain. Ghostfolio uses nearly identical architecture (NestJS + Prisma + PostgreSQL). Adapter pattern, bridge currency, and snapshot aggregation are proven patterns. |
| Pitfalls | HIGH | Core pitfalls (floating-point, Docker networking, scheduling) verified with multiple sources. Integration pitfalls (CCXT per-exchange issues, T212 deprecation) verified against library issues and community reports. |

**Overall confidence:** HIGH

### Gaps to Address

- **Trading 212 API replacement timeline:** The Pies API is deprecated with "major upgrade coming." No concrete date. Monitor the T212 community forum during Phase 3. If the new API launches during development, the adapter pattern makes migration contained.
- **CCXT per-exchange testing:** Research covers CCXT generically. Each specific exchange the user connects will need individual testing during Phase 3 for: balance type params, symbol normalization, rate limit behavior.
- **Recharts React 19 stability:** The react-is fix is community-sourced (blog post + GitHub issue). It works as of the research date but Recharts has not issued an official fix. If it breaks, Nivo is the ready fallback. Verify during Phase 5.
- **Intra-category rebalancing interaction model:** How two-tier rebalancing suggestions interact (macro vs intra) is not fully specified. Research this during Phase 7 planning -- conflicting suggestions between tiers need a clear resolution strategy.
- **Prisma Decimal round-trip handling:** Verify no precision loss when serializing Prisma Decimal through decimal.js and back during Phase 1 schema validation.

## Sources

### Primary (HIGH confidence)
- [NestJS 11 Docs and Announcement](https://trilon.io/blog/announcing-nestjs-11-whats-new) -- version, features, scheduling, validation
- [Prisma 7 Announcement](https://www.prisma.io/blog/announcing-prisma-orm-7-0-0) -- Rust-free client, performance, schema reference
- [CCXT GitHub + npm](https://github.com/ccxt/ccxt) -- 100+ exchanges, version 4.5.44, unified API surface
- [Trading 212 API Docs](https://docs.trading212.com/api) -- endpoints, rate limits, beta status
- [Frankfurter API](https://frankfurter.dev/) -- free ECB exchange rates, no key required
- [React 19 Release](https://react.dev/versions) -- version 19.2.4 confirmed
- [TanStack Query npm](https://www.npmjs.com/package/@tanstack/react-query) -- version 5.91.3 confirmed
- [shadcn/ui](https://ui.shadcn.com/) -- React 19 + Tailwind v4 support confirmed
- [Docker Compose Health Checks](https://last9.io/blog/docker-compose-health-checks/) -- depends_on with service_healthy
- [Modern Treasury - Floats and Money](https://www.moderntreasury.com/journal/floats-dont-work-for-storing-cents) -- financial precision requirements
- [Crunchy Data - PostgreSQL Money Types](https://www.crunchydata.com/blog/working-with-money-in-postgres) -- NUMERIC vs MONEY

### Secondary (MEDIUM confidence)
- [Recharts React 19 Fix](https://www.bstefanski.com/blog/recharts-empty-chart-react-19) -- react-is workaround, community-sourced
- [Nivo React 19 Support](https://github.com/plouc/nivo/issues/2618) -- confirmed compatible, GitHub issue
- [Best ORM for NestJS 2025](https://dev.to/sasithwarnakafonseka/best-orm-for-nestjs-in-2025-drizzle-orm-vs-typeorm-vs-prisma-229c) -- ORM comparison
- [Trading 212 Community: API Update](https://community.trading212.com/t/trading-212-api-update/87988) -- deprecation status, upcoming changes
- [trading212-api GitHub](https://github.com/bennycode/trading212-api) -- package staleness assessment
- [nestjs-ccxt GitHub](https://github.com/fasenderos/nestjs-ccxt) -- unmaintained status confirmed
- [Ghostfolio GitHub](https://github.com/ghostfolio/ghostfolio) -- competitor architecture reference
- [Capitally Blog - Portfolio Tracker Comparison](https://www.mycapitally.com/blog/best-portfolio-tracker-for-the-modern-diy-investor) -- feature landscape

---
*Research completed: 2026-03-21*
*Ready for roadmap: yes*
