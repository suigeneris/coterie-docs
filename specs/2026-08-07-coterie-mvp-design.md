# Coterie — MVP Design Spec

**Date:** 2026-08-07
**Status:** Design approved, pending written-spec review
**Codename:** Coterie
**Author:** Jonathan Aguele (with Claude)

---

## 1. Concept & Goals

Coterie is a service for **co-owning luxury goods**. A small group (2–4 people) splits the cost of buying a specific item — a designer bag or a pair of luxury shoes — and each owner gets an equal share of both the asset and the time they can use it. Once funded, the item is purchased and held at a platform concierge hub; owners book their days on a shared calendar, and the item ships to whoever's turn it is.

The behavioural insight: people want **access** to luxury more than constant **possession** of it.

**v1 purpose — a real-money pilot at small scale.** The goal is to prove people will hand over real money for co-ownership. Real payments and real identity verification are in; physical logistics are run manually behind the scenes for a handful of real items. This is deliberately not a demo prototype and not a full scalable product — it is the smallest thing that answers "will people pay?"

**Group formation is a blend:** a creator can invite their own friends into a pool *and* let strangers fill the remaining slots. Because strangers can end up co-owning, trust must be *engineered* (verification, ratings, deposits, condition tracking) — this is not optional.

---

## 2. Scope

### In the MVP
1. **Identity verification** — real people only, via Stripe Identity.
2. **Open pool origination** — any verified user can create a co-ownership pool for a specific item.
3. **Buy a share with real money** — funds captured and held in escrow until the pool is fully funded, then the item is purchased.
4. **Ownership registry (cap table)** — who owns which equal share of which item.
5. **Shared calendar booking** + **per-use fee** on each booking.
6. **Ratings & reputation** — trust between strangers.
7. **Group chat** — per-pool communication.
8. **Logistics = manual** — the app shows item/custody status; the operator (you) runs the hub by hand.

### Deferred (post-MVP)
- **Share resale / exit marketplace** — pilot owners are locked in for the pilot term (communicated upfront). Biggest single scope-saver.
- Automated logistics / shipping automation.
- Depreciation-pricing / share-valuation engine.
- Stranger-matching engine (size-aware) beyond simple browse + self-select.

### Product constraints (locked)
- **Categories: bags and shoes only** for MVP.
- **Equal shares only** — every owner holds exactly one equal share; a pool is **2, 3, or 4 people**. No fractional or weighted ownership.
- **Creator is automatically owner #1** and pays their share on publish (skin in the game).
- **Size is load-bearing for shoes** — for wearables, "same size" is effectively a join requirement; the size badge must be impossible to miss. Bags are size-agnostic.
- **Time allocation:** equal entitlement (`term_days / owner_count`), booked on a shared calendar, **plus a per-use fee** per booking.
- **Exit reality:** shares are (post-MVP) tradeable but depreciate — a share is a stake in a *used* asset, so owners may not recover full value. Framed as "buy access, recover some value later," not an investment.

---

## 3. Architecture

The system is an **iOS client (SwiftUI)** + a **Supabase backend**, with **Stripe** as the money/identity system of record.

**Guiding principle: the app never moves money or mutates ownership directly.** Those operations go through server-side Edge Functions that enforce the rules, so a buggy or hostile client can never corrupt a cap table or an escrow balance.

```
iOS App (SwiftUI)
  Feature modules: Onboarding/Auth · Discover · Pool detail/fund ·
    Create pool · My Ownership · Calendar/Booking · Chat · Profile/Ratings ·
    Notifications/Settings
        │  Supabase SDK (auth, realtime, RLS-guarded reads)
        │  + Edge Function calls for anything money/ownership
        ▼
Supabase
  Postgres  → users, pools, memberships, bookings, ledger, ratings, messages, custody
  Auth      → sign-in, sessions (JWT)
  Realtime  → chat + live funding progress
  Storage   → item photos, ID docs, condition photos
  RLS       → row-level security on every read
  Edge Fns  → createPool, buyShare, bookItem, stripeWebhook (the money/rules)
        │  server-to-server
        ▼
Stripe
  Connect   → escrow, payouts (share payments, per-use fees, deposits)
  Identity  → verification
```

