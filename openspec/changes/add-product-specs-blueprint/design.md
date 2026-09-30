## Context

Product specs are currently OpenSpec markdown files in git. Git supplies
history, but the specs are disconnected from the product's knowledge graph:
requirements cannot be queried next to code, delivery cannot be tied to
releases, and "when did this ship" is not a first-class query. This change
introduces a blueprint that installs a schema pack modelling the OpenSpec
grammar as Memory graph objects. This repository is the blueprint's home; it is
a standalone repo (like `code-memory-blueprint`), installed via
`memory blueprints install`.

Constraints that shaped the design:

- **Blueprint pack format** (`apps/cli/internal/blueprints/{types.go,loader.go}`
  in the Memory monorepo): a pack lives at `schemas/<name>.yaml`; top-level
  `name`/`version` are required (version a quoted string) and `objectTypes` must
  be non-empty.
- **In-pack relationship endpoints:** the blueprint validator rejects a
  relationship whose `sourceType`/`targetType` is not an object type declared in
  the same pack, and the server validates relationship types against the
  installed schema at write time. A single pack therefore cannot reference
  another pack's types, and undeclared relationships are not viable.
- **Native history already exists:** graph objects/relationships carry
  `created_at`, `version`, `supersedes_id`, `change_summary`, `revision_count`
  and are exposed via a per-object history endpoint; the project `journal`
  records every mutation (including manual `note` events). History is therefore
  not re-modelled as a schema type.

## Goals / Non-Goals

**Goals**

- Model capabilities, requirements, scenarios, changes, artifacts, tasks,
  releases, and test suites as graph objects with a coherent relationship
  contract.
- Make delivery an explicit, queryable act (release-anchored), distinct from
  verification.
- Keep the pack self-contained so it validates and installs standalone.
- Document the graph-only operating model and the native history primitives.

**Non-Goals**

- No CLI, import/export tooling, archive-guard replacement, or PR ingestion —
  those are later phases.
- No cross-pack linkage to code-structure or code-review types (blocked by the
  in-pack endpoint rule).

## Decisions

### D1 — One pack, endpoints in-pack

All ten relationships resolve entirely within `product-specs`. Where a link
would naturally cross into another pack (`verified_by` → `TestSuite`, which the
code-structure pack also declares), the pack declares its own thin type rather
than reaching across packs. Trade-off: `Scenario` and `TestSuite` collide by
name with `code-memory-blueprint`; the bundle targets a specs-only graph, and
the collision is documented.

### D2 — Requirement identity is stable; status carries the delta

A `Requirement` is linked to its `Capability` from the moment it is authored,
with `status: proposed`. On change archive the delta is reconciled:
`added`/`modified` → `active`; `removed` → `deprecated` (never deleted).
Alternative considered: stage requirements under the `Change` and re-parent them
at archive — rejected because it breaks a stable identity, forcing
`delivered_in`/`verified_by` edges to be re-created.

Modifying an active requirement edits its `text`, which the platform records as
a new object version (`version++`, `supersedes_id`, `change_summary`) — the
git-diff equivalent — rather than a new object.

### D3 — Delivery is release-anchored, not PR-derived

`delivery` is modelled as `delivered_in` from `Requirement`/`Change` to a
first-class `Release`. `Requirement.status` carries a derived `delivered`
mirror for fast filtering, but the `Release` edge is authoritative.
Alternatives rejected: a bare `delivered_at` property (cannot answer "what
shipped in this release"); deriving delivery from PR merge state (PR state is
written by an external, eventually-consistent sync — merge is evidence toward
delivery, not proof of it).

### D4 — Verification orthogonal to delivery

`verified_by` links a `Requirement` to a `TestSuite` independently of delivery,
so "merged but unreleased" and "released but unverified" remain distinct,
queryable states.

### D5 — History stays native

The product timeline is served by `created_at`, per-object versioning, the
object history endpoint, and the journal. No `HistoryEvent`/`Milestone` type is
introduced; `Release` is the only new timeline node.

## Risks / Trade-offs

- **Type-name collisions** with `code-memory-blueprint` (`Scenario`,
  `TestSuite`) → documented; specs-only graph is the supported target.
- **No PR traceability yet** → deliberately deferred; a future change adds a
  `PullRequest` type and an agent/CI writer.
- **Graph-only breaks the OpenSpec CLI and the repo archive guard** → expected;
  a future phase replaces them with graph-backed equivalents.
- **Pack schema drift** → mitigated by `memory blueprints validate` in CI.

## Migration Plan

Not applicable — this is a new repository with no existing data. Rollout is
`memory blueprints validate` then `memory blueprints install`.

## Open Questions

- None blocking. Cross-pack/shared-type-registry support (to retire the thin
  duplicated types) is a platform follow-up, not part of this change.
