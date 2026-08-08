# Coterie Plan 1 — Foundation & Auth — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Coterie iOS app + local Supabase backend so a user can see the first-run intro, sign up, sign in (session persists), and complete their profile.

**Architecture:** SwiftUI MVVM + coordinators. View models are `@Observable`/`@MainActor` and depend on repository *protocols*. Repository implementations own the Supabase SDK and the DTO/DTI + mapper layer. Manual DI via an `AppContainer` in the SwiftUI environment. Backend is local Supabase (Postgres/Auth) with migrations-as-code.

**Tech Stack:** Swift 6 / SwiftUI (iOS 18), XcodeGen, Supabase Swift SDK, Supabase CLI (local Docker), Swift Testing, SwiftLint.

## Global Constraints

- **Three separate repos** under a plain `coterie/` container directory: `docs`, `ios`, `backend`. Each repo is self-contained (own `Makefile`, `Brewfile`, `.gitignore`, `README.md`) and clonable standalone. Paths below are repo-qualified; commands run from inside the named repo.
- **Min iOS deployment target: 18.0.** Single app target; folders (not SPM packages) for grouping; boundaries enforced via repository protocols.
- **Project generation: XcodeGen** — `project.yml` is source of truth; `.xcodeproj` is git-ignored and generated.
- **MVVM + coordinators.** Views hold no business logic or networking. View models depend on protocols, never `SupabaseClient`.
- **Four-tier data layering:** domain models (used by view models/UI) · DTO (`Decodable`, inbound) · DTI (`Encodable`, outbound) · mappers between them. DTO/DTI/mappers live *inside* repository implementations only.
- **DTO/DTI properties are camelCase with an explicit `CodingKeys` mapping to the snake_case column names.** Swift identifiers containing underscores are rejected by the project's SwiftLint `identifier_name` rule, so mirroring Postgres column names directly does not compile clean. Because the wire contract then lives in `CodingKeys` rather than in the property names, each DTO/DTI needs a round-trip test asserting the snake_case JSON keys — otherwise a rename silently breaks the API contract with no test failure.
- **Concurrency:** `async/await`; view models `@MainActor`.
- **Testing:** Swift Testing (`@Test`/`#expect`); view-model tests run against mock repositories; DB tests via pgTAP (`supabase test db`). **No snapshot tests.**
- **Where a mock records call counts, assert them.** Verifying only the *result* of a call leaves branch-routing untested — both branches of `AuthViewModel.submit()` return an `AuthedUser` with the same email, so a `signUp` branch wrongly calling `signIn` passed every result-based test. Confirmed by mutation: the plan's five original tests all stayed green under that bug.
- **No code comments** unless explicitly required.
- **Roles/verification are columns from day one:** `profiles.role` (`member|admin`), `profiles.verification_status` (`unverified|pending|verified`).
- **Every migration that creates a table must GRANT explicitly.** In this Supabase version the default ACL for `public` tables owned by `postgres` gives `anon`/`authenticated` only `Dxtm` (truncate/references/trigger/maintain) — **no SELECT/INSERT/UPDATE/DELETE**. A table with RLS policies but no grants is unreadable by the app, and its policies never evaluate. Grant `select` for readable tables, and scope `update` to a **column list** so privilege-bearing columns (`role`, `verification_status`, `rating_*`) cannot be self-assigned. RLS scopes *rows*; GRANTs scope *columns* — both are required.
- **pgTAP runs as `postgres`, a superuser that bypasses RLS and grants.** Assertions must therefore check the `authenticated` role explicitly via `has_table_privilege` / `has_column_privilege`, or switch role with `set local role authenticated` plus `set local request.jwt.claims`. A suite that only inspects schema will pass against a table the app cannot actually use.
- **Verified-only actions come later** (Plan 2+); Plan 1 only establishes the columns and defaults (`unverified`, `member`).
- **Simulator for tests:** `platform=iOS Simulator,name=iPhone 17 Pro`.
- **Commits:** conventional-commit messages; **no Claude co-author trailers.**

---

## File Structure

Three sibling git repositories under a plain `coterie/` container directory. Paths in
this plan are repo-qualified; each task's commands run from inside the repo it names.

```
coterie/
  docs/                                      # repo — this plan lives here
    specs/2026-08-07-coterie-mvp-design.md
    plans/2026-08-07-plan-1-foundation-and-auth.md

  ios/                                       # repo — SwiftUI app
    project.yml                              # XcodeGen (Task 1)
    .gitignore .swiftlint.yml                # (Task 1)
    Brewfile Makefile .xcode-version         # (Task 0)
    scripts/bootstrap.sh                     # (Task 0)
    Config/Debug.xcconfig                    # (Task 1, key filled Task 2)
    Config/Release.xcconfig                  # (Task 1, key filled Task 2)
    Coterie/
      App/
        CoterieApp.swift                     # @main entry (Task 1, updated Task 11)
        RootView.swift                       # root routing view (Task 1, updated Task 11)
        AppContainer.swift                   # manual DI container (Task 11)
        AppCoordinator.swift                 # navigation/route state (Task 11)
      Core/
        Config/AppEnvironment.swift          # dev/QA/prod config (Task 2)
        Supabase/SupabaseClientProvider.swift# builds SupabaseClient (Task 2)
        Models/
          Profile.swift                      # domain (Task 4)
          AuthedUser.swift                   # domain (Task 5)
      Features/
        Onboarding/
          IntroPage.swift                    # model (Task 10)
          IntroViewModel.swift               # (Task 10)
          IntroCarouselView.swift            # (Task 10)
        Auth/
          AuthRepository.swift               # protocol (Task 5)
          SupabaseAuthRepository.swift       # impl + DTO/DTI/mapper (Task 6)
          AuthViewModel.swift                # (Task 5)
          SignInView.swift                   # (Task 7)
          SignUpView.swift                   # (Task 7)
        Profile/
          ProfileRepository.swift            # protocol (Task 8)
          ProfileDTO.swift                   # DTO/DTI + mapper (Task 4)
          SupabaseProfileRepository.swift    # impl (Task 8)
          ProfileViewModel.swift             # (Task 8)
          ProfileSetupView.swift             # (Task 9)
      Resources/
        Assets.xcassets/                     # (Task 1)
    CoterieTests/
      Mocks/
        MockAuthRepository.swift             # (Task 5)
        MockProfileRepository.swift          # (Task 8)
      AppEnvironmentTests.swift              # (Task 2)
      ProfileMapperTests.swift               # (Task 4)
      AuthViewModelTests.swift               # (Task 5)
      ProfileViewModelTests.swift            # (Task 8)
      IntroViewModelTests.swift              # (Task 10)
      AppCoordinatorTests.swift              # (Task 11)

  backend/                                   # repo — Supabase schema
    Brewfile Makefile .gitignore             # (Task 0)
    supabase/
      config.toml                            # (Task 2)
      migrations/
        0001_profiles.sql                    # (Task 3)
      tests/
        0001_profiles_test.sql               # pgTAP (Task 3)
      seed.sql                               # (Task 3)
```

---

## Task 0: Repos & reproducible dev environment ✅ DONE

**Goal:** three self-contained repos, each of which a fresh clone can bring up on any Mac
with a single command. All tooling declared per repo, Docker runtime included, no hidden
per-machine steps.

**Files:**
- Create: `docs/README.md`, `docs/.gitignore`
- Create: `ios/Brewfile`, `ios/Makefile`, `ios/.xcode-version`, `ios/scripts/bootstrap.sh`, `ios/README.md`
- Create: `backend/Brewfile`, `backend/Makefile`, `backend/.gitignore`, `backend/README.md`

**Interfaces:**
- Produces: `ios` → `make setup` (bootstrap → generate), `make build`, `make test`, `make lint`.
- Produces: `backend` → `make setup` (bootstrap → docker → db), `make db`, `make db-reset`, `make db-test`, `make db-stop`, `make db-status`.

- [x] **Step 1: Create the container directory and three repos**

```bash
mkdir -p ~/Developer/coterie/{docs,ios,backend}
```
Each of `docs`, `ios`, `backend` is its own `git init` repo. `coterie/` itself is a plain
folder, not a repo.

- [x] **Step 2: Create `ios/Brewfile`**

```ruby
brew "xcodegen"
brew "swiftlint"
```

- [x] **Step 3: Create `ios/.xcode-version`**

```
26.2
```

- [x] **Step 4: Create `ios/Makefile`**

