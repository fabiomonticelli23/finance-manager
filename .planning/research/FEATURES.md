# Feature Research

**Domain:** Personal portfolio management / multi-source finance aggregation
**Researched:** 2026-03-21
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete. Drawn from analysis of Ghostfolio, Kubera, Rotki, Capitally, Sharesight, Snowball Analytics, and numerous portfolio trackers.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Unified net worth display** | The core value proposition. Every competitor has this front and center. Without it the app has no reason to exist. | LOW | Single number combining all sources, converted to base currency. Kubera, Capitally, Ghostfolio all lead with this. |
| **Multi-source data aggregation** | Users expect to connect accounts, not manually re-enter everything. CCXT for crypto, Trading 212 API for stocks. | HIGH | CCXT covers 100+ exchanges. T212 Pies API is deprecated but still functional -- build defensively (see Pitfalls). Manual entry is the fallback for unsupported sources. |
| **Manual asset entry** | Bank accounts, real estate, cash, and unsupported exchanges need manual tracking. Every competitor supports this. | LOW | Simple: name, currency, balance. Capitally and Kubera both support "anything" via manual entry. Don't over-engineer with categories or metadata upfront. |
| **Base currency selection** | International users and multi-currency holders require seeing everything in one currency. Standard across all competitors. | MEDIUM | EUR, USD, GBP minimum. Frankfurter API (free, ECB data, no rate limits) for fiat. CCXT tickers for crypto rates. Historical rates needed for accurate snapshots. |
| **Currency conversion** | Automatic conversion of all holdings to the selected base currency. Not optional when you hold assets in multiple currencies. | MEDIUM | Must handle fiat-to-fiat and crypto-to-fiat. Frankfurter API for fiat. CCXT fetchTicker for crypto. Store original currency alongside converted value. Use decimal.js for all conversion math. |
| **Portfolio allocation view** | Pie/donut chart showing category breakdown is the most basic visualization in any portfolio tracker. Every single competitor has this. | LOW | Macro categories: Crypto, Stocks, Liquidity, Trading Bots (per PROJECT.md). Recharts PieChart (with react-is fix for React 19) or Nivo as fallback. |
| **Basic P&L display** | Users need to know if they are making or losing money. Absolute and percentage gains/losses. | MEDIUM | Unrealized P&L = current value minus cost basis. For assets without purchase history (manual entries), P&L tracks change from first recorded balance. |
| **Historical portfolio value** | Users expect to see their net worth over time. Ghostfolio, Rotki, Capitally, Portfolio Performance all provide this. | MEDIUM | Daily snapshots approach (as specified in PROJECT.md) is the standard for non-real-time trackers. Store daily snapshot of total value and per-category values. Display as Recharts LineChart with daily/weekly/monthly granularity. |
| **Data refresh / sync** | Users need a way to pull latest data from connected sources. Even if not real-time, on-demand or scheduled sync is expected. | MEDIUM | On-demand "refresh" button plus daily sync via @nestjs/schedule cron. Respect API rate limits (CCXT and T212 both have them). |
| **Responsive web dashboard** | Users expect the dashboard to work on desktop and mobile browsers. | MEDIUM | React + Tailwind CSS + shadcn/ui. Desktop-first is fine for a personal tool, but must not break on mobile. |

### Differentiators (Competitive Advantage)

