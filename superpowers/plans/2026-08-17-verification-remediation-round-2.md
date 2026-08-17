# Verification Remediation, Round 2 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the one original review finding that was never actually fixed, plus five regressions introduced by the first remediation, so the verification feature stops claiming states it has not verified.

**Architecture:** Two repositories, both local. The backend is Supabase (Postgres + Edge Functions in Deno); the app is SwiftUI with MVVM and repository protocols. Every fix follows the same rule: a failure must be representable and reported, never converted into a decided state. Where an invariant can live in the database it does.

**Tech Stack:** Swift 6 / SwiftUI / Swift Testing / SwiftLint; Supabase CLI, Postgres, pgTAP, Deno; Stripe Identity.

**Spec:** No separate spec document exists. This plan carries its own findings — they came from two review agents whose reports live only in the conversation that produced this file, so the **Findings** section below is the source of truth and must not be summarised away.

---

## Global Constraints

- **Work on a branch.** `git checkout -b fix/verification-round-2` in each repo. Nothing goes to `main` in this plan.
- **Two separate git repositories:** `/Users/jonathan.aguele/Developer/coterie/ios` and `/Users/jonathan.aguele/Developer/coterie/backend`. Commit in whichever repo the task touches.
- **SwiftLint must be clean.** `identifier_name` rejects underscores; no force unwrapping; `line_length` 140; `cyclomatic_complexity` ≤ 10; `file_length` 400; `vertical_whitespace` one blank line. Note `make lint` currently exits 0 on warnings — Task 8 changes that.
- **Mutation testing is the standard.** A fix is not done because tests pass. Break the fix, run the suite, watch a *named* test fail, restore. Every task has a mutation step; do not skip it.
- **No test may hang.** If a test awaits something a mutation could stop delivering, it must terminate on its own. Two hanging tests were written during the first remediation and only found when a mutation deadlocked the suite for ten minutes.
- **Migrations are never amended.** `0001`–`0003` are applied. New changes go in `0004_…sql`.
- **Comments explain WHY, not what.** The codebase deliberately carries reasoning comments. Do not add narration; do not delete existing reasoning. Two comments were found describing behaviour their code did not have — if you change behaviour, change the comment with it.
- **Never claim verification you did not perform.** Say what you ran and what it showed.

---

## Findings

Reproduced here because the reports exist nowhere else. `SFH` = silent-failure hunter, `CR` = code reviewer. Both reviewed independently.

