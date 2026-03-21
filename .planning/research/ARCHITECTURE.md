# Architecture Research

**Domain:** Personal portfolio management / finance aggregation
**Researched:** 2026-03-21
**Confidence:** HIGH

## Standard Architecture

### System Overview

```
+---------------------------------------------------------------------------+
|                         PRESENTATION LAYER                                |
|  +--------------------------------------------------------------------+  |
|  |              React SPA (Vite + TypeScript)                         |  |
|  |  +----------+  +----------+  +-----------+  +-----------+         |  |
|  |  |Dashboard |  |Portfolio |  |Rebalancing|  | Settings  |         |  |
|  |  |  Views   |  |  Views   |  |   Views   |  |   Views   |         |  |
|  |  +----------+  +----------+  +-----------+  +-----------+         |  |
|  +--------------------------------------------------------------------+  |
|                              | HTTP/REST                                  |
+------------------------------+--------------------------------------------+
|                         API LAYER (NestJS)                                |
|  +----------+  +----------+  +-----------+  +------------------+         |
|  |Portfolio |  | Assets   |  |Rebalancing|  |   Snapshots      |         |
|  |Controller|  |Controller|  |Controller |  |   Controller     |         |
|  +----+-----+  +----+-----+  +-----+-----+  +-------+----------+        |
|       |              |              |                |                     |
+-------+--------------+--------------+----------------+--------------------+
|                        SERVICE LAYER                                      |
|  +----------+  +----------+  +-----------+  +------------------+         |
|  |Portfolio |  |  Asset   |  |Rebalancing|  |    Snapshot      |         |
|  | Service  |  | Service  |  |  Engine   |  |    Service       |         |
|  +----+-----+  +----+-----+  +-----------+  +-------+----------+        |
|       |              |                               |                    |
+-------+--------------+-------------------------------+--------------------+
|                     INTEGRATION LAYER                                     |
|  +----------+  +----------+  +-----------+  +------------------+         |
|  |  CCXT    |  |Trading212|  | Currency  |  |    Scheduler     |         |
|  | Adapter  |  | Adapter  |  |  Service  |  |   (Cron Jobs)    |         |
|  +----------+  +----------+  +-----------+  +------------------+         |
|                                                                           |
+---------------------------------------------------------------------------+
|                        DATA LAYER                                         |
|  +--------------------------------------------------------------------+  |
|  |                  Prisma ORM + PostgreSQL                            |  |
|  |  +---------+  +----------+  +----------+  +-------------+         |  |
|  |  | Assets  |  |Snapshots |  |Allocation|  |  Exchange   |         |  |
|  |  |         |  |          |  | Targets  |  |   Rates     |         |  |
|  |  +---------+  +----------+  +----------+  +-------------+         |  |
|  +--------------------------------------------------------------------+  |
+---------------------------------------------------------------------------+
```

This is a standard **layered monolith** inside a single NestJS application. No microservices, no message queues, no event buses. For a single-user personal tool, a layered monolith is the correct architecture -- anything more is over-engineering.

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **React SPA** | All UI: dashboards, charts, tables, forms | Vite + React 19 + TypeScript, Recharts for visualization, TanStack Query for server state, Zustand for client state |
| **NestJS Controllers** | HTTP request handling, input validation, response shaping | NestJS controllers with class-validator DTOs |
| **Portfolio Service** | Aggregate all assets into unified portfolio view, compute net worth | Business logic service combining automated + manual assets |
| **Asset Service** | CRUD for manual assets, orchestrate automated asset fetching | Coordinates between adapters and database |
| **Rebalancing Engine** | Compute current vs target allocations, generate suggestions | Pure calculation service with no side effects |
| **Snapshot Service** | Create and retrieve historical portfolio snapshots | Stores daily snapshots, provides time-series queries |
| **CCXT Adapter** | Fetch balances from crypto exchanges via CCXT | Wraps CCXT library directly (not nestjs-ccxt), normalizes responses to internal format |
| **Trading 212 Adapter** | Fetch portfolio data from Trading 212 API | Direct HTTP calls via @nestjs/axios, normalizes to internal format |
| **Currency Service** | Convert between currencies using exchange rates | Caches rates from Frankfurter API (fiat) and CCXT tickers (crypto), provides conversion methods |
| **Scheduler** | Trigger daily snapshot capture and rate updates | @nestjs/schedule with @Cron decorators |
| **Prisma ORM** | Database schema definition, type-safe queries, migrations | Prisma schema file, generated client, prisma migrate |

