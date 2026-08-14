# Coterie Plan 2 — Identity Verification (Stripe Identity) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Status: DRAFT — awaiting approval.** Nothing in this plan has been executed. Per the Plan 1
> post-mortem, every environment- and SDK-dependent code block below should be treated as a
> *prediction*, not a fact, and replaced with the file on disk once it has actually run.

**Goal:** a user can tap "Verify identity", complete a Stripe Identity check, and have
`profiles.verification_status` move `unverified → pending → verified` — written **only** by a
service-role backend path, never by the client. The pending screen learns about the outcome
without the user restarting the app, and a rejected user is told why and can retry.

**Architecture:** unchanged from Plan 1 — SwiftUI MVVM + coordinators, view models depend on
repository *protocols*, DTO/DTI + mappers inside repository implementations, manual DI via
`AppContainer`. New: two Supabase Edge Functions (Deno/TypeScript) as the only components that
hold a Stripe secret key, and one `security definer` SQL function as the only writer of
`profiles.verification_status`.

**Tech Stack:** Swift 6 / SwiftUI (iOS 18), XcodeGen, supabase-swift 2.54.1, StripeIdentity iOS
SDK (26.x), Supabase CLI 2.112.0 (local Docker), Deno 2 Edge Runtime, `npm:stripe@^22`,
`npm:@supabase/server@^1`, Swift Testing, SwiftLint, pgTAP.

---

## Global Constraints

Plan 1's constraints all still apply. These are the additions and the ones that bite hardest here.

- **`profiles.verification_status` stays outside the client GRANT, permanently.** Migration
  `0001` grants `update (display_name, bio, avatar_url)` and nothing else. That column list is
  the entire security model of this feature. No task below widens it, and no new table gives
  `authenticated` an `insert` or `update` grant.
- **The only writer of `verification_status` is a `security definer` SQL function**, owned by
  `postgres`, with `execute` revoked from `public`/`anon`/`authenticated` and granted only to
  `service_role`. The Edge Function calls it by RPC with the service-role key. A direct
  service-role `update` on `profiles` from TypeScript would work too, but would scatter the
  transition rules across a runtime we cannot test with pgTAP, and would not port off Supabase
  (spec §10).
- **pgTAP runs as `postgres` and bypasses RLS *and* GRANTs *and* `security definer` checks.**
  A test that calls `apply_verification_outcome()` as superuser proves nothing about
  authorisation. Every authorisation assertion must either use
  `has_function_privilege` / `has_table_privilege` / `has_column_privilege` against the
  `authenticated` role, or `set local role authenticated` and assert the call **throws**. This
  caught two real security bugs in Plan 1; assume it will again.
- **No PII in our database.** We store the Stripe session id, our user id, a status, and an
  error code/reason. We never store `verified_outputs` — not names, not dates of birth, not
  document numbers, not images. Reading those requires a *restricted* API key (`rk_`) rather
  than the secret key, and only within 48 hours; we deliberately never build that path.
- **Webhook handling is idempotent and order-independent.** Stripe does not guarantee delivery
  order and retries for up to 3 days in live mode. Dedupe on `event.id`; guard the state
  machine so `verified` can never be downgraded by a late `processing` event.
- **No Stripe secret ever enters the repo.** `sk_test_…`, `sk_live_…` and `whsec_…` live in
  `backend/supabase/functions/.env` (gitignored) locally and in `supabase secrets` for hosted
  environments. `backend/.gitignore` already ignores `.env`; confirm it covers the nested path
  before the first run.
- **The client never sees a Stripe secret key.** It receives a VerificationSession id plus a
  single-use ephemeral key secret that expires in one hour.
- **The claim "Verification is securely handled by Stripe" must be gated on the integration
  actually being live** (Task 9). Until then it is a false statement about where someone's
  passport goes.
- **Commits:** conventional-commit messages; **no Claude co-author trailers.** Backend and iOS
  are separate repos — land the migration before the client change that depends on it.

---

## Key Decisions

### D1 — Use the native `StripeIdentity` iOS SDK, not the hosted redirect URL

Both are fully supported by the same VerificationSession API, and **the checks are identical** —
selfie/liveness (`options[document][require_matching_selfie]`, `require_live_capture`), allowed
document types and ID-number checks are all *server-side session options*, not SDK features. So
this is purely a question of how the user gets in and out of the flow.

The hosted flow loses on the return path, and we have nothing to return to:

- The session `url` is a web page. Getting the user back into the app cleanly needs
  `return_url`, and every example Stripe publishes is an `https://` page on your own domain.
  **Stripe's API reference documents `return_url` in one sentence and states no constraint at
  all** — it neither permits nor forbids a custom scheme. Coterie has no domain, no website, and
  no associated-domains entitlement (confirmed: no `applinks`, `CFBundleURLTypes` or
  `associatedDomains` anywhere in `ios/`). So a clean return would require standing up a web
  property first.
- Without `return_url` the user finishes on Stripe's own confirmation page inside an in-app
  browser and has to tap Done. Survivable, but it makes the completion signal unobservable to
  the app.
- **Unresolved risk:** whether Stripe's hosted capture works inside `SFSafariViewController` at
  all. Live document capture uses the camera; in-app browser camera support is a long-standing
  WebKit grey area and **neither Stripe's docs nor Apple's state a guarantee either way**. If it
  does not work, the only remaining hosted option is punting to external Safari — leaving the app
  entirely, with no way back.

The native SDK sidesteps all of it: `IdentityVerificationSheet.present(from:completion:)` returns
`.flowCompleted` / `.flowCanceled` / `.flowFailed(error)` in-process, so the app knows the moment
the user has submitted and can show the pending state immediately.

**Costs we are accepting**, stated plainly:
- Four new binary modules (`StripeIdentity`, `StripeCore`, `StripeCameraCore`, `StripeUICore`)
  via `https://github.com/stripe/stripe-ios-spm`. **Stripe publishes no per-module app-size
  figure** and the 74 MB `Stripe.xcframework.zip` release asset is all modules × all slices, so
  it is not a proxy. Measure it with an Xcode app size report in Task 7 and record the number.
- `NSCameraUsageDescription` in Info.plist, and an App Privacy disclosure in App Store Connect.
- The SDK needs an **ephemeral key**, which must be created against a **pinned Stripe API
  version matching the SDK**. Bumping `StripeIdentity` may require bumping that string.
- Stripe's licence forbids modifying the SDK when used with Identity.
- Minimum iOS: **15.0** per `StripeIdentity/README.md`. (The docs page says 13.0 — that is
  boilerplate for the whole stripe-ios repo. We target 18.0 regardless.)

**This decision is client-side only.** The `create-verification-session` Edge Function returns
`{ session_id, ephemeral_key_secret, url }` — all three. Switching to hosted later is a change to
one Swift file and nothing else. If the Task 7 size report is unacceptable, that is the exit.

### D2 — A `verification_sessions` table, not just a status column

Spec §9 calls for RBAC + an audit trail, and names the append-only `ledger` and `item_custody`
log as the audit record for *money and custody*. Identity has no such log, and a bare enum on
`profiles` answers none of the questions an operator will actually ask: when did they submit, how
many times, why were they rejected, which Stripe session was it. It also gives us nowhere to put
the Stripe session id — which we need in order to *reuse* a session on retry, as Stripe
explicitly recommends.

So: one row per Stripe VerificationSession, owned by the backend, readable by its owner, writable
by nobody except the `security definer` function. Plus a separate `stripe_webhook_events` table
whose primary key is the Stripe event id — that table *is* the idempotency mechanism (D4).

### D3 — Status mapping, and the two states we cannot represent

| Stripe session `status` | Event | `profiles.verification_status` | Notes |
|---|---|---|---|
| — (session created) | `identity.verification_session.created` | `unverified` | no user-visible change |
| `requires_input` (never submitted) | — | `unverified` | the normal starting state |
| `processing` | `identity.verification_session.processing` | **`pending`** | the only writer of `pending` |
| `verified` | `identity.verification_session.verified` | **`verified`** | terminal, never downgraded |
| `requires_input` (after a failed check) | `identity.verification_session.requires_input` | **`unverified`** | + `last_error_code` / `last_error_reason` on the session row |
| `canceled` | `identity.verification_session.canceled` | `unverified` | also fired on redaction |
| — | `identity.verification_session.redacted` | unchanged | must be subscribed to *explicitly*; "subscribe to all events" does not include it |