**Boundaries:**
- **Reads** (browse pools, view cap table, chat history) go app → Postgres, guarded by RLS.
- **Writes that touch money or ownership** go *only* through Edge Functions that validate rules and call Stripe. Never trust the client here.
- **Realtime** drives live chat and the live funding bar (shares filling).

---

## 3A. Frontend Architecture (iOS)

The internal architecture of the SwiftUI client. Chosen for testability, clean boundaries, and the portability goals in §10.

**Pattern: MVVM + Coordinators**
- **View (SwiftUI)** — dumb; renders state, forwards intent to its view model. No business logic, no networking.
- **ViewModel (`@Observable`, `@MainActor`)** — owns screen state and logic; depends on **repository protocols**, never on `SupabaseClient`. Fully unit-testable against mock repositories.
- **Coordinator** — owns navigation. Flow logic (onboarding → auth → verify → pools → booking) lives here, not in views. Routes modelled as enums over `NavigationStack` paths.

**Data layering (four tiers, mappers between them):**
- **Domain models** — the app's own types (`Pool`, `Profile`, `Booking`). View models and UI use *only* these. Backend-agnostic.
- **DTO (Data Transfer Object)** — `Decodable` types mirroring backend/wire shapes; what we **receive**. Live inside the repository implementation.
- **DTI (Data Transfer Input)** — `Encodable` request payloads; what we **send**. Also repository-internal.
- **Mappers** — pure functions translating DTO → domain and domain/input → DTI. The only place that knows both shapes.

```
NewPoolInput (domain) ─Mapper.dti→ CreatePoolDTI (Encodable) ─POST→ backend
Pool (domain) ←Mapper.domain─ PoolDTO (Decodable) ←row──────────── backend
        │
        ▼
   PoolViewModel (@Observable) ──▶ SwiftUI View
```

**Repository layer (the portability seam):**
- Each domain area exposes a protocol (`AuthRepository`, `ProfileRepository`, `PoolRepository`, …) returning **domain models**.
- Supabase-backed implementations (`SupabaseAuthRepository`, …) own the DTO/DTI + mapping and the Supabase SDK calls.
- Swapping to the custom backend (§10) or adding Android's shared contract changes only the implementation + DTO/DTI/mappers — domain models and view models are untouched.

**Dependency injection:** lightweight manual DI via an `AppContainer` (holds repository instances), passed through the SwiftUI environment. No DI framework.

**Concurrency:** `async/await` throughout; view models are `@MainActor`; repositories are `actor`- or `Sendable`-safe as needed.

**Testing:** Swift Testing (`@Test` / `#expect`). View-model unit tests run against mock repositories. No snapshot tests (per §11).

---

## 4. Data Model

Everything money/ownership-related is relational with hard invariants enforced server-side.

