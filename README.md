# Coterie — Docs

Whole-project documentation: product spec, implementation plans, and decisions that
span more than one platform. Platform-specific docs live in their own repo.

Part of a four-repo project:

| Repo | Local | Contents |
|---|---|---|
| [`coterie-ios`](https://github.com/suigeneris/coterie-ios) | `coterie/ios` | SwiftUI app |
| [`coterie-backend`](https://github.com/suigeneris/coterie-backend) | `coterie/backend` | Supabase schema, migrations, RLS, pgTAP tests |
| [`coterie-design`](https://github.com/suigeneris/coterie-design) | `coterie/design` | Ours design system as a Swift package |
| [`coterie-docs`](https://github.com/suigeneris/coterie-docs) | `coterie/docs` | this repo — spec, plans, and the **design system source of truth** |

## Getting all four

All four repos are **private**, so authentication must be working before any clone will
succeed. The GitHub repos are `coterie-*` prefixed; locally they sit unprefixed inside a
plain `coterie/` container folder, which is not itself a repo — so each clone needs an
explicit target directory.

### Step 1 — authenticate as `suigeneris`

```bash
brew install gh
gh auth login          # github.com → HTTPS → browser → sign in as suigeneris
gh auth status         # confirm: "Logged in to github.com account suigeneris"
```

Choosing **HTTPS** is deliberate: `gh` then acts as a git credential helper and no SSH key
is needed. If you prefer SSH, generate a key and register it (`gh ssh-key add`) first.

### Step 2 — clone

```bash
mkdir -p ~/Developer/coterie && cd ~/Developer/coterie
git clone https://github.com/suigeneris/coterie-docs.git    docs
git clone https://github.com/suigeneris/coterie-ios.git     ios
git clone https://github.com/suigeneris/coterie-backend.git backend
git clone https://github.com/suigeneris/coterie-design.git  design
```

The GitHub names are `coterie-*` prefixed; locally they are not. `coterie/` itself is a plain
folder, not a repo.

Explicit HTTPS URLs, not `gh repo clone`. `gh repo clone` rewrites the URL using gh's
*per-host* `git_protocol`, which can be `ssh` even when the global setting is `https`
(check with `gh config get git_protocol -h github.com`). On a machine whose SSH key belongs
to a different GitHub account that silently resolves to the wrong identity and fails with a
misleading `Repository not found` — the repo exists, you just aren't who GitHub thinks you
are. Explicit `https://` URLs always use the credential helper from Step 1.

### Step 3 — set the commit identity in each repo

Not inherited from a clone, and deliberately not set globally:

```bash
for r in docs ios backend design; do
  git -C $r config user.name  "suigeneris"
  git -C $r config user.email "jona2k5@yahoo.com"
done
```

### Machines with more than one GitHub account

If the machine is also signed in to a work GitHub account, plain `git@github.com` may
resolve to the *wrong* account and clones of these private repos will fail with
`Permission denied` or `Repository not found`. Pin the personal key with an SSH host alias
in `~/.ssh/config`:

```
Host github-personal
  HostName github.com
  User git
  IdentitiesOnly yes
  IdentityFile ~/.ssh/<your-personal-key>
```

then clone with `git@github-personal:suigeneris/<repo>.git` and set remotes to match. An
`ssh -T git@github-personal` must greet you as `suigeneris` before this works. If it fails
with `Permission denied (publickey)` and verbose output mentions `read_passphrase`, the key
simply isn't loaded — `ssh-add --apple-use-keychain ~/.ssh/<your-personal-key>`.

This is how the original development machine is set up; its remotes use the
`github-personal` alias rather than plain `github.com`.

### Step 4 — bring each repo up

```bash
cd backend && make setup    # Supabase CLI, Docker, local database
cd ../ios   && make setup && make test
```

## Layout
```
specs/    product & technical specification
plans/    phased implementation plans, one per delivery slice
design/   the Ours design system and its 14 plates
```

## Contents
| Document | Purpose |
|---|---|
| [specs/2026-08-07-coterie-mvp-design.md](specs/2026-08-07-coterie-mvp-design.md) | MVP design spec — product, data model, architecture |
| [plans/2026-08-07-plan-1-foundation-and-auth.md](plans/2026-08-07-plan-1-foundation-and-auth.md) | Plan 1 — foundation, auth, profile setup |
| [plans/2026-08-09-ci-setup.md](plans/2026-08-09-ci-setup.md) | CI — both pipelines and the self-hosted runner |
| [design/ours-design-system.md](design/ours-design-system.md) | **Ours design system** — tokens, components, plates |

## Conventions
- File paths in plans are **repo-qualified** (`ios/…`, `backend/…`) because the code lives
  in sibling repos. Commands are run from inside the repo named by the task.
- Plans use checkbox steps and are ticked off as they are executed.