## Recommended Project Structure

### Backend (NestJS)

```
backend/
+-- src/
|   +-- app.module.ts              # Root module, imports all feature modules
|   +-- main.ts                    # Bootstrap, CORS, global pipes
|   |
|   +-- prisma/                    # Prisma ORM layer
|   |   +-- prisma.module.ts       # Global module
|   |   +-- prisma.service.ts      # PrismaClient wrapper with onModuleInit/onModuleDestroy
|   |   +-- schema.prisma          # Single source of truth for DB schema
|   |   +-- migrations/            # Generated by prisma migrate
|   |
|   +-- portfolio/                 # Portfolio aggregation feature
|   |   +-- portfolio.module.ts
|   |   +-- portfolio.controller.ts
|   |   +-- portfolio.service.ts
|   |   +-- dto/
|   |       +-- portfolio-summary.dto.ts
|   |
|   +-- assets/                    # Asset management feature
|   |   +-- assets.module.ts
|   |   +-- assets.controller.ts
|   |   +-- assets.service.ts
|   |   +-- dto/
|   |       +-- create-asset.dto.ts
|   |       +-- update-asset.dto.ts
|   |
|   +-- snapshots/                 # Historical snapshot feature
|   |   +-- snapshots.module.ts
|   |   +-- snapshots.controller.ts
|   |   +-- snapshots.service.ts
|   |
|   +-- rebalancing/               # Rebalancing engine feature
|   |   +-- rebalancing.module.ts
|   |   +-- rebalancing.controller.ts
|   |   +-- rebalancing.service.ts
|   |   +-- dto/
|   |       +-- allocation-target.dto.ts
|   |
|   +-- integrations/              # External API adapters
|   |   +-- integrations.module.ts
|   |   +-- exchange-adapter.interface.ts   # Shared adapter contract
|   |   +-- ccxt/
|   |   |   +-- ccxt.service.ts
|   |   |   +-- ccxt.config.ts
|   |   +-- trading212/
|   |   |   +-- trading212.service.ts
|   |   |   +-- trading212.config.ts
|   |   +-- currency/
|   |       +-- currency.service.ts
|   |
|   +-- scheduler/                 # Cron jobs and scheduled tasks
|   |   +-- scheduler.module.ts
|   |   +-- scheduler.service.ts
|   |
|   +-- common/                    # Shared utilities
|       +-- decorators/
|       +-- filters/
|       +-- interceptors/
|       +-- types/
|           +-- money.ts           # Money value object (Decimal + currency)
|
+-- test/
+-- Dockerfile
+-- .env.example
+-- tsconfig.json
```

### Frontend (React)

```
frontend/
+-- src/
|   +-- main.tsx
|   +-- App.tsx
|   +-- api/                       # API client layer
|   |   +-- client.ts              # Axios instance with base config
|   |   +-- portfolio.api.ts
|   |   +-- assets.api.ts
|   |   +-- snapshots.api.ts
|   |   +-- rebalancing.api.ts
|   |
|   +-- pages/                     # Route-level page components
|   |   +-- Dashboard.tsx
|   |   +-- Portfolio.tsx
|   |   +-- Rebalancing.tsx
|   |   +-- History.tsx
|   |   +-- Settings.tsx
|   |
|   +-- components/                # Reusable UI components
|   |   +-- charts/
|   |   |   +-- AllocationPieChart.tsx
|   |   |   +-- HistoryLineChart.tsx
|   |   |   +-- DeviationBarChart.tsx
|   |   +-- tables/
|   |   |   +-- AssetTable.tsx
|   |   |   +-- RebalancingTable.tsx
|   |   +-- ui/                    # shadcn/ui components (copy-pasted)
|   |       +-- button.tsx
|   |       +-- card.tsx
|   |       +-- table.tsx
|   |       +-- ...
|   |
|   +-- hooks/                     # Custom React hooks (TanStack Query wrappers)
|   |   +-- usePortfolio.ts
|   |   +-- useAssets.ts
|   |   +-- useSnapshots.ts
|   |   +-- useRebalancing.ts
|   |
|   +-- stores/                    # Zustand stores (client-only state)
|   |   +-- useSettingsStore.ts    # Base currency, UI preferences
|   |
|   +-- types/                     # TypeScript interfaces
|   |   +-- index.ts
|   |
|   +-- lib/                       # Utility functions
|       +-- formatting.ts          # Currency/percentage formatting
|       +-- decimal.ts             # decimal.js helpers for display math
|
+-- Dockerfile
+-- vite.config.ts
+-- tailwind.config.ts
+-- tsconfig.json
```

