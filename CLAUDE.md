<!-- GSD:project-start source:PROJECT.md -->
## Project

**Finance Manager**

A personal portfolio management web application that aggregates all assets and investments into a single unified view. It pulls data automatically from crypto exchanges (via CCXT) and Trading 212 (via pies API), with manual entry for bank accounts and unsupported exchanges. Built for one user who wants total visibility, clear analytics, and intelligent rebalancing across their entire portfolio.

**Core Value:** Complete, accurate net worth visibility across all asset sources — if the numbers aren't right and unified, nothing else matters.

### Constraints

- **Tech stack**: NestJS + React + PostgreSQL — decided upfront, non-negotiable
- **Containerization**: Docker with three separate Dockerfiles + docker-compose.yml — required for deployment
- **Trading 212**: Pies-only integration — API limitations and user preference
- **Data freshness**: Daily snapshots — not real-time, reduces API load and complexity
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| NestJS | ^11.0.0 | Backend framework | Decided upfront. v11 is current stable (11.1.16). Decorator-based architecture maps cleanly to service modules (CCXT service, T212 service, snapshot service). Native TypeScript, built-in DI, module system, and first-class support for scheduling, validation, and HTTP clients. |
| React | ^19.0.0 | Frontend UI | Decided upfront. v19.2.4 is current stable. React 19 brings performance improvements and Server Components support (not needed here, but ecosystem compatibility matters). |
| PostgreSQL | 16 | Primary database | Decided upfront. v16 is the current production standard in Docker images. Excellent for time-series snapshot data, JSON support for flexible asset metadata, and strong numerical precision for financial calculations. |
| Docker + Compose | latest | Containerization | Decided upfront. Three-service architecture (backend, frontend, db). Multi-stage builds for NestJS reduce image size by excluding dev dependencies and TypeScript source. |
| TypeScript | ^5.7.0 | Language | NestJS 11 scaffolds with TS 5.7 natively. Strict mode essential for financial calculations -- catches type errors at compile time. |
### ORM / Database Access
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Prisma ORM | ^7.0.0 | Database ORM | **Use Prisma, not TypeORM or Drizzle.** Prisma 7 (current: 7.4.x) is the best fit for this project: schema-first approach generates fully type-safe queries from a single `schema.prisma` file. The generated client catches errors at compile time -- critical for financial data. No separate NestJS adapter needed; just inject PrismaService. Migration tooling (`prisma migrate`) is superior to TypeORM's fragile migration system. Drizzle is faster in raw benchmarks but Prisma's DX advantage matters more for a single-developer project. |
| @prisma/client | ^7.0.0 | Generated query client | Auto-generated from schema. Provides typed queries, relations, and aggregations. |
| prisma (CLI) | ^7.0.0 | Migrations and schema tooling | `prisma migrate dev` for development, `prisma migrate deploy` for Docker containers. |
### Crypto Exchange Integration
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| ccxt | ^4.5.0 | Crypto exchange API abstraction | Current: 4.5.44. Supports 100+ exchanges with unified API. Fetch balances, positions, and market data with one interface regardless of exchange. JavaScript/TypeScript native. Active maintenance (releases every few days). |
### Trading 212 Integration
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| @nestjs/axios + axios | ^4.0.0 / ^1.7.0 | HTTP client for T212 API | NestJS's built-in HttpModule wraps Axios with Observable support, interceptors, and retry logic. The Trading 212 API is simple REST (v0) -- a dedicated SDK is overkill. |
- Base URLs: `https://live.trading212.com` (live) / `https://demo.trading212.com` (demo)
- Auth: API key in header
- Endpoints: Pies (GET/POST/PUT/DELETE), Account info, Instruments list
- The API is in beta -- expect breaking changes. Isolate all T212 logic in one module.
### Scheduling (Daily Snapshots)
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| @nestjs/schedule | ^6.1.0 | Cron job scheduling | Current: 6.1.1. Official NestJS package wrapping `cron` under the hood. Decorator-based: `@Cron('0 2 * * *')` on a method triggers daily at 2 AM. Supports timezone configuration, dynamic scheduling via SchedulerRegistry, and intervals/timeouts. |
### Currency Conversion
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Frankfurter API | N/A (external) | Fiat exchange rates (EUR, USD, GBP) | Free, no API key required, no rate limits. Data sourced from the European Central Bank, updated daily ~16:00 CET. Supports 30+ fiat currencies and historical rates back 20 years. Perfect for daily snapshot cadence. |
| ccxt (built-in) | (same as above) | Crypto-to-fiat rates | CCXT exchanges provide ticker prices. Use a major exchange (Binance/Kraken) as price oracle for crypto-to-fiat conversion. No additional library needed. |
### Charting (Frontend)
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Recharts | ^3.8.0 | Charts (pie, bar, line, area) | Current: 3.8.0. Built on D3, idiomatic React components. Covers all chart types needed: PieChart for allocation, BarChart for category comparison, LineChart/AreaChart for historical portfolio value. Large ecosystem, extensive examples. |
| react-is | ^19.0.0 | Recharts React 19 fix | **CRITICAL:** Recharts has a known blank-rendering bug with React 19.2.x. Fix: install `react-is` at the same major version as React. Without this, charts render as empty containers with no console errors. |
- **vs Tremor:** Tremor is built ON Recharts but limits customization. It provides high-level dashboard components but you cannot drop to lower-level D3 when needed. For a portfolio app where you will want custom tooltips showing asset values and allocation percentages, Recharts gives more control.
- **vs Nivo:** Nivo is excellent (React 19 compatible) but more complex API. Recharts is simpler for the chart types this app needs. If Recharts React 19 issues persist, Nivo (`@nivo/pie`, `@nivo/line`, `@nivo/bar`) is the fallback.
- **vs ApexCharts:** Better for candlestick/financial trading charts. This app does not need candlesticks -- it needs allocation pies and value-over-time lines.
### Frontend API Communication
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| @tanstack/react-query | ^5.91.0 | Server state management | Current: 5.91.3. Handles caching, background refetching, stale data, and loading/error states. Eliminates manual `useEffect` + `useState` fetch patterns. Devtools for debugging cache. The standard for React data fetching in 2025-2026. |
| axios | ^1.7.0 | HTTP client (frontend) | Consistent with backend (NestJS HttpModule uses Axios). Interceptors for error handling, request/response transformation. Lighter alternatives exist (ky, ofetch) but consistency across frontend/backend has value. |
### Frontend State Management
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Zustand | ^5.0.0 | Client-side state | Current: 5.0.12. Minimal (~1KB), no providers, no boilerplate. Perfect for UI state (selected base currency, active filters, sidebar toggle). Server state lives in TanStack Query -- Zustand handles only what TanStack Query does not. |
### Frontend UI Framework
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Tailwind CSS | ^4.0.0 | Utility-first CSS | Current standard. v4 is stable and works with React 19. No CSS-in-JS runtime overhead. Faster iteration than component library CSS. |
| shadcn/ui | latest | UI component library | Copy-paste components built on Radix UI + Tailwind. You own the code -- no dependency to update. Accessible, keyboard-navigable, and production-ready. Covers tables, cards, dialogs, dropdowns, and forms needed for a dashboard. |
### Backend Validation & Configuration
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| class-validator | ^0.15.0 | DTO validation | Current: 0.15.1. Decorator-based validation integrated into NestJS ValidationPipe. Validates all incoming request DTOs. |
| class-transformer | ^0.5.0 | DTO transformation | Transforms plain JSON into class instances for validation pipeline. Required by NestJS ValidationPipe. |
| @nestjs/config | ^4.0.0 | Environment configuration | Current: 4.0.3. Manages env vars (API keys, DB connection, base currency). Supports `.env` files in dev, environment variables in Docker. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| decimal.js | ^10.4.0 | Precise decimal arithmetic | ALL financial calculations. JavaScript floats lose precision (0.1 + 0.2 !== 0.3). Use Decimal for portfolio values, percentages, allocation math. Store as string/numeric in PostgreSQL, convert to Decimal in application layer. |
| date-fns | ^4.0.0 | Date manipulation | Formatting snapshot dates, calculating time ranges (daily/weekly/monthly views), timezone handling. Tree-shakeable unlike Moment.js. |
| zod | ^3.23.0 | Schema validation (shared) | Optional but recommended for validating external API responses (CCXT balance shape, T212 pie shape, Frankfurter rate shape). class-validator handles inbound DTOs; Zod validates outbound API data you do not control. |
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| @tanstack/react-query-devtools | ^5.91.0 | Debug TanStack Query cache | Only included in dev builds. Shows query status, cache state, and refetch triggers. |
| prisma studio | (bundled) | Visual database browser | `npx prisma studio` opens a GUI for browsing/editing DB records. Invaluable during development for verifying snapshot data. |
| @nestjs/testing | ^11.0.0 | Backend unit/integration tests | NestJS's built-in testing utilities. Mock services, test controllers in isolation. |
| Vite | ^6.0.0 | Frontend build tool | Fast HMR, ESM-native. The standard React build tool post-CRA deprecation. |
| ESLint + Prettier | latest | Code quality | NestJS CLI scaffolds with ESLint. Add Prettier for consistent formatting. |
## Installation
# === Backend (NestJS) ===
# Core NestJS (scaffolded by CLI)
# Database
# Crypto exchanges
# HTTP client (for Trading 212, Frankfurter)
# Scheduling
# Validation
# Configuration
# Financial math
# Date utilities
# External API response validation (optional)
# Dev dependencies
# === Frontend (React) ===
# Core (scaffolded by Vite)
# React 19 compatibility fix for Recharts
# Data fetching
# State management
# Charts
# UI
# shadcn/ui: use `npx shadcn@latest init` then add components as needed
# Dev dependencies
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Prisma | Drizzle ORM (0.45.x) | If query performance becomes a measurable bottleneck. Drizzle is ~14x faster for complex joins. Unlikely to matter for daily snapshots. |
| Prisma | TypeORM | If you already have an existing TypeORM codebase. Do not choose it for greenfield. |
| Recharts | @nivo/pie + @nivo/line + @nivo/bar | If Recharts React 19 rendering bugs are unresolvable. Nivo has confirmed React 19 support. |
| Recharts | Tremor | If you want zero-config dashboard aesthetics and do not need custom chart interactions. |
| @nestjs/schedule | BullMQ + @nestjs/bullmq | If you need job persistence, retry with backoff, or job queues. Requires Redis. |
| Frankfurter API | Open Exchange Rates | If you need more than 30 fiat currencies or intraday rates. Requires API key (free tier: 1000 req/month). |
| Zustand | React Context | If you have literally 1-2 pieces of global state and no performance concerns. |
| Axios (frontend) | ky or ofetch | If bundle size is critical. ky is ~3KB vs Axios ~13KB. Minor for a dashboard app. |
| Custom T212 service | trading212-api npm | If the package gets active maintenance and the T212 API stabilizes out of beta. Check before deciding. |
| Custom CCXT service | nestjs-ccxt | Never. The package is dead. Wrap CCXT directly. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| nestjs-ccxt | Unmaintained (last release 2+ years ago). Will break with CCXT 4.x and NestJS 11. | Wrap `ccxt` directly in a NestJS service |
| trading212-api | Unmaintained (~1 year stale), T212 API is in beta with breaking changes | Direct HTTP calls via @nestjs/axios |
| Moment.js | Deprecated, massive bundle (300KB+), mutable API | date-fns (tree-shakeable, immutable) |
| TypeORM | Fragile migrations, weaker type safety, decorator bloat | Prisma ORM |
| Redux | Unnecessary complexity for single-user dashboard | Zustand (client state) + TanStack Query (server state) |
| JavaScript floats for money | 0.1 + 0.2 = 0.30000000000000004. Financial calculations will be wrong. | decimal.js for all arithmetic, PostgreSQL NUMERIC/DECIMAL for storage |
| Sequelize | Legacy ORM, poor TypeScript support compared to Prisma | Prisma ORM |
| Create React App (CRA) | Deprecated, slow, no longer maintained | Vite |
| node-cron | Works but no NestJS integration, no DI, no decorator support | @nestjs/schedule |
## Stack Patterns by Variant
- Add WebSocket gateway via `@nestjs/websockets`
- Replace daily cron with streaming price feeds
- Add Redis for pub/sub (BullMQ becomes justified)
- This is explicitly out of scope today
- Add `@nestjs/passport` + `@nestjs/jwt` for auth
- Add user_id foreign key to all tables
- Add row-level security in PostgreSQL
- This is explicitly out of scope today
- Replace Recharts with Nivo: `npm install @nivo/core @nivo/pie @nivo/line @nivo/bar`
- Nivo has confirmed React 19 support
- API is more verbose but equally capable for the chart types needed
## Version Compatibility
| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| NestJS 11.x | Prisma 7.x | Use PrismaService injected as a standard provider. No adapter package needed. |
| NestJS 11.x | @nestjs/schedule 6.x | Official package, version-locked to NestJS major. |
| NestJS 11.x | @nestjs/axios 4.x | Official package, version-locked to NestJS major. |
| NestJS 11.x | @nestjs/config 4.x | Official package, version-locked to NestJS major. |
| React 19.2.x | Recharts 3.8.x | Requires `react-is@^19.0.0` installed as peer dependency fix. Test before committing. |
| React 19.2.x | @tanstack/react-query 5.x | Fully compatible. No known issues. |
| React 19.2.x | Zustand 5.x | Fully compatible. No known issues. |
| React 19.2.x | shadcn/ui (latest) | Updated for React 19 and Tailwind v4. |
| CCXT 4.x | Node.js 18+ | CCXT 4.x uses modern JS features. Ensure Docker base image is Node 20 LTS. |
| PostgreSQL 16 | Prisma 7.x | Full support including identity columns (recommended over serial). |
## Sources
- [NestJS 11 Announcement - Trilon](https://trilon.io/blog/announcing-nestjs-11-whats-new) -- NestJS 11 features and version (HIGH confidence)
- [NestJS npm releases](https://www.npmjs.com/package/@nestjs/core) -- Version 11.1.16 confirmed (HIGH confidence)
- [Prisma 7 Announcement](https://www.prisma.io/blog/announcing-prisma-orm-7-0-0) -- Prisma 7 Rust-free client, performance (HIGH confidence)
- [Prisma 7.2.0 Release](https://www.prisma.io/blog/announcing-prisma-orm-7-2-0) -- Latest patch version (HIGH confidence)
- [CCXT npm](https://www.npmjs.com/package/ccxt) -- Version 4.5.44 confirmed (HIGH confidence)
- [CCXT GitHub](https://github.com/ccxt/ccxt) -- 100+ exchanges, JS/TS support (HIGH confidence)
- [nestjs-ccxt GitHub](https://github.com/fasenderos/nestjs-ccxt) -- Last publish 2+ years ago, unmaintained (HIGH confidence)
- [Trading 212 API Docs](https://docs.trading212.com/api) -- API endpoints, beta status (HIGH confidence)
- [trading212-api GitHub](https://github.com/bennycode/trading212-api) -- Package assessment (MEDIUM confidence)
- [NestJS Task Scheduling Docs](https://docs.nestjs.com/techniques/task-scheduling) -- @nestjs/schedule usage (HIGH confidence)
- [@nestjs/schedule npm](https://www.npmjs.com/package/@nestjs/schedule) -- Version 6.1.1 confirmed (HIGH confidence)
- [Frankfurter API](https://frankfurter.dev/) -- Free, no key, ECB data (HIGH confidence)
- [Recharts npm](https://www.npmjs.com/package/recharts) -- Version 3.8.0, React 19 issue (HIGH confidence)
- [Recharts React 19 Fix](https://www.bstefanski.com/blog/recharts-empty-chart-react-19) -- react-is workaround (MEDIUM confidence)
- [Nivo React 19 Support](https://github.com/plouc/nivo/issues/2618) -- Confirmed compatible (MEDIUM confidence)
- [TanStack Query npm](https://www.npmjs.com/package/@tanstack/react-query) -- Version 5.91.3 confirmed (HIGH confidence)
- [Zustand npm](https://www.npmjs.com/package/zustand) -- Version 5.0.12 confirmed (HIGH confidence)
- [React Versions](https://react.dev/versions) -- React 19.2.4 confirmed (HIGH confidence)
- [Best ORM for NestJS 2025 - DEV](https://dev.to/sasithwarnakafonseka/best-orm-for-nestjs-in-2025-drizzle-orm-vs-typeorm-vs-prisma-229c) -- ORM comparison (MEDIUM confidence)
- [NestJS Drizzle Integration - Trilon](https://trilon.io/blog/nestjs-drizzleorm-a-great-match) -- Drizzle alternative assessment (MEDIUM confidence)
- [shadcn/ui](https://ui.shadcn.com/) -- Tailwind v4 + React 19 support (HIGH confidence)
- [class-validator npm](https://www.npmjs.com/package/class-validator) -- Version 0.15.1 confirmed (HIGH confidence)
- [NestJS Validation Docs](https://docs.nestjs.com/techniques/validation) -- ValidationPipe best practices (HIGH confidence)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