**The gap:** our three-value enum cannot distinguish "never tried" from "tried and was rejected".
Both are `unverified`.

**We are not adding a fourth enum value.** The spec fixes three states, `verification_status` is a
*trust gate* (may this person hold a share — yes/no/deciding) and a rejection does not create a
fourth answer to that question. Instead the *reason* lives on the session row, and the UI reads
it: `.unverified` with a non-nil rejection reason renders "We couldn't verify that document —
<reason>" and a "Try again" button, rather than the first-run copy.

Note the naming collision to avoid: `VerificationViewModel.State.failed(String)` already exists
and means *"we could not load your status"*, i.e. a network failure. Do not overload it with
"Stripe rejected you". Task 8 changes `.unverified` to carry the reason instead.

`last_error.reason` is documented as a user-displayable string, so it can be shown directly — but
`consent_declined` and `under_supported_age` deserve bespoke copy (see Task 8).

### D4 — Idempotency: the event id is a primary key

The webhook function's **first** database write is
`insert into stripe_webhook_events (event_id, …) on conflict do nothing`, checking the affected
row count. Zero rows means we have already processed this event; return `200` immediately and do
nothing else. This is stronger than an application-level "have I seen this?" check because the
uniqueness is enforced by Postgres, and it survives two Edge Function instances racing on a
retry.

Out-of-order delivery is handled separately, in the SQL function: the transition is monotonic.
`verified` is terminal — a late `processing` or `requires_input` event updates the session row's
audit fields but **cannot** move the profile back off `verified`. Stripe's own docs warn that
ordering is not guaranteed, and this exact case (`verified` arriving before `processing`) is the
one that would silently un-verify a paying user.

### D5 — Pending-screen refresh: poll on appear + foreground, not Realtime, not push

- **Push is out of scope** — it is an undecided project-wide question (no FCM equivalent in
  Supabase, needs a `device_tokens` table) and blocking identity verification on it is wrong.
- **Realtime is deferred.** A `postgres_changes` subscription on the user's own `profiles` row
  would work, but it is a new moving part (publication config, RLS-through-realtime, reconnect
  handling) for a screen where Stripe says most checks finish in **under a minute**. Realtime
  earns its keep on the funding bar in a later plan, where the data genuinely streams.
- **What we build:** `VerificationViewModel.load()` on appear, plus a refetch when the app
  returns to the foreground (`ScenePhase`), plus — only while `state == .pending` — a bounded
  poll: every 5 seconds for 2 minutes, then stop and show a "Still checking — we'll update this
  when it's done" affordance with a manual refresh. Bounded so a backgrounded-forever screen
  cannot hammer PostgREST.
- **Belt and braces for a missed webhook** (Task 5, step 6): when the app foregrounds onto a
  `pending` state whose session row is older than ~5 minutes, it calls a
  `sync-verification-status` Edge Function that *retrieves* the session from Stripe and applies
  the same SQL function. Stripe is the source of truth; this makes a dropped webhook
  self-healing without a cron job.

### D6 — Edge Function shape: the CLI's own scaffold

`supabase functions new` on CLI 2.112.0 emits the **new** `@supabase/server` pattern — verified
by running it in a throwaway directory:

```ts
import "@supabase/functions-js/edge-runtime.d.ts";
import { withSupabase } from "@supabase/server";

export default {
  fetch: withSupabase({ auth: ["publishable", "secret"] }, async (req, ctx) => { … }),
};
```

with a per-function `deno.json` import map and — importantly — the CLI writes
`[functions.<name>] verify_jwt = false` into `config.toml` by default, moving the auth decision
into the handler's `auth:` mode. `ctx.supabase` is RLS-scoped to the caller; `ctx.supabaseAdmin`
bypasses RLS. `@supabase/server` is officially **public beta**; the older
`Deno.serve` + `createClient` + `auth.getUser()` pattern still works and is not deprecated, but
it is no longer what any current doc or the CLI shows. **We follow the CLI scaffold**, pinned to
`npm:@supabase/server@^1`, and record the beta status as a risk.

We use `npm:` specifiers throughout (Supabase's current guidance explicitly says to avoid
`esm.sh` / `deno.land/x` / `unpkg`) and pin every version.

---

## File Structure

```
coterie/
  docs/
    plans/2026-08-13-plan-2-identity-verification.md   # this file

  backend/
    Brewfile                                    # + stripe CLI            (Task 0)
    Makefile                                    # + fn-serve, fn-test     (Task 2)
    .gitignore                                  # confirm nested .env     (Task 2)
    supabase/
      config.toml                               # + [functions.*]         (Task 2)
      migrations/
        0002_verification.sql                                             (Task 1)
      tests/
        0002_verification_test.sql              # pgTAP                   (Task 1)
      functions/
        .env                                    # GITIGNORED, never committed
        _shared/
          verification.ts                       # status mapping, pure    (Task 4)
          verification_test.ts                  # Deno test, offline      (Task 4)
        create-verification-session/
          deno.json  index.ts                                             (Task 3)
        stripe-identity-webhook/
          deno.json  index.ts  index_test.ts                              (Task 4)
        sync-verification-status/
          deno.json  index.ts                                             (Task 5)

  ios/
    project.yml                                 # + stripe-ios-spm        (Task 7)
    Config/Debug.xcconfig  Release.xcconfig     # + IDENTITY_ENABLED      (Task 9)
    Coterie/
      Core/Config/AppEnvironment.swift          # + isIdentityEnabled     (Task 9)
      Core/Models/VerificationSession.swift     # domain                  (Task 6)
      Features/Verification/
        VerificationRepository.swift            # protocol                (Task 6)
        VerificationDTO.swift                   # DTO/DTI + mapper        (Task 6)
        SupabaseVerificationRepository.swift    # impl + mapped(_:)       (Task 6)
        IdentityVerificationPresenter.swift     # UIKit bridge            (Task 7)
        VerificationViewModel.swift             # MODIFIED                (Task 8)
        VerificationIntroView.swift             # MODIFIED                (Task 8, 9)
      App/
        RootView.swift                          # MODIFIED — onVerify     (Task 7)
        AppContainer.swift                      # MODIFIED                (Task 7)
      Features/Shell/MainTabView.swift          # MODIFIED — onVerify     (Task 7)
      Resources/Assets.xcassets/
        StripeWordmark.imageset/                # official asset          (Task 9)
    CoterieTests/
      Mocks/MockVerificationRepository.swift                              (Task 6)
      VerificationMapperTests.swift                                       (Task 6)
      VerificationErrorMappingTests.swift                                 (Task 6)
      VerificationViewModelTests.swift          # MODIFIED                (Task 8)
```

---

## Task 0: Manual prerequisites — Jonathan only

**Nobody can automate any of this, and Tasks 3, 5, 7 and 10 are blocked until it is done.**
Tasks 1, 2, 4 and 6 are not — start those in parallel.

**Do not paste any key into a file inside a repo, into a commit message, or into a chat.**

- [ ] **Step 1: Create the Stripe account**

Go to <https://dashboard.stripe.com/register>. Set the **business location to United Kingdom** —
GB is one of only three countries where Stripe Identity is *generally available* (with JP and US);
the other 31 supported countries are self-serve public beta.

- [ ] **Step 2: Complete business verification**

<https://dashboard.stripe.com/account/onboarding>. Stripe lists this as step 1 of "Before you
begin" on every Identity guide. Expect to supply business details and ID. This can take days —
**do it first**.

- [ ] **Step 3: Submit the Stripe Identity application**

<https://dashboard.stripe.com/identity/application>. This is a separate approval from the account
itself. Describe the use case honestly: *consumer marketplace for co-ownership of physical goods;
identity verification is used to gate money movement and co-ownership between strangers.*

Check the use-case restrictions at <https://docs.stripe.com/identity/use-cases> before applying.
Two are directly relevant to Coterie:
- **Stripe will not verify anyone under 16.** Coterie's terms must therefore set a minimum age.
  Rejections arrive as `last_error.code = under_supported_age`.
