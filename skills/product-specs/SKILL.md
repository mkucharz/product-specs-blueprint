---
name: product-specs
description: Author, deliver, and verify product specs in the Memory knowledge graph using the product-specs schema — Capability, Requirement, Scenario, Change, Artifact, TaskGroup, Task, Release, TestSuite — instead of keeping openspec/ files in the repo. Use when creating or revising a spec change, listing capabilities or requirements, marking a feature delivered in a release, linking verification, or tracing task→requirement coverage.
metadata:
  author: emergent
  version: "0.1"
---

Operate product specs as knowledge-graph objects. Specs live in the graph, not
as `openspec/` files — there is no spec directory to edit. All authoring is
graph writes (`memory graph` or the MCP graph tools).

## Rules

- **Graph-only.** Do not create or edit `openspec/` files. The graph is the
  source of truth.
- **Keyed objects.** Give every object a stable `key` so re-applies are
  idempotent. Never leave a spec object keyless.
- **Author a `Change`.** Never edit a live capability wholesale; every
  capability change flows through a `Change`.
- **Stage, then archive.** Requirements start `status: proposed`; they become
  `status: active` only when the owning `Change` is archived.
- **Delivery is explicit.** Marking a feature delivered means linking it to a
  `Release` via `delivered_in` — not just flipping a status.
- **Never delete.** Removed requirements become `status: deprecated`; the object
  and its history are retained.

## Object model

| Type | Key prefix | Role |
|---|---|---|
| `Capability` | `cap:` | durable, cumulative spec domain (e.g. `cap:auth`) |
| `Requirement` | `req:` | a behavioural requirement of a capability |
| `Scenario` | `scen:` | a WHEN/THEN example of a requirement |
| `Change` | `chg:` | a proposed change (authoring/delivery unit) |
| `Artifact` | `art:` | a change's proposal or design markdown |
| `TaskGroup` | `tg:` | a numbered section of the task list |
| `Task` | `task:` | a checklist item |
| `Release` | `rel:` | a shipped release (delivery anchor, timeline spine) |
| `TestSuite` | `ts:` | the suite that verifies a requirement |

## Relationships

| Relationship | From → To |
|---|---|
| `has_requirement` | Capability → Requirement |
| `has_scenario` | Requirement → Scenario |
| `depends_on` | Capability → Capability |
| `proposes_change` | Change → Capability (property `operation`: added/modified/removed) |
| `has_artifact` | Change → Artifact |
| `has_task_group` | Change → TaskGroup |
| `contains_task` | TaskGroup → Task |
| `addresses` | Task → Requirement |
| `delivered_in` | Requirement, Change → Release |
| `verified_by` | Requirement → TestSuite |

## The loop

Author → implement → deliver → verify → archive.

1. **Open a change.** Create `chg:<slug>` with `status: draft`, `workflow:
   spec-driven`, `created`, `base_ref`.
2. **Attach artifacts.** Create `art:<slug>:proposal` and `art:<slug>:design`
   (full markdown in `body`) and wire `has_artifact`.
3. **Declare impact.** For each affected capability wire `proposes_change` with
   `operation: added | modified | removed`. If the capability is new, create
   `cap:<slug>` first.
4. **Write the delta.** Create each `Requirement` (`status: proposed`, `origin:
   added | modified | removed`), wire `has_requirement` to its capability, and
   add `Scenario`s via `has_scenario`.
5. **Plan the work.** Create `TaskGroup`s and `Task`s (`has_task_group`,
   `contains_task`), and wire `addresses` from each task to the requirement it
   delivers.
6. **Implement.** Do the code work on a branch / PR. (PR objects are not yet in
   this schema — a later change adds them.)
7. **Tick tasks.** Set `Task.done: true` as work completes.
8. **Deliver.** Create or reuse `rel:<version>` and wire `delivered_in` from the
   delivered requirements and the change. `Requirement.status` becomes
   `delivered`.
9. **Verify.** Wire `verified_by` from a requirement to its `TestSuite`;
   the requirement becomes `verified`. Verification is independent of delivery —
   "delivered but unverified" and "verified but unreleased" are both valid.
10. **Archive.** Set `Change.status: archived` and `archived_at`, then reconcile
    the delta onto the capability: `origin: added|modified` → `status: active`;
    `origin: removed` → `status: deprecated`.

Modifying an active requirement is an **edit to its `text`** — the platform
records a new object version (`version++`, `supersedes_id`, `change_summary`),
which is the git-diff equivalent. Do not create a second requirement object.

## Capability construction

- **Emergent:** the first change that adds a capability creates it
  (`proposes_change` with `operation: added`).
- **Pre-authored:** seed the capability set up front, then have each change
  modify an existing node.

A capability accumulates requirements across many changes; each requirement's
`created_at`, `origin`, and `version` trace how and when the behaviour changed.

## History (no git needed)

- `created_at` on every object — when a capability/requirement first existed.
- Object version fields (`version`, `supersedes_id`, `change_summary`) and the
  object history endpoint — per-object changelog.
- The project journal — every mutation plus manual `note` events.

## Querying

Use graph queries/`memory graph` to answer: what a capability requires, what a
change proposes, what shipped in a release (traverse `delivered_in`), which
requirements are delivered but unverified, and which tasks address a
requirement.