```makefile
SCHEME = Coterie
DESTINATION = platform=iOS Simulator,name=iPhone 17 Pro

.PHONY: setup bootstrap generate build test lint

setup: bootstrap generate
	@echo "Coterie iOS ready. Run 'make test'."

bootstrap:
	brew bundle

generate:
	xcodegen generate

build:
	xcodebuild build -scheme $(SCHEME) -destination '$(DESTINATION)' -quiet

test:
	xcodebuild test -scheme $(SCHEME) -destination '$(DESTINATION)' -quiet

lint:
	swiftlint lint --quiet
```

- [x] **Step 5: Create `ios/scripts/bootstrap.sh` and make it executable**

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "→ Installing tools via Brewfile…"
brew bundle
echo "→ Generating Xcode project…"
xcodegen generate
echo "Done. Open Coterie.xcodeproj in Xcode, or run 'make test'."
echo "Note: the backend repo must be running for anything that talks to Supabase — see ../backend."
```
Then: `chmod +x ios/scripts/bootstrap.sh`

- [x] **Step 6: Create `backend/Brewfile`**

```ruby
tap "supabase/tap"
brew "supabase/tap/supabase"
brew "colima"
brew "docker"
```

- [x] **Step 7: Create `backend/Makefile`**

```makefile
.PHONY: setup bootstrap docker db db-reset db-test db-stop db-status

setup: bootstrap docker db
	@echo "Coterie backend ready. Run 'make db-status' for local credentials."

bootstrap:
	brew trust supabase/tap 2>/dev/null || true
	brew bundle

docker:
	colima status >/dev/null 2>&1 || colima start

db:
	supabase start

db-reset:
	supabase db reset

db-test:
	supabase test db

db-stop:
	supabase stop

db-status:
	supabase status
```

- [x] **Step 8: Create `backend/.gitignore`**

```gitignore
.DS_Store
supabase/.branches/
supabase/.temp/
.env
```

- [x] **Step 9: Create the three READMEs**

Each README states the three-repo table, prerequisites, one-command setup, and the
command reference for that repo. `backend/README.md` additionally records that migrations
are forward-only and that local Supabase keys are safe to commit while QA/prod keys are not.

- [x] **Step 10: Install tooling and start services**

Run:
```bash
cd ios && brew bundle
cd ../backend && brew bundle && colima start
```
Expected: `xcodegen`, `swiftlint`, `supabase`, `colima`, `docker` installed; colima running.

- [x] **Step 11: Commit in each repo**

```bash
git -C docs    add -A && git -C docs    commit -m "chore: docs repo with spec and Plan 1"
git -C ios     add -A && git -C ios     commit -m "chore: iOS repo scaffold and dev environment"
git -C backend add -A && git -C backend commit -m "chore: backend repo scaffold and dev environment"
```

---

## Task 1: iOS scaffold & XcodeGen project ✅ DONE

All paths in this task are inside the `ios` repo; run commands from `coterie/ios`.

**Files:**
- Create: `ios/project.yml`, `ios/.gitignore`, `ios/.swiftlint.yml`
- Create: `ios/Config/Debug.xcconfig`, `ios/Config/Release.xcconfig`
- Create: `ios/Coterie/App/CoterieApp.swift`, `ios/Coterie/App/RootView.swift`
- Create: `ios/Coterie/Resources/Assets.xcassets/` (empty asset catalog)

**Interfaces:**
- Produces: an app target named `Coterie` and a test target `CoterieTests`, buildable via `xcodebuild`.

- [x] **Step 1: Create `ios/.gitignore`**

```gitignore
.DS_Store
*.xcodeproj
*.xcworkspace
xcuserdata/
DerivedData/
.build/
Coterie/Info.plist
.env
```

- [x] **Step 2: Create `ios/.swiftlint.yml`**

```yaml
disabled_rules:
  - todo
opt_in_rules:
  - empty_count
  - force_unwrapping
line_length:
  warning: 140
  error: 200
identifier_name:
  min_length: 2
excluded:
  - DerivedData
  - .build
```

- [x] **Step 3: Create `ios/project.yml`**

```yaml
name: Coterie
options:
  bundleIdPrefix: com.coterie
  deploymentTarget:
    iOS: "18.0"
  createIntermediateGroups: true
settings:
  base:
    SWIFT_VERSION: "6.0"
    MARKETING_VERSION: "0.1.0"
    CURRENT_PROJECT_VERSION: "1"
packages:
  Supabase:
    url: https://github.com/supabase-community/supabase-swift
    from: "2.0.0"
targets:
  Coterie:
    type: application
    platform: iOS
    sources:
      - Coterie
    dependencies:
      - package: Supabase
        product: Supabase
    configFiles:
      Debug: Config/Debug.xcconfig
      Release: Config/Release.xcconfig
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.coterie.app
    info:
      path: Coterie/Info.plist
      properties:
        SUPABASE_URL: $(SUPABASE_URL)
        SUPABASE_ANON_KEY: $(SUPABASE_ANON_KEY)
        APP_ENV: $(APP_ENV)
        UILaunchScreen: {}
  CoterieTests:
    type: bundle.unit-test
    platform: iOS
    sources:
      - CoterieTests
    dependencies:
      - target: Coterie
schemes:
  Coterie:
    build:
      targets:
        Coterie: all
    test:
      targets:
        - CoterieTests
```

The generated `Coterie/Info.plist` is git-ignored — XcodeGen rewrites it from `project.yml`
on every `make generate`.

- [x] **Step 4: Create xcconfig files**

`ios/Config/Debug.xcconfig`:
```
SUPABASE_URL = http:$()/$()/127.0.0.1:54321
SUPABASE_ANON_KEY = REPLACE_WITH_LOCAL_ANON_KEY
APP_ENV = dev
```
`ios/Config/Release.xcconfig`:
```
SUPABASE_URL = http:$()/$()/127.0.0.1:54321
SUPABASE_ANON_KEY = REPLACE_WITH_LOCAL_ANON_KEY
APP_ENV = prod
```
(The `$()` splits `//` so xcconfig doesn't treat it as a comment. The anon key is filled in
Task 2 Step 2 after `make db` prints it.)

- [x] **Step 5: Create the minimal app entry**

`ios/Coterie/App/CoterieApp.swift`:
```swift
import SwiftUI

@main
struct CoterieApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
    }
}
```

`ios/Coterie/App/RootView.swift`:
```swift
import SwiftUI

struct RootView: View {
    var body: some View {
        Text("Coterie")
            .font(.largeTitle.bold())
    }
}
```

- [x] **Step 6: Create the empty asset catalog**

`ios/Coterie/Resources/Assets.xcassets/Contents.json`:
```json
{ "info": { "author": "xcode", "version": 1 } }
```

- [x] **Step 7: Generate the project and build**

Run from `coterie/ios`:
```bash
make generate
make build
```
Expected: `** BUILD SUCCEEDED **`.

- [x] **Step 8: Commit**

```bash
git add -A
git commit -m "chore: scaffold Coterie app with XcodeGen"
```

---

## Task 2: Supabase local stack & environment config ✅ DONE

**Repos:** `backend` (Steps 1) then `ios` (Steps 2–8). Each step says which.

**Files:**
- Create: `backend/supabase/config.toml` (via CLI)
- Create: `ios/Coterie/Core/Config/AppEnvironment.swift`
- Create: `ios/Coterie/Core/Supabase/SupabaseClientProvider.swift`
- Test: `ios/CoterieTests/AppEnvironmentTests.swift`

**Interfaces:**
- Produces: `AppEnvironment.current` → `{ supabaseURL: URL, supabaseAnonKey: String, name: String }`; `SupabaseClientProvider.make() -> SupabaseClient`.

- [x] **Step 1: Initialise and start local Supabase**

Run from `coterie/backend`:
```bash
supabase init
make db
```
Expected: `supabase init` creates `backend/supabase/config.toml`; `make db` prints
`API URL: http://127.0.0.1:54321` and an `anon key`. Copy the anon key
(`make db-status` reprints it). Commit `backend/supabase/config.toml`.

`supabase init` defaults `project_id` to the directory name (`backend`), which names the
local Docker containers. Change it to `coterie` so the containers are `supabase_*_coterie`
and cannot collide with another project checked out in a folder also called `backend`:

```toml
project_id = "coterie"
```
Changing it after `supabase start` requires `supabase stop` then `make db` again.

- [x] **Step 2: Paste the anon key into both xcconfig files**

In the `ios` repo, replace `REPLACE_WITH_LOCAL_ANON_KEY` in `ios/Config/Debug.xcconfig` and
`ios/Config/Release.xcconfig` with the key from Step 1, then run `make generate`.