- Identity may not be resold or used for FCRA "permissible purposes". Neither applies to us.

- [ ] **Step 4: Enable the mobile SDK**

<https://dashboard.stripe.com/settings/identity> → click **Enable**. Verbatim from the iOS guide:
*"To get access to the Identity iOS SDK, visit the Identity Settings page and click Enable."*
Without this, Task 7 cannot work no matter how correct the code is.

- [ ] **Step 5: Copy the test secret key**

<https://dashboard.stripe.com/test/apikeys> → reveal the **secret** key, `sk_test_…`.
Use the **test** key only. Do not create or reveal a live key in this plan.

- [ ] **Step 6: Install and log in to the Stripe CLI**

```bash
brew install stripe/stripe-cli/stripe
stripe login
```
(Task 2 adds this to `backend/Brewfile` so a fresh clone gets it.)

- [ ] **Step 7: Note the pricing and the free allowance**

From the UK pricing page: **£1.25 per document verification (including selfie)**, 35p per ID-number
lookup (US SSN only), and **the first 50 verifications are free**. Test-mode verifications do not
process checks and are not charged. 50 free is comfortably enough for the pilot's development.

- [ ] **Step 8: Download the official Stripe wordmark** (can be done now, no account needed)

See Task 9 — the asset is a hard requirement and Stripe's terms forbid redrawing it.

- [ ] **Step 9: Record what happened**

Append to this file, under Task 0: the date the account was created, the date Identity was
approved (or rejected, and why), and whether test-mode API calls worked *before* approval —
**Stripe's docs never state the pre-approval failure mode, so this is genuinely unknown until
tried.** If they do work, Tasks 3 and 5 unblock earlier than assumed.

---

## Task 1: Migration `0002_verification.sql` — data model and the privileged writer ✅ DONE

**Repo:** `backend`. **Not blocked by Task 0.**

**Files:**
- Create: `backend/supabase/migrations/0002_verification.sql`
- Create: `backend/supabase/tests/0002_verification_test.sql`

**Interfaces:**
- Produces: `public.verification_sessions`, `public.stripe_webhook_events`,
  `public.apply_verification_outcome(...)`.

- [ ] **Step 1: Write the migration**

Sketch — verify column types and the `on conflict` semantics by running it:

```sql
create table public.verification_sessions (
    id uuid primary key default gen_random_uuid(),
    user_id uuid not null references public.profiles(id) on delete cascade,
    stripe_session_id text not null unique,
    status text not null default 'requires_input',
    last_error_code text,
    last_error_reason text,
    submitted_at timestamptz,
    completed_at timestamptz,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now()
);

create index verification_sessions_user_id_idx on public.verification_sessions (user_id);

alter table public.verification_sessions enable row level security;

-- select only. No insert, no update, no delete, for anyone but service_role.
grant select on public.verification_sessions to authenticated;

create policy "users read their own verification sessions"
    on public.verification_sessions for select
    to authenticated
    using (auth.uid() = user_id);

-- The idempotency ledger. Not exposed to the client at all.
create table public.stripe_webhook_events (
    event_id text primary key,
    event_type text not null,
    stripe_session_id text,
    received_at timestamptz not null default now()
);

alter table public.stripe_webhook_events enable row level security;
-- deliberately no grants: service_role only.

create function public.apply_verification_outcome(
    p_stripe_session_id text,
    p_user_id uuid,
    p_stripe_status text,
    p_error_code text default null,
    p_error_reason text default null
)
returns public.verification_sessions
language plpgsql
security definer set search_path = public
as $$
declare
    v_session public.verification_sessions;
    v_profile_status verification_status;
begin
    insert into public.verification_sessions
        (user_id, stripe_session_id, status, last_error_code, last_error_reason)
    values
        (p_user_id, p_stripe_session_id, p_stripe_status, p_error_code, p_error_reason)
    on conflict (stripe_session_id) do update set
        status            = excluded.status,
        last_error_code   = excluded.last_error_code,
        last_error_reason = excluded.last_error_reason,
        submitted_at      = coalesce(public.verification_sessions.submitted_at,
                                     case when excluded.status = 'processing' then now() end),
        completed_at      = coalesce(public.verification_sessions.completed_at,
                                     case when excluded.status = 'verified' then now() end),
        updated_at        = now()
    returning * into v_session;

    v_profile_status := case p_stripe_status
        when 'processing' then 'pending'::verification_status
        when 'verified'   then 'verified'::verification_status
        else 'unverified'::verification_status
    end;

    -- Monotonic: `verified` is terminal. Stripe does not guarantee event order, so a late
    -- `processing` must never un-verify someone who is already through.
    update public.profiles
       set verification_status = v_profile_status
     where id = p_user_id
       and verification_status is distinct from 'verified'::verification_status;

    return v_session;
end;
$$;

revoke all on function public.apply_verification_outcome(text, uuid, text, text, text) from public;
revoke all on function public.apply_verification_outcome(text, uuid, text, text, text) from anon;
revoke all on function public.apply_verification_outcome(text, uuid, text, text, text) from authenticated;
grant execute on function public.apply_verification_outcome(text, uuid, text, text, text) to service_role;
```

**Both checked 2026-08-13.** A newly created function grants `execute` to `PUBLIC` by default
and `authenticated` and `anon` inherit it — so the revokes are load-bearing, not defensive.
Removing them fails the pgTAP suite (tests 8 and 9). Original notes:
1. Whether `revoke … from public` is enough on its own in this Postgres version, or whether the
   per-role revokes are redundant. Keep them either way — they are what the pgTAP assertions read.
2. Whether `default privileges` in this Supabase version already grant `execute` on new functions
   to `authenticated`. **If they do and the revoke is omitted, any signed-in user can verify
   themselves with one RPC call.** This is the single highest-severity failure mode in the plan.

- [ ] **Step 2: Write the pgTAP test**

The assertions that matter are the negative ones, and they must be written against the
`authenticated` role — **pgTAP runs as `postgres`, a superuser, so calling the function
successfully in a test proves nothing.**

```sql
begin;
select plan(16);

select has_table('public', 'verification_sessions', 'verification_sessions exists');
select has_table('public', 'stripe_webhook_events', 'stripe_webhook_events exists');
select col_is_pk('public', 'stripe_webhook_events', 'event_id', 'event id is the primary key');
select has_function('public', 'apply_verification_outcome', 'the privileged writer exists');

select is((select relrowsecurity from pg_class where oid = 'public.verification_sessions'::regclass),
          true, 'RLS is enabled on verification_sessions');
select is((select relrowsecurity from pg_class where oid = 'public.stripe_webhook_events'::regclass),
          true, 'RLS is enabled on stripe_webhook_events');

-- The load-bearing negatives.
select ok(not has_function_privilege('authenticated',
          'public.apply_verification_outcome(text, uuid, text, text, text)', 'EXECUTE'),
          'authenticated CANNOT execute the privileged writer');
select ok(not has_function_privilege('anon',
          'public.apply_verification_outcome(text, uuid, text, text, text)', 'EXECUTE'),
          'anon CANNOT execute the privileged writer');
select ok(has_function_privilege('service_role',
          'public.apply_verification_outcome(text, uuid, text, text, text)', 'EXECUTE'),
          'service_role CAN execute the privileged writer');

select ok(has_table_privilege('authenticated', 'public.verification_sessions', 'SELECT'),
          'authenticated can read verification sessions');
select ok(not has_table_privilege('authenticated', 'public.verification_sessions', 'INSERT'),
          'authenticated cannot insert verification sessions');
select ok(not has_table_privilege('authenticated', 'public.verification_sessions', 'UPDATE'),
          'authenticated cannot update verification sessions');
select ok(not has_table_privilege('authenticated', 'public.stripe_webhook_events', 'SELECT'),
          'authenticated cannot even read the webhook ledger');

-- Still true after this migration: the Plan 1 guarantee.
select ok(not has_column_privilege('authenticated', 'public.profiles', 'verification_status', 'UPDATE'),
          'authenticated still cannot update verification_status');

-- Behaviour: verified is terminal.
insert into auth.users (id, email) values ('00000000-0000-0000-0000-0000000000a1', 'v@b.com');
select public.apply_verification_outcome('vs_test_1', '00000000-0000-0000-0000-0000000000a1', 'verified');
select public.apply_verification_outcome('vs_test_1', '00000000-0000-0000-0000-0000000000a1', 'processing');
select is((select verification_status::text from public.profiles
           where id = '00000000-0000-0000-0000-0000000000a1'),
          'verified',
          'a late processing event does not downgrade a verified profile');

select is((select count(*)::int from public.verification_sessions where stripe_session_id = 'vs_test_1'),
          1,
          'reapplying the same stripe session updates rather than duplicating');

select finish();
rollback;
```

