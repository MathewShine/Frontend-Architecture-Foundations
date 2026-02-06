# 02 — Boundaries & Layering (So the App Doesn’t Rot)

Frontend apps don’t collapse because React is “slow” or because we didn’t pick the right library.  
They collapse because **boundaries weren’t enforced**, so every feature becomes a negotiation and every bug becomes archaeology.

In fintech/trading, boundaries matter even more:
- **Money flows are unforgiving**
- **Latency + correctness** both matter
- **Edge cases are the product**
- **Users are anxious**, and your UI needs recovery paths, not just happy paths

This document defines **how I layer code**, **where logic belongs**, and **how dependencies must flow**.

---

## Goal

Build a system where:
- UI can change without breaking business rules
- API shapes can change without rewriting the entire app
- Features remain testable without rendering the whole app
- Trading-grade flows are resilient: error states, retry states, offline/slow network states, stale cache states

---

## The Layer Model (My Default)

Think of the app as **four layers**.

### 1) UI Layer (Presentation + Interaction)
**What it is:** Components, styling, layout, interactions  
**What it does:** Render state, collect user intent, show feedback  
**What it must NOT do:** Business rules, validation that defines correctness, API mapping

Examples (trading):
- Price panel, order form, watchlist table, portfolio cards
- Showing “Order rejected” / “Partial fill” states
- Input formatting (₹, $, decimals, commas)

✅ Allowed:
- Input masking, formatting, visual validation (“this field is empty”)
- Triggering a command: `placeOrder()`

❌ Not allowed:
- “Is this order allowed?” logic
- “How to calculate margin / risk / exposure” logic

---

### 2) Domain Layer (Rules + Meaning)
**What it is:** The brain of the product  
**What it does:** Defines business intent and invariants  
**What it must NOT do:** React, UI, network calls, browser storage

Examples (trading):
- Order rules: min quantity, tick size, allowed order types
- Risk checks: max loss, exposure, leverage constraints
- State machine: Draft → Confirm → Submit → Pending → Filled/Rejected

✅ Allowed:
- Domain models and pure functions (no side effects)
- Deterministic transformations: “Given X, return Y”
- Domain validation: “This is invalid because…”

❌ Not allowed:
- `fetch()`, localStorage, sockets
- reading DOM, rendering components

---

### 3) Application Layer (Use Cases / Orchestration)
**What it is:** The coordinator  
**What it does:** Orchestrates domain + data, handles flows, manages side effects boundaries  
**What it must NOT do:** Heavy UI rendering, low-level network details

Examples (trading):
- “Place Order” flow:
  - validate order
  - check cache/stale quote policy
  - call API
  - update optimistic state
  - handle retries and reconciliation
- “Load Portfolio Overview” flow:
  - return cached data immediately
  - refresh in background if online
  - reconcile and show “Updated just now”

✅ Allowed:
- Use-case functions: `placeOrderUseCase`, `refreshQuotesUseCase`
- Managing request state: loading/success/error
- Retry rules, fallback decisions

❌ Not allowed:
- UI layout code
- “how to call axios” implementation details inside use cases

---

### 4) Data/Infrastructure Layer (IO: API, Cache, WebSocket, Storage)
**What it is:** The plumbing  
**What it does:** Fetching, caching, persistence, sockets, logging  
**What it must NOT do:** Product rules or UI decisions

Examples (trading):
- WebSocket quote streaming
- REST calls for order placement / order status
- Cache: memory → localStorage/IndexedDB fallback
- Token refresh interceptors (access token renewal)

✅ Allowed:
- API clients, socket clients, cache adapters
- DTO mapping (server ↔ domain)
- Storage drivers

❌ Not allowed:
- Risk calculations
- UI formatting decisions

---

## Dependency Rules (Non-Negotiable)

The most important rule:
> **Dependencies flow inward. UI depends on Application. Application depends on Domain. Data depends on nothing in UI.**