- [x] **Step 3: Write the failing test for `AppEnvironment`**

`ios/CoterieTests/AppEnvironmentTests.swift`:
```swift
import Testing
import Foundation
@testable import Coterie

struct AppEnvironmentTests {
    @Test func currentEnvironmentHasValidURLAndKey() {
        let env = AppEnvironment.current
        #expect(env.supabaseURL.scheme == "http" || env.supabaseURL.scheme == "https")
        #expect(!env.supabaseAnonKey.isEmpty)
        #expect(!env.name.isEmpty)
    }
}
```

- [x] **Step 4: Run test to verify it fails**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AppEnvironmentTests -quiet
```
Expected: FAIL — `cannot find 'AppEnvironment' in scope`.

- [x] **Step 5: Implement `AppEnvironment`**

`ios/Coterie/Core/Config/AppEnvironment.swift`:
```swift
import Foundation

struct AppEnvironment {
    let name: String
    let supabaseURL: URL
    let supabaseAnonKey: String

    static let current: AppEnvironment = {
        let info = Bundle.main.infoDictionary ?? [:]
        let urlString = (info["SUPABASE_URL"] as? String) ?? ""
        let key = (info["SUPABASE_ANON_KEY"] as? String) ?? ""
        let name = (info["APP_ENV"] as? String) ?? "unknown"
        guard let url = URL(string: urlString) else {
            fatalError("Invalid SUPABASE_URL: \(urlString)")
        }
        return AppEnvironment(name: name, supabaseURL: url, supabaseAnonKey: key)
    }()
}
```

Add the keys to Info.plist generation in `project.yml` under `targets.Coterie.settings.base`:
```yaml
        INFOPLIST_KEY_SUPABASE_URL: $(SUPABASE_URL)
```
(already added in Task 1 Step 4 — confirm present). Re-run `xcodegen generate`.

- [x] **Step 6: Run test to verify it passes**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AppEnvironmentTests -quiet
```
Expected: PASS.

- [x] **Step 7: Implement `SupabaseClientProvider`**

`ios/Coterie/Core/Supabase/SupabaseClientProvider.swift`:
```swift
import Foundation
import Supabase

enum SupabaseClientProvider {
    static func make(environment: AppEnvironment = .current) -> SupabaseClient {
        SupabaseClient(
            supabaseURL: environment.supabaseURL,
            supabaseKey: environment.supabaseAnonKey
        )
    }
}
```

- [x] **Step 8: Build and commit both repos**

Run from `coterie/ios`:
```bash
make build
git add -A
git commit -m "feat: add environment config and Supabase client provider"
```
Run from `coterie/backend`:
```bash
git add -A
git commit -m "chore: initialise local Supabase project"
```

---

## Task 3: `profiles` migration, RLS & trigger (pgTAP) ✅ DONE

**Repo:** `backend` — run all commands from `coterie/backend`.

**Files:**
- Create: `backend/supabase/migrations/0001_profiles.sql`
- Create: `backend/supabase/tests/0001_profiles_test.sql`
- Modify: `backend/supabase/seed.sql`

**Interfaces:**
- Produces: `public.profiles` table with columns `id, display_name, bio, avatar_url, verification_status, role, rating_avg, rating_count, created_at`; a trigger that inserts a profile row on `auth.users` insert; RLS: anyone authenticated may `select`, only the owner may `update`.

- [x] **Step 1: Write the migration**

`backend/supabase/migrations/0001_profiles.sql`:
```sql
create type verification_status as enum ('unverified', 'pending', 'verified');
create type user_role as enum ('member', 'admin');

create table public.profiles (
    id uuid primary key references auth.users(id) on delete cascade,
    display_name text not null default '',
    bio text not null default '',
    avatar_url text,
    verification_status verification_status not null default 'unverified',
    role user_role not null default 'member',
    rating_avg numeric(3,2) not null default 0,
    rating_count integer not null default 0,
    created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;

grant select on public.profiles to authenticated;
grant update (display_name, bio, avatar_url) on public.profiles to authenticated;

create policy "profiles are readable by authenticated users"
    on public.profiles for select
    to authenticated
    using (true);

create policy "users can update their own profile"
    on public.profiles for update
    to authenticated
    using (auth.uid() = id)
    with check (auth.uid() = id);

create function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = public
as $$
begin
    insert into public.profiles (id, display_name)
    values (new.id, coalesce(new.raw_user_meta_data->>'display_name', ''));
    return new;
end;
$$;

create trigger on_auth_user_created
    after insert on auth.users
    for each row execute function public.handle_new_user();
```

- [x] **Step 2: Write the pgTAP test**

`backend/supabase/tests/0001_profiles_test.sql`:
```sql
begin;
select plan(13);

select has_table('public', 'profiles', 'profiles table exists');
select col_is_pk('public', 'profiles', 'id', 'id is primary key');
select col_default_is('public', 'profiles', 'verification_status', 'unverified', 'default verification is unverified');
select col_default_is('public', 'profiles', 'role', 'member', 'default role is member');

select is(
    (select relrowsecurity from pg_class where oid = 'public.profiles'::regclass),
    true,
    'RLS is enabled on profiles'
);

insert into auth.users (id, email) values ('00000000-0000-0000-0000-000000000001', 'a@b.com');
select is(
    (select count(*)::int from public.profiles where id = '00000000-0000-0000-0000-000000000001'),
    1,
    'inserting an auth user auto-creates a profile'
);

select ok(
    has_table_privilege('authenticated', 'public.profiles', 'SELECT'),
    'authenticated can read profiles'
);
select ok(
    has_column_privilege('authenticated', 'public.profiles', 'display_name', 'UPDATE'),
    'authenticated can update display_name'
);
select ok(
    has_column_privilege('authenticated', 'public.profiles', 'bio', 'UPDATE'),
    'authenticated can update bio'
);
select ok(
    not has_column_privilege('authenticated', 'public.profiles', 'role', 'UPDATE'),
    'authenticated cannot update role'
);
select ok(
    not has_column_privilege('authenticated', 'public.profiles', 'verification_status', 'UPDATE'),
    'authenticated cannot update verification_status'
);
select ok(
    not has_column_privilege('authenticated', 'public.profiles', 'rating_avg', 'UPDATE'),
    'authenticated cannot update rating_avg'
);
select ok(
    not has_column_privilege('authenticated', 'public.profiles', 'rating_count', 'UPDATE'),
    'authenticated cannot update rating_count'
);

select finish();
rollback;
```
**As executed:** `tests.rls_enabled` is *not* available — local Supabase has no `tests`
schema — so the fallback above was used, keyed on `'public.profiles'::regclass` rather than
`relname` to avoid matching a same-named table in another schema. The assertion count stays
at `plan(6)`.

No `create extension pgtap` is needed: `supabase test db` installs it into the running
database before executing the test files. (Verified — pgTAP reports `NOT INSTALLED`
immediately after `make db-reset`, and the suite still passes.)

- [x] **Step 3: Add a seed comment placeholder**

`backend/supabase/seed.sql`:
```sql
-- Seed data is added in later plans (pools, memberships). Intentionally empty for Plan 1.
```

- [x] **Step 4: Reset DB and run the migration test**

Run:
```bash
make db-reset
make db-test
```
Expected: `0001_profiles_test.sql .. ok` and all assertions pass.

- [x] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(db): profiles table with RLS and auto-create trigger"
```

---

## Task 4: Profile domain model, DTO/DTI & mapper ✅ DONE

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Core/Models/Profile.swift`
- Create: `ios/Coterie/Features/Profile/ProfileDTO.swift`
- Test: `ios/CoterieTests/ProfileMapperTests.swift`

**Interfaces:**
- Produces: domain `Profile`, `VerificationStatus`, `UserRole`; `ProfileDTO` (Decodable), `ProfileUpdateDTI` (Encodable), `ProfileMapper.domain(from:)`, `ProfileMapper.updateDTI(displayName:bio:)`.

- [x] **Step 1: Write the domain model**

`ios/Coterie/Core/Models/Profile.swift`:
```swift
import Foundation

enum VerificationStatus: String, Sendable, Equatable {
    case unverified, pending, verified
}

enum UserRole: String, Sendable, Equatable {
    case member, admin
}

struct Profile: Identifiable, Sendable, Equatable {
    let id: UUID
    var displayName: String
    var bio: String
    var avatarURL: URL?
    var verificationStatus: VerificationStatus
    var role: UserRole
}
```