- [ ] **Step 3: Mutation-check the negatives**

Non-negotiable, and the reason Plan 1's security bugs were found. Temporarily add
`grant execute on function public.apply_verification_outcome(...) to authenticated;` to the
migration, run `make db-reset && make db-test`, and confirm the privilege assertion **fails**.
Then remove it. Repeat for the `verified`-is-terminal test by deleting the
`and verification_status is distinct from 'verified'` clause. If either mutation leaves the
suite green, the test is decorative — rewrite it.

- [ ] **Step 4: Run and commit**

```bash
make db-reset && make db-test
git add -A && git commit -m "feat(db): verification sessions, webhook ledger and privileged writer"
```

---

## Task 2: Edge Function plumbing — config, secrets, Makefile, and proving caller auth ✅ DONE

**Repo:** `backend`. **Not blocked by Task 0** (uses a fake key).

**Files:**
- Modify: `backend/supabase/config.toml`, `backend/Makefile`, `backend/Brewfile`, `backend/.gitignore`
- Create: `backend/supabase/functions/.env.example`

**Interfaces:**
- Produces: `make fn-serve`, `make fn-test`; a documented, verified answer to "can a caller with
  only the anon/publishable key invoke a function that requires a user?"

- [ ] **Step 1: Confirm `.env` is ignored at the nested path**

`backend/.gitignore` currently has a bare `.env`. Git's bare pattern matches at any depth, so
`supabase/functions/.env` is covered — **verify it rather than assume**:

```bash
cd ~/Developer/coterie/backend
mkdir -p supabase/functions && touch supabase/functions/.env
git check-ignore -v supabase/functions/.env   # must print a matching rule
```
If it does not print a rule, add `supabase/functions/.env` explicitly. Commit a
`.env.example` with placeholder values and key names only.

- [ ] **Step 2: Add the Stripe CLI to the Brewfile**

```ruby
tap "stripe/stripe-cli"
brew "stripe/stripe-cli/stripe"
```

- [ ] **Step 3: Add function targets to the Makefile**

```makefile
fn-serve:
	supabase functions serve

fn-test:
	deno test --allow-env --allow-net supabase/functions
```

- [x] **Step 4: Settle the caller-auth question empirically — do not guess** ✅ ANSWERED 2026-08-13

**Result, against CLI 2.112.0 / edge-runtime 1.74.3:**

| Caller | Result |
|---|---|
| No credentials | `401` |
| Publishable key only | `401 INVALID_CREDENTIALS` |
| Legacy anon JWT | `401 INVALID_CREDENTIALS` |
| Real user JWT | `200`, `authMode: "user"` |

`auth: "user"` is sufficient on its own — the extra guard this step contemplated is not needed
to keep publishable-key callers out. The positive control is what makes this conclusive; four
401s alone would equally have described a broken function.

**Correction to Task 3:** the caller's id is at **`ctx.userClaims.id`**, not
`ctx.userClaims.sub`. `sub` appears only nested inside `userMetadata`, so the guard sketched
below would have rejected every legitimate caller. `ctx.supabase.auth.getUser()` also returns
it. Full context keys: `supabase`, `supabaseAdmin`, `userClaims`, `jwtClaims`, `authMode`,
`authKeyName`.


CLI 2.112.0 writes `verify_jwt = false` into `config.toml` for every new function and moves the
decision to the handler's `withSupabase({ auth: … })` mode. The new API-key system
(`sb_publishable_…` / `sb_secret_…`) is not a JWT, so it is **not obvious** that platform-level
`verify_jwt = true` and handler-level `auth: 'user'` compose the way we want.

Scaffold a throwaway `auth-probe` function with `auth: 'user'` and hit it three ways:

```bash
supabase functions new auth-probe   # then set auth: "user" in the handler
supabase functions serve auth-probe

# 1. no credentials at all            → expect 401
curl -i -X POST http://127.0.0.1:54321/functions/v1/auth-probe

# 2. publishable/anon key only        → MUST be rejected. This is the whole point.
curl -i -X POST http://127.0.0.1:54321/functions/v1/auth-probe \
  -H "apikey: $PUBLISHABLE_KEY"

# 3. a real user JWT from a signed-in test user → expect 200 with the right user id
curl -i -X POST http://127.0.0.1:54321/functions/v1/auth-probe \
  -H "Authorization: Bearer $USER_JWT"
```

**Record the actual results in this file.** If case 2 returns 200, `auth: 'user'` is not
sufficient on its own and the function must additionally assert `ctx.userClaims?.sub` is present
and non-anonymous before doing anything — otherwise anyone holding the publicly shipped anon key
could create verification sessions on the account's dime (£1.25 each in live mode). Delete
`auth-probe` afterwards.

- [ ] **Step 5: Add the function config entries**

```toml
[functions.create-verification-session]
enabled = true
verify_jwt = false          # handler enforces auth: "user"; see Task 2 Step 4

[functions.stripe-identity-webhook]
enabled = true
verify_jwt = false          # Stripe sends no JWT; the signature is the authentication

[functions.sync-verification-status]
enabled = true
verify_jwt = false
```

`config.toml` covers both `functions serve` and `functions deploy`, which is why we use it rather
than the `--no-verify-jwt` flag (the flag still exists and is not deprecated, but drifts between
environments).

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "chore(functions): edge function config, secrets scaffolding and make targets"
```

---

## Task 3: `create-verification-session` Edge Function ✅ DONE

**Repo:** `backend`. **Blocked on Task 0 Step 5** for a live run; the code and its unit tests are
not blocked.

**Files:**
- Create: `backend/supabase/functions/create-verification-session/{index.ts,deno.json}`

**Interfaces:**
- Consumes: caller's user JWT; `STRIPE_SECRET_KEY`.
- Produces: `POST /functions/v1/create-verification-session` →
  `{ session_id, ephemeral_key_secret, url, status }`.

- [ ] **Step 1: Scaffold and pin dependencies**

```bash
supabase functions new create-verification-session
```
Then in its `deno.json`, add `"stripe": "npm:stripe@^22"` alongside the scaffolded
`@supabase/server` and `@supabase/functions-js` entries.

- [ ] **Step 2: Implement**

Sketch. The rules it must enforce, in order:

1. `withSupabase({ auth: "user" })`; derive the user id **from `ctx.userClaims.id` only** (see Task 2 Step 4 — `.sub` is not populated at the top level). Never
   from the request body — that is the entire "a user can only create a session for themselves"
   guarantee, and a `user_id` parameter in the body would hand it away.
2. Short-circuit if the profile is already `verified` → `409`. No point charging for it.
3. **Reuse before create.** Look for an existing `verification_sessions` row for this user whose
   status is not `canceled`. Stripe explicitly recommends reusing a session across retries so the
   failed attempts stay attached to one object — and it avoids being billed twice.
   - Reusing: `stripe.identity.verificationSessions.retrieve(id)` to get a **fresh** `url`
     (the old one expired after 48h and is single-use).
   - Creating: `type: 'document'`, `options.document.require_matching_selfie: true`,
     `metadata: { user_id }`, `client_reference_id: user_id`, and an **idempotency key derived
     from the user id** so a double-tap cannot create two sessions.
4. Create the ephemeral key — required by the iOS SDK, single-use, expires in 1 hour:
   ```ts
   const ephemeralKey = await stripe.ephemeralKeys.create(
     { verification_session: session.id },
     { apiVersion: "2026-07-29.dahlia" },   // must match the StripeIdentity SDK version
   );
   ```
   Pin this string in one place with a comment tying it to the SPM version from Task 7.
5. Record the session via `ctx.supabaseAdmin.rpc("apply_verification_outcome", …)` with the
   session's current status, so the audit row exists before the user ever opens the sheet.
6. Return `{ session_id, ephemeral_key_secret, url, status }`. **Never** return the secret key,
   the full session object, or `verified_outputs`.

Store both `metadata.user_id` **and** `client_reference_id`. `client_reference_id` is
Dashboard-searchable and is the reconciliation handle an operator will actually use — and note
that **redaction wipes `metadata`**, so metadata alone is not a durable link.

- [ ] **Step 3: Test what can be tested without a key**

Extract the request-shaping into a pure function in `_shared/verification.ts` (which session
params we send for a given user) and unit-test it with `deno test`. The Stripe call itself is
untestable offline; do not fake it with a hand-rolled mock of the Stripe SDK — that tests the
mock. Cover it in Task 5's live run instead.

- [ ] **Step 4: Live smoke (blocked on Task 0)**

```bash
echo "STRIPE_SECRET_KEY=sk_test_…" >> supabase/functions/.env    # gitignored
make fn-serve
curl -i -X POST http://127.0.0.1:54321/functions/v1/create-verification-session \
  -H "Authorization: Bearer $USER_JWT"