Features that set this product apart from Ghostfolio, Rotki, and spreadsheet-based trackers. These align with the project's stated requirements and fill gaps in the open-source ecosystem.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Macro-level rebalancing** | Define target allocations per category (e.g., 40% Crypto, 30% Stocks, 20% Liquidity, 10% Trading Bots) and see current vs target deviation with suggested actions. Ghostfolio has basic allocation but no target-based rebalancing. Rotki has none. | HIGH | Core differentiator vs open-source alternatives. Show deviation percentage, dollar amount over/under, and specific "buy X / sell Y" suggestions. Suggestions only -- no auto-execution (per PROJECT.md). Threshold-based tolerance bands (e.g., >5% deviation). |
| **Intra-category rebalancing** | Rebalancing within each macro category (e.g., within Crypto: 50% BTC, 30% ETH, 20% altcoins). Very few tools offer two-tier rebalancing. | HIGH | Unique combination when paired with macro rebalancing. Users define sub-allocations within each category. Same deviation/suggestion pattern. Key differentiator no open-source tool does well. |
| **Trading 212 pies integration** | Direct integration with T212 pies API to fetch holdings and allocation data. No open-source portfolio tracker integrates T212. | MEDIUM | Pies API is deprecated but still functional. Fetch pie structure, holdings, and values via @nestjs/axios. Build a fallback path for when the API eventually changes. |
| **CCXT-based trading bot grouping** | CCXT accounts grouped under "Trading Bot" macro category with separate allocation tracking and performance metrics. Purpose-built for users running algorithmic strategies alongside long-term holdings. | MEDIUM | Distinct from general crypto holdings. Track per-bot P&L, allocation within the bot category. |
| **P&L drill-down by category and asset** | Hierarchical P&L: total portfolio > category > individual asset. Absolute and percentage. Most open-source tools show only portfolio-level P&L. | MEDIUM | Category-level P&L (how is my Crypto doing vs Stocks?) is a natural extension of the allocation view. Per-asset P&L requires tracking cost basis or historical balance changes. |
| **Allocation visualization with targets overlay** | Bar charts showing current allocation with target allocation overlaid. Visual gap analysis. | MEDIUM | Recharts BarChart with current vs target side-by-side. Deviation highlighted in red/green. More intuitive than a table of numbers. |
| **Historical performance by category** | Not just total portfolio over time, but per-category historical performance lines. | MEDIUM | Requires per-category values in daily snapshots. Recharts LineChart with one line per category. |
| **Containerized self-hosted deployment** | Docker Compose with three services (backend, frontend, database). Privacy-first, no cloud dependency, no subscription. | LOW | Already specified in PROJECT.md. Docker Compose is well-understood. Multi-stage builds for NestJS. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems for this specific project scope.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Real-time price streaming / WebSocket feeds** | "I want live prices updating on my dashboard." | Massively increases complexity (WebSocket infrastructure, connection management, rate limits). For a daily-snapshot portfolio tracker, real-time adds no meaningful value. | Daily snapshots with on-demand refresh button. |
| **Automated trade execution** | "The rebalancing tool should just execute the trades for me." | Enormous liability and complexity. A bug could lose real money. | Suggestions-only rebalancing (per PROJECT.md). |
| **Multi-user authentication system** | "What if my partner wants to use it too?" | Auth adds session management, user isolation, RBAC, password reset. For a personal tool used by one person, this is pure overhead. | Single-user, no login. Network-level security. |
| **Individual stock tracking on Trading 212** | "I want to see each stock, not just pies." | T212 Pies API only exposes pie-level data. Individual tracking would require screen scraping or CSV import. | Pies-only integration as designed. |
| **Tax reporting / tax-loss harvesting** | "Calculate my capital gains taxes." | Tax rules vary by country, asset type, holding period, and change annually. A black hole of complexity. | Export data for use in dedicated tax tools. Track cost basis and gains data, but don't compute tax liability. |
| **Mobile native app** | "I want an iPhone/Android app." | Doubles the frontend codebase. | Responsive web app works on mobile browsers. PWA wrapper later if truly needed. |
| **AI-powered insights / recommendations** | "Use AI to tell me what to invest in." | Regulatory minefield (financial advice). Unreliable for investment decisions. | Clear, honest metrics: allocation deviation, P&L trends. Let the user decide. |
| **Social features / portfolio sharing** | "Let me share my portfolio performance." | Privacy risk, scope creep. Antithetical to a self-hosted privacy-first tool. | None needed. Export/screenshot if sharing is desired. |

## Feature Dependencies