```
profiles                (1:1 with auth.users)
  id, display_name, avatar_url, bio
  verification_status   (unverified | pending | verified)
  role                  (member | admin)
  stripe_customer_id, stripe_connect_account_id
  rating_avg, rating_count, created_at

pools                   (a co-ownership pool for ONE specific item)
  id, creator_id → profiles
  category              (bag | shoes)
  item_name, brand, description, photos[]
  size                  (nullable; REQUIRED & prominent for shoes, hidden for bags)
  item_attributes       JSONB (color, material, condition, release year, ...)
  target_amount         (item price + fees)
  owner_count           (2 | 3 | 4)
  price_per_share       (= target_amount / owner_count)
  per_use_fee
  slots_filled          (0 → owner_count)
  status                (draft | funding | funded | active | closed | cancelled)
  funding_deadline, term_end_date, created_at

pool_memberships        (THE CAP TABLE — one equal share each)
  id, pool_id → pools, owner_id → profiles, joined_at
  UNIQUE(pool_id, owner_id)

share_purchases         (immutable log of each buy transaction)
  id, pool_id, buyer_id, amount
  stripe_payment_intent_id
  status                (escrowed | settled | refunded)
  created_at

ledger                  (append-only money record — source of truth for $)
  id, pool_id, user_id
  type                  (share_purchase | per_use_fee | deposit_hold | deposit_release | refund | payout)
  amount, currency, stripe_ref, status, created_at

bookings
  id, pool_id, owner_id
  start_date, end_date, days
  per_use_fee_amount, deposit_amount, stripe_payment_intent_id
  status                (confirmed | active | completed | cancelled)
  created_at

item_custody            (physical location / logistics state)
  id, pool_id
  state                 (at_hub | shipping_to_owner | with_owner | returning_to_hub)
  current_holder_id, condition_notes, condition_photos[], updated_at

ratings
  id, pool_id, booking_id, rater_id, ratee_id, score(1-5), comment, created_at

messages                (per-pool group chat)
  id, pool_id, sender_id, body, created_at
```

**Invariants — enforced server-side, never in the client:**
1. **No overselling** — a pool never exceeds `owner_count` members; checked transactionally inside `buyShare`.
2. **Verified-only actions** — creating a pool, buying a share, or booking requires `verification_status = verified`.
3. **Equal-time entitlement** — each owner's allowance = `term_days / owner_count`; `bookItem` rejects dates exceeding remaining allowance or colliding with an existing booking.
4. **Ledger is append-only** — money state is *derived* by summing ledger rows, never edited in place. This keeps escrow honest and doubles as an audit trail.

---

## 5. Escrow / Money Lifecycle

The part we cannot get wrong. A pool takes time to fill (days/weeks), so card authorizations (which expire in ~7 days) won't work. We use **capture-and-hold**: money is captured into the platform escrow balance when a member joins, tagged to the pool in the ledger, and either **settled** (pool fills → item bought) or **refunded** (deadline passes → everyone made whole).

**Pool funding state machine:**
```
draft ─publish→ funding ─last slot fills→ funded ─item procured→ active ─term ends→ closed
                   │
                   └─deadline passes, not full→ cancelled (refund all)
```

**Money movements:**

| Event | Money action | Ledger entry |
|---|---|---|
| Member joins (incl. creator) | Stripe captures `price_per_share` → held in platform escrow | `share_purchase` / `escrowed` |
| Final slot fills → **funded** | Funds committed to buying the item | (no move) |
| Item procured (manual, pilot) | Escrow settles to cover the purchase | `share_purchase` → `settled` |
| Deadline hits, not full | Every captured payment auto-refunded | `refund` per member |
| Owner books dates | `per_use_fee` captured; refundable **deposit hold** placed | `per_use_fee`, `deposit_hold` |
| Item returned clean | Deposit hold released | `deposit_release` |

**Why capture-on-join, not authorize:** a pool may take weeks to fill; auth holds die in one. Capturing real money means slots are backed by committed funds — no "funded on paper, declines on capture" surprises.

**Critical edge cases (server-side):**
- **Race on the last slot** — `buyShare` takes a transactional lock; first commits, second is rejected *before* Stripe captures (no charge).
- **Payment fails mid-join** — slot never granted, nothing held.
- **Refund on cancellation** — driven off the append-only ledger → idempotent and auditable; failed refunds retry and alert the operator.
- **Creator's money** — captured/escrowed like everyone else on publish.

---

## 6. User Flows & Screens

### Three hero journeys

**1. Join a pool (demand side — most common)**
```
Discover feed → Pool detail (item, size, price/share, slots 3/4, co-owners, deadline)
 → [Buy this share] → verify identity (if first time) → confirm payment
 → "You're in — pool funding complete!" → My Ownership
```

**2. Create a pool (supply side — origination)**
```
Create → category (bag/shoes) → item details (name, brand, photos, size if shoes, condition)
 → target price + owner_count (2/3/4) → review your share cost
 → verify identity → pay your share → live in Discover as "1/N filled"
```