- [x] **Step 2: Write the failing mapper test**

`ios/CoterieTests/ProfileMapperTests.swift`:
```swift
import Testing
import Foundation
@testable import Coterie

struct ProfileMapperTests {
    @Test func mapsDTOToDomain() throws {
        let dto = ProfileDTO(
            id: try #require(UUID(uuidString: "00000000-0000-0000-0000-000000000001")),
            displayName: "Jonathan",
            bio: "collector",
            avatarURL: "https://example.com/a.png",
            verificationStatus: "verified",
            role: "admin"
        )
        let profile = ProfileMapper.domain(from: dto)
        #expect(profile.displayName == "Jonathan")
        #expect(profile.bio == "collector")
        #expect(profile.avatarURL?.absoluteString == "https://example.com/a.png")
        #expect(profile.verificationStatus == .verified)
        #expect(profile.role == .admin)
    }

    @Test func unknownEnumStringsFallBackToSafeDefaults() {
        let dto = ProfileDTO(id: UUID(), displayName: "X", bio: nil,
                             avatarURL: nil, verificationStatus: "garbage", role: "garbage")
        let profile = ProfileMapper.domain(from: dto)
        #expect(profile.verificationStatus == .unverified)
        #expect(profile.role == .member)
        #expect(profile.bio == "")
    }

    @Test func buildsUpdateDTI() {
        let dti = ProfileMapper.updateDTI(displayName: "New Name", bio: "hi")
        #expect(dti.displayName == "New Name")
        #expect(dti.bio == "hi")
    }

    @Test func decodesSnakeCaseColumnsFromPostgrest() throws {
        let json = """
        {
          "id": "00000000-0000-0000-0000-000000000001",
          "display_name": "Jonathan",
          "bio": null,
          "avatar_url": "https://example.com/a.png",
          "verification_status": "pending",
          "role": "member"
        }
        """
        let dto = try JSONDecoder().decode(ProfileDTO.self, from: Data(json.utf8))
        #expect(dto.displayName == "Jonathan")
        #expect(dto.avatarURL == "https://example.com/a.png")
        #expect(dto.verificationStatus == "pending")
        #expect(dto.bio == nil)
    }

    @Test func encodesUpdateDTIWithSnakeCaseColumns() throws {
        let dti = ProfileMapper.updateDTI(displayName: "New Name", bio: "hi")
        let data = try JSONEncoder().encode(dti)
        let object = try #require(
            try JSONSerialization.jsonObject(with: data) as? [String: Any]
        )
        #expect(object["display_name"] as? String == "New Name")
        #expect(object["bio"] as? String == "hi")
        #expect(object["displayName"] == nil)
    }
}
```

- [x] **Step 3: Run test to verify it fails**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/ProfileMapperTests -quiet
```
Expected: FAIL — `cannot find 'ProfileDTO' in scope`.

- [x] **Step 4: Implement DTO/DTI and mapper**

`ios/Coterie/Features/Profile/ProfileDTO.swift`:
```swift
import Foundation

struct ProfileDTO: Decodable {
    let id: UUID
    let displayName: String
    let bio: String?
    let avatarURL: String?
    let verificationStatus: String
    let role: String

    enum CodingKeys: String, CodingKey {
        case id
        case displayName = "display_name"
        case bio
        case avatarURL = "avatar_url"
        case verificationStatus = "verification_status"
        case role
    }
}

struct ProfileUpdateDTI: Encodable {
    let displayName: String
    let bio: String

    enum CodingKeys: String, CodingKey {
        case displayName = "display_name"
        case bio
    }
}

enum ProfileMapper {
    static func domain(from dto: ProfileDTO) -> Profile {
        Profile(
            id: dto.id,
            displayName: dto.displayName,
            bio: dto.bio ?? "",
            avatarURL: dto.avatarURL.flatMap(URL.init(string:)),
            verificationStatus: VerificationStatus(rawValue: dto.verificationStatus) ?? .unverified,
            role: UserRole(rawValue: dto.role) ?? .member
        )
    }

    static func updateDTI(displayName: String, bio: String) -> ProfileUpdateDTI {
        ProfileUpdateDTI(displayName: displayName, bio: bio)
    }
}
```

- [x] **Step 5: Run test to verify it passes**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/ProfileMapperTests -quiet
```
Expected: PASS (5 tests — 3 mapper, 2 wire-format round trip).

- [x] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: Profile domain model with DTO/DTI mapper"
```

---

## Task 5: AuthRepository protocol, AuthedUser & AuthViewModel ✅ DONE

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Core/Models/AuthedUser.swift`
- Create: `ios/Coterie/Features/Auth/AuthRepository.swift`
- Create: `ios/Coterie/Features/Auth/AuthViewModel.swift`
- Create: `ios/CoterieTests/Mocks/MockAuthRepository.swift`
- Test: `ios/CoterieTests/AuthViewModelTests.swift`

**Interfaces:**
- Consumes: nothing.
- Produces: `AuthedUser { id: UUID, email: String }`; `protocol AuthRepository` with `currentUser: AuthedUser? { get async }`, `signUp(email:password:displayName:)`, `signIn(email:password:)`, `signOut()`; `@MainActor @Observable AuthViewModel(repo:)` with `email/password/displayName/mode/isLoading/errorMessage/authedUser`, `canSubmit: Bool`, `submit() async`.

- [x] **Step 1: Write the domain model and protocol**

`ios/Coterie/Core/Models/AuthedUser.swift`:
```swift
import Foundation

struct AuthedUser: Sendable, Equatable {
    let id: UUID
    let email: String
}
```

`ios/Coterie/Features/Auth/AuthRepository.swift`:
```swift
import Foundation

protocol AuthRepository: Sendable {
    var currentUser: AuthedUser? { get async }
    func signUp(email: String, password: String, displayName: String) async throws -> AuthedUser
    func signIn(email: String, password: String) async throws -> AuthedUser
    func signOut() async throws
}
```

- [x] **Step 2: Write the mock repository**

`ios/CoterieTests/Mocks/MockAuthRepository.swift`:
```swift
import Foundation
@testable import Coterie

actor MockAuthRepository: AuthRepository {
    enum Failure: Error { case forced }

    var stubbedUser: AuthedUser?
    var shouldThrow = false
    private(set) var signUpCallCount = 0
    private(set) var signInCallCount = 0

    init(stubbedUser: AuthedUser? = nil, shouldThrow: Bool = false) {
        self.stubbedUser = stubbedUser
        self.shouldThrow = shouldThrow
    }

    var currentUser: AuthedUser? { stubbedUser }

    func signUp(email: String, password: String, displayName: String) async throws -> AuthedUser {
        signUpCallCount += 1
        if shouldThrow { throw Failure.forced }
        let user = AuthedUser(id: UUID(), email: email)
        stubbedUser = user
        return user
    }

    func signIn(email: String, password: String) async throws -> AuthedUser {
        signInCallCount += 1
        if shouldThrow { throw Failure.forced }
        let user = AuthedUser(id: UUID(), email: email)
        stubbedUser = user
        return user
    }

    func signOut() async throws { stubbedUser = nil }
}
```

- [x] **Step 3: Write failing AuthViewModel tests**