### Allowed direction
- UI → Application → Domain
- Application → Data (via interfaces/adapters)
- Data → Domain (only for mapping types, not rules)

### Forbidden direction
- Domain → UI (never)
- Domain → Data (never)
- UI → Data directly for business flows (avoid; only allowed for trivial reads like “theme preference”)

If this rule breaks, the app becomes a spiderweb and refactors become a risk.

---

## Boundaries in Practice (What Goes Where)

### UI Layer owns:
- Components
- rendering states
- skeletons/loading UI
- error boundaries and error views (visual)
- user intent: clicks, input, gestures

### Domain Layer owns:
- invariants and correctness
- trading rules, risk rules
- calculations that must be consistent everywhere
- state machines (order lifecycle, session lifecycle)

### Application Layer owns:
- use cases (place order, cancel order, refresh portfolio)
- orchestration of retries and reconciliation
- optimistic updates (with rollback rules)
- cache policies (freshness windows)

### Data Layer owns:
- api clients, websocket streams
- DTO mapping and normalization
- persistence
- caching adapters
- logging/telemetry

---

## Boundary Example: “Place Order” (Fintech Reality)

### UI does:
- Collect inputs (symbol, qty, price, order type)
- Show confirmation modal
- Trigger use case: `placeOrderUseCase(draftOrder)`

### Domain does:
- Validate order invariants:
  - tick size
  - min qty
  - price bounds
  - allowed order types per instrument
- Produce either:
  - `ValidOrder` or `DomainError[]`

### Application does:
- Apply flow:
  - if invalid → return errors for UI to render
  - if valid → call API, optimistic state update
  - handle 401 → token refresh → retry once
  - reconcile final order status from websocket/polling

### Data does:
- `POST /orders`
- subscribe websocket channel for order updates
- map backend response → domain format

---

## Boundary Example: Quotes Streaming (WebSocket)

UI should never decide “how quotes are streamed.”  
UI should only consume:
- `quotesStore` / `useQuotes(symbols)`  
and render states:
- live / delayed / stale / disconnected

Application/Data decide:
- subscription batching
- reconnection backoff
- stale quote policy (e.g., show stale but label it)

---

## Layering for Confirmations (High-Stress UX)

Fintech confirmations are not “nice-to-have”.
They are the difference between trust and churn.

### Confirmation belongs to UI + Application
- UI: confirmation modal, risk warning UI, summary view
- Application: determines if confirmation is required:
  - high quantity
  - high leverage
  - market volatility flag
  - user’s last action pattern
- Domain: defines the rule (what qualifies as risky)

---

## Shared Types vs Layer Types

Avoid dumping everything into a single `types.ts`.

### Domain Types (stable)
- Order, Quote, Position, Portfolio
- Domain errors: InvalidTickSize, InsufficientMargin, StaleQuote

### DTO Types (unstable)
- API response models (changes often)
- should stay in data layer

Rule:
> Domain models are stable. DTOs are disposable.

---

## Where Layering Usually Fails (Common Violations)

- API response shape used directly in UI components
- business validation written in UI forms
- websocket handlers updating UI state directly
- “helpers” folder with random logic (no owner layer)
- “services” folder that becomes a trash bin

If I see these, I refactor **before** adding more features.

---

## Folder Mapping (Repo Structure)

This repo separates concerns intentionally.

- `principles/` → rules + philosophy
- `playbooks/` → how to implement consistently
- `case-studies/` → proof in real product scenarios
- `decision-log/` → why I made key architecture calls
- `diagrams/` → mental models + flows + boundaries
- `templates/` → reusable docs and patterns

---

## Personal Rules I Follow (Hard Checks)

1. If a component contains business rules, it’s leaking domain.
2. If a hook directly calls the API for a product-critical flow, it’s leaking app orchestration.
3. If DTO fields show up in UI, I’m coupling to backend changes.
4. If I can’t test the domain without rendering React, boundaries are broken.
5. If retry/recovery isn’t explicit, fintech UX becomes fragile.

