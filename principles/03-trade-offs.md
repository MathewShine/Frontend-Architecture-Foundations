# 03 — Trade-offs (What I Choose, and Why)

Architecture is not about picking the “best” option.
It’s about making **explicit decisions under constraints**.

Every real product I’ve worked on — trading platforms, finance automation systems, lending workflows —
has involved trade-offs between speed, safety, complexity, and long-term maintainability.

This document captures the trade-offs I consciously make, why I default to certain choices,
and when I intentionally break my own rules.

---

## 1. Client State vs Server State

**The trade-off**  
- Client state gives control and speed  
- Server state gives truth and consistency  

**My default**
- UI-only concerns → client state  
- Anything fetched or derived from backend → server state (cached)

**Why**
In trading dashboards, things like:
- selected tab
- expanded rows
- temporary filters  

belong to client state.

But:
- portfolio snapshot
- market movers
- watchlist prices  

are server state and should be cached, refetched, and invalidated properly.

**When I break this**
For very high-frequency updates (e.g., streaming prices),
I keep a client-side representation backed by WebSocket events,
and periodically reconcile with server snapshots.

---

## 2. Global State vs Local State

**The trade-off**
- Global state reduces prop drilling  
- Local state reduces mental overhead  

**My default**
- Local state unless multiple screens depend on it
- Global state only for long-lived, shared concepts

**Examples**
Global:
- authenticated user
- active portfolio
- watchlist
- feature flags

Local:
- form inputs
- modal open/close
- inline validation errors

**Why**
In finance automation systems, overusing global state quickly turns
“small workflow changes” into cross-app side effects.

---

## 3. Redux  vs Server-State Libraries

**The trade-off**
- Global stores give control
- Server-state libraries give correctness and caching for free

**My default**
- Server-state tools for data fetching & caching
- Global state for UI/session/domain context

**Example**
In a trading dashboard:
- market movers list → server state
- selected symbol → global UI state
- order draft → local + domain validation

**Why**
Mixing fetching logic into global stores made refactors harder
and caching behaviour unpredictable over time.

---

## 4. DTOs Directly in UI vs Domain Mapping

**The trade-off**
- Using backend DTOs is fast
- Mapping to domain models is stable

**My default**
- Map DTOs at the data layer
- UI only consumes domain-friendly shapes

**Why**
In finance automation, backend APIs evolve:
- new status codes
- additional flags
- region-specific fields

Letting DTOs leak into UI meant UI changes every time backend evolved.

**When I don’t map**
Internal tools or throwaway admin screens with low longevity.

---

## 5. Frontend Validation vs Backend Enforcement

**The trade-off**
- Frontend validation improves UX
- Backend validation enforces correctness

**My default**
- Frontend domain handles *pre-validation*
- Backend remains final authority

**Examples**
Frontend:
- required fields
- tick size
- obvious invalid inputs

Backend:
- margin sufficiency
- compliance rules
- authorization checks

**Why**
Waiting for backend to tell the user something obvious
creates friction and unnecessary load.

---

## 6. Optimistic UI vs Confirmed UI

**The trade-off**
- Optimistic UI feels fast
- Confirmed UI feels safe

**My default**
- Optimistic UI for reversible actions
- Confirmed UI for irreversible actions

**Examples**
Optimistic:
- marking an invoice as “processing”
- updating watchlist

Confirmed:
- placing an order
- approving a payment
- submitting a loan decision

**Why**
In trading and finance, incorrect optimism damages trust faster than slowness.

---

## 7. Polling vs WebSocket

**The trade-off**
- Polling is simple and predictable
- WebSockets are fast but complex

**My default**
- Polling for low-frequency updates
- WebSockets for price streams and order status

**Why**
WebSockets require:
- reconnect logic
- backoff
- stale data reconciliation

I only use them where the value clearly outweighs the complexity.

---

## 8. Caching Aggressively vs Freshness

**The trade-off**
- Caching improves speed
- Freshness ensures correctness

**My default**
- Cache read-heavy, non-critical data
- Always label freshness in UI

**Examples**
Cache:
- last portfolio snapshot
- market movers
- historical analytics

Never cache:
- tokens
- irreversible actions
- sensitive PII

**Why**
In trading, showing *stale but labelled* data is often better
than showing nothing.

---

## 9. Reusable Components vs Feature-Owned Components

**The trade-off**
- Reusability reduces duplication
- Feature ownership reduces coupling

**My default**
- Shared components for primitives
- Feature-owned components for workflows

**Why**
In finance workflows, reuse at the wrong level
creates invisible dependencies across teams.

---

## 10. Strict Design System vs Shipping Speed

**The trade-off**
- Strict systems improve consistency
- Flexibility helps early delivery

**My default**
- Strict primitives
- Flexible composition

**Why**
In lending and finance automation, visual consistency builds trust.
But forcing every edge case into the system early slows teams down.

---

## 11. Error Recovery Explicit vs Implicit

**The trade-off**
- Implicit recovery is simpler
- Explicit recovery is safer

**My default**
- Explicit retry and recovery paths

**Examples**
- token refresh → retry once
- network failure → read-only fallback
- partial failures → explain clearly

**Why**
In high-stress products, users need clarity more than optimism.

---

## Closing Note

Trade-offs are not permanent rules.
They are **context-aware decisions**.

What matters is not choosing “correctly”,
but choosing **intentionally** and documenting why.

This document evolves as products, teams, and constraints evolve.