`ios/CoterieTests/AuthViewModelTests.swift`:
```swift
import Testing
import Foundation
@testable import Coterie

@MainActor
struct AuthViewModelTests {
    @Test func signInFormInvalidWhenEmailOrPasswordEmpty() {
        let viewModel = AuthViewModel(repo: MockAuthRepository())
        viewModel.mode = .signIn
        viewModel.email = ""
        viewModel.password = ""
        #expect(viewModel.canSubmit == false)
    }

    @Test func signInFormValidWithGoodEmailAndPassword() {
        let viewModel = AuthViewModel(repo: MockAuthRepository())
        viewModel.mode = .signIn
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        #expect(viewModel.canSubmit == true)
    }

    @Test func signUpRequiresDisplayName() {
        let viewModel = AuthViewModel(repo: MockAuthRepository())
        viewModel.mode = .signUp
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        viewModel.displayName = ""
        #expect(viewModel.canSubmit == false)
        viewModel.displayName = "Jonathan"
        #expect(viewModel.canSubmit == true)
    }

    @Test func submitSuccessSetsAuthedUser() async {
        let viewModel = AuthViewModel(repo: MockAuthRepository())
        viewModel.mode = .signIn
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        await viewModel.submit()
        #expect(viewModel.authedUser?.email == "a@b.com")
        #expect(viewModel.errorMessage == nil)
        #expect(viewModel.isLoading == false)
    }

    @Test func submitFailureSetsErrorMessage() async {
        let viewModel = AuthViewModel(repo: MockAuthRepository(shouldThrow: true))
        viewModel.mode = .signIn
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        await viewModel.submit()
        #expect(viewModel.authedUser == nil)
        #expect(viewModel.errorMessage != nil)
        #expect(viewModel.isLoading == false)
    }

    @Test func signUpModeCallsSignUpNotSignIn() async {
        let repo = MockAuthRepository()
        let viewModel = AuthViewModel(repo: repo)
        viewModel.mode = .signUp
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        viewModel.displayName = "Jonathan"
        await viewModel.submit()
        #expect(await repo.signUpCallCount == 1)
        #expect(await repo.signInCallCount == 0)
    }

    @Test func signInModeCallsSignInNotSignUp() async {
        let repo = MockAuthRepository()
        let viewModel = AuthViewModel(repo: repo)
        viewModel.mode = .signIn
        viewModel.email = "a@b.com"
        viewModel.password = "password1"
        await viewModel.submit()
        #expect(await repo.signInCallCount == 1)
        #expect(await repo.signUpCallCount == 0)
    }

    @Test func submitWithInvalidFormDoesNotCallRepository() async {
        let repo = MockAuthRepository()
        let viewModel = AuthViewModel(repo: repo)
        viewModel.mode = .signIn
        viewModel.email = "not-an-email"
        viewModel.password = "short"
        await viewModel.submit()
        #expect(await repo.signInCallCount == 0)
        #expect(viewModel.authedUser == nil)
    }
}
```

- [x] **Step 4: Run tests to verify they fail**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AuthViewModelTests -quiet
```
Expected: FAIL — `cannot find 'AuthViewModel' in scope`.

- [x] **Step 5: Implement AuthViewModel**

`ios/Coterie/Features/Auth/AuthViewModel.swift`:
```swift
import Foundation
import Observation

@MainActor
@Observable
final class AuthViewModel {
    enum Mode { case signIn, signUp }

    var mode: Mode = .signIn
    var email = ""
    var password = ""
    var displayName = ""
    var isLoading = false
    var errorMessage: String?
    private(set) var authedUser: AuthedUser?

    private let repo: AuthRepository

    init(repo: AuthRepository) {
        self.repo = repo
    }

    var canSubmit: Bool {
        let emailOK = email.contains("@") && email.contains(".")
        let passwordOK = password.count >= 8
        let nameOK = mode == .signIn || !displayName.trimmingCharacters(in: .whitespaces).isEmpty
        return emailOK && passwordOK && nameOK && !isLoading
    }

    func submit() async {
        guard canSubmit else { return }
        isLoading = true
        errorMessage = nil
        do {
            switch mode {
            case .signIn:
                authedUser = try await repo.signIn(email: email, password: password)
            case .signUp:
                authedUser = try await repo.signUp(email: email, password: password, displayName: displayName)
            }
        } catch {
            errorMessage = "Something went wrong. Please try again."
        }
        isLoading = false
    }
}
```

- [x] **Step 6: Run tests to verify they pass**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AuthViewModelTests -quiet
```
Expected: PASS (8 tests — 5 from the plan, 3 added for call-count routing).

- [x] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: AuthRepository protocol and tested AuthViewModel"
```

---

## Task 6: SupabaseAuthRepository implementation ✅ DONE

**Repos:** `ios` for the code; Step 3 also uses `backend` for the integration check.

**Files:**
- Create: `ios/Coterie/Features/Auth/SupabaseAuthRepository.swift`

**Interfaces:**
- Consumes: `AuthRepository`, `AuthedUser`, `SupabaseClientProvider`.
- Produces: `SupabaseAuthRepository(client:)` conforming to `AuthRepository`.

- [x] **Step 1: Implement the repository**

`ios/Coterie/Features/Auth/SupabaseAuthRepository.swift`:
```swift
import Foundation
import Supabase

enum AuthRepositoryError: LocalizedError {
    case emailConfirmationRequired

    var errorDescription: String? {
        switch self {
        case .emailConfirmationRequired:
            "Check your email to confirm your account before signing in."
        }
    }
}

struct SupabaseAuthRepository: AuthRepository {
    private let client: SupabaseClient

    init(client: SupabaseClient) {
        self.client = client
    }

    var currentUser: AuthedUser? {
        get async {
            guard let session = try? await client.auth.session else { return nil }
            return AuthedUser(id: session.user.id, email: session.user.email ?? "")
        }
    }

    func signUp(email: String, password: String, displayName: String) async throws -> AuthedUser {
        let response = try await client.auth.signUp(
            email: email,
            password: password,
            data: ["display_name": .string(displayName)]
        )
        guard response.session != nil else {
            throw AuthRepositoryError.emailConfirmationRequired
        }
        return AuthedUser(id: response.user.id, email: response.user.email ?? email)
    }

    func signIn(email: String, password: String) async throws -> AuthedUser {
        let session = try await client.auth.signIn(email: email, password: password)
        return AuthedUser(id: session.user.id, email: session.user.email ?? email)
    }

    func signOut() async throws {
        try await client.auth.signOut()
    }
}
```

- [x] **Step 2: Build to verify it compiles against the SDK**

Run from `coterie/ios`:
```bash
make build
```
Expected: `** BUILD SUCCEEDED **`.

**As executed against supabase-swift 2.54.1** — the plan's caveat about `data:` was unfounded,
but a different mismatch was found:
- `data:` is `[String: AnyJSON]?`, so `.string(displayName)` is correct.
- `signIn(email:password:captchaToken:)` returns `Session`. Correct as written.
- `client.auth.session` is `get async throws`, so `try? await` is correct.
- **`AuthResponse` is an `enum`** — `.session(Session)` or `.user(User)` — not a struct. It
  exposes a `user: User` accessor covering both cases (so `response.user.id` compiles) and a
  `session: Session?` that is `nil` when only a user record came back.

That last point is the real one. `signUp` returns `.user` with **no session** when email
confirmation is enabled. Local `config.toml` has `enable_confirmations = false`, so a session
is always issued and the plan's code appears to work — but hosted projects default it **on**,
where the original code would return an `AuthedUser` for a user who is not signed in. Task 11
would then route to profile setup and fail on `permission denied`. Hence the explicit
`guard response.session != nil` above.

- [x] **Step 3: Integration check**

Done against the HTTP API rather than a throwaway button, so it is reproducible and exercises
the same endpoints the SDK calls.

Run from `coterie/backend` (`ANON` is the local anon key from `make db-status`):
```bash
make db-reset
curl -s -X POST "http://127.0.0.1:54321/auth/v1/signup" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"jonathan@example.com","password":"password1","data":{"display_name":"Jonathan"}}'
docker exec supabase_db_coterie psql -U postgres -c \
  "select display_name, verification_status, role from public.profiles;"
```
Expected: response contains `access_token`; one row with the display name, `unverified`, `member`.

Then, using the `access_token` from that response, confirm the client contract Task 8 depends on:
```bash
curl -s "http://127.0.0.1:54321/rest/v1/profiles?select=*" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK"                       # 200, row returned
curl -s -X PATCH "http://127.0.0.1:54321/rest/v1/profiles?id=eq.$UID" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" -d '{"display_name":"X","bio":"y"}'  # 200, updated
curl -s -X PATCH "http://127.0.0.1:54321/rest/v1/profiles?id=eq.$UID" \
  -H "apikey: $ANON" -H "Authorization: Bearer $TOK" \
  -H "Content-Type: application/json" -d '{"role":"admin"}'                # 403, code 42501
```
Verified: read and safe-column write succeed; the escalation attempt returns
`HTTP 403 / 42501 permission denied for table profiles`, confirming the Task 3 column grants
hold over the real API and not just in pgTAP.

- [x] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: Supabase-backed AuthRepository implementation"
```

---

## Task 7: Auth UI (Sign In / Sign Up)

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Features/Auth/SignInView.swift`
- Create: `ios/Coterie/Features/Auth/SignUpView.swift`

**Interfaces:**
- Consumes: `AuthViewModel`.
- Produces: `SignInView(viewModel:)`, `SignUpView(viewModel:)`. Both bind to a shared `AuthViewModel`; on success `viewModel.authedUser` is non-nil (routing handled in Task 11).

- [ ] **Step 1: Implement SignInView**

`ios/Coterie/Features/Auth/SignInView.swift`:
```swift
import SwiftUI

