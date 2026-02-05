# Trading Platform — Frontend Architecture & Product Decisions (Case Study)

This document describes how I approach designing and delivering a **trading + portfolio intelligence platform** as a frontend lead / product-minded UI architect.  
The goal is to show *how I make decisions* when requirements are given, deadlines are real, and the product must feel trustworthy under financial pressure.

---

## 0) Context (what we were building)

A trading platform that supports:
- Registration + verification (India context)
- Secure login
- Dashboard (portfolio summary, allocation, market snapshots, performance)
- Portfolio (holdings, P&L, transactions)
- Watchlist (custom lists, quick actions)
- Screeners (filter/rule builder, multiple screener sets)
- News (ticker-linked + market news)
- Alerts (price / rule alerts)
- Account settings (KYC status, preferences)

Target: medium-scale product (not millions of concurrent users), but still requires:
- predictable UI states
- strong trust signals
- fast performance
- safe workflows (no accidental destructive actions)

---

## 1) Step 1 — Product Map (modules, roles, critical flows)

### Modules
1. Auth & Registration
2. KYC & Verification (PAN/Aadhaar + phone/email)
3. Dashboard
4. Portfolio
5. Watchlist
6. Screeners
7. News
8. Alerts
9. Settings / Profile

### Roles
- Retail user (primary)
- Support / Ops (secondary, optional)
- Admin (optional later)

### Critical flows (user journeys)
- Register → Verify identity → Login
- Onboard → connect broker / portfolio import (if applicable)
- Daily use: open dashboard → check performance → open ticker → decide action
- Use watchlist and alerts to reduce monitoring effort
- Use screeners to discover opportunities
- Consume news tied to portfolio/watchlist to reduce context switching

### Failure modes (what can go wrong)
- Verification failures (PAN/Aadhaar mismatch)
- API slow/stale data
- WebSocket drops (if live pricing exists)
- Partial data: portfolio loads but news fails
- Inconsistent states: alert saved but not visible, watchlist mismatch

**Decision:** I treat these failure modes as product requirements (not edge cases).  
A trading app loses trust quickly if it feels inconsistent.

---

## 2) Step 2 — Architecture & Layering Model (industry standard)

I standardise a predictable layering model:

### UI Layer (presentation)
- pages, layouts, UI components
- no business logic beyond view formatting

### Feature Layer (orchestration)
- `features/dashboard`, `features/portfolio`, etc.
- coordinates fetching, state transitions, view models
- owns page composition decisions

### Domain Layer (business rules)
- models: Portfolio, Holding, Alert, ScreenerRule, VerificationStatus
- pure functions: validations, transitions, computations

### Data Layer (contracts)
- API clients
- DTO → Domain mapping
- caching & retry logic
- WebSocket client (if applicable)

### Platform Layer (cross-cutting)
- auth/session
- routing
- design system
- error boundaries
- telemetry

**Why this matters:** UI changes frequently; domain logic should not.  
This separation prevents “UI refactors breaking trading rules”.

---

## 3) Step 3 — Folder Structure (so the team scales)

A structure that a team of 5–6 engineers can follow without confusion:

src/
  app/
    routes/
    providers/
    AppShell.tsx
  platform/
    auth/
    telemetry/
    websocket/
    config/
  design-system/
    tokens/
    components/
    patterns/
  shared/
    ui/
    hooks/
    utils/
    types/
  domain/
    trading/
    portfolio/
    verification/
  data/
    http/
    websocket/
    mappers/
  features/
    auth/
    onboarding/
    dashboard/
    portfolio/
    watchlist/
    screeners/
    news/
    alerts/

**Rules I enforce**
- UI consumes domain models, not raw API responses
- feature modules own orchestration, not domain rules
- data mapping is central, not sprinkled inside components
- shared is for truly generic utilities only (avoid dumping ground)

---

## 4) Step 4 — Auth + Verification Design (PAN/Aadhaar + email/phone)

### Registration steps (India context)
1. Email + Phone verification (OTP)
2. PAN verification (name + PAN match)
3. Aadhaar verification (OTP or offline Aadhaar verification approach)
4. Optional “Penny drop” bank verification (if needed for withdrawals)
5. Create user profile and enable trading features

### Token strategy (JWT) and where to store
**Decision: use HttpOnly secure cookies for access/refresh tokens** where possible.
- Prevents XSS token theft (important in finance)
- Centralised session management
- Easier to refresh tokens silently

If the backend cannot support cookie-based auth, fallback:
- access token in memory
- refresh token in HttpOnly cookie (preferred) OR secure storage based on platform constraints

**Why not localStorage for tokens**
- XSS risk (token theft)
- trading/finance products are high trust; we avoid avoidable risks

### Session behavior
- Access token short-lived
- Refresh flow automatic
- On 401: refresh → retry once → if fails, logout + friendly UI

**User trust UX**
- show session expiry warning only when required
- avoid surprising logouts during actions

---