```
[Currency Conversion (Frankfurter + CCXT tickers)]
    +-- requires --> [Base Currency Selection]
    +-- required by --> [Unified Net Worth Display]
    +-- required by --> [P&L Calculations]
    +-- required by --> [Historical Snapshots]

[Multi-Source Data Aggregation]
    +-- includes --> [CCXT Integration (ccxt npm)]
    +-- includes --> [Trading 212 Pies Integration (@nestjs/axios)]
    +-- includes --> [Manual Asset Entry (Prisma CRUD)]
    +-- required by --> [Unified Net Worth Display]

[Unified Net Worth Display]
    +-- requires --> [Multi-Source Data Aggregation]
    +-- requires --> [Currency Conversion]
    +-- required by --> [Portfolio Allocation View (Recharts PieChart)]
    +-- required by --> [Historical Portfolio Value]

[Portfolio Allocation View]
    +-- requires --> [Unified Net Worth Display]
    +-- required by --> [Macro-Level Rebalancing]
    +-- required by --> [Intra-Category Rebalancing]

[Daily Snapshot System (@nestjs/schedule)]
    +-- requires --> [Unified Net Worth Display]
    +-- required by --> [Historical Portfolio Value (Recharts LineChart)]
    +-- required by --> [Historical Performance by Category]
    +-- required by --> [P&L Drill-Down]

[Macro-Level Rebalancing]
    +-- requires --> [Portfolio Allocation View]
    +-- enhances --> [Allocation Visualization with Targets]

[Intra-Category Rebalancing]
    +-- requires --> [Macro-Level Rebalancing]
    +-- requires --> [Per-Asset Data within Categories]

[P&L Drill-Down]
    +-- requires --> [Daily Snapshot System]
    +-- requires --> [Per-Asset Tracking within Categories]
```

### Dependency Notes

- **Currency Conversion requires Base Currency Selection:** Must know the target currency before converting anything. Base currency setting is the foundation.
- **Unified Net Worth requires both Aggregation and Currency Conversion:** Cannot show a total without pulling data from sources and converting to a common currency.
- **Allocation View requires Unified Net Worth:** Need to know the total to compute percentages.
- **Rebalancing requires Allocation View:** Need current allocations before comparing to targets.
- **Intra-Category Rebalancing requires Macro-Level:** The two-tier model means macro categories must exist before drill-down.
- **Historical views require Daily Snapshot System:** Time-series data comes from stored snapshots, not recalculated on the fly.

## MVP Definition

### Launch With (v1)

Minimum viable product -- what is needed to validate the core value: "Complete, accurate net worth visibility across all asset sources."

- [ ] **Manual asset entry** -- simplest data source, validates the data model and UI
- [ ] **Base currency selection + currency conversion** -- required for any multi-currency portfolio
- [ ] **Unified net worth display** -- the core value proposition, must work immediately
- [ ] **CCXT integration (read-only)** -- fetch balances from crypto exchanges, the primary automated source
- [ ] **Portfolio allocation view (macro categories)** -- basic Recharts PieChart, validates the category model
- [ ] **Basic P&L display (portfolio-level)** -- users need to know if they're up or down
- [ ] **Daily snapshot system** -- start capturing data from day one, even if historical views come later
- [ ] **Docker Compose deployment** -- containerized from the start per project constraints

### Add After Validation (v1.x)

Features to add once core aggregation and display are working correctly.

- [ ] **Trading 212 pies integration** -- second automated data source, adds stock/ETF coverage
- [ ] **Historical portfolio value charts** -- once snapshots are accumulating, visualize the history
- [ ] **P&L drill-down by category and asset** -- deeper analytics once basic P&L is validated
- [ ] **Macro-level rebalancing** -- target allocation definition, deviation display, suggestions
- [ ] **Allocation visualization with targets overlay** -- visual rebalancing aid
- [ ] **Historical performance by category** -- per-category trend lines

### Future Consideration (v2+)

Features to defer until the core product is stable and daily-driven.