### Docker Compose Structure

```
finance-manager/
+-- docker-compose.yml
+-- backend/
|   +-- Dockerfile
+-- frontend/
|   +-- Dockerfile
+-- .env                           # Shared env vars (in .gitignore)
+-- .env.example                   # Template for .env
+-- .planning/                     # Project planning (not containerized)
```

### Structure Rationale

- **Feature-based modules (backend):** Each NestJS module owns its controller, service, and DTOs. A developer looking at the `rebalancing/` folder sees everything related to rebalancing.
- **Prisma in its own module:** The PrismaService is a global module injected into any service that needs database access. The `schema.prisma` file is the single source of truth for the data model -- no entity files scattered across modules.
- **Integrations as a separate module:** External API adapters are isolated from business logic. If Trading 212 changes their API or CCXT updates, changes are contained to one folder. The adapter interface ensures all data sources return a common format.
- **Scheduler as its own module:** The scheduler depends on snapshot and integration services but has no business logic of its own. It simply triggers existing services on a cron schedule.
- **API client layer (frontend):** Thin wrappers around Axios keep API calls out of components. If an endpoint changes, update one file.
- **Pages vs Components (frontend):** Pages are route-level containers that compose components. Components are reusable visual building blocks. shadcn/ui components live in `components/ui/` as owned source code.
- **Stores separated from hooks:** Zustand stores handle client-only state (settings, UI preferences). TanStack Query hooks handle server state. Clear separation prevents confusion about where state lives.

## Architectural Patterns

### Pattern 1: Adapter Pattern for External APIs

**What:** Define a common interface (`ExchangeAdapter`) that all data source integrations implement. CCXT, Trading 212, and future integrations all conform to this interface.

**When to use:** Always for external API integrations. This project has two automated data sources today and may add more.

**Trade-offs:** Slight upfront abstraction cost, but massive payoff when adding new exchanges or when an API changes.

**Example:**
```typescript
// integrations/exchange-adapter.interface.ts
interface NormalizedBalance {
  assetName: string;
  symbol: string;
  amount: Decimal;          // decimal.js -- never number
  valueCurrency: string;    // ISO 4217 or crypto symbol
  valueInCurrency: Decimal;
}

interface ExchangeAdapter {
  getBalances(): Promise<NormalizedBalance[]>;
  getName(): string;
  getCategory(): 'Crypto' | 'Stocks' | 'TradingBot';
}

// integrations/ccxt/ccxt.service.ts
@Injectable()
class CcxtAdapter implements ExchangeAdapter {
  async getBalances(): Promise<NormalizedBalance[]> {
    const exchange = new ccxt.binance({ apiKey, secret });
    const balances = await exchange.fetchBalance({ type: 'spot' });
    return this.normalize(balances);
  }
}

// integrations/trading212/trading212.service.ts
@Injectable()
class Trading212Adapter implements ExchangeAdapter {
  constructor(private readonly httpService: HttpService) {}

  async getBalances(): Promise<NormalizedBalance[]> {
    const { data } = await firstValueFrom(
      this.httpService.get('/api/v0/equity/pies')
    );
    return this.normalize(data);
  }
}
```

