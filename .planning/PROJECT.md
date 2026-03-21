# Finance Manager

## What This Is

A personal portfolio management web application that aggregates all assets and investments into a single unified view. It pulls data automatically from crypto exchanges (via CCXT) and Trading 212 (via pies API), with manual entry for bank accounts and unsupported exchanges. Built for one user who wants total visibility, clear analytics, and intelligent rebalancing across their entire portfolio.

## Core Value

Complete, accurate net worth visibility across all asset sources — if the numbers aren't right and unified, nothing else matters.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Aggregate balances from CCXT-connected crypto exchanges (automated)
- [ ] Aggregate balances from Trading 212 pies via API (automated, pies only — no individual stocks)
- [ ] Manual asset entry for bank accounts and unsupported exchanges (name, currency, balance)
- [ ] Display total net worth combining all automated and manual data
- [ ] User-selectable base currency with multi-currency conversion
- [ ] Portfolio allocation view across macro categories: Crypto, Stocks, Liquidity, Trading Bots
- [ ] Visual dashboards with pie charts and bar charts for allocation and structure
- [ ] Historical portfolio value tracking via daily snapshots (daily, weekly, monthly views)
- [ ] Profit & loss tracking (absolute and percentage) with drill-down by category and asset
- [ ] Macro-level rebalancing: define target allocations per category, see current vs target, deviation, and suggested actions
- [ ] Intra-category rebalancing: define allocations within each category, track deviations, get suggestions
- [ ] Trading 212 rebalancing via pies only (fetch and manage pies via API)
- [ ] CCXT trading accounts grouped under "Trading Bot" macro category with full allocation/performance/rebalancing support
- [ ] NestJS backend, React frontend, PostgreSQL database
- [ ] Fully containerized with Docker: three services (backend, frontend, database) with docker-compose, shared network, env vars, and persistent volumes

### Out of Scope

- Multi-user / authentication system — single-user app, no login needed
- Automated trade execution — rebalancing shows suggestions only, user executes manually
- Individual stock tracking on Trading 212 — pies only
- Mobile app — web-first
- Real-time price streaming / WebSocket feeds — daily snapshots are sufficient
- OAuth / social login — no auth needed for single user

## Context

- User has working Trading 212 API access (key available)
- CCXT library covers most crypto exchanges the user needs
- Manual assets are deliberately simple (name + currency + balance) to avoid over-engineering
- Daily snapshots capture portfolio state for historical tracking — no need for intraday granularity
- "Trading Bot" is a macro category for CCXT-connected accounts used for active/algorithmic trading, distinct from long-term crypto holdings
- Base currency is user-selectable (not hardcoded), supporting EUR, USD, GBP and potentially others

## Constraints

- **Tech stack**: NestJS + React + PostgreSQL — decided upfront, non-negotiable
- **Containerization**: Docker with three separate Dockerfiles + docker-compose.yml — required for deployment
- **Trading 212**: Pies-only integration — API limitations and user preference
- **Data freshness**: Daily snapshots — not real-time, reduces API load and complexity

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single-user, no auth | Personal tool — auth adds complexity without value | — Pending |
| Suggestions-only rebalancing | User wants control over execution, reduces risk | — Pending |
| Daily snapshots for history | Sufficient granularity, avoids API rate limits | — Pending |
| Pies-only T212 integration | Matches user's T212 usage pattern, simpler API surface | — Pending |
| User-selectable base currency | Flexibility across EUR/USD/GBP without hardcoding | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-21 after initialization*