```
Expected: `200` with a `vs_…` id, an `ek_…` secret, and a `https://verify.stripe.com/start/…`
URL. Then confirm a second call for the same user returns the **same** `vs_…` id.

Also confirm the negative: the same call with only the publishable key must fail (Task 2 Step 4).

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(functions): create Stripe Identity verification sessions"
```

---

## Task 4: `stripe-identity-webhook` Edge Function ✅ DONE

**Repo:** `backend`. **Not blocked by Task 0** — signature verification is testable entirely
offline with a made-up signing secret.

**Files:**
- Create: `backend/supabase/functions/stripe-identity-webhook/{index.ts,deno.json,index_test.ts}`
- Create: `backend/supabase/functions/_shared/{verification.ts,verification_test.ts}`

- [ ] **Step 1: Extract the status mapping as a pure function**

`_shared/verification.ts` — the D3 table as code, with no I/O:

```ts
export type StripeStatus = "requires_input" | "processing" | "verified" | "canceled";
export type ProfileStatus = "unverified" | "pending" | "verified";
export function profileStatus(stripe: StripeStatus): ProfileStatus { … }
```

This is the piece worth testing hardest, and it is free to test.

- [ ] **Step 2: Implement the handler**

```ts
import Stripe from "npm:stripe@^22";
import { withSupabase } from "@supabase/server";

const stripe = new Stripe(Deno.env.get("STRIPE_SECRET_KEY")!);
const cryptoProvider = Stripe.createSubtleCryptoProvider();

export default {
  fetch: withSupabase({ auth: "none" }, async (req, ctx) => {
    const signature = req.headers.get("Stripe-Signature");
    const body = await req.text();          // RAW. Never req.json() — any reserialisation
                                            // changes the bytes and the HMAC fails.
    let event;
    try {
      event = await stripe.webhooks.constructEventAsync(
        body, signature!, Deno.env.get("STRIPE_WEBHOOK_SIGNING_SECRET")!,
        undefined, cryptoProvider,
      );
    } catch (err) {
      return new Response(err.message, { status: 400 });
    }
    …
  }),
};
```

`constructEventAsync` + `createSubtleCryptoProvider()` is required, not optional: WebCrypto's
`SubtleCrypto` is async-only, so the synchronous `constructEvent` that every Stripe doc sample
shows throws on Deno. **This pattern is documented by Supabase, not by Stripe** — flagging that
because it means Stripe's own docs will look like they contradict our code.

Then, in order:

1. **Idempotency first.** `insert into stripe_webhook_events (event_id, event_type, stripe_session_id)
   … on conflict (event_id) do nothing`. If zero rows were inserted, return `200 {"duplicate":true}`
   and stop. Also guard the documented edge case where Stripe emits *two distinct* event objects
   for one state change, by treating `(stripe_session_id, event_type)` as a secondary dedupe key.
2. Resolve the user id: prefer `client_reference_id`, fall back to `metadata.user_id`, and if
   neither is present **fail loudly with a 400 and a log line** rather than silently dropping the
   event. A verification we cannot attribute is a bug, not a no-op.
3. Call `ctx.supabaseAdmin.rpc("apply_verification_outcome", { … })` with
   `last_error.code` / `last_error.reason` when present.
4. Return `200` fast. Do not fetch `verified_outputs` — we deliberately never read them.
5. Ignore `identity.verification_session.created` beyond the ledger row (no user-visible change).

- [ ] **Step 3: Test it offline**

`deno test` with a signing secret we invent (`whsec_test_…`), signing a synthetic payload
ourselves. Cover:
- valid signature → `200`
- tampered body → `400`
- stale timestamp beyond the 5-minute tolerance → `400`
- a `v0`-only signature header → `400` (downgrade protection: only `v1` counts)
- the same `event.id` twice → second returns duplicate, and the RPC is called **once**
- `verified` then a late `processing` for the same session → profile stays `verified`
- an event with neither `client_reference_id` nor `metadata.user_id` → `400`, not `200`

Stripe's Node lib exposes `webhooks.generateTestHeaderString` for the signing side; **there is a
known Deno incompatibility with it** (supabase discussion #33011). If it misbehaves, compute the
HMAC-SHA256 of `` `${timestamp}.${body}` `` with WebCrypto directly — six lines, and it documents
the format rather than hiding it.

- [ ] **Step 4: Mutation-check**

Delete the signature verification entirely and confirm the tampered-body test fails. Delete the
`on conflict do nothing` guard and confirm the duplicate test fails. A webhook suite that stays
green without signature checking is worse than no suite.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(functions): Stripe Identity webhook with signature checks and idempotency"
```

---

## Task 5: End-to-end locally, plus the missed-webhook fallback ⚠️ PARTIALLY DONE

**Live run passed 2026-08-14.** `create-verification-session` returns a real `vs_…`, an `ek_…`
and a `https://verify.stripe.com/start/test_…` URL; a second call returns the **same** session
id, so reuse works and a double tap cannot be billed twice. The audit row is written before the
sheet opens, the profile correctly stays `unverified`, and a publishable-key-only caller is
refused with 401.

**Answers two open questions:**
- The pinned API version `2026-07-29.dahlia` **is accepted by Stripe** when minting the
  ephemeral key. Whether the iOS SDK accepts the same string is still open until Task 7.
- Identity in test mode requires **mobile SDK access enabled** at
  `dashboard.stripe.com/settings/identity` — separate from submitting the application. Until
  it is on, every call fails with *"This account does not have access to use Identity's mobile
  SDKs"*, and **no session is created**, so there is nothing to clean up.

**Gotcha that cost real time:** the Stripe CLI's logged-in account and the `sk_test_` key in
`.env` can be different accounts — a sandbox versus the main account. Stripe's error log URL
names the key's account, which is how it was spotted. `stripe listen` forwards from the CLI's
account, so if they disagree the webhook can never match a session. Check both agree before
any live run:

```bash
curl -s https://api.stripe.com/v1/account -u "$STRIPE_SECRET_KEY:" | grep -o '"id": "[^"]*"'
stripe config --list | grep account_id
```

**Still to do:** the missed-webhook fallback (`sync-verification-status`), and an end-to-end run
through the hosted URL to observe real status transitions.


**Repo:** `backend`. **Blocked on Task 0.**

- [ ] **Step 1: Forward Stripe events to the local function**

```bash
stripe listen \
  --events identity.verification_session.created,identity.verification_session.processing,identity.verification_session.verified,identity.verification_session.requires_input,identity.verification_session.canceled,identity.verification_session.redacted \
  --forward-to http://127.0.0.1:54321/functions/v1/stripe-identity-webhook
```

`stripe listen` prints its **own** `whsec_…` on startup — that is the secret the local function
must use, not the one from a Dashboard endpoint. Put it in `supabase/functions/.env`.

Subscribe to `redacted` explicitly: Stripe documents that endpoints subscribed to "all events"
silently exclude it, and we will need it for GDPR deletion later.