## 5) Step 5 — State Strategy (server, UI, workflow, derived, realtime)

I split state intentionally to avoid duplication and “ghost bugs”.

### (A) Server state (API data)
- portfolio summary
- holdings
- watchlists
- screener results
- news feeds
- alerts list

Strategy:
- cache per module
- pagination support for large lists
- stale-while-revalidate patterns where safe

### (B) UI state (local)
- selected tab
- filter inputs
- dialog open/close
- sorting, pagination

Keep local to feature module unless shared across routes.

### (C) Workflow state (explicit steps)
- registration and verification steps
- screener builder steps
- alert creation steps

Keep workflow transitions explicit (reducer/state machine style),
not scattered in components.

### (D) Derived state
- P&L values
- allocation breakdown
- top performers
- computed trend summaries

Derived state must be computed from single source of truth (selectors / memoised computations)
to avoid mismatched numbers across pages.

### (E) Real-time state (if live market feed)
- WebSocket ticks keyed by symbol
- throttled updates
- UI reads from selectors (not direct event subscriptions)

**Why this is critical**
Trading UIs fail when:
- the same number appears differently on dashboard vs holdings
- a price updates but P&L doesn’t
- watchlist updates too frequently causing jitter

---

## 6) Step 6 — Dashboard Composition (why these widgets, why minimal)

I keep the dashboard minimal because it’s the “decision entry point”.
Overloaded dashboards create anxiety and reduce clarity.

### Dashboard widgets (example)
1. Portfolio Summary (value, day change, overall return)
2. Allocation (sector/asset breakdown)
3. Top Movers in Portfolio (best/worst performers)
4. Market Snapshot (indices + key movers)
5. Watchlist highlights (only top 5)
6. News relevant to portfolio/watchlist (context-linked)

**Decision principles**
- Prefer “what changed” over “everything”
- Show top-level clarity → allow drill-down
- Each widget loads independently (resilience)
- No single API should block the entire dashboard

### Loading and error UX
- skeletons per card
- partial failure allowed (news can fail without breaking portfolio)
- “last updated” timestamp for trust

---

## 7) Portfolio (holdings, transactions, performance analysis)

### Key UX decisions
- Separate “Holdings” and “Transactions” (different mental models)
- Provide a quick filter: gainers/losers, sector, asset class
- Keep a consistent number formatting strategy (currency, commas, decimals)

### Architecture decisions
- Holdings list uses virtualization if large
- Domain model ensures all P&L calculations come from one place
- “Performance analysis” uses derived state, not duplicated calculations

---

## 8) Watchlist (what I consider and why)

### Watchlist requirements
- multiple lists (e.g., Shortlist, Long-term, Earnings watch)
- reorder symbols
- quick view for last price, change %, volume, key indicator
- deep link to company/ticker page
- optional “alert” shortcut

### Decisions
- Watchlist is a “monitoring tool”; it must be fast and stable
- Store watchlist as domain model (`Watchlist`, `WatchlistItem`)
- Real-time price update uses same symbol store as dashboard
- Offline fallback: show last known price + timestamp

---

## 9) Screeners (rule builder and results)

### Why screeners need careful architecture
Screeners become complex fast:
- rule builder UI
- saved screener sets
- query persistence
- results rendering
- comparators and validation

### Approach
- Screener rules are domain objects: `ScreenerRule`, `Operator`, `Value`
- Builder is a workflow: add rule → validate → save → run
- Results are paginated and cached
- Rule validation lives in domain layer (not UI)

**Key UX decisions**
- validate rules inline (prevent “run with invalid rules”)
- show how many matches before loading full table (optional)
- save and reuse screeners (reduces repeated work)

---

## 10) News (portfolio-linked + ticker-linked)

### Why this matters
News is the biggest “context switching reducer” if designed well.

### Approach
- News feed keyed by:
  - portfolio symbols
  - watchlist symbols
  - manually searched ticker
- UI shows why it’s relevant (“from your holdings”, “from watchlist”)

Resilience:
- news can fail without impacting trading experience
- cached with timestamps

---

## 11) Alerts (price alerts + condition alerts)

### Alert types
- price above/below
- percentage change
- volume spike (optional)
- event-based alerts (earnings, macro events) if supported

### Architecture choices
- alert creation is a workflow (wizard style):
  select ticker → choose condition → confirm → saved
- backend is source of truth for evaluation
- UI emphasises clarity and confirmation to avoid mistakes

UX trust signals:
- show active/inactive status
- show last triggered time
- show which channel (email/push/in-app)

---

## 12) Best practices I enforce as frontend lead

### Code quality
- strict lint rules to prevent unsafe patterns
- consistent formatting
- review checklist focused on boundaries and state duplication

### Observability
- logging for:
  - auth failures
  - websocket disconnects
  - screener errors
  - alert save failures
- frontend telemetry for performance and crash tracking

### Accessibility
- keyboard navigation for critical actions
- readable contrast for numbers (gains/losses)
- stable layouts to avoid motion fatigue


