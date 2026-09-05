# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

`metap-org` is a **root repo of git submodules**, not a monorepo — `metap`, `metap-docs`,
`metap-lowcode`, `metap-demo-waf`, `metap-themes`, `design-system`, and `platform-ui` are each an
independent git repo with its own remote/history, linked in here as a submodule (`.gitmodules`).
There is still no root build file and no cross-repo CI — this repo exists to pin which commit of
each sibling repo goes together and to hold shared cross-repo docs/scripts, not to build anything
itself. Each submodule is its own repo with its own history, and several of them already have
their own `CLAUDE.md`/`README.md` — **read the target repo's own docs before working in it.**

`metap-demo-crm` and `metap-demo-jira` are the two exceptions: they exist on disk here but have no
GitHub remote yet, so they are **not submodules** (see `.gitignore`) — plain local git repos this
root repo doesn't track at all. Add them as submodules once they have a remote (`git submodule add
<url> metap-demo-crm`, then drop the two `.gitignore` lines that currently exclude them).

**Precedence: this file sets the default/baseline for every repo here; a repo's own `CLAUDE.md`
can add repo-specific rules or override a default from this file, and wins inside that repo when
the two disagree.** A repo with no `CLAUDE.md` of its own (`metap-lowcode`, `metap-demo-crm`,
`metap-demo-jira`, `metap-docs`, `design-system`, `platform-ui`, `metap-themes` — none exist yet)
follows this file's defaults with nothing overriding them. Silence in a repo's own `CLAUDE.md` on a
given point means this file's default still applies there too — a repo's file only needs to state
what it changes, not restate everything from here.

## Submodules

- **Fresh clone**: `git clone --recurse-submodules <this-repo-url>`, or if already cloned plain:
  `git submodule update --init --recursive`. Without this, every submodule directory is empty.
- **A submodule's own commits/branches/PRs work exactly like before** — `cd metap && git checkout
  -b ... && ... && git push` is unchanged; this root repo does not intercept or wrap that. Nothing
  about working inside one repo (this whole file's rules included) changes because it's now also a
  submodule.
- **This root repo only records a pointer** (one commit SHA per submodule, shown as `A` no
  content diff in `git status`/`git diff` here) — after a submodule's own change is pushed to its
  own remote, bump the pointer here deliberately: `git add <submodule-path> && git commit -m
  "Bump <submodule> to <short-sha-or-reason>"`. This is a separate, explicit step, never automatic
  — `git status` here shows a submodule as "modified content" only when its own working tree has
  uncommitted changes, and shows the gitlink itself as changed only once you've actually checked
  out a different commit/branch inside it.
- **Never commit a submodule bump for a change nobody asked to persist** — same "never commit
  without being explicitly asked" rule as everywhere else in this file, applied one level up: don't
  bump a pointer just because you happened to `cd` into a submodule and made a local commit there
  during unrelated work.
- `git submodule foreach 'git status --short'` is the fast way to check every submodule for
  uncommitted/unpushed work at once before assuming the tree is clean.

```
metap/            Rust backend core — metadata-driven CRUD/permission/workflow platform library
metap-docs/       Design docs, roadmap, feature briefs, ADRs for `metap` (docs-only repo)
metap-lowcode/    SaaS low-code control-plane layer on top of `metap` (entity builder, tenant provisioning API)
metap-demo-crm/   Downstream example app #1: CRM (shared-schema tenant, generic `records` table)
metap-demo-jira/  Downstream example app #2: Jira-like (dedicated-DB tenant, table-per-entity, kanban board)
metap-demo-waf/   Downstream product: WAAP/Cloudflare-style portal — data-plane + control-plane + edge-plane, see its own CLAUDE.md
design-system/    `@metap/ui` — Tailwind + Radix + shadcn-style component library (the design-system layer)
platform-ui/      `@metap/platform-ui` — generic list/form/detail/workflow admin UI, built on `@metap/ui`
metap-themes/     Theme packages (enterprise/fintech/saas/storefront/creative) built on top of `@metap/ui`
```

Everything is wired together via sibling-relative paths, not package versions:
- Rust repos depend on `../metap/crates/*` (and, for `metap-demo-crm`, also `../metap-lowcode`'s
  crates) as Cargo **`path` dependencies** — local dev only, meant to be swapped to `git`/version
  deps for CI/deploy. This means none of the demo repos build standalone in Docker yet (their build
  context can't see `../metap`).
- Frontend repos depend on `design-system` and `platform-ui` via pnpm **`link:../design-system`** /
  **`link:../platform-ui`** — real sibling repos, not published npm packages. `pnpm install` in a
  `web/` directory requires the sibling repos to exist at that exact relative path.

**Split history**: `metap` used to contain everything (frontend, demo apps, low-code layer, docs)
and was progressively split into these sibling repos through 2026-08-28 → 2026-08-31 (frontend →
Phase 47, demo apps → Phase 51, low-code control-plane → Phase 52, docs → the `54-docs-repo-split`
entry) so that `metap` itself is a pure Rust library workspace. A large number of comments/docs
inside `metap` still say `docs/...` meaning `../metap-docs/docs/...` — this was a deliberate choice
made when splitting docs out, not a bug to fix.

## Where to actually start work

There is no single build/test command for this whole directory — go into the relevant repo and use
its own tooling:

| Task | Go to |
|---|---|
| Backend platform primitives (CRUD, permission, workflow, query, outbox, reconciler) | `metap/` — `cargo build --workspace` / `cargo test --workspace` from its root |
| Run something end-to-end (mint a token, migrate a DB, serve HTTP) | `metap-demo-crm/`, `metap-demo-jira/`, or `metap-demo-waf/data-plane/` — each has its own README with the full local-run recipe |
| Low-code entity builder / self-service tenant provisioning HTTP API | `metap-lowcode/` — `cargo build --workspace` / `cargo test --workspace` |
| Shared UI atoms/primitives (Button, Dialog, Table, form controls...) | `design-system/` — `pnpm install && pnpm test` / `pnpm storybook` |
| Generic admin screens (list/detail/form/workflow, permission UI) composed from those atoms | `platform-ui/` — `pnpm typecheck && pnpm lint && pnpm format:check` (no build step of its own; consumed straight from `src/` by whichever app's Vite bundler) |
| Architecture docs, roadmap, ADRs for `metap` | `metap-docs/` (docs only, no code) |

## Architectural layering (applies across every repo here)

```
design-system (@metap/ui)         — pure UI atoms, Tailwind + Radix, no business logic
        ↑ composed by
platform-ui (@metap/platform-ui)  — generic business screens (list/detail/form/workflow), no backend coupling
        ↑ consumed by
metap-demo-crm/web, metap-demo-jira/web, metap-demo-waf/data-plane/web
        ↕ HTTP (REST/GraphQL/gRPC)
metap-demo-* / metap-demo-waf backend binaries
        ↑ built on
metap (crates/metap-*)            — entity-agnostic core: metadata, permission, query, workflow, outbox
        ↑ optionally extended by
metap-lowcode                     — lets a downstream product define entities via API instead of Rust code
```

**Strict rule enforced by convention across all of these**: no core library crate/package
(`metap`'s `crates/`, `design-system`) is allowed to know about a specific business entity
(`crm.customers`, `jira.issues`, WAF's `Zone`, etc.). Business-entity knowledge lives only in a
downstream app (`metap-demo-*`) or in composed screens (`platform-ui`). If you find yourself adding
entity-specific logic to `metap` or `design-system`, it's in the wrong repo.

Same one-way rule for UI: `platform-ui` must not hand-roll a new UI primitive — every atom, even a
small one, belongs in `design-system` (`@metap/ui`) and gets imported back. `platform-ui` only
*composes* existing `@metap/ui` components into business screens. See `platform-ui/README.md`'s
"Nguyên tắc kiến trúc" section for the history of this rule being enforced after violations were
found.

## Cross-repo conventions worth knowing before you dig into any one repo

- **Docs are written in Vietnamese by default, in every repo here** — a deliberate, explicit
  policy while the team is small (see `metap/CLAUDE.md`, where this was first stated before being
  promoted to this file). Applies unless a repo's own `CLAUDE.md` says otherwise for that repo (see
  "Precedence" above) — none currently do. Code identifiers, comments, commit messages, and each
  repo's own `CLAUDE.md` stay in English regardless, everywhere.
  **Known gap, not yet fixed**: `metap`, `metap-docs`, `metap-demo-waf`, and `platform-ui`'s own
  `README.md` already follow this. `metap-lowcode` (new repo, split out 2026-08-31) does not —
  its `README.md`/`docs/architecture.md` were written in English by mistake (matching the English
  doc-comments in the code it was extracted from, rather than this policy) and haven't been
  translated back. Under this file's default, that's non-compliant, not an exception — translate
  them next time `metap-lowcode`'s docs are touched, or ask before assuming it's fine to leave.
- **Frontend verification policy**: don't self-verify frontend changes with browser automation
  (Playwright etc.) — write the code, run typecheck/lint, then hand it to the user to check in a
  real browser themselves.
- **Rust repos all follow the same path-dependency convention**: point at `../metap/crates/*` for
  local dev, swap to `git`/version deps for CI/deploy — check the comment above the dependency in
  each repo's `Cargo.toml` before "fixing" what looks like a hardcoded local path. `metap`'s own
  `crates/metap-runtime` is cross-cutting plumbing (HTTP client with a default timeout, bearer-token
  parsing, env-var helpers, the `{"error":{"code":...}}` axum response shape) meant to be reused the
  same way by any Rust repo here that needs it, not just `metap`'s own binaries — check there before
  hand-rolling something it already has (`metap-lowcode`'s crates already depend on it).
- `metap-demo-crm` and `metap-demo-jira` intentionally demonstrate two different tenancy models
  side by side: shared-schema (`crm`, generic `records` table) vs. dedicated-DB + table-per-entity
  (`jira`). Don't assume one app's data-access pattern generalizes to the other.
- Ports are deliberately staggered so demo apps can run side by side: `metap-demo-crm` backend
  :3000 / frontend :5173, `metap-demo-jira` backend :3100 / frontend :5174.
- **`metap` and `metap-lowcode` deliberately disagree on monolith vs. microservices** — this is
  not an inconsistency to "fix" if you notice it. `metap`-core stays a modular monolith (see its own
  `docs/architectures/09-adr.md` in `../metap-docs`, "Không tách microservice cho hướng SaaS
  multi-tenant") because its business CRUD hot path needs ACID across audit/outbox/lock that
  splitting into services would cost. `metap-lowcode` (control-plane admin: entity authoring,
  tenant provisioning — no CRUD hot path) chose the opposite, on purpose, from day one, so its
  crates can be handed to different agents/people to build as independent services — see
  `metap-lowcode/docs/architecture.md`.

## Operational conventions that apply across every repo here, not just one

Defaults from this file per "Precedence" above — only `metap/CLAUDE.md` states them explicitly
today (that's where they were first written), but they've governed work across every repo in this
directory in practice, not just `metap`:

- **Respond in Vietnamese** (chat/explanations to the human) regardless of which repo the work is
  in — established by `metap/CLAUDE.md` but applied the same way working in `metap-lowcode`/
  `metap-demo-crm`/`metap-demo-jira`/`metap-docs` too, none of which have their own `CLAUDE.md` yet
  to say so themselves.
- **Never commit without being explicitly asked, and only the repo(s) actually named** — stage
  changes (`git add`), leave them for review, and don't treat "commit" after discussing 2 repos as
  blanket permission for every repo that happens to have pending changes at that moment.
- **Check the shared target dir's size before a Rust build session in *any* Rust repo here**
  (`metap`, `metap-lowcode`, `metap-demo-crm`, `metap-demo-jira`, `metap-demo-waf/data-plane`).
  Since 2026-09-02 these 5 repos no longer have their own `target/` — each one's `.cargo/config.toml`
  redirects `[build] target-dir` to one shared `metap-org/.shared-target` (relative path, so it
  works regardless of where `metap-org` is checked out; `metap-demo-waf/data-plane` points 2 levels
  up instead of 1, since it sits 1 directory deeper than the other 4). This was done specifically
  because they path-depend on `../metap/crates/*` and were compiling a large overlapping set of the
  same third-party crates independently — ~49GB combined across 4 repos' own `target/` before the
  merge. **The 40GB `cargo clean` threshold below (originally per-repo, `metap/CLAUDE.md`) now
  applies to this one shared directory's total size, not to any single repo** — `du -sh
  metap-org/.shared-target`, not `du -sh target` from inside a repo (that path doesn't exist
  there anymore). **`cargo clean` run from inside any one of the 5 repos wipes the whole shared
  directory**, not just that repo's own artifacts — it deletes every other repo's cached
  incremental build too, so the next build in any of them starts cold. See `metap/CLAUDE.md` for
  the original 96GB-crash incident this rule came from (pre-dates the shared-dir merge, but the
  same disk-pressure risk is now shared across all 5 repos instead of isolated to one).
- **Splitting something out of a repo into a new sibling repo** (done 4 times so far: frontend
  library → `../design-system`/`../platform-ui`, demo apps → `../metap-demo-crm`/`../metap-demo-jira`,
  low-code control-plane → `../metap-lowcode`, docs → `../metap-docs`) follows the same order every
  time: create the new repo and verify it builds/tests standalone *before* deleting anything from
  the source; copy tracked files via `git ls-files` (never `find`, to avoid picking up local
  `.env`/`keys/`); `git init` the new repo and stage everything but never auto-commit it; update
  living docs (`CLAUDE.md`/`README.md`) in both repos, but never rewrite a historical
  `docs/roadmap/NN-*.md` phase entry — only add a new one, or collapse an old row to a short pointer.