### Pattern 2: Bridge Currency for Multi-Currency Conversion

**What:** Convert all values through a single bridge currency (USD) rather than maintaining direct conversion rates between every currency pair. Store the exchange rate used at the time of each snapshot for auditability.

**When to use:** When supporting multiple currencies (EUR, USD, GBP) without needing to fetch N-squared exchange rate pairs.

**Trade-offs:** Introduces a small rounding compound (two conversions instead of one for non-USD pairs), which is negligible for portfolio tracking. Simplifies the exchange rate data model significantly.

**Example:**
```typescript
// integrations/currency/currency.service.ts
@Injectable()
class CurrencyService {
  private ratesCache: Map<string, Decimal>;

  async convert(amount: Decimal, from: string, to: string): Promise<Decimal> {
    if (from === to) return amount;
    const amountInUsd = amount.div(this.getRate(from));
    return amountInUsd.mul(this.getRate(to));
  }

  async refreshRates(): Promise<void> {
    // Fiat rates from Frankfurter API (free, no key, ECB data)
    const { data } = await firstValueFrom(
      this.httpService.get('https://api.frankfurter.dev/latest?from=USD')
    );
    // Persist to DB for snapshot auditing
    await this.prisma.exchangeRate.createMany({
      data: Object.entries(data.rates).map(([currency, rate]) => ({
        baseCurrency: 'USD',
        targetCurrency: currency,
        rate: rate as number,
        fetchedAt: new Date(),
      })),
    });
  }
}
```

### Pattern 3: Snapshot Capture as Scheduled Aggregation

**What:** A daily cron job triggers the snapshot service, which fetches all current balances (automated + manual), converts to base currency, and stores a complete portfolio snapshot with individual asset entries.

**When to use:** For historical tracking without real-time requirements. Daily granularity avoids API rate limit issues and keeps the data model simple.

**Trade-offs:** No intraday visibility, but the PROJECT.md explicitly scopes this out. The snapshot is a full capture, not a delta -- simpler to query and reason about.

**Example:**
```typescript
// scheduler/scheduler.service.ts
@Injectable()
class SchedulerService {
  @Cron('0 2 * * *', { timeZone: 'Europe/London' })
  async captureDaily() {
    this.logger.log('Starting daily snapshot capture');
    await this.currencyService.refreshRates();
    const portfolio = await this.portfolioService.aggregateAll();
    await this.snapshotService.capture(portfolio);
    this.logger.log('Daily snapshot capture complete');
  }
}
```

### Pattern 4: TanStack Query for Server State, Zustand for Client State

**What:** Split frontend state management cleanly. Server data (portfolio, snapshots, assets) lives in TanStack Query cache. Client-only data (base currency preference, UI toggles) lives in Zustand stores.

**When to use:** Always in this project. Almost all state comes from the server.

**Trade-offs:** Two state libraries instead of one, but each handles its domain better than a single library would.

**Example:**
```typescript
// hooks/usePortfolio.ts
export function usePortfolioSummary() {
  const baseCurrency = useSettingsStore(s => s.baseCurrency);
  return useQuery({
    queryKey: ['portfolio', 'summary', baseCurrency],
    queryFn: () => api.getPortfolioSummary(baseCurrency),
    staleTime: 5 * 60 * 1000, // 5 min -- data is daily
  });
}

// stores/useSettingsStore.ts
export const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      baseCurrency: 'EUR',
      setBaseCurrency: (c: string) => set({ baseCurrency: c }),
    }),
    { name: 'finance-settings' }
  )
);
```

## Data Flow

### Request Flow: Fetching Portfolio Summary