- [ ] **Intra-category rebalancing** -- most complex rebalancing feature, needs macro rebalancing working first
- [ ] **Trading bot grouping under CCXT** -- specialized category logic for algorithmic trading accounts
- [ ] **Data export (CSV/JSON)** -- useful for tax tools and backup, but not critical for daily use
- [ ] **PWA wrapper** -- if mobile access becomes a real need rather than theoretical

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Unified net worth display | HIGH | LOW | P1 |
| Manual asset entry | HIGH | LOW | P1 |
| Base currency + conversion | HIGH | MEDIUM | P1 |
| CCXT integration (read-only) | HIGH | MEDIUM | P1 |
| Portfolio allocation view | HIGH | LOW | P1 |
| Basic P&L display | HIGH | MEDIUM | P1 |
| Daily snapshot system | HIGH | MEDIUM | P1 |
| Docker Compose deployment | HIGH | LOW | P1 |
| Trading 212 pies integration | MEDIUM | MEDIUM | P2 |
| Historical portfolio value charts | HIGH | LOW | P2 |
| P&L drill-down (category + asset) | MEDIUM | MEDIUM | P2 |
| Macro-level rebalancing | HIGH | HIGH | P2 |
| Allocation with targets overlay | MEDIUM | LOW | P2 |
| Historical performance by category | MEDIUM | LOW | P2 |
| Intra-category rebalancing | MEDIUM | HIGH | P3 |
| CCXT trading bot grouping | LOW | MEDIUM | P3 |
| Data export (CSV/JSON) | LOW | LOW | P3 |
| PWA wrapper | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for launch -- validates core value proposition
- P2: Should have, add once core is working -- the differentiating features
- P3: Nice to have, future consideration -- refinements and edge cases

## Competitor Feature Analysis

| Feature | Ghostfolio | Rotki | Kubera | Snowball Analytics | Our Approach |
|---------|------------|-------|--------|-------------------|--------------|
| Multi-source aggregation | Manual transactions, some broker imports | CEX + blockchain + manual | Plaid/Yodlee auto-sync, manual | Yodlee auto-sync, CSV | CCXT + T212 API + manual. Fewer sources but deeper integration. |
| Net worth tracking | Yes (total only) | Yes (total + per-chain) | Yes (total + per-category) | Yes (total) | Total + per-macro-category. Four explicit categories. |
| Allocation visualization | Basic pie chart | Basic charts | Allocation breakdown | Allocation with targets | Recharts pie/bar with target overlay. Visual deviation. |
| Rebalancing | None | None | None | Target allocations with suggestions | Two-tier (macro + intra-category). Primary differentiator. |
| Historical performance | Line chart, multiple timeframes | Net worth over time | Limited | Portfolio growth chart | Daily snapshots, per-category historical lines. |
| P&L tracking | ROAI metric, multiple timeframes | PnL over custom periods | Basic gain/loss | Basic percentage | Hierarchical: portfolio > category > asset. Absolute + percentage. |
| Currency handling | Multi-currency, auto-conversion | Main currency setting | Basic multi-currency | Basic conversion | User-selectable base, separate fiat + crypto rate sources, historical rate storage. |
| Self-hosted | Yes (Docker, NestJS + Angular) | Yes (Desktop app + Docker) | No (SaaS only) | No (SaaS only) | Yes (Docker Compose, NestJS + React + PostgreSQL). |
| T212 integration | No | No | No | No | Yes -- unique among open-source tools. |
| Tech stack | NestJS + Angular + PostgreSQL + Prisma | Python + SQLite/PostgreSQL | Proprietary | Proprietary | NestJS + React + PostgreSQL + Prisma. Similar to Ghostfolio backend but React frontend. |

## Sources

- [Capitally: Best Portfolio Tracker Comparison](https://www.mycapitally.com/blog/best-portfolio-tracker-for-the-modern-diy-investor) -- detailed feature comparison
- [Ghostfolio GitHub](https://github.com/ghostfolio/ghostfolio) -- feature set, tech stack
- [Rotki GitHub](https://github.com/rotki/rotki) -- privacy-first portfolio tracking
- [Trading 212 API Docs](https://docs.trading212.com/api) -- Pies API capabilities
- [Trading 212 Community: API Update](https://community.trading212.com/t/trading-212-api-update/87988) -- Pies API status
- [Frankfurter API](https://frankfurter.dev/) -- free ECB exchange rate data
- [CCXT GitHub](https://github.com/ccxt/ccxt) -- 100+ exchange support

---
*Feature research for: Personal portfolio management / multi-source finance aggregation*
*Researched: 2026-03-21*