**3. Use the item (post-funding — the payoff)**
```
My Ownership → item → Calendar (remaining allowance e.g. 68/91 days, others' bookings greyed)
 → select dates → per-use fee + deposit → confirm & pay
 → booking confirmed → item ships from hub → status card → after return → rate co-owners
```

### First-run education (pre-auth, shown once, skippable)
3-slide value-prop carousel:
- *"Own luxury together"* — split the cost of a bag or pair with 1–3 others.
- *"Book your time"* — everyone gets equal days; reserve on a shared calendar.
- *"Safe & verified"* — every owner ID-verified; items cleaned & insured at our hub.

Then a **Get started / I already have an account** fork.

### Screen inventory

| Module | Screens |
|---|---|
| Onboarding & Auth | First-run intro carousel, Sign up/in, Profile setup, ID verification (deferred to first money action) |
| Discover | Feed (pool cards), Filters (category, size, price) |
| Pool detail | Item + funding status, Co-owners, Buy-share sheet |
| Create pool | Multi-step origination wizard |
| My Ownership | Owned items list, Item detail (cap table + custody status) |
| Booking | Calendar, Booking confirm + fee/deposit sheet, My bookings |
| Chat | Per-pool thread list + conversation |
| Profile & ratings | My profile, Public profile, Rating sheet |
| System | Notifications, Settings |

### UX direction principles
- **Trust is the product** — every pool card leads with co-owner avatars + ratings and a clear funding bar.
- **Size impossible to miss** on shoe pools (big badge on the card).
- **The funding bar is the emotional core** — "3/4 filled, £6,000 committed," ticking up live via realtime.
- **Two-sided, one app** — creating and joining live side by side; no separate seller mode.
- **Browse before you verify** — the feed is the salesperson; verification appears exactly when the user taps "Buy this share."

*(Visual look-and-feel is out of scope for this spec — handled at build time via frontend-design.)*

---

## 7. Trust & Safety

Strangers co-owning a multi-thousand-pound item only works if trust is engineered. Four mechanisms, scaled to a pilot:

1. **Identity verification (the gate)** — Stripe Identity (government ID + selfie), triggered at first money action. `verification_status` drives a ✓ badge everywhere the name appears. No unverified user can hold a share or booking.
2. **Reputation** — after each handoff/booking, co-owners rate each other (1–5 + note). `rating_avg` on pool cards and profiles. Thin at first → seed trust with the verification badge and "new member" framing rather than a scary empty 0.
3. **Custody & condition tracking (manual for pilot)** — item lives at the hub between bookings; `item_custody` tracks state; **condition is photo-logged at every handoff** (operator in/out, owner on receipt) as the evidence trail. App shows status; operator drives transitions.
4. **Damage & disputes (lightweight MVP)** — per-use fee bundles cleaning + insurance; a **refundable security deposit hold** is placed at booking and released on clean return; dispute resolution is **manual (operator)** for the pilot — no automated arbitration.

**MVP simplifications:** simple star ratings (no weighting); no automated dispute engine; operator custody view can be a dead-simple filtered admin list.

---

## 8. Environments & Delivery

Enterprise-shaped from day one, even at pilot scale.

- **Three environments:** `dev`, `QA`, `prod` — each a separate Supabase project (isolated DB, auth, storage).
- **Stripe keys:** test mode in `dev` + `QA`; **live keys only in `prod`**.
- **Database migrations as code** — schema changes are versioned SQL migrations, applied through the pipeline, never hand-edited in a console. Portable to self-hosted Postgres later (see §10).
- **CI on every PR** — build, lint (SwiftLint), run tests, run Edge Function integration tests against Stripe test mode.
- **Secrets per environment** — no shared credentials; prod secrets restricted.
- **Promotion path:** dev → QA (sign-off) → prod.

---

## 9. Roles & Access (RBAC + Audit)

Foundation for the later admin surfaces (§12), designed in now because access control can't be bolted on cleanly.