- [ ] **Step 2: Understand what the CLI can and cannot fake**

Verified against the `stripe-cli` fixture source: `stripe trigger` has fixtures for
`identity.verification_session.created`, `.canceled` and `.redacted` **only**. There is **no
fixture for `.verified`, `.requires_input` or `.processing`** — the fixtures work by making real
API calls, and there is no API that forces a session to `verified`.

So:
- `stripe trigger identity.verification_session.created` / `.canceled` exercise the real
  transport, signature and idempotency path.
- To see a real `verified` you must walk a test-mode session to completion in the hosted URL or
  the SDK sheet and pick the success test case. **Test mode does not run real checks** — "If your
  API key is in test mode, verification checks won't actually process, though everything else
  will occur as if in live mode."
- **Unknown:** the exact list of predefined test cases Stripe's flow offers, and whether the
  native iOS sheet offers the same picker as the web flow. There is no Stripe Identity testing
  page (`docs.stripe.com/identity/testing` is a 404). Record what you actually see.
- **Trap:** search results about "test images take precedence over test ID numbers" and document
  file tokens belong to **Connect** account verification, a different product. They do not apply
  to Identity VerificationSessions.

- [ ] **Step 3: Walk the full path once**

Create a session via Task 3's endpoint, open the returned `url` in a desktop browser, complete it
with the success test case, and watch: `stripe listen` logs the events → the function returns
`200` → `select * from public.verification_sessions` shows the transitions → `profiles` shows
`verified`. Then check `stripe_webhook_events` has one row per event and no duplicates.

- [ ] **Step 4: Prove idempotency against the real transport**

```bash
stripe events resend <event_id> --webhook-endpoint <id>
```
Expected: `200`, no second row, no state change.

- [ ] **Step 5: Prove the client still cannot cheat**

With a real user JWT, against the live local API:

```bash
curl -X PATCH "http://127.0.0.1:54321/rest/v1/profiles?id=eq.$UID" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" -d '{"verification_status":"verified"}'   # expect 403 / 42501

curl -X POST "http://127.0.0.1:54321/rest/v1/rpc/apply_verification_outcome" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" \
  -d '{"p_stripe_session_id":"vs_x","p_user_id":"'$UID'","p_stripe_status":"verified"}'  # expect 403/404

curl -X POST "http://127.0.0.1:54321/rest/v1/verification_sessions" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" -d '{"user_id":"'$UID'","stripe_session_id":"vs_y","status":"verified"}'  # expect 403
```

This is the step that actually proves the security model, in the way pgTAP structurally cannot.
Do not skip it, and paste the real responses into this file.

- [ ] **Step 6: Build `sync-verification-status`**

A third function, `auth: "user"`, that retrieves the caller's current session from Stripe and
applies the same RPC. It exists so a dropped webhook self-heals: the app calls it when it
foregrounds onto a stale `pending` (D5). Rate-limit it to one call per user per 30s.

Test it by stopping `stripe listen`, completing a verification (so the webhook is genuinely
missed), then calling the sync endpoint and confirming the profile catches up.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat(functions): verification status sync and end-to-end local verification"
```

---

## Task 6: iOS — `VerificationRepository`, DTO/DTI, error mapping

**Repo:** `ios`. **Not blocked by Task 0** — no Stripe dependency yet, mock-driven throughout.

**Files:**
- Create: `Coterie/Core/Models/VerificationSession.swift`
- Create: `Coterie/Features/Verification/{VerificationRepository.swift,VerificationDTO.swift,SupabaseVerificationRepository.swift}`
- Create: `CoterieTests/Mocks/MockVerificationRepository.swift`
- Test: `CoterieTests/{VerificationMapperTests.swift,VerificationErrorMappingTests.swift}`

**Interfaces:**
```swift
struct StripeVerificationSession: Sendable, Equatable {
    let sessionID: String
    let ephemeralKeySecret: String
    let hostedURL: URL?
}

protocol VerificationRepository: Sendable {
    func createSession() async throws -> StripeVerificationSession
    func syncStatus() async throws -> VerificationStatus
    func latestRejection() async throws -> VerificationRejection?
}
```

- [ ] **Step 1: DTO/DTI with explicit `CodingKeys`**

Per the Plan 1 constraint, Swift identifiers may not contain underscores (SwiftLint
`identifier_name`), so `session_id` / `ephemeral_key_secret` / `last_error_code` must be mapped in
`CodingKeys`. And per the Plan 1 lesson, **that means the wire contract lives in `CodingKeys` and
nothing else will fail if it drifts** — so each DTO needs a round-trip test asserting the exact
snake_case JSON keys, and `VerificationSessionDTO` needs a fixture decoded from a captured real
response (Task 5).

- [ ] **Step 2: Error mapping as a pure function**

Follow `SupabaseAuthRepository.mapped(_:)` exactly. Map `FunctionsError` onto a domain
`VerificationRepositoryError`:

| Source | Domain case | User-facing |
|---|---|---|
| `FunctionsError.httpError(409, _)` | `.alreadyVerified` | "You're already verified." |
| `FunctionsError.httpError(401, _)` | `.notSignedIn` | "Please sign in again." |
| `FunctionsError.httpError(429, _)` | `.rateLimited` | "Too many attempts. Try again shortly." |
| `FunctionsError.relayError` | `.serviceUnavailable` | "Verification is unavailable right now." |
| `URLError.notConnectedToInternet` | `.offline` | "You're offline." |
| anything else | `.unknown` | generic |

Confirmed against supabase-swift 2.54.1: `FunctionsError` has **exactly two cases**,
`.httpError(code:data:)` and `.relayError`. There is **no `FunctionsHttpError`** — that is the
supabase-js name. Also note v2.54.0 changed the semantics so non-2xx relay failures throw
`.relayError`; verify which case a 409 actually produces by running it, and correct the table.

12+ tests, no network, mirroring `AuthErrorMappingTests`.

- [ ] **Step 3: Implement `SupabaseVerificationRepository`**

`client.functions.invoke("create-verification-session", options: FunctionInvokeOptions())`,
decoding into the DTO. The user's JWT is attached automatically from the client's session — the
docs state this, but **confirm it on the wire** in Task 7's live run rather than trusting it.

`latestRejection()` reads `verification_sessions` through PostgREST (RLS scopes it to the caller).

- [ ] **Step 4: Mock, tests, run, commit**

`MockVerificationRepository` as an `actor` with **call counts**, per the Plan 1 constraint —
result-only assertions let a branch-routing bug pass.

```bash
make test && make lint
git add -A && git commit -m "feat: verification repository, DTOs and error mapping"
```

---

## Task 7: iOS — StripeIdentity SDK and wiring the inert button

**Repo:** `ios`. **Blocked on Task 0 Steps 4 and 5.**

**Files:**
- Modify: `project.yml`, `Coterie/App/AppContainer.swift`, `Coterie/App/RootView.swift`,
  `Coterie/Features/Shell/MainTabView.swift`
- Create: `Coterie/Features/Verification/IdentityVerificationPresenter.swift`

- [ ] **Step 1: Add the package**

```yaml
packages:
  StripeIdentity:
    url: https://github.com/stripe/stripe-ios-spm
    from: "26.6.0"
targets:
  Coterie:
    dependencies:
      - package: StripeIdentity
        product: StripeIdentity
```
Note the repo is `stripe-ios-spm`, **not** `stripe-ios` — the SPM distribution is a separate
repository. Then `make generate && make build`.

- [ ] **Step 2: Add the camera usage string**

In `project.yml` under the Coterie target's `info.properties`:
```yaml
        NSCameraUsageDescription: We use your camera to photograph your ID and take a selfie, so we can confirm you're a real person.