struct SignInView: View {
    @Bindable var viewModel: AuthViewModel
    var onSwitchToSignUp: () -> Void

    var body: some View {
        VStack(spacing: 16) {
            Text("Welcome back")
                .font(.title.bold())
            TextField("Email", text: $viewModel.email)
                .textContentType(.emailAddress)
                .keyboardType(.emailAddress)
                .textInputAutocapitalization(.never)
            SecureField("Password", text: $viewModel.password)
                .textContentType(.password)
            if let error = viewModel.errorMessage {
                Text(error).foregroundStyle(.red).font(.footnote)
            }
            Button {
                Task { await viewModel.submit() }
            } label: {
                if viewModel.isLoading { ProgressView() } else { Text("Sign in") }
            }
            .buttonStyle(.borderedProminent)
            .disabled(!viewModel.canSubmit)
            Button("Create an account", action: onSwitchToSignUp)
                .font(.footnote)
        }
        .textFieldStyle(.roundedBorder)
        .padding()
        .onAppear { viewModel.mode = .signIn }
    }
}
```

- [ ] **Step 2: Implement SignUpView**

`ios/Coterie/Features/Auth/SignUpView.swift`:
```swift
import SwiftUI

struct SignUpView: View {
    @Bindable var viewModel: AuthViewModel
    var onSwitchToSignIn: () -> Void

    var body: some View {
        VStack(spacing: 16) {
            Text("Create your account")
                .font(.title.bold())
            TextField("Display name", text: $viewModel.displayName)
                .textContentType(.name)
            TextField("Email", text: $viewModel.email)
                .textContentType(.emailAddress)
                .keyboardType(.emailAddress)
                .textInputAutocapitalization(.never)
            SecureField("Password (min 8 chars)", text: $viewModel.password)
                .textContentType(.newPassword)
            if let error = viewModel.errorMessage {
                Text(error).foregroundStyle(.red).font(.footnote)
            }
            Button {
                Task { await viewModel.submit() }
            } label: {
                if viewModel.isLoading { ProgressView() } else { Text("Sign up") }
            }
            .buttonStyle(.borderedProminent)
            .disabled(!viewModel.canSubmit)
            Button("I already have an account", action: onSwitchToSignIn)
                .font(.footnote)
        }
        .textFieldStyle(.roundedBorder)
        .padding()
        .onAppear { viewModel.mode = .signUp }
    }
}
```

- [ ] **Step 3: Build and run lint**

Run:
```bash
make build
swiftlint lint --quiet Coterie/Features/Auth
```
Expected: build succeeds; no SwiftLint errors.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: sign in and sign up screens"
```

---

## Task 8: ProfileRepository, SupabaseProfileRepository & ProfileViewModel

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Features/Profile/ProfileRepository.swift`
- Create: `ios/Coterie/Features/Profile/SupabaseProfileRepository.swift`
- Create: `ios/Coterie/Features/Profile/ProfileViewModel.swift`
- Create: `ios/CoterieTests/Mocks/MockProfileRepository.swift`
- Test: `ios/CoterieTests/ProfileViewModelTests.swift`

**Interfaces:**
- Consumes: `Profile`, `ProfileDTO`, `ProfileUpdateDTI`, `ProfileMapper`, `SupabaseClient`.
- Produces: `protocol ProfileRepository { fetchMyProfile() async throws -> Profile; updateMyProfile(displayName:bio:) async throws -> Profile }`; `@MainActor @Observable ProfileViewModel(repo:)` with `displayName/bio/isSaving/errorMessage/profile`, `loadProfile() async`, `save() async`, `canSave: Bool`.

- [ ] **Step 1: Write the protocol and mock**

`ios/Coterie/Features/Profile/ProfileRepository.swift`:
```swift
import Foundation

protocol ProfileRepository: Sendable {
    func fetchMyProfile() async throws -> Profile
    func updateMyProfile(displayName: String, bio: String) async throws -> Profile
}
```

`ios/CoterieTests/Mocks/MockProfileRepository.swift`:
```swift
import Foundation
@testable import Coterie

actor MockProfileRepository: ProfileRepository {
    enum Failure: Error { case forced }

    var stubbedProfile: Profile
    var shouldThrow: Bool

    init(stubbedProfile: Profile? = nil, shouldThrow: Bool = false) {
        self.stubbedProfile = stubbedProfile ?? Profile(
            id: UUID(), displayName: "Existing", bio: "",
            avatarURL: nil, verificationStatus: .unverified, role: .member
        )
        self.shouldThrow = shouldThrow
    }

    func fetchMyProfile() async throws -> Profile {
        if shouldThrow { throw Failure.forced }
        return stubbedProfile
    }

    func updateMyProfile(displayName: String, bio: String) async throws -> Profile {
        if shouldThrow { throw Failure.forced }
        stubbedProfile.displayName = displayName
        stubbedProfile.bio = bio
        return stubbedProfile
    }
}
```

- [ ] **Step 2: Write failing ProfileViewModel tests**

`ios/CoterieTests/ProfileViewModelTests.swift`:
```swift
import Testing
import Foundation
@testable import Coterie

@MainActor
struct ProfileViewModelTests {
    @Test func loadPopulatesFields() async {
        let profile = Profile(id: UUID(), displayName: "Jon", bio: "hi",
                              avatarURL: nil, verificationStatus: .unverified, role: .member)
        let vm = ProfileViewModel(repo: MockProfileRepository(stubbedProfile: profile))
        await vm.loadProfile()
        #expect(vm.displayName == "Jon")
        #expect(vm.bio == "hi")
    }

    @Test func canSaveFalseWhenDisplayNameBlank() {
        let vm = ProfileViewModel(repo: MockProfileRepository())
        vm.displayName = "   "
        #expect(vm.canSave == false)
        vm.displayName = "Jon"
        #expect(vm.canSave == true)
    }

    @Test func saveSuccessUpdatesProfile() async {
        let vm = ProfileViewModel(repo: MockProfileRepository())
        vm.displayName = "New Name"
        vm.bio = "bio"
        await vm.save()
        #expect(vm.profile?.displayName == "New Name")
        #expect(vm.errorMessage == nil)
        #expect(vm.isSaving == false)
    }

