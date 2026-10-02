# product-specs-blueprint

An [Emergent](https://emergent.company) blueprint that installs the
**product-specs** schema pack into a Memory knowledge-graph project. It models
an OpenSpec-style spec workflow — capabilities, requirements, scenarios,
changes, artifacts, task groups, and tasks — as graph objects, with
release-anchored delivery and test-suite verification.

The goal: **keep product specs in the graph, not in the repository.** Git gave
history; the graph gives history *and* queryability *and* a product timeline
*and* traceability from a requirement to the release that shipped it.

---

## What it installs

- **Schema pack `product-specs` (v0.1.0)** — 9 object types, 10 relationship
  types.
- **Skill `product-specs`** — a bundled authoring skill describing the
  author → implement → deliver → verify → archive loop.

No agents, no seed data.

---

## The model

Product specs map onto graph objects like this:

| OpenSpec artifact | Graph object | Lifecycle |
|---|---|---|
| a capability spec | `Capability` (durable, cumulative) | `active` / `deprecated` |
| a requirement line | `Requirement` (owned by a capability) | `proposed → active → deprecated` |
| a `#### Scenario` | `Scenario` (owned by a requirement) | — |
| a change directory | `Change` + its `Artifact`s | `draft → active → implemented → archived` |
| `tasks.md` checklist | `TaskGroup` → `Task` | `done` flag |
| "shipped" | `Release` | timeline spine |
| the tests that prove it | `TestSuite` | verification |

```
Change ──has_artifact──► Artifact (proposal / design markdown)
  │
  ├──proposes_change (added | modified | removed)──► Capability
  │                                                     │
  │ has_task_group                                      │ has_requirement
  ▼                                                     ▼
TaskGroup ──contains_task──► Task ──addresses──► Requirement ──has_scenario──► Scenario
                                                     │
                              delivered_in ──────────┼──────────► Release
                              verified_by ───────────┴──────────► TestSuite
```

---

## Object types (9)

| Type | Key prefix | Properties |
|---|---|---|
| `Capability` | `cap:` | `name`, `purpose`, `status` (active/deprecated) |
| `Requirement` | `req:` | `text`, `strength` (SHALL/SHOULD/MAY), `origin` (added/modified/removed), `reason`, `migration`, `status` (proposed/active/deprecated) |
| `Scenario` | `scen:` | `when[]`, `then[]` |
| `Change` | `chg:` | `name`, `status` (draft/active/implemented/archived), `workflow`, `created`, `archived_at`, `base_ref` |
| `Artifact` | `art:` | `kind` (proposal/design), `body` (markdown), `order` |
| `TaskGroup` | `tg:` | `ordinal`, `title` |
| `Task` | `task:` | `number`, `text`, `done` |
| `Release` | `rel:` | `version`, `name`, `released_at`, `notes`, `channel` |
| `TestSuite` | `ts:` | `name`, `status`, `passed_at` |

## Relationship types (10)

| Relationship | From → To | Meaning |
|---|---|---|
| `has_requirement` | Capability → Requirement | a capability owns a requirement |
| `has_scenario` | Requirement → Scenario | a requirement owns a scenario |
| `depends_on` | Capability → Capability | capability dependency |
| `proposes_change` | Change → Capability | the change adds/modifies/removes the capability (property `operation`) |
| `has_artifact` | Change → Artifact | the change's proposal/design |
| `has_task_group` | Change → TaskGroup | task-list section |
| `contains_task` | TaskGroup → Task | a section contains a task |
| `addresses` | Task → Requirement | the task delivers this requirement |
| `delivered_in` | Requirement, Change → Release | the authoritative delivery edge |
| `verified_by` | Requirement → TestSuite | verification, orthogonal to delivery |

---

## The workflow loop

**Author → implement → deliver → verify → archive.**

1. **Open a change** — create `chg:<slug>` (`status: draft`).
2. **Attach artifacts** — `art:<slug>:proposal` / `:design` (markdown in `body`).
3. **Declare impact** — `proposes_change` to each affected `Capability` with
   `operation: added|modified|removed` (create the capability if new).
4. **Write the delta** — `Requirement` (`status: proposed`, `origin: …`) +
   `Scenario`s.
5. **Plan the work** — `TaskGroup`s and `Task`s; `addresses` each task to a
   requirement.
6. **Implement** — code branch / PR (PR objects are a later change).
7. **Tick tasks** — `Task.done: true`.
8. **Deliver** — `delivered_in` → `Release` (creates the product timeline).
9. **Verify** — `verified_by` → `TestSuite`.
10. **Archive** — `Change.status: archived`; reconcile the delta:
    `added`/`modified` → `active`; `removed` → `deprecated`.

### Capabilities: how they are constructed

- **Emergent:** the first change that adds a capability creates it.
- **Pre-authored:** seed the capability set up front, then each change modifies
  an existing node.

A capability is cumulative — many changes deposit requirements into it. Each
requirement's `created_at`, `origin`, and `version` record how and when the
behaviour was defined or changed.

---

## Delivery and verification

Delivery and verification are deliberately **separate**:

- **delivered** = a `Release` link exists (`delivered_in`). Answers *when/where
  it shipped*.
- **verified** = a `TestSuite` link exists (`verified_by`). Answers *does it
  work*.

A merged pull request is evidence toward delivery, not proof of it, and not
proof of working — so it is neither. "Delivered but unverified" and "verified
but unreleased" are both valid, queryable states.

## History (what replaces git)

Nothing is lost by leaving files behind — the graph already keeps the history
git used to provide:

- **`created_at`** on every object — when a capability/requirement first existed.
- **Object versioning** — `version`, `supersedes_id`, `change_summary`,
  `revision_count`, and a per-object history endpoint
  (`GET /api/graph/objects/{id}/history`) — the per-object changelog.
- **The project journal** — every mutation (create/update/delete/relate/merge)
  with actor and timestamp, plus manual `note` events for milestones.
- **Archives are retained** — removed requirements become `deprecated`, never
  deleted.

---

## Installation

```bash
memory blueprints validate ./product-specs-blueprint        # offline check
memory blueprints install ./product-specs-blueprint --project <project-slug>
```

Or from GitHub:

```bash
memory blueprints install https://github.com/mkucharz/product-specs-blueprint --project <project-slug>
```

---

## Design decisions

- **Graph-only source of truth.** Specs are graph objects; there is no
  `openspec/` directory to edit. The bundled skill enforces this.
- **Requirements have a stable identity.** A requirement links to its capability
  from the moment it is authored (`status: proposed`); archiving flips it to
  `active`. This keeps `delivered_in`/`verified_by` edges stable across the
  change lifecycle. Modifying a requirement edits its `text` (a new object
  version), it does not create a second object.
- **Delivery is release-anchored, not PR-derived.** A `Release` edge is
  authoritative; `Requirement.status` carries a derived `delivered` mirror only
  for fast filtering.
- **Verification is orthogonal to delivery.**
- **History is native.** No `HistoryEvent` type; `Release` is the only new
  timeline node.
- **Self-contained pack.** The blueprint validator requires relationship
  endpoints to be object types declared in the same pack, so `TestSuite` is a
  thin in-pack type (see caveats).

---

## Caveats

- **Type-name collisions.** `Scenario` and `TestSuite` collide by name with the
  `code-memory-blueprint` pack. The two packs are not intended to be installed
  into the same project — this bundle targets a **specs-only graph**.
- **No pull-request traceability yet.** This first cut ships the schema pack and
  skill only, so there is no `PullRequest` object or PR→spec link. A later
  change adds a `PullRequest` type and an agent/CI writer that records PRs (and
  their merge state) into the graph.
- **Graph-only retires the OpenSpec CLI path.** Once specs live only in the
  graph, file-oriented tooling (the `openspec` CLI, repo archive guards) no
  longer applies; a later change provides graph-backed equivalents.
- **Cross-pack traceability is a platform limitation.** Linking a requirement to
  code-structure/code-review types would require either duplicated types or
  cross-pack relationship support. Deferred.

## Prerequisites

- A Memory project and the `memory` CLI.
- No agents, no seed data, no external MCP servers required.

## License

MIT