```
Regenerate. Note this string is user-visible and is reviewed by App Store review.

- [ ] **Step 3: Measure the size cost and record it**

Archive before and after, or use Xcode's app size report, and write the delta into this file. If
it is unacceptable, D1's exit is to drop the SDK and use the hosted `url` the Edge Function
already returns — the backend does not change.

- [ ] **Step 4: Bridge the UIKit sheet into SwiftUI**

`IdentityVerificationSheet` is UIKit — `present(from: UIViewController)`. Wrap it in a small
presenter that resolves the topmost view controller, so `VerificationIntroView` stays a plain
SwiftUI view and the view model never imports StripeIdentity. Map the three results:

| Result | App behaviour |
|---|---|
| `.flowCompleted` | optimistically set `.pending`, start the bounded poll (Task 8) |
| `.flowCanceled` | back to `.unverified`, no error banner — cancelling is not a failure |
| `.flowFailed(error)` | show `error.localizedDescription` (Stripe's is localised) |

`IdentityVerificationSheet.Configuration(brandLogo:)` takes a `UIImage`, recommended 32×32 pt —
**our** logo, not Stripe's.

- [ ] **Step 5: Fill the two inert closures**

`onVerify: {}` appears in exactly two places:
- `Coterie/App/RootView.swift:54` — origin `.onboarding`
- `Coterie/Features/Shell/MainTabView.swift:51` — origin `.onDemand`

Both must call the same view-model entry point, so cancellation and retry behave identically
regardless of where verification was raised. (`MainTabView.swift:198`'s
`OursButton("Verify identity")` merely presents the sheet and is already correct.)

- [ ] **Step 6: Add the repository to `AppContainer`** and inject it into `VerificationViewModel`.

- [ ] **Step 7: Live run on a device**

The simulator has no camera. **Document capture needs a physical device** — the first task in this
plan that does. Walk the whole flow, then confirm in the Dashboard that the session shows the
right `client_reference_id`.

- [ ] **Step 8: Build, lint, test, commit**

---

## Task 8: iOS — pending refresh, retry, and the rejection reason

**Repo:** `ios`. **Not blocked by Task 0** (mock-driven).

**Files:** modify `VerificationViewModel.swift`, `VerificationIntroView.swift`,
`VerificationViewModelTests.swift`.

- [ ] **Step 1: Carry the rejection reason in the state**

Change `case unverified` to `case unverified(reason: String?)`. This touches the existing
`@Test(arguments:)` parameterised test and `shouldOfferVerification` — both need updating, and
`.failed(String)` must keep meaning "could not load", not "Stripe said no" (D3).

- [ ] **Step 2: Map the error codes to copy**

`last_error.reason` is documented as user-displayable, so it is the default. Three codes need
bespoke handling because the generic "try again" is wrong for all of them:

| Code | Behaviour |
|---|---|
| `consent_declined` | Explain that verification is required to co-own; offer retry. Stripe's docs suggest checking with legal counsel about offering a non-biometric alternative — **flag for Jonathan, not a code decision.** |
| `under_supported_age` | Terminal. Do **not** offer retry. Needs a "you must be 16+" line, and a matching clause in Coterie's terms. |
| `country_not_supported` | Terminal for now. Offer support contact, not retry. |

- [ ] **Step 3: Bounded polling and foreground refresh (D5)**

- refetch on `.task`/appear (already present)
- refetch on `ScenePhase` → `.active`
- while `.pending`: poll every 5s, stop after 2 minutes, then show a manual refresh
- on foreground onto a `pending` older than ~5 minutes: call `sync-verification-status`

Tests must assert **call counts on the mock** — a poll that silently stops, or never stops, is
exactly the kind of bug a result-only assertion misses. Include a test that the poll terminates,
and one that it stops the moment the status becomes `verified`.

- [ ] **Step 4: Mutation-check** — make the poll never cancel, and confirm a test fails.

- [ ] **Step 5: Build, lint, test, commit**

---

## Task 9: The Stripe wordmark, and gating the claim

**Repo:** `ios`. Step 1 is not blocked by anything.

### The claim

`Coterie/Features/Verification/VerificationIntroView.swift` line ~243 renders:

> "Verification is securely handled by Stripe"

It appears only in the `.unverified` branch (line 99 passes `showStripe: true`; every other branch
passes `false`). **Today it is false** — `onVerify` is an empty closure, so nothing is handled by
Stripe at all. It is a claim about where someone's passport goes, on the screen where they decide
whether to hand it over.

- [ ] **Step 1: Gate it on the integration actually being live**

Add `IDENTITY_ENABLED` to `Config/Debug.xcconfig` and `Config/Release.xcconfig`, surface it as
`AppEnvironment.isIdentityVerificationEnabled`, and thread it into `VerificationIntroView`. When
false:
- the Stripe line is **not** rendered, and
- the primary button does not claim to start a verification (disable it, or label it so it does
  not promise something that cannot happen).

Gating only the sentence would leave a "Verify identity" button that silently does nothing, which
is its own lie. Set it `true` only in the build where Task 7's live run passed.

Add a test asserting the line is absent when the flag is false. (No snapshot tests — assert on the
view model / a small `shouldShowStripeAttribution` computed property rather than the rendered
view.)

### The asset

Stripe's Marks Usage Terms forbid modifying, abbreviating, recolouring or redrawing the marks, and
forbid displaying them more prominently than our own. The current code sets the word "Stripe" in
our own typeface, which is a redraw — the in-code comment already acknowledges this and names the
intended asset `StripeWordmark`, matching how `GoogleMark` is handled in `FederatedSignInButton`.

- [ ] **Step 2: Obtain the official asset**

From <https://stripe.com/newsroom/brand-assets> → **The Stripe Logo** → downloads
`Stripe_logo_kit.zip`
(`https://assets.stripeassets.com/fzn2n1nzq965/7q0dJGs6fRS1LRmMpChoAF/87def4edfbb7fd5aef4ab9baf904b2db/Stripe_logo_kit.zip`).

Take the **wordmark**, in **slate** (or blurple) for our light background — Stripe's page states:
*"Use slate and blurple on light backgrounds, and white on dark backgrounds"* and permits no other
colour. The app is `UIUserInterfaceStyle: Light`, so one slate variant is sufficient; add the
white variant only if dark mode is ever enabled.

Add as `Coterie/Resources/Assets.xcassets/StripeWordmark.imageset/`, `Contents.json` with
`"template-rendering-intent": "original"` so it can never be tinted by an accent colour. Do not
scale non-uniformly, do not recolour, do not add effects.

- [ ] **Step 3: Do not use the "Powered by Stripe" badge**

The same page offers `Powered_by_Stripe-badge.zip`. **It is a payments/checkout badge.** Using it
next to identity verification would misstate what Stripe is doing for us and edges toward implying
a partnership, which the Marks Terms prohibit. Plain wordmark plus our own sentence.

- [ ] **Step 4: Keep the sentence, set the name in Stripe's mark**

"Verification is securely handled by" in our type, followed by the wordmark image, sized no larger
than our own brand elements on that screen. Give the image an accessibility label of "Stripe" so
VoiceOver reads the sentence whole.

- [ ] **Step 5: Build, lint, test, commit**

---

## Task 10: CI, environments, and the go-live checklist

**Repos:** `backend`, `ios`.

- [ ] **Step 1: Backend CI currently cannot run Edge Function tests**

`.github/workflows` starts Supabase with
`-x gotrue,realtime,…,edge-runtime,…` — **`edge-runtime` is explicitly excluded**, deliberately,
to pull one image instead of twelve. Options:
- (preferred) run `deno test` on `supabase/functions` as a **separate job with no Supabase stack
  at all** — the handler tests in Task 4 are pure and need no database. Fast, and keeps the pgTAP
  job as lean as it is.
- Add a third, slower job that starts the edge runtime only if integration coverage is later
  needed.

Add `deno` to the workflow via `denoland/setup-deno`.

- [ ] **Step 2: Secrets for hosted environments**

Per-environment Stripe keys (spec §8: test mode in dev and QA, live only in prod):
```bash
supabase link --project-ref <ref>
supabase secrets set STRIPE_SECRET_KEY=sk_test_… STRIPE_WEBHOOK_SIGNING_SECRET=whsec_…
supabase secrets list
supabase functions deploy create-verification-session stripe-identity-webhook sync-verification-status
```
Secrets take effect immediately with no redeploy. **Edge Functions cannot be deployed to the local
stack — a hosted Supabase project is required**, which QA/prod do not yet have (open item from
Plan 1). This is a hard prerequisite for anything beyond local development.

Note `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` and `SUPABASE_DB_URL` are
auto-injected and **reserved** — they cannot be set as custom secrets.