    @Test func saveFailureSetsError() async {
        let vm = ProfileViewModel(repo: MockProfileRepository(shouldThrow: true))
        vm.displayName = "New Name"
        await vm.save()
        #expect(vm.errorMessage != nil)
        #expect(vm.isSaving == false)
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/ProfileViewModelTests -quiet
```
Expected: FAIL — `cannot find 'ProfileViewModel' in scope`.

- [ ] **Step 4: Implement ProfileViewModel**

`ios/Coterie/Features/Profile/ProfileViewModel.swift`:
```swift
import Foundation
import Observation

@MainActor
@Observable
final class ProfileViewModel {
    var displayName = ""
    var bio = ""
    var isSaving = false
    var errorMessage: String?
    private(set) var profile: Profile?

    private let repo: ProfileRepository

    init(repo: ProfileRepository) {
        self.repo = repo
    }

    var canSave: Bool {
        !displayName.trimmingCharacters(in: .whitespaces).isEmpty && !isSaving
    }

    func loadProfile() async {
        do {
            let loaded = try await repo.fetchMyProfile()
            profile = loaded
            displayName = loaded.displayName
            bio = loaded.bio
        } catch {
            errorMessage = "Couldn't load your profile."
        }
    }

    func save() async {
        guard canSave else { return }
        isSaving = true
        errorMessage = nil
        do {
            profile = try await repo.updateMyProfile(displayName: displayName, bio: bio)
        } catch {
            errorMessage = "Couldn't save your profile. Please try again."
        }
        isSaving = false
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/ProfileViewModelTests -quiet
```
Expected: PASS (4 tests).

- [ ] **Step 6: Implement SupabaseProfileRepository**

`ios/Coterie/Features/Profile/SupabaseProfileRepository.swift`:
```swift
import Foundation
import Supabase

struct SupabaseProfileRepository: ProfileRepository {
    private let client: SupabaseClient

    init(client: SupabaseClient) {
        self.client = client
    }

    private func currentUserID() async throws -> UUID {
        try await client.auth.session.user.id
    }

    func fetchMyProfile() async throws -> Profile {
        let id = try await currentUserID()
        let dto: ProfileDTO = try await client
            .from("profiles")
            .select()
            .eq("id", value: id)
            .single()
            .execute()
            .value
        return ProfileMapper.domain(from: dto)
    }

    func updateMyProfile(displayName: String, bio: String) async throws -> Profile {
        let id = try await currentUserID()
        let dti = ProfileMapper.updateDTI(displayName: displayName, bio: bio)
        let dto: ProfileDTO = try await client
            .from("profiles")
            .update(dti)
            .eq("id", value: id)
            .select()
            .single()
            .execute()
            .value
        return ProfileMapper.domain(from: dto)
    }
}
```

- [ ] **Step 7: Build and commit**

Run:
```bash
make build
git add -A
git commit -m "feat: ProfileRepository and tested ProfileViewModel"
```

---

## Task 9: Profile setup screen

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Features/Profile/ProfileSetupView.swift`

**Interfaces:**
- Consumes: `ProfileViewModel`.
- Produces: `ProfileSetupView(viewModel:onDone:)`; calls `onDone()` after a successful save.

- [ ] **Step 1: Implement ProfileSetupView**

`ios/Coterie/Features/Profile/ProfileSetupView.swift`:
```swift
import SwiftUI

struct ProfileSetupView: View {
    @Bindable var viewModel: ProfileViewModel
    var onDone: () -> Void

    var body: some View {
        VStack(spacing: 16) {
            Text("Set up your profile")
                .font(.title.bold())
            TextField("Display name", text: $viewModel.displayName)
                .textContentType(.name)
            TextField("Short bio (optional)", text: $viewModel.bio, axis: .vertical)
                .lineLimit(2...4)
            if let error = viewModel.errorMessage {
                Text(error).foregroundStyle(.red).font(.footnote)
            }
            Button {
                Task {
                    await viewModel.save()
                    if viewModel.errorMessage == nil { onDone() }
                }
            } label: {
                if viewModel.isSaving { ProgressView() } else { Text("Continue") }
            }
            .buttonStyle(.borderedProminent)
            .disabled(!viewModel.canSave)
        }
        .textFieldStyle(.roundedBorder)
        .padding()
        .task { await viewModel.loadProfile() }
    }
}
```

- [ ] **Step 2: Build, lint, commit**

Run:
```bash
make build
swiftlint lint --quiet Coterie/Features/Profile
git add -A
git commit -m "feat: profile setup screen"
```

---

## Task 10: First-run intro carousel

**Repo:** `ios` — run all commands from `coterie/ios`.

**Files:**
- Create: `ios/Coterie/Features/Onboarding/IntroPage.swift`
- Create: `ios/Coterie/Features/Onboarding/IntroViewModel.swift`
- Create: `ios/Coterie/Features/Onboarding/IntroCarouselView.swift`
- Test: `ios/CoterieTests/IntroViewModelTests.swift`

**Interfaces:**
- Produces: `IntroPage { title, subtitle, systemImage }`; `@Observable IntroViewModel` with `pages: [IntroPage]`, `currentIndex`, `isLastPage: Bool`, `advance()`; `IntroCarouselView(onFinished:)`.

- [ ] **Step 1: Write the page model**

`ios/Coterie/Features/Onboarding/IntroPage.swift`:
```swift
import Foundation

struct IntroPage: Identifiable, Equatable {
    let id = UUID()
    let title: String
    let subtitle: String
    let systemImage: String
}
```

- [ ] **Step 2: Write failing IntroViewModel tests**

`ios/CoterieTests/IntroViewModelTests.swift`:
```swift
import Testing
@testable import Coterie

@MainActor
struct IntroViewModelTests {
    @Test func hasThreePages() {
        let vm = IntroViewModel()
        #expect(vm.pages.count == 3)
    }

    @Test func advanceMovesForwardAndDetectsLastPage() {
        let vm = IntroViewModel()
        #expect(vm.currentIndex == 0)
        #expect(vm.isLastPage == false)
        vm.advance()
        #expect(vm.currentIndex == 1)
        vm.advance()
        #expect(vm.currentIndex == 2)
        #expect(vm.isLastPage == true)
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/IntroViewModelTests -quiet
```
Expected: FAIL — `cannot find 'IntroViewModel' in scope`.

- [ ] **Step 4: Implement IntroViewModel**

`ios/Coterie/Features/Onboarding/IntroViewModel.swift`:
```swift
import Foundation
import Observation

@MainActor
@Observable
final class IntroViewModel {
    let pages: [IntroPage] = [
        IntroPage(title: "Own luxury together",
                  subtitle: "Split the cost of a bag or pair with up to three others.",
                  systemImage: "bag.fill"),
        IntroPage(title: "Book your time",
                  subtitle: "Everyone gets equal days. Reserve yours on a shared calendar.",
                  systemImage: "calendar"),
        IntroPage(title: "Safe & verified",
                  subtitle: "Every owner is ID-verified. Items are cleaned and insured at our hub.",
                  systemImage: "checkmark.shield.fill")
    ]

    var currentIndex = 0

    var isLastPage: Bool { currentIndex == pages.count - 1 }

    func advance() {
        if currentIndex < pages.count - 1 { currentIndex += 1 }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/IntroViewModelTests -quiet
```
Expected: PASS (2 tests).

- [ ] **Step 6: Implement IntroCarouselView**

`ios/Coterie/Features/Onboarding/IntroCarouselView.swift`:
```swift
import SwiftUI

struct IntroCarouselView: View {
    @State private var viewModel = IntroViewModel()
    var onFinished: () -> Void

    var body: some View {
        VStack {
            TabView(selection: $viewModel.currentIndex) {
                ForEach(Array(viewModel.pages.enumerated()), id: \.element.id) { index, page in
                    VStack(spacing: 20) {
                        Image(systemName: page.systemImage)
                            .font(.system(size: 64))
                            .foregroundStyle(.tint)
                        Text(page.title).font(.title.bold())
                        Text(page.subtitle)
                            .multilineTextAlignment(.center)
                            .foregroundStyle(.secondary)
                            .padding(.horizontal)
                    }
                    .tag(index)
                }
            }
            .tabViewStyle(.page)

            Button(viewModel.isLastPage ? "Get started" : "Next") {
                if viewModel.isLastPage { onFinished() } else { viewModel.advance() }
            }
            .buttonStyle(.borderedProminent)
            .padding()

            Button("Skip", action: onFinished)
                .font(.footnote)
                .padding(.bottom)
        }
    }
}
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: first-run intro carousel"
```

---

## Task 11: AppContainer DI, AppCoordinator & root routing

**Repos:** `ios` for the code; Step 8 also uses `backend` for the smoke test.

**Files:**
- Create: `ios/Coterie/App/AppContainer.swift`
- Create: `ios/Coterie/App/AppCoordinator.swift`
- Modify: `ios/Coterie/App/RootView.swift` (replace placeholder)
- Modify: `ios/Coterie/App/CoterieApp.swift` (inject container)
- Test: `ios/CoterieTests/AppCoordinatorTests.swift`

**Interfaces:**
- Consumes: `AuthRepository`, `ProfileRepository`, `SupabaseClientProvider`, `AuthViewModel`, `ProfileViewModel`, `IntroCarouselView`, `SignInView`, `SignUpView`, `ProfileSetupView`.
- Produces: `AppContainer { authRepository, profileRepository }`; `@Observable AppCoordinator` with `route: Route` (`intro | auth | profileSetup | home`), `hasSeenIntro` (AppStorage-backed), `completeIntro()`, `didAuthenticate()`, `didCompleteProfile()`, `start(currentUser:)`.

- [ ] **Step 1: Implement AppContainer**

`ios/Coterie/App/AppContainer.swift`:
```swift
import Foundation
import Supabase

@MainActor
final class AppContainer {
    let client: SupabaseClient
    let authRepository: AuthRepository
    let profileRepository: ProfileRepository

    init() {
        let client = SupabaseClientProvider.make()
        self.client = client
        self.authRepository = SupabaseAuthRepository(client: client)
        self.profileRepository = SupabaseProfileRepository(client: client)
    }

    init(client: SupabaseClient, authRepository: AuthRepository, profileRepository: ProfileRepository) {
        self.client = client
        self.authRepository = authRepository
        self.profileRepository = profileRepository
    }
}
```

- [ ] **Step 2: Write failing AppCoordinator tests**

`ios/CoterieTests/AppCoordinatorTests.swift`:
```swift
import Testing
@testable import Coterie

@MainActor
struct AppCoordinatorTests {
    @Test func startsAtIntroWhenNotSeenAndSignedOut() {
        let c = AppCoordinator(hasSeenIntro: false)
        c.start(currentUser: nil)
        #expect(c.route == .intro)
    }

    @Test func startsAtAuthWhenIntroSeenAndSignedOut() {
        let c = AppCoordinator(hasSeenIntro: true)
        c.start(currentUser: nil)
        #expect(c.route == .auth)
    }

    @Test func startsAtHomeWhenSignedIn() {
        let c = AppCoordinator(hasSeenIntro: true)
        c.start(currentUser: AuthedUser(id: .init(), email: "a@b.com"))
        #expect(c.route == .home)
    }

    @Test func completingIntroMovesToAuth() {
        let c = AppCoordinator(hasSeenIntro: false)
        c.start(currentUser: nil)
        c.completeIntro()
        #expect(c.route == .auth)
    }

    @Test func authThenProfileThenHome() {
        let c = AppCoordinator(hasSeenIntro: true)
        c.start(currentUser: nil)
        c.didAuthenticate()
        #expect(c.route == .profileSetup)
        c.didCompleteProfile()
        #expect(c.route == .home)
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AppCoordinatorTests -quiet
```
Expected: FAIL — `cannot find 'AppCoordinator' in scope`.

- [ ] **Step 4: Implement AppCoordinator**

`ios/Coterie/App/AppCoordinator.swift`:
```swift
import Foundation
import Observation

@MainActor
@Observable
final class AppCoordinator {
    enum Route: Equatable { case intro, auth, profileSetup, home }

    private(set) var route: Route = .intro
    private var hasSeenIntro: Bool

    init(hasSeenIntro: Bool) {
        self.hasSeenIntro = hasSeenIntro
    }

    func start(currentUser: AuthedUser?) {
        if currentUser != nil {
            route = .home
        } else if hasSeenIntro {
            route = .auth
        } else {
            route = .intro
        }
    }

    func completeIntro() {
        hasSeenIntro = true
        route = .auth
    }

    func didAuthenticate() {
        route = .profileSetup
    }

    func didCompleteProfile() {
        route = .home
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run:
```bash
xcodebuild test -scheme Coterie -destination 'platform=iOS Simulator,name=iPhone 17 Pro' -only-testing:CoterieTests/AppCoordinatorTests -quiet
```
Expected: PASS (8 tests — 5 from the plan, 3 added for call-count routing).

- [ ] **Step 6: Wire RootView to the coordinator**

Replace `ios/Coterie/App/RootView.swift`:
```swift
import SwiftUI

struct RootView: View {
    let container: AppContainer
    @State private var coordinator: AppCoordinator
    @State private var authViewModel: AuthViewModel
    @State private var showSignUp = false
    @AppStorage("hasSeenIntro") private var hasSeenIntro = false

    init(container: AppContainer) {
        self.container = container
        _coordinator = State(initialValue: AppCoordinator(hasSeenIntro: UserDefaults.standard.bool(forKey: "hasSeenIntro")))
        _authViewModel = State(initialValue: AuthViewModel(repo: container.authRepository))
    }

    var body: some View {
        Group {
            switch coordinator.route {
            case .intro:
                IntroCarouselView {
                    hasSeenIntro = true
                    coordinator.completeIntro()
                }
            case .auth:
                if showSignUp {
                    SignUpView(viewModel: authViewModel, onSwitchToSignIn: { showSignUp = false })
                } else {
                    SignInView(viewModel: authViewModel, onSwitchToSignUp: { showSignUp = true })
                }
            case .profileSetup:
                ProfileSetupView(
                    viewModel: ProfileViewModel(repo: container.profileRepository),
                    onDone: { coordinator.didCompleteProfile() }
                )
            case .home:
                HomePlaceholderView()
            }
        }
        .onChange(of: authViewModel.authedUser) { _, newValue in
            if newValue != nil { coordinator.didAuthenticate() }
        }
        .task {
            let user = await container.authRepository.currentUser
            coordinator.start(currentUser: user)
        }
    }
}

private struct HomePlaceholderView: View {
    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: "bag.fill").font(.largeTitle)
            Text("You're in.").font(.title2.bold())
            Text("Pools arrive in Plan 2.").foregroundStyle(.secondary)
        }
    }
}
```

- [ ] **Step 7: Inject the container at app entry**

Replace `ios/Coterie/App/CoterieApp.swift`:
```swift
import SwiftUI

@main
struct CoterieApp: App {
    @State private var container = AppContainer()