### F1 — CRITICAL. `latestSessionID` discards its error, and the type makes it unreachable
Found by **both** reviewers (SFH N1, CR #1). The only original finding not fixed by the first remediation.

`backend/supabase/functions/_shared/reconcile.ts:48-57` destructures `{ data }` only, and the `Admin` interface at `:17` gives `maybeSingle()` a return type with **no `error` field**, so the error cannot be read even deliberately. The comment at `:22-30` in the same file explains precisely why that is fatal — and applies the reasoning only to `rpc`.

Failure: `sync-verification-status/index.ts:25` calls it and `:33` gates the reconcile on the result. A transient read failure (statement timeout, pool exhaustion, PostgREST 5xx) yields `null`, so **Stripe is never asked**, and the handler returns HTTP 200 with the stored column. That is the exact missed-webhook case the endpoint exists for, now failing in the one place the client is required to trust. Because it is a 200, the client's error handling never runs and nothing is logged anywhere.

Note also `create-verification-session/index.ts:41-54` runs the same query inline **and does check its error**, so two copies of one query disagree about whether errors exist.

### F2 — CRITICAL. `actionError` is never cleared on success
SFH N5, CR #2. **Introduced by the first remediation:** hoisting the banner out of `case .unverified` is what made a stale message visible in every state.

`VerificationViewModel.swift` — `startVerification()` clears `actionError`; `load()`, `refresh()` and `pollWhilePending()` never do. `dismissActionError()` is dead code: `OursMessageBanner.init(_ message: String, tone:)` takes no dismiss handler, so there is no affordance to clear it.

Failure: poll exhausts unanswered → `actionError` set. Connectivity returns, user taps "Check again", Stripe says verified. Screen reads "You're verified" with a red banner beneath saying verification is unavailable.

### F3 — CRITICAL. "Check again" deletes itself and does not restart the poll
SFH N2, CR #3.

`VerificationIntroView.swift:180` renders "Check again" only `if viewModel.isPollExhausted`. `VerificationViewModel.refresh()` sets `isPollExhausted = false` on success. The poll is keyed on `.task(id: viewModel.isPending)`, and `isPending` does not change when a refresh returns "still pending" — so the poll does not restart.

Failure: poll exhausts → button offered → user taps it → refresh succeeds, still pending → button **disappears**, poll does **not** restart → screen says "We're checking your ID… there's nothing else for you to do" with nothing checking and no control to ask again.

### F4 — IMPORTANT. `refreshOnForeground` resets the poll budget and wipes the footer
CR #4. **Introduced by the first remediation** when `refreshOnForeground` became a plain `load()`.

`load()` writes `state = .loading` unconditionally. That flips `isPending` false, which cancels the `.task(id:)`, then resolves back to `.pending`, which restarts it — with a fresh 24-call budget and `isPollExhausted` reset. Ten foregrounds gives 240 reconciles against a documented bound of 24, silently removes the "Check again" escape hatch mid-wait, and flashes `OursSpinner("Checking your account…")` over an answer already on screen every single time.

### F5 — HIGH. A failed sign-out signs the user out anyway
SFH N3. **Introduced by the first remediation** — the fix meant to "ask rather than assume" asks a question whose failure answer is indistinguishable from "nobody".

`AuthViewModel.signOut()`'s catch does `authedUser = await repo.currentUser`. `SupabaseAuthRepository.currentUser` is `guard let session = try? await client.auth.session else { return nil }`, and `client.auth.session` performs a **token refresh** — a network call. The condition that makes `signOut()` throw is the same one that makes that read throw.

Failure: user taps Sign out on a dead connection. Both calls fail, `currentUser` yields nil, `authedUser` is cleared, routing sends them to `.auth`. They are shown as signed out while the refresh token is still in the keychain and the session is live on the server; next launch signs them back in without authenticating. The "Couldn't sign out" banner renders on the sign-up screen, where it is meaningless.

The existing test passes for the wrong reason: `MockAuthRepository(shouldThrow: true)` leaves `stubbedUser` nil, so `currentUser` returns nil regardless of the bug.

### F6 — HIGH. `load()` blames the user's network for every error
SFH N4. `VerificationViewModel.swift:164` and `ProfileViewModel.swift:81` both hardcode `"Check your connection and try again."` for an error they never inspect. `refresh()` was fixed to surface the mapped error; `load()` — called from four places — was not. An RLS denial, an expired JWT, a 500, or a `DecodingError` all tell the user to check their wifi, behind a retry button that will fail identically forever.

`ProfileRepository` has **no** mapped error type at all (confirmed: no `ProfileRepositoryError` exists), which is why the literal is there.

### F7 — MEDIUM. The new `verification_refused` 409 is unmapped on the client
SFH N10, CR #7. `create-verification-session` now returns `{"error": "verification_refused"}` with 409. `SupabaseVerificationRepository.conflict(from:)` recognises only `already_verified` and maps everything else to `.verificationInProgress`, whose copy is "We're still checking your ID" — a lie for a terminal refusal. Currently masked because `startVerification` treats that case as "not a failure" and calls `load()`; it becomes visible the moment any other caller surfaces it.

### F8 — MEDIUM. Six new SwiftLint warnings, and `make lint` cannot fail
CR #8. Baseline `5b623c4` had one warning (`large_tuple`, pre-existing). HEAD has seven: `vertical_whitespace` at `VerificationViewModel.swift:42,50,67`, `ProfileViewModel.swift:57`, `VerificationPollingTests.swift:371`, and `file_length` at `VerificationPollingTests.swift:495` (limit 400). All six are deletion residue from the first remediation. `make lint` runs `swiftlint lint --quiet`, which exits 0 on warnings, so CI is green and silent.

### F9 — MEDIUM. The Deno handler tests are not in CI
CR, process gap. `backend/.github/workflows/db-test.yml` runs `supabase test db` only. `make fn-test` exists and is not called, so the five handler tests added *because* that is where the backend bugs live are ungated.

---

## Deferred, with reasons

Do **not** implement these. They are recorded so nobody has to rediscover them.

- **Logging (SFH N10 second half).** There is zero logging in the app — no `os_log`, `Logger`, `print`, or `assertionFailure` anywhere in the target. It is why every one of these bugs was invisible. Deliberately deferred to its own plan: an app-wide logging facility is cross-cutting, and folding it in would make this plan about two things.
- **Unstructured `Task {}` lifetimes (CR #5).** Six `Task { }` closures survive `SignedInScreens` being destroyed and keep the view model alive. Real mechanism — a `createSession()` in flight across a user change can complete as the new user and put a Stripe sheet over their screen, billing them — but a narrow window, and the fix changes how every button invokes work.
- **`23505` constraint-name matching (CR).** The webhook maps SQLSTATE `23505` to "duplicate delivery". Verified empirically correct against the current schema; it becomes wrong if any unique index is later added to `verification_sessions` or `profiles`. Matching the constraint name would make it robust rather than incidentally right.
- **`didCreateAccount` never cleared after profile setup (CR, Suspicion 1).** Combined with an email-fallback divergence between `currentUser`/stream (`?? ""`) and `signIn`/`signUp` (`?? email`), a non-equal session emission could route a signed-up user back to `.profileSetup`. Unreachable today.
- **`default:` in `SignedInScreens.body` (CR, Suspicion 2).** Masks nothing today; removes the compile-time forcing function when `Route` gains a case.

## Deployment note — not a task

`0003` drops the five-argument `apply_verification_outcome` **and** revokes `insert, delete on stripe_webhook_events from service_role`. Migration and Edge Functions are therefore mutually dependent and **must deploy together**:

- migration first, functions later → the old handler's own ledger insert hits `permission denied` → 500
- functions first, migration later → the new handler passes 7 args to a 5-arg function → `PGRST202` → 500

Both self-heal because Stripe retries for up to 3 days, but there is a window where every webhook fails. Also expect a few `PGRST202` until PostgREST reloads its schema cache after the function is recreated.

---

## File Structure

**backend** (`/Users/jonathan.aguele/Developer/coterie/backend`)
- `supabase/functions/_shared/reconcile.ts` — MODIFY. `Admin` interface gains `error` on the select; `latestSessionID` throws.
- `supabase/functions/sync-verification-status/index.ts` — MODIFY. Wrap the read and the reconcile; 503 rather than reporting a status derived from a read that did not happen.
- `supabase/functions/_shared/reconcile_test.ts` — MODIFY. The fake must be able to express a failing select.
- `.github/workflows/db-test.yml` — MODIFY. Gate `make fn-test`.

**ios** (`/Users/jonathan.aguele/Developer/coterie/ios`)
- `Coterie/Features/Verification/VerificationViewModel.swift` — MODIFY. Clear `actionError`; poll restart trigger; stop flapping `state`; mapped message in `load()`.
- `Coterie/Features/Verification/VerificationIntroView.swift` — MODIFY. Key the poll task on the new trigger.
- `Coterie/Features/Verification/SupabaseVerificationRepository.swift` — MODIFY. Map `verification_refused`.
- `Coterie/Features/Verification/VerificationRepository.swift` — MODIFY. New error case.
- `Coterie/Features/Profile/ProfileRepository.swift` — MODIFY. Add `ProfileRepositoryError`.
- `Coterie/Features/Profile/SupabaseProfileRepository.swift` — MODIFY. Map errors.
- `Coterie/Features/Profile/ProfileViewModel.swift` — MODIFY. Use the mapped message.
- `Coterie/Features/Auth/AuthRepository.swift` — MODIFY. `currentUser` becomes throwing.
- `Coterie/Features/Auth/SupabaseAuthRepository.swift` — MODIFY. Stop swallowing with `try?`.
- `Coterie/Features/Auth/AuthViewModel.swift` — MODIFY. A failed read is not evidence of sign-out.
- `Coterie/App/RootView.swift` — MODIFY. `try await` on `currentUser`.
- `CoterieTests/Mocks/MockAuthRepository.swift` — MODIFY. `currentUser` can fail.
- `CoterieTests/VerificationPollingTests.swift` — SPLIT. Over the 400-line limit; extract the poll-lifecycle cases into `CoterieTests/VerificationPollLifecycleTests.swift`.
- `Makefile` — MODIFY. Lint fails on warnings.

---

### Task 1: Restore a working local environment and a dev account

The local database was reset during review, so `auth.users` is empty: `admin@yahoo.com`, `agent1/2/3@example.com` and `jona2k5@yahoo.com` are all gone, along with their Stripe session history. Without an account there is no way to exercise any fix beyond unit tests, and the lesson of the first remediation is that green tests were not enough.

**Files:** none — environment only.

**Interfaces:**
- Produces: a signed-in-able account `dev@example.com` / `Passw0rd123`, and a running local stack. Later tasks' manual verification steps assume both.

- [ ] **Step 1: Start the stack and confirm the migrations are applied**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
make db
docker exec supabase_db_coterie psql -U postgres -d postgres -c \
  "select version from supabase_migrations.schema_migrations order by version;"
```

Expected: rows including `0001`, `0002`, `0003`. If `0003` is missing, run `supabase migration up`.

- [ ] **Step 2: Confirm the account table really is empty**

```bash
docker exec supabase_db_coterie psql -U postgres -d postgres -t -c "select count(*) from auth.users;"
```

Expected: `0`. If it is not 0, do not delete anything — record what is there and carry on.

- [ ] **Step 3: Create the dev account through GoTrue, not SQL**

Going through the API means the same triggers and profile row creation happen as for a real sign-up.

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
ANON=$(grep '^SUPABASE_ANON_KEY' ../ios/Config/Debug.xcconfig | sed 's/.*= //')
curl -s -X POST "http://127.0.0.1:54321/auth/v1/signup" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"dev@example.com","password":"Passw0rd123"}'
docker exec supabase_db_coterie psql -U postgres -d postgres -c \
  "update profiles set display_name = 'Dev' where id = (select id from auth.users where email = 'dev@example.com');"
```

- [ ] **Step 4: Prove the account can sign in and that a profile row exists**

```bash
ANON=$(grep '^SUPABASE_ANON_KEY' ../ios/Config/Debug.xcconfig | sed 's/.*= //')
curl -s -X POST "http://127.0.0.1:54321/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"dev@example.com","password":"Passw0rd123"}' | head -c 120
docker exec supabase_db_coterie psql -U postgres -d postgres -t -c \
  "select display_name, verification_status from profiles where id = (select id from auth.users where email='dev@example.com');"
```

Expected: an `access_token` in the first output; `Dev | unverified` in the second.

- [ ] **Step 5: Start the Edge Functions and confirm both are served**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
nohup supabase functions serve --env-file supabase/functions/.env > /tmp/fserve.log 2>&1 &
sleep 14
grep -E "create-verification-session|sync-verification-status" /tmp/fserve.log
```

Expected: both function URLs listed. `functions serve` is not part of `supabase start`, so it must be started by hand every session — this is the cause of an earlier 500 that looked like an app bug.

- [ ] **Step 6: Record the account in the plan and stop**

Append to this file under Task 1: the account created, and that `jona2k5@yahoo.com`'s Stripe history is unrecoverable. Report to the human that the dev accounts were destroyed by the review and what replaced them. No commit — nothing in the repo changed.

---

### Task 2: Make the `verification_sessions` read's error reachable and act on it

Fixes **F1**. Both reviewers found this independently; it is the only original finding still live, and it defeats the reconcile design.

**Files:**
- Modify: `backend/supabase/functions/_shared/reconcile.ts:17`, `:48-57`
- Modify: `backend/supabase/functions/sync-verification-status/index.ts:25-40`
- Test: `backend/supabase/functions/_shared/reconcile_test.ts`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: `latestSessionID(admin, userID)` now **throws** rather than returning `null` on a read failure. `sync-verification-status` returns `503 {"error":"status_unavailable"}` on any read or reconcile failure.

- [ ] **Step 1: Write the failing test**

Add to `backend/supabase/functions/_shared/reconcile_test.ts`. The existing `fakeAdmin` cannot express a failing select, which is why no test caught this — so extend it first.

In `fakeAdmin`'s options add `selectError`, and change the `maybeSingle` to return it:

```ts
function fakeAdmin(opts: {
  latest?: string | null;
  selectError?: { code?: string; message: string } | null;
  rpcError?: { code?: string; message: string } | null;
} = {}) {
```

```ts
                      return {
                        maybeSingle: () =>
                          Promise.resolve({
                            data: opts.latest === undefined || opts.latest === null
                              ? null
                              : { stripe_session_id: opts.latest },
                            error: opts.selectError ?? null,
                          }),
                      };
```

Then the test:

```ts
Deno.test("latestSessionID throws rather than reporting no session", async () => {
  // A failed read used to be indistinguishable from "this user has never started". That made
  // sync-verification-status skip Stripe and answer 200 with the stored column — the exact
  // missed-webhook case it exists to fix, failing in the one place the client must trust.
  const { admin } = fakeAdmin({ selectError: { message: "statement timeout" } });

  await assertRejects(
    () => latestSessionID(admin, "user-1"),
    Error,
    "could not read verification sessions",
  );
});
```

- [ ] **Step 2: Run it and watch it fail**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend/supabase/functions
deno test _shared/reconcile_test.ts
```

Expected: FAIL — `latestSessionID` resolves with `null` instead of rejecting.

- [ ] **Step 3: Make the error reachable, then act on it**

In `reconcile.ts`, the `Admin` interface — add `error` to the select's return type:

```ts
          limit(n: number): {
            maybeSingle(): PromiseLike<{
              data: { stripe_session_id: string } | null;
              error: { code?: string; message: string } | null;
            }>;
          };
```

And `latestSessionID`:

```ts
export async function latestSessionID(admin: Admin, userID: string): Promise<string | null> {
  const { data, error } = await admin
    .from("verification_sessions")
    .select("stripe_session_id")
    .eq("user_id", userID)
    .order("created_at", { ascending: false })
    .limit(1)
    .maybeSingle();

  // Throws rather than returning null. A failed read and "never started verification" are
  // different facts, and treating them alike let the caller skip Stripe and then report the
  // stored column as though it had asked.
  if (error) {
    throw new Error(`could not read verification sessions: ${error.message}`);
  }

  return data?.stripe_session_id ?? null;
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
deno test _shared/reconcile_test.ts
```

Expected: PASS, 6 tests.

- [ ] **Step 5: Handle the throw in the handler**

`sync-verification-status/index.ts` has no try/catch at all, so both `latestSessionID` and `reconcile` now throw into nothing. Replace the body between the auth guard and the profile read:

```ts
    let lastErrorCode: string | null = null;
    let lastErrorReason: string | null = null;

    try {
      const sessionID = await latestSessionID(ctx.supabaseAdmin, userID);

      // A session that exists is reconciled first, so the column we are about to read is
      // current. `reconcile` throws if it could not store what Stripe said — reporting Stripe's
      // answer without having persisted it is how the app ends up claiming a state the database
      // denies.
      if (sessionID) {
        const session = await reconcile(stripe, ctx.supabaseAdmin, userID, sessionID);
        lastErrorCode = session.last_error?.code ?? null;
        lastErrorReason = session.last_error?.reason ?? null;
      }
    } catch (error) {
      // 503, never a status. This endpoint's only job is to be trustworthy about whether it
      // asked Stripe; answering 200 with the stored column after failing to ask is the failure
      // the whole reconcile design was built to prevent.
      console.error(`sync failed for ${userID}: ${error}`);
      return Response.json({ error: "status_unavailable" }, { status: 503 });
    }
```

- [ ] **Step 6: Verify live against the dev account from Task 1**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
ANON=$(grep '^SUPABASE_ANON_KEY' ../ios/Config/Debug.xcconfig | sed 's/.*= //')
TOK=$(curl -s -X POST "http://127.0.0.1:54321/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"dev@example.com","password":"Passw0rd123"}' \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['access_token'])")
curl -s -w "|%{http_code}\n" -X POST http://127.0.0.1:54321/functions/v1/sync-verification-status \
  -H "Authorization: Bearer $TOK" -H "apikey: $ANON"
```

Expected: `{"status":"unverified","last_error_code":null,"last_error_reason":null}|200` — a user with no session still gets a clean answer, not a 503.

- [ ] **Step 7: Mutation check**

Revert only the `if (error) throw` in `latestSessionID` and re-run `deno test _shared/reconcile_test.ts`. Expected: the named test fails. Restore.

- [ ] **Step 8: Commit**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
git add supabase/functions/_shared/reconcile.ts supabase/functions/_shared/reconcile_test.ts \
        supabase/functions/sync-verification-status/index.ts
git commit -m "fix(verification): a failed session read is not 'never started'

Both reviewers found this independently and it is the one original finding
the first remediation missed. latestSessionID destructured { data } only, and
the Admin interface gave the select no error field at all — so the failure was
unreachable, not merely unchecked, in the same file whose comment explains why
that is fatal.

A transient read failure therefore read as 'this user has never verified',
sync-verification-status skipped Stripe entirely, and it answered 200 with the
stored column. That is the missed-webhook case the endpoint exists for,
failing in the one place the client is required to trust — and because it was
a 200, no client error handling ran and nothing was logged.

It now throws, and the handler 503s rather than reporting a status derived
from a read that did not happen. The test fake could not express a failing
select, which is why the suite was silent; it can now."
```

---

### Task 3: Clear `actionError` on success

Fixes **F2**. Introduced by the first remediation.

**Files:**
- Modify: `ios/Coterie/Features/Verification/VerificationViewModel.swift`
- Test: `ios/CoterieTests/VerificationPollingTests.swift`

**Interfaces:**
- Consumes: nothing.
- Produces: `actionError` is nil after any successful `load()` or `refresh()`. `dismissActionError()` is deleted.

- [ ] **Step 1: Write the failing test**

Append to `ios/CoterieTests/VerificationPollingTests.swift`, inside `struct PollExhaustionTests`:

```swift
    /// A stale failure must not outlive the thing that failed. Hoisting the banner out of
    /// `case .unverified` is what made this visible: "You're verified" rendered above
    /// "Verification is unavailable right now."
    @Test func aSuccessfulRefreshClearsTheErrorFromBefore() async {
        let repo = MockVerificationRepository()
        let vm = makeViewModel(repo)
        await vm.load()

        await repo.setStubbedError(VerificationRepositoryError.serviceUnavailable)
        await vm.refresh()
        #expect(vm.actionError != nil, "precondition: an error is showing")

        await repo.setStubbedError(nil)
        await repo.setStubbedSync(VerificationSync(status: .verified, refusal: nil))
        await vm.refresh()

        #expect(vm.state == .verified)
        #expect(vm.actionError == nil, "the error belonged to the attempt that failed")
    }
```

- [ ] **Step 2: Run it and watch it fail**

```bash
cd /Users/jonathan.aguele/Developer/coterie/ios
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | grep -aE "recorded an issue"
```

Expected: FAIL on `vm.actionError == nil`.

- [ ] **Step 3: Clear it where work begins**

In `VerificationViewModel.swift`, add as the first statement after the re-entrancy guard in `load()`:

```swift
        // Cleared when work starts, not left for the next screen. An error belongs to the
        // attempt that produced it; carrying it forward put a failure banner under a success.
        actionError = nil
```

And as the first statement inside `refresh()` after its guard and `defer`:

```swift
        actionError = nil
```

Then delete `dismissActionError()` entirely — it has no caller and `OursMessageBanner(_ message: String, tone:)` accepts no dismiss handler, so it can never acquire one without a design-system change.

- [ ] **Step 4: Run the tests to verify they pass**

```bash
make test
```

Expected: exit 0.

- [ ] **Step 5: Mutation check**

Remove the `actionError = nil` from `refresh()`. Run `make test`. Expected: `aSuccessfulRefreshClearsTheErrorFromBefore` fails. Restore.

- [ ] **Step 6: Commit**

```bash
git add Coterie/Features/Verification/VerificationViewModel.swift CoterieTests/VerificationPollingTests.swift
git commit -m "fix(verification): an error belongs to the attempt that produced it

actionError was set by load, refresh and the poll and cleared by none of
them, so once shown it survived for the life of the view model. Hoisting the
banner out of case .unverified — done to make refresh failures visible at all
— is what turned that into 'You're verified' rendered above 'Verification is
unavailable right now.'

Cleared when work begins. dismissActionError went with it: it had no caller,
and OursMessageBanner takes no dismiss handler, so it could never have had one."
```

---

### Task 4: Make the poll's lifecycle honest

Fixes **F3** and **F4** together — they are one mechanism, and fixing either alone leaves the screen lying about whether anything is watching.

**Files:**
- Modify: `ios/Coterie/Features/Verification/VerificationViewModel.swift`
- Modify: `ios/Coterie/Features/Verification/VerificationIntroView.swift`
- Create: `ios/CoterieTests/VerificationPollLifecycleTests.swift`

**Interfaces:**
- Consumes: Task 3's cleared `actionError`.
- Produces: `pollTrigger: PollTrigger` (an `Equatable` struct of `isPending` and `attempt`) which the view keys its poll task on. `refresh()` no longer touches `isPollExhausted`. `load()` no longer writes `.loading` over an answer already on screen.

- [ ] **Step 1: Write the failing tests**

Create `ios/CoterieTests/VerificationPollLifecycleTests.swift`. A new file rather than an addition: `VerificationPollingTests.swift` is already 495 lines against a 400 limit (see Task 8).

```swift
import Testing
import Foundation
@testable import Coterie

/// When the app is watching, and when it has stopped.
///
/// The screen must never present itself as watching while nothing is. Two ways it did: a
/// successful "Check again" cleared isPollExhausted, which removed the only button that could
/// ask again without restarting the poll; and a foreground reload flipped the state to .loading,
/// which cancelled and restarted the poll with a fresh 24-call budget.
@MainActor
struct VerificationPollLifecycleTests {
    private func profile(_ status: VerificationStatus) -> Profile {
        Profile(id: UUID(), displayName: "Jon", bio: "",
                avatarURL: nil, verificationStatus: status, role: .member)
    }

    private func makeViewModel(_ repo: MockVerificationRepository,
                               maxPolls: Int = 3) -> VerificationViewModel {
        VerificationViewModel(
            repo: MockProfileRepository(stubbedProfile: profile(.pending)),
            verification: repo,
            maxPolls: maxPolls,
            sleep: { _ in }
        )
    }

    /// The escape hatch must survive being used. It is the whole mechanism now that the
    /// staleness clock is gone.
    @Test func aRefreshThatIsStillPendingKeepsTheButtonAndRestartsTheWatch() async {
        let repo = MockVerificationRepository()
        await repo.setStubbedSync(VerificationSync(status: .pending, refusal: nil))
        let vm = makeViewModel(repo)
        await vm.load()
        await vm.pollWhilePending()
        #expect(vm.isPollExhausted, "precondition: the poll gave up")

        let before = vm.pollTrigger
        await vm.refresh()

        #expect(vm.isPollExhausted, "the button must not delete itself")
        #expect(vm.pollTrigger != before, "and the watch must be asked to start again")
    }

    /// Foregrounding must not silently buy another 24 calls, nor blank the footer mid-wait.
    @Test func foregroundingDoesNotResetThePollBudget() async {
        let repo = MockVerificationRepository()
        await repo.setStubbedSync(VerificationSync(status: .pending, refusal: nil))
        let vm = makeViewModel(repo)
        await vm.load()
        await vm.pollWhilePending()
        #expect(vm.isPollExhausted)

        let before = vm.pollTrigger
        await vm.refreshOnForeground()

        #expect(vm.isPollExhausted, "the exhausted footer must survive a foreground")
        #expect(vm.pollTrigger == before, "and the poll must not be restarted behind the user")
    }

    /// A reload must not flash a spinner over an answer already on screen.
    @Test func reloadingKeepsTheCurrentAnswerVisible() async {
        let repo = MockVerificationRepository()
        await repo.setStubbedSync(VerificationSync(status: .pending, refusal: nil))
        let vm = makeViewModel(repo)
        await vm.load()
        #expect(vm.state == .pending)

        await vm.refreshOnForeground()
        #expect(vm.state == .pending, "never .loading once an answer exists")
    }
}
```

- [ ] **Step 2: Run them and watch them fail**

```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | grep -aE "recorded an issue|error:"
```

Expected: compile failure on `vm.pollTrigger` (does not exist yet). That counts as failing — implement, then re-run.

- [ ] **Step 3: Add the trigger and stop `refresh()` clearing exhaustion**

In `VerificationViewModel.swift`, add near `isPollExhausted`:

```swift
    /// What the view keys its poll task on.
    ///
    /// `isPending` alone was not enough: a refresh that comes back "still pending" does not
    /// change it, so the task never re-ran and the button that triggered it had nothing to
    /// restart. `attempt` is bumped when someone asks again.
    struct PollTrigger: Equatable {
        let isPending: Bool
        let attempt: Int
    }

    var pollTrigger: PollTrigger { PollTrigger(isPending: isPending, attempt: pollAttempt) }

    private(set) var pollAttempt = 0
```

In `refresh()`, delete the `isPollExhausted = false` line and add, after `apply(sync)`:

```swift
            // Asked again, so watch again. Clearing `isPollExhausted` here instead would remove
            // the only control that can ask, without anything taking over the watching.
            if state == .pending { pollAttempt += 1 }
```

- [ ] **Step 4: Stop `load()` flapping the state**

In `load()`, delete the `state = .loading` line. The initial value of `state` is already `.loading`, so a first load still shows the spinner; a re-read now keeps the current answer on screen until a new one arrives. This is what stops `isPending` flipping false and cancelling, restarting, and re-budgeting the poll on every foreground.

- [ ] **Step 5: Key the view's task on the trigger**

In `VerificationIntroView.swift`, change the poll task:

```swift
        .task(id: viewModel.pollTrigger) {
            guard viewModel.isPending else { return }
            await viewModel.pollWhilePending()
        }
```

Update the comment above it so it explains the trigger rather than `isPending`.

- [ ] **Step 6: Run the tests to verify they pass**

```bash
xcodegen generate --quiet && make test
```

Expected: exit 0. `xcodegen` is required because a new test file was created — sources are globbed and the `.xcodeproj` is git-ignored.

- [ ] **Step 7: Mutation check, twice**

First: restore `isPollExhausted = false` in `refresh()`. Run `make test`. Expected: `aRefreshThatIsStillPendingKeepsTheButtonAndRestartsTheWatch` fails. Restore.

Second: put `state = .loading` back at the top of `load()`. Run `make test`. Expected: `foregroundingDoesNotResetThePollBudget` and/or `reloadingKeepsTheCurrentAnswerVisible` fail. Restore.

- [ ] **Step 8: Verify on the simulator**

Build and install, sign in as `dev@example.com` / `Passw0rd123`, reach the pending screen, and confirm: backgrounding and foregrounding does **not** flash a spinner, and the "Check again" button is still there afterwards. Report what you saw, not what you expected.

- [ ] **Step 9: Commit**

```bash
git add Coterie/Features/Verification/VerificationViewModel.swift \
        Coterie/Features/Verification/VerificationIntroView.swift \
        CoterieTests/VerificationPollLifecycleTests.swift
git commit -m "fix(verification): stop the screen claiming to watch when nothing is

Two faults, one mechanism.

A successful 'Check again' cleared isPollExhausted, which is the only thing
that renders the button — so the control deleted itself on use. The poll did
not restart to take over, because it was keyed on isPending, which a
still-pending answer does not change. The screen went back to 'we're checking
your ID' with nothing checking and no way to ask again. Removing the staleness
clock had made that button the whole mechanism.

And refreshOnForeground -> load() wrote .loading unconditionally, flipping
isPending false, which cancelled the poll and then restarted it with a fresh
24-call budget: ten foregrounds bought 240 reconciles against a documented
bound of 24, wiped the exhausted footer mid-wait, and flashed a spinner over
an answer already on screen.

The view now keys on a trigger carrying both isPending and an attempt count,
so asking again restarts the watch; refresh leaves isPollExhausted alone; and
load keeps the current answer visible while re-reading."
```

---

### Task 5: A failed sign-out must not sign the user out

Fixes **F5**. Introduced by the first remediation.

**Files:**
- Modify: `ios/Coterie/Features/Auth/AuthRepository.swift`
- Modify: `ios/Coterie/Features/Auth/SupabaseAuthRepository.swift`
- Modify: `ios/Coterie/Features/Auth/AuthViewModel.swift`
- Modify: `ios/Coterie/App/RootView.swift`
- Modify: `ios/CoterieTests/Mocks/MockAuthRepository.swift`
- Test: `ios/CoterieTests/AuthViewModelTests.swift`

**Interfaces:**
- Consumes: nothing.
- Produces: `AuthRepository.currentUser` becomes `{ get async throws }`. `MockAuthRepository` gains `currentUserError` and `setCurrentUserError(_:)`.

- [ ] **Step 1: Write the failing test**

The existing `aFailedSignOutAdoptsWhateverTheSessionActuallyIs` passes for the wrong reason — `stubbedUser` is nil, so `currentUser` returns nil whether or not the bug exists. Replace it in `ios/CoterieTests/AuthViewModelTests.swift`:

```swift
    /// A failed sign-out is not evidence that the session survived — but a failed *read* is not
    /// evidence that it died. `client.auth.session` performs a token refresh, so the network
    /// failure that makes sign-out fail is the same one that makes the read fail. Clearing the
    /// user on that basis showed people as signed out while their session stayed live on the
    /// server, and signed them back in on the next launch without authenticating.
    @Test func aFailedSignOutKeepsTheSessionWhenItCannotBeChecked() async {
        let repo = MockAuthRepository(shouldThrow: true)
        await repo.setStubbedUser(user)
        await repo.setCurrentUserError(MockAuthRepository.Failure.forced)
        let vm = AuthViewModel(repo: repo)
        vm.adopt(restoredSession: user)

        let succeeded = await vm.signOut()

        #expect(succeeded == false)
        #expect(vm.formError != nil, "and it says so")
        #expect(vm.authedUser != nil, "we could not confirm they are out, so they are not out")
    }

    /// When the read succeeds and reports nobody, the local session really is gone and the app
    /// must follow — that is the case the original fix was reaching for.
    @Test func aFailedSignOutFollowsAConfirmedEmptySession() async {
        let repo = MockAuthRepository(shouldThrow: true)
        let vm = AuthViewModel(repo: repo)
        vm.adopt(restoredSession: user)

        _ = await vm.signOut()

        #expect(vm.authedUser == nil, "the read succeeded and said nobody")
    }
```

- [ ] **Step 2: Run and watch it fail**

```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' 2>&1 | grep -aE "recorded an issue|error:"
```

Expected: compile failure on `setCurrentUserError` / `setStubbedUser`. Implement, then re-run.

- [ ] **Step 3: Make "could not determine" representable**

`AuthRepository.swift`:

```swift
    /// Throws when it cannot tell. Returning nil for both "nobody is signed in" and "the check
    /// failed" made a failed read look like a confirmed sign-out.
    var currentUser: AuthedUser? { get async throws }
```

`SupabaseAuthRepository.swift`:

```swift
    var currentUser: AuthedUser? {
        get async throws {
            do {
                let session = try await client.auth.session
                return AuthedUser(id: session.user.id, email: session.user.email ?? "")
            } catch let error as AuthError where error.isSessionMissing {
                // A genuinely absent session is an answer, not a failure.
                return nil
            }
        }
    }
```

If `AuthError.isSessionMissing` does not exist in supabase-swift 2.54.1, check what the SDK exposes and match on that instead; if nothing distinguishes them, return nil only for the documented "session missing" case and rethrow everything else. Do **not** fall back to `try?`.

- [ ] **Step 4: Only clear the user on a confirmed empty session**

`AuthViewModel.signOut()`'s catch:

```swift
        } catch {
            do {
                // Adopt the truth only if we could learn it. A failed read is not evidence.
                authedUser = try await repo.currentUser
                formError = "Couldn't sign out. Please try again."
            } catch {
                formError = "Couldn't sign out — you're still signed in on this device."
            }
            return false
        }
```

- [ ] **Step 5: Update the two other call sites**

`RootView.swift`'s first `.task`: `let user = try? await container.authRepository.currentUser` yields `AuthedUser??`. Write it explicitly instead:

```swift
        .task {
            let user: AuthedUser?
            do {
                user = try await container.authRepository.currentUser
            } catch {
                // Launch with nobody rather than block on it. The sessionChanges subscription
                // below will correct this the moment the SDK knows better.
                user = nil
            }
            authViewModel.adopt(restoredSession: user)
            coordinator.start(currentUser: user)
        }
```

`MockAuthRepository.swift`:

```swift
    var currentUserError: Error?
    func setCurrentUserError(_ error: Error?) { currentUserError = error }
    func setStubbedUser(_ user: AuthedUser?) { stubbedUser = user }

    var currentUser: AuthedUser? {
        get throws {
            if let currentUserError { throw currentUserError }
            return stubbedUser
        }
    }
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
make test
```

Expected: exit 0.

- [ ] **Step 7: Mutation check**

Change the catch in `signOut()` back to `authedUser = try? await repo.currentUser ?? nil`. Run `make test`. Expected: `aFailedSignOutKeepsTheSessionWhenItCannotBeChecked` fails. Restore.

- [ ] **Step 8: Commit**

```bash
git add Coterie/Features/Auth CoterieTests/Mocks/MockAuthRepository.swift \
        CoterieTests/AuthViewModelTests.swift Coterie/App/RootView.swift
git commit -m "fix(auth): a failed check is not a confirmed sign-out

The previous fix said 'ask, rather than assume' and then asked a question
whose failure answer was indistinguishable from 'nobody': currentUser was
try? -> nil, and client.auth.session performs a token refresh, so the network
failure that makes signOut throw is the same one that makes the read throw.

Result: tapping Sign out on a dead connection showed the user as signed out
while the refresh token stayed in the keychain and the session stayed live on
the server — then signed them back in on the next launch without
authenticating, with the 'Couldn't sign out' banner rendered on the sign-up
screen where it means nothing.

currentUser now throws, so 'could not determine' is representable, and the
user is only cleared when the read succeeds and reports nobody.

The old test passed for the wrong reason: stubbedUser was nil, so currentUser
returned nil whether or not the bug existed. Both directions are now covered."
```

---

### Task 6: Stop blaming the user's network for every error

Fixes **F6**.

**Files:**
- Modify: `ios/Coterie/Features/Profile/ProfileRepository.swift`
- Modify: `ios/Coterie/Features/Profile/SupabaseProfileRepository.swift`
- Modify: `ios/Coterie/Features/Profile/ProfileViewModel.swift`
- Modify: `ios/Coterie/Features/Verification/VerificationViewModel.swift`
- Test: `ios/CoterieTests/ProfileErrorMappingTests.swift` (create)

**Interfaces:**
- Consumes: nothing.
- Produces: `ProfileRepositoryError` with cases `notSignedIn`, `serviceUnavailable`, `offline`, `unknown`, each with an `errorDescription`, and `static func mapped(_ error: Error) -> ProfileRepositoryError`.

- [ ] **Step 1: Write the failing test**

Create `ios/CoterieTests/ProfileErrorMappingTests.swift`, mirroring `VerificationErrorMappingTests`:

```swift
import Testing
import Foundation
@testable import Coterie

/// Pure mapping, no network. The profile read is the one call behind "Check your connection and
/// try again." — a message it produced for an expired JWT, an RLS denial and a 500 alike, behind
/// a retry button that would fail identically forever.
struct ProfileErrorMappingTests {
    @Test func anExpiredSessionAsksTheUserToSignInAgain() {
        #expect(SupabaseProfileRepository.mapped(mockPostgrestError(code: 401)) == .notSignedIn)
    }

    @Test func aServerFaultIsNotTheUsersConnection() {
        #expect(SupabaseProfileRepository.mapped(mockPostgrestError(code: 500)) == .serviceUnavailable)
    }

    @Test func genuineConnectivityIsReportedAsConnectivity() {
        #expect(SupabaseProfileRepository.mapped(URLError(.notConnectedToInternet)) == .offline)
    }

    /// A broken wire contract must not read as a network problem — retrying will never fix it.
    @Test func aDecodingFailureIsNotAConnectionProblem() {
        let decoding = DecodingError.keyNotFound(
            AnyCodingKey(stringValue: "verification_status"),
            .init(codingPath: [], debugDescription: "")
        )
        #expect(SupabaseProfileRepository.mapped(decoding) != .offline)
    }

    private struct AnyCodingKey: CodingKey {
        var stringValue: String
        var intValue: Int? { nil }
        init(stringValue: String) { self.stringValue = stringValue }
        init?(intValue: Int) { nil }
    }
}
```

Implementing `mockPostgrestError(code:)` requires knowing what `SupabaseProfileRepository` actually catches. **Read that file first.** If it surfaces `PostgrestError`, construct one; if it surfaces `URLError` and generic `Error` only, drop the two HTTP cases and keep the connectivity and decoding ones — do not invent a type that does not exist.

- [ ] **Step 2: Run and watch it fail**

Expected: compile failure — `SupabaseProfileRepository.mapped` does not exist.

- [ ] **Step 3: Add the mapped error type**

In `ProfileRepository.swift`, add `ProfileRepositoryError` following `VerificationRepositoryError`'s shape exactly (same cases, same message style). In `SupabaseProfileRepository.swift`, add `static func mapped(_ error: Error) -> ProfileRepositoryError` and a private `mappingErrors` wrapper around each call, mirroring `SupabaseVerificationRepository:42-48`. Keep `.offline` for genuine connectivity only — `notConnectedToInternet`, `networkConnectionLost`, `timedOut`, `cannotFindHost`, `cannotConnectToHost`, `dataNotAllowed`.

- [ ] **Step 4: Use it in both view models**

`ProfileViewModel.swift:81` and `VerificationViewModel.swift:164` both hardcode the literal. Replace with the mapped description. `VerificationViewModel` already has the helper:

```swift
        } catch {
            state = .failed(message(for: error) ?? "Something went wrong. Please try again.")
        }
```

and extend `message(for:)` to try `ProfileRepositoryError` as well as `VerificationRepositoryError`.

- [ ] **Step 5: Run the tests to verify they pass**

```bash
xcodegen generate --quiet && make test
```

- [ ] **Step 6: Mutation check**

Make `mapped` return `.offline` unconditionally. Run `make test`. Expected: at least two named tests fail. Restore.

- [ ] **Step 7: Commit**

```bash
git add Coterie/Features/Profile Coterie/Features/Verification/VerificationViewModel.swift \
        CoterieTests/ProfileErrorMappingTests.swift
git commit -m "fix(profile): map the profile read's errors instead of blaming the wifi

'Check your connection and try again.' was hardcoded for an error nobody
inspected, in the one call every verification screen depends on. An expired
JWT, an RLS denial after a policy change, a 500 and a renamed column all told
the user to check their connection, behind a retry that would fail
identically forever. refresh was fixed to surface the mapped error; load —
called from four places — was not, because ProfileRepository had no mapped
error type at all.

It has one now, shaped like VerificationRepositoryError, and both view models
use it."
```

---

### Task 7: Map the `verification_refused` conflict

Fixes **F7**.

**Files:**
- Modify: `ios/Coterie/Features/Verification/VerificationRepository.swift`
- Modify: `ios/Coterie/Features/Verification/SupabaseVerificationRepository.swift`
- Test: `ios/CoterieTests/VerificationErrorMappingTests.swift`

**Interfaces:**
- Consumes: nothing.
- Produces: `VerificationRepositoryError.refused` with an honest description.

- [ ] **Step 1: Write the failing test**

Add to `ios/CoterieTests/VerificationErrorMappingTests.swift`:

```swift
    /// The backend now returns three distinct 409 bodies. Mapping an unrecognised one to
    /// `.verificationInProgress` means a terminally refused user is told "We're still checking
    /// your ID" — the exact lie the enum was split apart to prevent.
    @Test func aRefusalIsNotAReviewInProgress() {
        #expect(SupabaseVerificationRepository.mapped(
            http(409, body: #"{"error":"verification_refused"}"#)) == .refused)
    }
```

- [ ] **Step 2: Run and watch it fail**

Expected: compile failure — `.refused` does not exist.

- [ ] **Step 3: Add the case and map it**

`VerificationRepository.swift`:

```swift
    /// Stripe refused, and another attempt cannot change that. Distinct from
    /// `verificationInProgress`: both arrive as 409 and mean opposite things to the user.
    case refused
```

with a description of `"We couldn't verify your identity."` — deliberately not offering a retry, because the reason lives on the session row and the screen renders it.

In `conflict(from:)`:

```swift
        switch code {
        case "already_verified": return .alreadyVerified
        case "verification_refused": return .refused
        // Anything unrecognised falls towards in-progress: it understates access rather than
        // granting it, and the screen it lands on re-reads the real status.
        default: return .verificationInProgress
        }
```

- [ ] **Step 4: Handle it where the conflicts are handled**

In `VerificationViewModel.startVerification()`, add `.refused` alongside the other two conflict cases that call `load()` — a refusal is not a failure to report, it is a state to re-read, and `load()` will surface the reason from the session row.

- [ ] **Step 5: Run tests, mutation-check, commit**

```bash
make test
```

Then map `verification_refused` back to the default and confirm `aRefusalIsNotAReviewInProgress` fails. Restore.

```bash
git add Coterie/Features/Verification CoterieTests/VerificationErrorMappingTests.swift
git commit -m "fix(verification): a refusal is not a review in progress

The backend returns three 409 bodies now; the app recognised one and mapped
the rest to verificationInProgress, whose copy is 'We're still checking your
ID.' For a terminal refusal that is the precise lie the enum was split apart
to prevent. Masked today only because startVerification treats that case as
're-read rather than report'."
```

---

### Task 8: Clear the lint regressions and make `make lint` able to fail

Fixes **F8**.

**Files:**
- Modify: `ios/Coterie/Features/Verification/VerificationViewModel.swift` (lines 42, 50, 67)
- Modify: `ios/Coterie/Features/Profile/ProfileViewModel.swift` (line 57)
- Modify: `ios/CoterieTests/VerificationPollingTests.swift` (line 371, and length)
- Modify: `ios/Makefile`

**Interfaces:**
- Consumes: Task 4's `VerificationPollLifecycleTests.swift`, which already removes some length from `VerificationPollingTests.swift`.
- Produces: `make lint` exits non-zero on warnings.

- [ ] **Step 1: See the current state**

```bash
cd /Users/jonathan.aguele/Developer/coterie/ios
swiftlint lint --quiet
```

Expected: 7 warnings — 6 from the first remediation's deletions, plus a pre-existing `large_tuple` at `VerificationViewModelTests.swift:89`.

- [ ] **Step 2: Fix the whitespace, and split the oversized test file**

Collapse the double blank lines at the four `vertical_whitespace` sites. For `file_length`: after Task 4 moved the lifecycle cases out, check the count. If `VerificationPollingTests.swift` is still over 400, move `PollExhaustionTests` into `CoterieTests/VerificationPollExhaustionTests.swift` — it is a self-contained `struct` with its own helpers.

For the pre-existing `large_tuple` at `VerificationViewModelTests.swift:89`, replace the 3-tuple return with a small named struct:

```swift
    private struct Harness {
        let viewModel: VerificationViewModel
        let presenter: StubPresenter
        let verification: MockVerificationRepository
    }
```

- [ ] **Step 3: Make the target able to fail**

`ios/Makefile` — change the lint target:

```makefile
lint:
	swiftlint lint --quiet --strict
```

`--strict` promotes warnings to errors, so the target exits non-zero. This is why six warnings accumulated unnoticed: the target reported success and CI believed it.

- [ ] **Step 4: Verify**

```bash
xcodegen generate --quiet && make lint && echo "LINT CLEAN" && make test
```

Expected: `LINT CLEAN`, then tests exit 0. If `make lint` fails, fix the reported warnings — do not weaken `--strict`.

- [ ] **Step 5: Commit**

```bash
git add Coterie CoterieTests Makefile
git commit -m "chore(ios): clear the lint regressions and let make lint fail

Six warnings accumulated during the first remediation — five blank lines left
by deletions, and a test file grown past the 400-line limit — while every
report said 'lint errors: 0'. Both were true and both were misleading: the
checks looked for 'error:' and swiftlint --quiet exits 0 on warnings, so
neither the local target nor CI could say otherwise.

--strict makes the target able to fail, which is the only reason to run it."
```

---

### Task 9: Gate the Deno handler tests in CI

Fixes **F9**.

**Files:**
- Modify: `backend/.github/workflows/db-test.yml`

**Interfaces:**
- Consumes: Task 2's new test.
- Produces: CI runs `make fn-test`.

- [ ] **Step 1: Read the workflow and confirm the gap**

```bash
cd /Users/jonathan.aguele/Developer/coterie/backend
cat .github/workflows/db-test.yml
grep -n "fn-test" Makefile
```

Expected: the workflow runs `supabase test db` and never `make fn-test`.

- [ ] **Step 2: Add the step**

Add a step after the pgTAP one, using the same Deno setup the `fn-test` target expects. If Deno is not already available in the runner, add `denoland/setup-deno@v1` before it. Name the step so a failure is legible: `Edge Function handler tests`.

- [ ] **Step 3: Verify locally, since CI cannot be run from here**

```bash
make fn-test
```

Expected: all Deno tests pass, including Task 2's. State plainly in the commit that the workflow change itself is unverified until it runs on GitHub.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/db-test.yml
git commit -m "ci(backend): gate the Edge Function handler tests

db-test.yml ran supabase test db only, so the handler tests — added precisely
because every backend bug the review found lived in the code that talks to
the database — were not gated by anything. make fn-test already existed.

The workflow change is unverified until it runs on GitHub; make fn-test passes
locally."
```

---

## Self-Review

**Findings coverage.** F1→Task 2. F2→Task 3. F3, F4→Task 4. F5→Task 5. F6→Task 6. F7→Task 7. F8→Task 8. F9→Task 9. Environment→Task 1. Deferred items are listed with reasons and deliberately have no task. Every finding is either implemented or explicitly deferred; none is silently dropped.

**Placeholder scan.** No TBDs. Two tasks contain a conditional instruction rather than fixed code — Task 5 Step 3 (`AuthError.isSessionMissing` may not exist in supabase-swift 2.54.1) and Task 6 Step 1 (the error type `SupabaseProfileRepository` actually surfaces). Both name the file to read, state the decision rule, and forbid the wrong answer (`try?`, inventing a type). That is deliberate: guessing an SDK API in a plan is how a plan becomes fiction.

**Type consistency.** `PollTrigger`/`pollTrigger`/`pollAttempt` are introduced in Task 4 and used only there and in `VerificationIntroView`. `ProfileRepositoryError` and `SupabaseProfileRepository.mapped` are introduced in Task 6 and used in Task 6. `.refused` is introduced and consumed in Task 7. `setCurrentUserError`/`setStubbedUser` are introduced in Task 5 Step 5 and used in Step 1 — the implementer hits the compile failure first, which Step 2 states as the expected failure.

**Ordering.** Task 1 must run first; nothing else can be verified live without it. Task 3 before Task 4 (both touch `refresh()`; Task 4's tests assume `actionError` is cleared). Task 4 before Task 8 (the file split reduces the length that Task 8 checks). Tasks 5, 6, 7 are independent of each other. Task 9 after Task 2.