- [ ] **Step 3: Register the hosted webhook endpoint** (Jonathan)

Workbench → Webhooks → Add destination → URL
`https://<project-ref>.supabase.co/functions/v1/stripe-identity-webhook`, subscribing to the six
`identity.verification_session.*` events from Task 5 Step 1 — including `redacted`, explicitly.
Copy that endpoint's `whsec_` (different from the CLI's) into `supabase secrets`.

- [ ] **Step 4: Pre-live checklist**

- [ ] App Store Connect App Privacy answers updated for the ID data Stripe collects in-app
- [ ] `NSCameraUsageDescription` reads well to a reviewer
- [ ] Terms updated: minimum age 16, identity verification required for money actions
- [ ] A GDPR deletion path exists on paper: `POST /v1/identity/verification_sessions/{id}/redact`,
      which takes **up to 4 days** and wipes `metadata` — so our own `verification_sessions` row
      must remain the durable link to the user, not Stripe's metadata
- [ ] Live keys set only in prod; `IDENTITY_ENABLED` true only where the flow actually works

---

## What is buildable now vs blocked on Stripe

| | |
|---|---|
| **Now, no Stripe account** | Task 1 (migration + pgTAP + mutation checks) · Task 2 (function config, secrets plumbing, the caller-auth probe) · Task 4 (webhook handler, signature verification, idempotency — all testable offline with an invented `whsec_`) · Task 6 (repository protocol, DTOs, error mapping, mocks, view-model tests) · Task 8 (pending refresh, retry, rejection copy — mock-driven) · Task 9 Step 1 (gating the claim) and Steps 2–4 (the wordmark asset is a public download) |
| **Blocked on the account + keys** | Task 3's live run · Task 5 entirely · Task 7 entirely (the SDK needs the "Enable" click, and the flow needs a device) · Task 10 Steps 2–3 |
| **Blocked on something else entirely** | Hosted QA/prod Supabase projects (Edge Functions cannot deploy to local) — an open item carried over from Plan 1 |

Roughly two-thirds of the work — and *all* of the security-critical work — can be done and proven
before Jonathan finishes Stripe's onboarding. Start Task 1 immediately.

---

## Risks

| Risk | Handling |
|---|---|
| **Default `execute` on new functions leaks the privileged writer to `authenticated`** | The highest-severity failure here. One RPC call would let any signed-in user verify themselves. Explicit `revoke`s in the migration, a pgTAP `has_function_privilege` negative, **and** a live curl in Task 5 Step 5. Mutation-check all three. |
| **Webhook missed** | Bounded polling plus `sync-verification-status`, which retrieves the session from Stripe (the source of truth) when the app foregrounds onto a stale `pending`. Also `stripe events resend`. Live mode retries for 3 days anyway; sandbox only retries 3 times over a few hours, so *local* misses are far more likely than production ones — do not read local flakiness as a production risk. |
| **Webhook delivered twice** | `stripe_webhook_events.event_id` is a primary key and the first write. Postgres enforces it, so two racing function instances cannot both proceed. Secondary dedupe on `(session id, event type)` for the documented case where Stripe emits two distinct events for one change. |
| **Events out of order** | `verified` is terminal in SQL; a late `processing` or `requires_input` updates audit fields only. Tested in pgTAP, mutation-checked. |
| **`@supabase/server` is public beta** | It is what CLI 2.112.0 scaffolds and what every current doc shows, so the legacy path is now the unusual one. Pin `^1`. If it breaks, the fallback is `Deno.serve` + `createClient` + `auth.getUser()` — still supported, just undocumented. |
| **`auth: "user"` might accept a publishable/anon key** | Unknown until measured — Task 2 Step 4 exists solely to settle it. If it does, add an explicit `ctx.userClaims?.sub` assertion. Getting this wrong means anyone with the shipped anon key can burn £1.25 per call in live mode. |
| **Ephemeral key ↔ SDK API-version coupling** | The `apiVersion` string passed to `ephemeralKeys.create` must match the SDK. Pin it in one place with a comment naming the SPM version, and re-check on every `StripeIdentity` bump. |
| **SDK app-size cost is unknown** | Stripe publishes no figure. Measure in Task 7 Step 3; the exit is the hosted URL the backend already returns. |
| **Camera in `SFSafariViewController`** | Only matters if we take the hosted exit. Undocumented by both Stripe and Apple. If we ever fall back, test it before committing — Stripe's demo at <https://identity.stripedemos.com> is a free way to check. |
| **PII scope creep** | We store no `verified_outputs`. Reading DOB/ID numbers needs a *restricted* key and only works within 48 hours — a path we intentionally never build. If a future plan wants "name matches the card", that is a new decision with new obligations, not a small change. |
| **Under-16 users** | Stripe refuses them (`under_supported_age`). Coterie's terms need a minimum age, and the UI must not offer a retry that can never succeed. |
| **Test mode cannot produce a `verified` event on demand** | No `stripe trigger` fixture exists for `verified`/`requires_input`/`processing`. Automated coverage therefore comes from hand-signed synthetic events (Task 4); the real transport is exercised manually (Task 5). Do not plan a CI job that expects `stripe trigger identity.verification_session.verified` to work. |
| **Cross-repo drift** | The migration (backend) and the DTOs (ios) can no longer change in one commit. Land Task 1 before Task 6, and pin the wire contract with a captured-response fixture, as Plan 1 did for `ProfileDTO`. |

---

## Open questions for Jonathan

1. **Do test-mode Identity API calls work before the Identity application is approved?** Stripe
   lists approval as a prerequisite but never states the failure mode. This decides whether Tasks
   3/5 unblock on day one or on approval day.
2. **Selfie check on or off?** `require_matching_selfie: true` is assumed above. It is the
   stronger anti-fraud posture for strangers co-owning goods, and it costs nothing extra
   (£1.25 covers document + selfie). It also adds `selfie_*` failure modes and a liveness step.
3. **Non-biometric alternative on `consent_declined`?** Stripe's docs say to check with legal
   counsel whether you are obliged to offer one. For a UK pilot this is likely a "document it and
   move on", but it is a legal call, not an engineering one.
4. **Retry limit.** Stripe recommends limiting submission attempts and gives no default. Three
   before requiring support contact seems reasonable — confirm.
5. **Do `stripe_customer_id` / `stripe_connect_account_id` go in now?** See below.

---

## Notes / Known follow-ups

- **Confirmed gap: `profiles` is missing `stripe_customer_id` and `stripe_connect_account_id`.**
  Spec §4 lists both; `0001_profiles.sql` has neither. **Recommendation: do not add them in this
  plan.** Identity does not use them — a VerificationSession has no Customer — and a column with
  no writer is a schema that lies about what the system does. They belong to the payments plan,
  in the same migration as the code that populates them. Recorded here so it is a decision rather
  than an oversight.
- **`related_customer` exists on the VerificationSession object** and links a verification to a
  Customer for fraud signals. Worth revisiting when payments land; irrelevant now.
- **Realtime is the natural successor to polling.** When the funding bar arrives and Realtime is
  set up anyway, move the pending screen onto a `postgres_changes` subscription on the user's own
  `profiles` row and delete the poll.
- **Push notifications remain undecided** (carried from Plan 1). "We'll let you know as soon as
  it's done" in `VerificationIntroView`'s pending copy is a promise the app cannot currently keep
  if the user leaves the screen. Either soften that copy in Task 8 or accept it as a known
  overstatement until push exists.
- **`identity.verification_session.redacted` must be subscribed to explicitly.** Endpoints
  subscribed to "all events" silently exclude it. Easy to lose in a later endpoint reconfiguration.
- **Stripe API version pinning.** The ephemeral key pins `2026-07-29.dahlia` per the current docs.
  Record the version actually used, since the docs sample will move.
- **The Deno `generateTestHeaderString` issue** (supabase discussion #33011) may force hand-rolled
  HMAC signing in tests. That is arguably better anyway — it documents the signature format
  instead of hiding it behind a helper.
- **Everything in this plan that touches an SDK or a runtime is a prediction.** Plan 1 shipped
  nine defects that only appeared on execution, all in exactly those places. Expect the same here,
  and replace these code blocks with what actually ran.