    var body: some Scene {
        WindowGroup {
            RootView(container: container)
        }
    }
}
```

- [ ] **Step 8: Full test suite, build, and manual smoke**

Run from `coterie/backend`:
```bash
make db
make db-reset
```
Run from `coterie/ios`:
```bash
make test
```
Expected: all tests pass. Then run the app and verify: intro shows on first launch → Get started → sign up → profile setup → "You're in." Relaunch → lands straight on "You're in." (session + intro persistence).

- [ ] **Step 9: Commit**

Run from `coterie/ios`:
```bash
git add -A
git commit -m "feat: DI container, coordinator, and root routing"
```

---

## Self-Review

**Spec coverage (Plan 1 slice of the spec):**
- §3A MVVM + coordinators + repository protocols + DTO/DTI/mapper + manual DI → Tasks 4, 5, 6, 8, 11. ✅
- §4 `profiles` table (incl. `role`, `verification_status` defaults) + RLS + trigger → Task 3. ✅
- §6 first-run education (3 slides) + sign up/in + profile setup → Tasks 7, 9, 10, 11. ✅
- §6 "browse before you verify" → verification is deferred (Plan 2); Plan 1 sets `unverified` default and does not gate. ✅
- §8 environments/migrations-as-code (dev config + versioned SQL migrations + Supabase local) → Tasks 1–3. (QA/prod project creation + CI is infra, tracked for Plan 2/CI setup.) ✅
- §9 `role` column present for later RBAC → Task 3. ✅
- §11 money paths fail loud (n/a Plan 1); no snapshot tests; Swift Testing → all test tasks. ✅

**Deferred to later plans (intentionally, not gaps):** Stripe Identity verification (Plan 2), pools/memberships/escrow/booking/chat/ratings (Plans 2–5), QA/prod Supabase projects + CI pipeline (CI setup task before Plan 2 execution).

**Placeholder scan:** no TBD/TODO; every code step shows complete code. ✅

**Type consistency:** `AuthRepository`, `ProfileRepository`, `AuthedUser`, `Profile`, `ProfileDTO`/`ProfileUpdateDTI`/`ProfileMapper`, `AuthViewModel`, `ProfileViewModel`, `IntroViewModel`, `AppCoordinator.Route`, `AppContainer` names match across all tasks. ✅

## Notes / Known Follow-ups
- **CI + QA/prod environments:** a short infra task should run before Plan 2 execution — it's an enabler, not a feature, so it lives outside the feature plans. With three repos this is now two pipelines: `ios` running `make test`, `backend` running `make db-test`. Also create QA/prod Supabase projects with their own keys.
- **Cross-repo drift:** the schema in `backend` and the DTOs in `ios` can no longer change in one commit. Task 3 (migration) and Tasks 4/6/8 (DTOs and repositories) straddle that boundary — land the migration first, then the client change. When the Android client arrives, `backend` is the single source of truth for both.
- **Remotes:** all three repos are private under `suigeneris` (`coterie-docs`, `coterie-ios`, `coterie-backend`). Each repo pins `user.name`/`user.email` locally to `suigeneris <jona2k5@yahoo.com>` — check it before the first commit in any fresh clone.
- **CI scopes:** the `suigeneris` token has no `workflow` scope, so pushing `.github/workflows/` will be rejected. Run `gh auth refresh -h github.com -s workflow` as part of the CI task.
- **Email confirmation must be decided before QA/prod.** `signUp` only returns a session when `enable_confirmations = false`. Local dev has it off. Turning it on — the hosted default — means sign-up no longer signs the user in, and the app needs a "check your email" state plus a deep link back into the app. `SupabaseAuthRepository` currently throws `AuthRepositoryError.emailConfirmationRequired` in that case, which fails loudly rather than silently but is not a usable flow. Either keep confirmations off for the pilot, or add that state in Plan 2.
- **Column-scoped writes are load-bearing for Plan 2+.** `verification_status` is only meaningful as a trust signal because users cannot set it themselves; the same will apply to escrow state, pool membership and booking ownership. Every future table needs the same treatment — grant `select`, scope `update` to a column list, and let a `security definer` function or service-role path own the privileged columns. Assert it in pgTAP against the `authenticated` role, not as superuser.
- **Secrets:** the local anon key is safe to commit (local-only). Real QA/prod keys must come from CI secrets / per-env xcconfig excluded from git — enforce when those environments are created.
