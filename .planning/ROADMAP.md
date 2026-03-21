# Roadmap: Finance Manager

## Overview

This roadmap delivers a personal portfolio management application from bare scaffold to full rebalancing engine. The journey starts with containerized infrastructure and data model (Phase 1), adds manual asset entry with currency conversion (Phase 2), integrates external exchanges (Phase 3), builds the unified portfolio dashboard (Phase 4), adds historical tracking via daily snapshots (Phase 5), and culminates in the two-tier rebalancing engine that is the primary differentiator (Phase 6). Each phase delivers a complete, verifiable capability that builds on the previous one.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Docker Compose infrastructure, NestJS + React scaffolding, Prisma schema with decimal precision
- [ ] **Phase 2: Manual Assets and Currency** - Manual asset CRUD and multi-currency conversion pipeline
- [ ] **Phase 3: Exchange Integrations** - CCXT crypto exchange and Trading 212 pies adapters with on-demand refresh
- [ ] **Phase 4: Core Portfolio Dashboard** - Unified net worth display with allocation views and charts
- [ ] **Phase 5: Snapshots and History** - Daily snapshot system with historical portfolio charts
- [ ] **Phase 6: Rebalancing Engine** - Macro-level and intra-category rebalancing with deviation tracking and suggestions

## Phase Details

### Phase 1: Foundation
**Goal**: The containerized development environment runs all three services and the database schema captures the full data model with correct decimal precision
**Depends on**: Nothing (first phase)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04, INFRA-05, INFRA-06
**Success Criteria** (what must be TRUE):
  1. Running `docker-compose up` starts backend, frontend, and database services that can communicate with each other
  2. PostgreSQL data survives container restarts (persistent volume)
  3. NestJS backend responds to a health-check endpoint and Prisma connects to the database
  4. React frontend loads in a browser and can reach the backend API
  5. All monetary columns in the database schema use NUMERIC type and the backend uses decimal.js for financial math
**Plans**: TBD

Plans:
- [ ] 01-01: TBD
- [ ] 01-02: TBD

### Phase 2: Manual Assets and Currency
**Goal**: Users can create and manage manual assets and see all values converted to their chosen base currency
**Depends on**: Phase 1
**Requirements**: DATA-01, DATA-02, CURR-01, CURR-02, CURR-03, CURR-04, CURR-05
**Success Criteria** (what must be TRUE):
  1. User can add a manual asset with name, currency, and balance, and it appears in the asset list
  2. User can edit and delete existing manual assets
  3. User can select a base currency (EUR, USD, GBP) and all asset values display in that currency
  4. Fiat exchange rates are fetched from Frankfurter API and crypto rates from CCXT ticker data
  5. Exchange rates are stored alongside asset data so historical conversions remain accurate
**Plans**: TBD

Plans:
- [ ] 02-01: TBD
- [ ] 02-02: TBD

### Phase 3: Exchange Integrations
**Goal**: Users can connect crypto exchanges and Trading 212 to automatically fetch balances, with bot/holding classification for crypto accounts
**Depends on**: Phase 2
**Requirements**: DATA-03, DATA-04, DATA-05, DATA-06, DATA-07, DATA-08
**Success Criteria** (what must be TRUE):
  1. User can add a CCXT-supported crypto exchange via API key and see fetched balances appear automatically
  2. User can connect a Trading 212 account via API key and see pie holdings and allocations
  3. User can trigger an on-demand refresh that updates balances from all connected sources
  4. User can tag CCXT accounts as "Trading Bot" or "Holding" and the classification persists
**Plans**: TBD

Plans:
- [ ] 03-01: TBD
- [ ] 03-02: TBD

### Phase 4: Core Portfolio Dashboard
**Goal**: Users see a unified dashboard showing total net worth and allocation breakdown across all asset sources with visual charts
**Depends on**: Phase 3
**Requirements**: PORT-01, PORT-02, PORT-03, PORT-04, PORT-05
**Success Criteria** (what must be TRUE):
  1. Dashboard displays total net worth combining all automated and manual sources in the selected base currency
  2. Portfolio allocation is shown across macro categories (Crypto, Stocks, Liquidity, Trading Bots) with correct percentages
  3. A pie chart visualizes macro category allocation
  4. A bar chart visualizes allocation structure
  5. User can drill into any macro category to see per-asset breakdown
**Plans**: TBD

Plans:
- [ ] 04-01: TBD
- [ ] 04-02: TBD

### Phase 5: Snapshots and History
**Goal**: The system autonomously captures daily portfolio snapshots and users can view historical net worth trends at multiple granularities
**Depends on**: Phase 4
**Requirements**: HIST-01, HIST-02, HIST-03, HIST-04, HIST-05
**Success Criteria** (what must be TRUE):
  1. A scheduled cron job captures a daily snapshot of all balances and exchange rates without user intervention
  2. User can view historical net worth as a line chart
  3. Historical view supports daily, weekly, and monthly granularity toggles
  4. User can view historical performance filtered by macro category
  5. If the system was down and missed snapshots, it catches up on startup
**Plans**: TBD

Plans:
- [ ] 05-01: TBD
- [ ] 05-02: TBD

### Phase 6: Rebalancing Engine
**Goal**: Users can define target allocations at both macro and intra-category levels, see deviations, and receive actionable rebalancing suggestions
**Depends on**: Phase 5
**Requirements**: REBL-01, REBL-02, REBL-03, REBL-04, REBL-05, REBL-06, REBL-07, REBL-08
**Success Criteria** (what must be TRUE):
  1. User can define target allocation percentages for macro categories and see current vs target with deviation
  2. System generates specific suggested actions to rebalance at the macro level
  3. User can define target allocation percentages within each macro category (intra-category) and see deviations
  4. System generates intra-category rebalancing suggestions
  5. Trading 212 rebalancing suggestions reference pies only (not individual stocks) and Trading Bot accounts are included in allocation tracking
**Plans**: TBD

Plans:
- [ ] 06-01: TBD
- [ ] 06-02: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/? | Not started | - |
| 2. Manual Assets and Currency | 0/? | Not started | - |
| 3. Exchange Integrations | 0/? | Not started | - |
| 4. Core Portfolio Dashboard | 0/? | Not started | - |
| 5. Snapshots and History | 0/? | Not started | - |
| 6. Rebalancing Engine | 0/? | Not started | - |
