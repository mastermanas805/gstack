# SYSTEM.md — the semantic contract graph

`SYSTEM.md` is a declarative file at the root of a repo that describes what
each component *is*, what it *owns*, and the role-level contracts it has with
other components. It's the input to upcoming `/plan-rollout` planning (landing
in a follow-up PR); consumers will include `/spill-check`, `/ship` (stack
mode), and `/review` (scope verification).

This PR lands only the schema spec. Library code (parser, reconciler,
scaffolder) and the consuming skills follow in subsequent PRs, gated on the
primitive landing here first.

## What SYSTEM.md is NOT

It is not a package manifest. It does not list:

- Import graphs or symbol-level callers
- npm / Cargo / Gem / Go module versions
- Build dependencies or linker flags
- Test framework wiring

Everything mechanical is discovered at runtime (AST, grep, package manifests).
Declaring it here would go stale within a week.

## What SYSTEM.md IS

The **semantic contract graph**: the relationships between components that
only a human knows.

| Kind | Example | Lives where |
|------|---------|-------------|
| Role/contract dep | "auth mints session tokens middleware enforces; format change without middleware redeploy breaks sessions" | SYSTEM.md |
| Package/import dep | "`auth.ts` imports `crypto-utils`; `middleware.ts` calls `auth.verify()`" | Discovered (NOT here) |

`/plan-rollout` reconciles the declared graph with the discovered graph.
Declared contracts give the *why*, discovered imports give the *what*, and
disagreements surface for human resolution rather than being silently
resolved either way.

## Schema (v1 — intra-repo)

```yaml
---
version: 1
components:
  - name: <unique identifier>
    path: <repo-relative path to the component root>
    repo: <owner/repo>         # reserved for v2 multi-repo; optional in v1
    kind: component            # or: leaf-util | types-only (optional; defaults to 'component')
    role: <one-line description of the component's job>
    owns:
      - <data surface, table, API, or feature this component is source-of-truth for>
    contracts:
      - with: <other component name>
        nature: <what the relationship is in plain English>
        breaks-if: <what human action causes the contract to break>
        rollout-edge: <hard | soft>
        note: <optional; 'runtime-only' | 'types-only' | 'legacy' | free text>
    rollout-order: <integer; lower ships first; same number can ship in parallel>
---

# System Map

<Free-form markdown narrative: component stability, anti-patterns, deploy-edge
semantics the team has learned from incidents. This section is for humans, not
for parsers.>
```

### Field reference

**`name`** — unique identifier used by other artifacts to reference this
component. Keep it short and stable. Renames cascade to every contract
reference.

**`path`** — where the component lives in the repo. Can be a file or
directory. Used for component-membership lookups (is `src/auth/session.ts` in
the `auth` component? yes).

**`kind`** — defaults to `component`. Use:
- `leaf-util` for shared utility dirs (`src/utils/`, `src/helpers/`) that
  components import freely without declaring contracts. Reconciler skips these.
- `types-only` for pure type/interface modules. Reconciler skips these.

**`role`** — one sentence describing what this component is FOR. Not what it
contains, not how it's built. What it does in the system.

**`owns`** — data surfaces, tables, APIs, or features this component is the
single source of truth for. Two components claiming ownership of the same
surface is a design smell.

**`contracts`** — the heart of SYSTEM.md. Each contract declares a role-level
relationship with another component.

- **`with`** — the other component's name. Must match a declared component.
- **`nature`** — plain-English description of the relationship.
- **`breaks-if`** — the specific human action that violates the contract.
  This is what rollout planning reads. "Session payload schema changes
  without middleware redeploy" tells `/plan-rollout` these two PRs must
  ship as a coordinated stage.
- **`rollout-edge`**:
  - `hard` = must deploy together (e.g., a session format change). The
    skill enforces same-step deploy or blocks with explanation.
  - `soft` = can lag (e.g., a logging metric addition). Noted but not
    enforced.