- **Roles:** `role` on `profiles` — `member | admin`.
- **RLS policies** gate every table; admin-only tables/actions require `role = admin`.
- **Audit trail:** the append-only `ledger` and `item_custody` log serve as the immutable audit record of money and physical-custody events. No separate audit system needed for the pilot.
- Admin-only capabilities (consumed later by the dashboard): issue refunds, moderate/cancel pools, oversee verification, update custody state, adjudicate disputes.

---

## 10. Portability: Client Contract, Android & Backend Migration

One discipline, two payoffs (Android + custom-BE migration). Adopted now as binding principles; the builds themselves are later cycles (§12).

- **Client-agnostic API contract** — the app talks to the backend through a defined set of operations, not ad-hoc Supabase calls scattered through the UI.
- **Client-side repository layer** — UI depends on abstractions (e.g. `PoolRepository`), not `SupabaseClient` directly. Swap the implementation without touching the UI.
- **Postgres is Postgres** — Supabase is managed Postgres; schema + SQL migrations port to self-hosted Postgres with no rewrite. (This is a reason we avoided Firestore — money/ownership is relational.)
- **Business logic in portable functions** — Edge Functions written as standard TypeScript with minimal Supabase-proprietary coupling; they become the custom backend's endpoints.
- **Auth via JWT** — plan for JWT-based sessions so the token contract survives a backend swap. The auth migration is the hard part; flagged now, solved at migration time.
- **Documented migration triggers** — scale, cost, compliance needs, or infrastructure Supabase can't provide.

---

## 11. Error Handling & Testing

**Error handling — money paths get the most rigor. Fail loud and safe; never silently swallow.**

| Failure | Handling |
|---|---|
| Stripe webhook missed/duplicated | All money ops **idempotent**, keyed on Stripe event ID + ledger; replay never double-charges/refunds |
| Last-slot race | Transactional lock in `buyShare`; loser rejected before any charge |
| Payment declines mid-join | Slot never granted, nothing held, clear retry prompt |
| Refund fails on cancellation | Retried from append-only ledger, flagged to operator; owner sees "refund pending," not silence |
| Verification fails/pending | Money action blocked with a clear state, not a dead end |
| Realtime drops | App reconnects and refetches; funding bar reconciles from DB, never trusts stale socket state |
| Offline | Reads from cache where safe; **all writes require connectivity** — never queue a money action locally |

**Testing strategy:**
- **Edge Functions (highest priority)** — integration tests against **Stripe test mode** for every money path: join → escrow → fund → settle, and join → deadline → refund-all. Explicitly test the oversell race and webhook idempotency.
- **Invariants as tests** — "never oversell," "ledger sums to escrow balance," "refund is idempotent," "equal-time allowance never exceeded."
- **iOS** — unit tests on view models (funding math, entitlement/allowance calc, booking validation).
- **Pre-real-money gate** — full end-to-end run on Stripe test cards before any live transaction.
- **No snapshot tests** at this stage.
- **No load/perf testing** — pilot scale is tiny.

---

## 12. Roadmap & Decomposition

This spec covers the **consumer iOS app + backend**. The following are **sibling sub-projects**, each getting its own brainstorm → spec → plan cycle, all sharing this backend and its RBAC/audit foundation:

1. **Admin dashboard (web, mobile-friendly)** — *next after this.* Refunds, pool moderation, verification oversight, custody/logistics, disputes. Mobile-responsive web, built first.
2. **Admin app (native operator/hub)** — *phase 2.* On-the-go hub custody transitions, condition photos, handoffs.
3. **Android app** — post-pilot. Native client reusing the same API contract (enabled by §10).
4. **Supabase → custom backend migration** — post-pilot, when triggers hit (enabled by §10).

**Project management:** the implementation plan (next step, via writing-plans) will be decomposed into epics/stories for tracking.

---

## Open Questions / To Confirm
- None outstanding at design level. Pending: user review of this written spec, then implementation planning.
