# Coterie — Docs

Whole-project documentation: product spec, implementation plans, and decisions that
span more than one platform. Platform-specific docs live in their own repo.

Part of a three-repo project:

| Repo | Contents |
|---|---|
| [`coterie-ios`](https://github.com/suigeneris/coterie-ios) | SwiftUI app |
| [`coterie-backend`](https://github.com/suigeneris/coterie-backend) | Supabase schema, migrations, RLS, pgTAP tests |
| [`coterie-docs`](https://github.com/suigeneris/coterie-docs) | this repo |

## Getting all three

The GitHub repos are `coterie-*` prefixed; locally they sit unprefixed inside a plain
`coterie/` container folder (which is not itself a repo):

```bash
mkdir -p ~/Developer/coterie && cd ~/Developer/coterie
git clone git@github.com:suigeneris/coterie-docs.git    docs
git clone git@github.com:suigeneris/coterie-ios.git     ios
git clone git@github.com:suigeneris/coterie-backend.git backend
```

## Layout
```
specs/    product & technical specification
plans/    phased implementation plans, one per delivery slice
```

## Contents
| Document | Purpose |
|---|---|
| [specs/2026-08-07-coterie-mvp-design.md](specs/2026-08-07-coterie-mvp-design.md) | MVP design spec — product, data model, architecture |
| [plans/2026-08-07-plan-1-foundation-and-auth.md](plans/2026-08-07-plan-1-foundation-and-auth.md) | Plan 1 — foundation, auth, profile setup |

## Conventions
- File paths in plans are **repo-qualified** (`ios/…`, `backend/…`) because the code lives
  in sibling repos. Commands are run from inside the repo named by the task.
- Plans use checkbox steps and are ticked off as they are executed.