- **`note`** (optional) — common values:
  - `runtime-only` — coupling via DB, message bus, HTTP, or filesystem; no
    code-level import edge exists. Suppresses the "contract without imports"
    reconciler flag.
  - `types-only` — TypeScript types only, not runtime values.
  - `legacy` — contract exists but is being phased out.

**`rollout-order`** — integer. Components with lower numbers ship first.
Equal numbers can ship in parallel. Used as the default ordering for PR-stack
decomposition; the user can override per-decomposition.

## Example

```yaml
---
version: 1
components:
  - name: auth
    path: src/auth
    role: authentication + session lifecycle
    owns:
      - user table
      - session table
      - JWT minting
    contracts:
      - with: middleware
        nature: middleware enforces session tokens auth mints
        breaks-if: session payload schema changes without middleware redeploy
        rollout-edge: hard
    rollout-order: 1

  - name: middleware
    path: src/middleware
    role: request routing + auth enforcement
    owns:
      - request context shape
    contracts:
      - with: gateway
        nature: gateway consumes req.user set by middleware
        breaks-if: req.user shape changes without gateway redeploy
        rollout-edge: hard
    rollout-order: 2

  - name: gateway
    path: src/gateway
    role: external HTTP surface
    owns:
      - public API schema
    contracts: []
    rollout-order: 3

  - name: utils
    path: src/utils
    kind: leaf-util
    role: shared helpers — imported freely without contracts
    owns: []
    contracts: []
    rollout-order: 0
---

# System Map

auth and middleware are the security boundary. Any change to session format
or the user-context shape is a coordinated deploy (rollout-edge: hard). We
learned this after the Feb 2025 incident where a session serializer change
shipped 40 minutes ahead of middleware and logged everyone out.

utils is declared `leaf-util` so the reconciler doesn't flag the many imports
into it from other components as missing contracts.
```

## Scaffolding

Repos without an existing SYSTEM.md can generate a draft via the scaffolder
(`scaffoldSystemMap` in `lib/plan-rollout`). It walks the top-level `src/`
subdirectories (or top-level directories if there's no `src/`), classifies
each as `component`, `leaf-util`, or `types-only` based on name heuristics,
infers a starting `role` from the directory's README or `package.json`
description, and writes to `SYSTEM.md.draft`.

The scaffolder never overwrites an existing `SYSTEM.md`. The user reviews the
draft, fills in `owns`, `contracts`, and `rollout-order`, and renames
`SYSTEM.md.draft` → `SYSTEM.md` when ready.

## Reconciliation

The reconciler (`reconcile` in `lib/plan-rollout`) takes a parsed `SystemMap`
and a list of discovered `ImportEdge[]` and returns `ReconcileFlag[]` for
three categories:

| Category | Meaning | Resolution |
|----------|---------|------------|
| `import-without-contract` | Components A and B are connected by imports, but SYSTEM.md declares no contract | Add a contract, or refactor to remove the layering violation |
| `contract-without-imports` | SYSTEM.md declares a contract but no import edges support it | Remove the stale contract, or add `note: runtime-only` if the coupling is via DB / message bus / HTTP / filesystem |
| `rollout-order-inversion` | A imports from B but A ships before B | Swap rollout-order values, or add `note: types-only` if the imports are types-only |

Leaf-util and types-only components are excluded from reconciliation —
edges to/from them are declared exceptions.

## Relationship to other declarative files

| File | Purpose | Who writes it |
|------|---------|---------------|
| `CLAUDE.md` | Project-specific instructions for Claude (routing rules, test commands) | Human |
| `CODEOWNERS` | Who reviews changes to which paths | Human |
| `SYSTEM.md` | Semantic contract graph | Human (scaffolded, then edited) |
| `decomposition.md` | Per-change PR stack | `/plan-rollout` (follow-up PR) |
| `rollout.md` | Per-change rollout plan | `/plan-rollout` (follow-up PR) |

SYSTEM.md is the long-lived, repo-wide truth. `decomposition.md` and
`rollout.md` are per-change artifacts that reference it.