```
[React Dashboard]
    | GET /api/portfolio/summary?currency=EUR
    v
[PortfolioController]
    | Validates currency param (class-validator)
    v
[PortfolioService.aggregateAll()]
    |
    +-->  [AssetService.getManualAssets()]
    |        v
    |    [Prisma Client] -> PostgreSQL
    |
    +-->  [CcxtAdapter.getBalances()]
    |        v
    |    [CCXT Library] -> Exchange APIs (Binance, etc.)
    |
    +-->  [Trading212Adapter.getBalances()]
    |        v
    |    [@nestjs/axios HttpService] -> Trading 212 REST API
    |
    +-->  [CurrencyService.convertAll(assets, targetCurrency)]
             v
         [Cached exchange rates / Frankfurter API]
    |
    v
[PortfolioController] -> JSON response
    v
[React Dashboard] -> TanStack Query cache -> Render charts and tables
```

### Data Flow: Daily Snapshot Capture

```
[Scheduler] -- 2:00 AM cron --> [SchedulerService]
    |
    +-->  [CurrencyService.refreshRates()]
    |        v
    |    [Frankfurter API] -> rates cached + persisted via Prisma
    |
    +-->  [PortfolioService.aggregateAll()]
    |        v
    |    (same flow as above: manual + CCXT + T212)
    |
    +-->  [SnapshotService.capture(portfolio)]
             v
         [Prisma] -> PostgreSQL (Snapshot + SnapshotEntry rows)
```

### Data Flow: Rebalancing Calculation

```
[React Rebalancing Page]
    | GET /api/rebalancing/macro
    v
[RebalancingController]
    v
[RebalancingService]
    |
    +-->  [Prisma] -> fetch target allocations (AllocationTarget)
    |
    +-->  [PortfolioService.aggregateAll()] -> current allocations
    |
    +-->  [Calculate deviations using decimal.js]
         For each category:
           deviation = current_pct - target_pct
           suggested_action = "Buy" or "Sell"
           suggested_amount = abs(deviation) * total_value
    |
    v
[JSON response: { category, current%, target%, deviation%, action, amount }]
    v
[React] -> Render deviation bars and suggestion table
```

## Database Schema Overview

```
+-------------------+     +------------------------+
|     Asset          |     |  ExchangeConnection     |
+-------------------+     +------------------------+
| id               |     | id                      |
| name             |     | type (ccxt|t212)        |
| type (manual|    |     | exchangeName            |
|   automated)     |     | apiKey (encrypted)      |
| category         |     | apiSecret (encrypted)   |
| currency         |     | category                |
| balance (Decimal)|     | isActive                |
| connectionId?    |     +------------------------+
+-------------------+
         |
         | (snapshot captures current state)
         v
+-------------------+     +------------------------+
|    Snapshot        |---->|   SnapshotEntry         |
+-------------------+     +------------------------+
| id               |     | id                      |
| capturedAt       |     | snapshotId              |
| baseCurrency     |     | assetName               |
| totalValue       |     | category                |
|                  |     | nativeCurrency          |
+-------------------+     | nativeValue (Decimal)   |
                         | convertedValue (Decimal) |
                         | exchangeRateUsed         |
                         +------------------------+

+------------------------+  +------------------------+
|  AllocationTarget       |  |    ExchangeRate         |
+------------------------+  +------------------------+
| id                      |  | id                      |
| level (macro|intra)     |  | baseCurrency            |
| category                |  | targetCurrency          |
| subcategory?            |  | rate (Decimal)          |
| targetPercentage        |  | fetchedAt               |
+------------------------+  +------------------------+

+------------------------+
|   Settings              |
+------------------------+
| id                      |
| key                     |
| value                   |
+------------------------+
```

**Key design decisions:**
- **Prisma schema as source of truth:** All tables defined in `schema.prisma`. Migrations generated with `prisma migrate dev`. No manual SQL for schema changes.
- **Decimal types:** All monetary values use PostgreSQL `Decimal` type (mapped to `@db.Decimal(19,8)` in Prisma for crypto precision). Parsed to `decimal.js` in the application layer.
- `SnapshotEntry` stores both native and converted values plus the exchange rate used -- enables re-computation and auditing.
- `ExchangeConnection` is separate from `Asset` because one connection (e.g., Binance) yields multiple assets (BTC, ETH, etc.).
- `AllocationTarget` supports both macro-level and intra-category targets via the `level` discriminator.
- `Settings` table stores user preferences (base currency, snapshot time, etc.) as key-value pairs. Simple and extensible for a single-user app.
- API keys stored encrypted at rest -- even for a single-user app, exchange API keys are sensitive.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Single user (this project) | Monolith is perfect. No caching layer needed. Direct DB queries. Daily cron is sufficient. |
| 10 users | Add user auth, tenant isolation in DB. Still monolith. |
| 1,000+ users | Queue-based snapshot processing (BullMQ). Rate limit external API calls per user. Consider caching exchange rates in Redis. |

