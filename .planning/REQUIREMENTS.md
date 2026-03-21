# Requirements: Finance Manager

**Defined:** 2026-03-21
**Core Value:** Complete, accurate net worth visibility across all asset sources

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Infrastructure

- [ ] **INFRA-01**: Docker Compose runs three services (backend, frontend, database) with shared network and env vars
- [ ] **INFRA-02**: PostgreSQL data persisted via Docker volumes
- [ ] **INFRA-03**: Separate Dockerfiles for backend (NestJS), frontend (React), and database (PostgreSQL)
- [ ] **INFRA-04**: NestJS backend scaffolded with modular architecture and Prisma ORM
- [ ] **INFRA-05**: React frontend scaffolded with responsive dashboard layout
- [ ] **INFRA-06**: All financial calculations use decimal precision (NUMERIC columns + decimal.js)

### Data Sources

- [ ] **DATA-01**: User can add manual assets with name, currency, and balance
- [ ] **DATA-02**: User can edit and delete manual assets
- [ ] **DATA-03**: User can connect CCXT-supported crypto exchanges via API key
- [ ] **DATA-04**: System fetches balances from connected CCXT exchanges automatically
- [ ] **DATA-05**: User can connect Trading 212 account via API key
- [ ] **DATA-06**: System fetches Trading 212 pie holdings and allocations via Pies API
- [ ] **DATA-07**: User can trigger on-demand data refresh from all connected sources
- [ ] **DATA-08**: CCXT accounts can be tagged as "Trading Bot" (active trading) vs "Holding" (long-term)

### Currency

- [ ] **CURR-01**: User can select base currency for portfolio display
- [ ] **CURR-02**: System converts all holdings to base currency automatically
- [ ] **CURR-03**: Fiat exchange rates fetched from Frankfurter API (ECB data)
- [ ] **CURR-04**: Crypto-to-fiat rates derived from CCXT ticker data
- [ ] **CURR-05**: Exchange rates stored with each snapshot for historical accuracy

### Portfolio

- [ ] **PORT-01**: Dashboard displays total net worth combining all automated and manual sources
- [ ] **PORT-02**: Portfolio allocation shown across macro categories: Crypto, Stocks, Liquidity, Trading Bots
- [ ] **PORT-03**: Pie chart visualizing macro category allocation
- [ ] **PORT-04**: Bar chart visualizing allocation structure
- [ ] **PORT-05**: Per-asset breakdown visible within each macro category

### Historical

- [ ] **HIST-01**: System captures daily snapshot of all balances and exchange rates via scheduled cron job
- [ ] **HIST-02**: User can view historical net worth as line chart
- [ ] **HIST-03**: Historical view supports daily, weekly, and monthly granularity
- [ ] **HIST-04**: Historical performance viewable per macro category
- [ ] **HIST-05**: Snapshot system handles missed snapshots (catch-up on startup)

### Rebalancing

- [ ] **REBL-01**: User can define target allocation percentages for macro categories
- [ ] **REBL-02**: Dashboard shows current vs target allocation with deviation (absolute and percentage)
- [ ] **REBL-03**: System suggests specific actions to rebalance at macro level
- [ ] **REBL-04**: User can define target allocation percentages within each macro category (intra-category)
- [ ] **REBL-05**: System shows intra-category current vs target deviation
- [ ] **REBL-06**: System suggests intra-category rebalancing actions
- [ ] **REBL-07**: Trading 212 rebalancing suggestions use pies only (not individual stocks)
- [ ] **REBL-08**: Trading Bot accounts included in allocation tracking and rebalancing logic

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Analytics

- **ANLT-01**: Profit & loss tracking (absolute and percentage) at portfolio level
- **ANLT-02**: P&L drill-down by macro category
- **ANLT-03**: P&L drill-down by individual asset
- **ANLT-04**: Cost basis tracking for assets with purchase history

### Export

- **EXPRT-01**: Export portfolio data as CSV
- **EXPRT-02**: Export portfolio data as JSON

### Enhancement

- **ENHN-01**: PWA wrapper for mobile access
- **ENHN-02**: Data export for tax tools

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Multi-user authentication | Single-user personal tool -- auth adds complexity without value |
| Automated trade execution | Rebalancing shows suggestions only -- a bug could lose real money |
| Individual stock tracking on T212 | Pies-only integration per user preference and API limitations |
| Real-time price streaming | Daily snapshots sufficient; real-time adds massive complexity |
| Tax reporting / tax-loss harvesting | Tax rules vary by country and change annually -- black hole of complexity |
| Mobile native app | Responsive web app works on mobile browsers |
| AI-powered insights | Regulatory minefield (financial advice); clear metrics are better |
| Social / portfolio sharing | Privacy risk, scope creep, antithetical to self-hosted tool |
| OAuth / social login | No auth needed for single user |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 1 | Pending |
| INFRA-02 | Phase 1 | Pending |
| INFRA-03 | Phase 1 | Pending |
| INFRA-04 | Phase 1 | Pending |
| INFRA-05 | Phase 1 | Pending |
| INFRA-06 | Phase 1 | Pending |
| DATA-01 | Phase 2 | Pending |
| DATA-02 | Phase 2 | Pending |
| DATA-03 | Phase 3 | Pending |
| DATA-04 | Phase 3 | Pending |
| DATA-05 | Phase 3 | Pending |
| DATA-06 | Phase 3 | Pending |
| DATA-07 | Phase 3 | Pending |
| DATA-08 | Phase 3 | Pending |
| CURR-01 | Phase 2 | Pending |
| CURR-02 | Phase 2 | Pending |
| CURR-03 | Phase 2 | Pending |
| CURR-04 | Phase 2 | Pending |
| CURR-05 | Phase 2 | Pending |
| PORT-01 | Phase 4 | Pending |
| PORT-02 | Phase 4 | Pending |
| PORT-03 | Phase 4 | Pending |
| PORT-04 | Phase 4 | Pending |
| PORT-05 | Phase 4 | Pending |
| HIST-01 | Phase 5 | Pending |
| HIST-02 | Phase 5 | Pending |
| HIST-03 | Phase 5 | Pending |
| HIST-04 | Phase 5 | Pending |
| HIST-05 | Phase 5 | Pending |
| REBL-01 | Phase 6 | Pending |
| REBL-02 | Phase 6 | Pending |
| REBL-03 | Phase 6 | Pending |
| REBL-04 | Phase 6 | Pending |
| REBL-05 | Phase 6 | Pending |
| REBL-06 | Phase 6 | Pending |
| REBL-07 | Phase 6 | Pending |
| REBL-08 | Phase 6 | Pending |

**Coverage:**
- v1 requirements: 37 total
- Mapped to phases: 37
- Unmapped: 0

---
*Requirements defined: 2026-03-21*
*Last updated: 2026-03-21 after roadmap creation*