**Bottom line:** This is a single-user tool. Do not optimize for scale. Build for correctness and maintainability.

## Anti-Patterns

### Anti-Pattern 1: Fetching External Data on Every Request

**What people do:** Call CCXT and Trading 212 APIs every time the user loads the dashboard.
**Why it is wrong:** External API calls are slow (500ms-3s each), rate-limited, and unreliable. The UI becomes sluggish and flaky.
**Do this instead:** Fetch external data on a schedule (daily cron) and on-demand via a manual "Refresh" button. Serve dashboard views from the latest snapshot data in PostgreSQL.

### Anti-Pattern 2: Storing Values Only in Base Currency

**What people do:** Convert everything to the user's base currency at time of storage and discard the original currency/amount.
**Why it is wrong:** If the user changes base currency, all historical data is wrong. Exchange rates fluctuate, so you lose the ability to recompute values at different rates.
**Do this instead:** Store each asset's value in its native currency. Store the exchange rate used at snapshot time. Compute base-currency values on read using stored rates.

### Anti-Pattern 3: Fat Controllers

**What people do:** Put business logic directly in NestJS controllers.
**Why it is wrong:** Controllers become untestable and unreusable. The scheduler cannot reuse the same logic since it does not go through HTTP.
**Do this instead:** Controllers are thin -- validate input, call a service, return the response. All business logic lives in services.

### Anti-Pattern 4: One Giant "Portfolio" Table

**What people do:** Store everything (manual assets, exchange balances, snapshots, targets) in a single table with a type column.
**Why it is wrong:** Conflates fundamentally different data with different lifecycles. Querying becomes a nightmare.
**Do this instead:** Separate Prisma models: Asset, Snapshot, SnapshotEntry, AllocationTarget, ExchangeRate, ExchangeConnection, Settings.

### Anti-Pattern 5: Client-Side Currency Conversion

**What people do:** Send raw multi-currency data to the frontend and convert in JavaScript.
**Why it is wrong:** Inconsistent calculations between frontend and backend. Precision issues with floating-point math in JS.
**Do this instead:** Backend converts to the requested base currency using decimal.js and returns pre-computed values. Frontend only formats for display.

### Anti-Pattern 6: Using nestjs-ccxt or trading212-api Wrappers

**What people do:** Install npm wrapper packages for external APIs to save time.
**Why it is wrong:** Both `nestjs-ccxt` (2+ years unmaintained) and `trading212-api` (~1 year unmaintained) are dead packages. They will break with current library versions and provide no updates when APIs change.
**Do this instead:** Wrap CCXT directly in a NestJS service (~50 lines). Call T212 REST API directly via @nestjs/axios (~100 lines). You own the code and can update when APIs change.

## Sources

- [NestJS Documentation - Modules](https://docs.nestjs.com/)
- [NestJS Task Scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [Prisma NestJS Integration](https://www.prisma.io/nestjs)
- [Prisma Schema Reference](https://www.prisma.io/docs/orm/reference/prisma-schema-reference)
- [CCXT Documentation](https://docs.ccxt.com/)
- [CCXT GitHub - Manual](https://github.com/ccxt/ccxt/wiki/manual)
- [Trading 212 API Documentation](https://docs.trading212.com/api)
- [Frankfurter Currency API](https://frankfurter.dev/)
- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Zustand Documentation](https://zustand.docs.pmnd.rs/)
- [shadcn/ui Documentation](https://ui.shadcn.com/)

---
*Architecture research for: Personal portfolio management / finance aggregation*
*Researched: 2026-03-21*
