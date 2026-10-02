## Why

Product specs in this organisation live as OpenSpec files in a git repo
(`openspec/changes/**`, `openspec/specs/**`). Git gives history, but the specs
stay outside the product's knowledge graph: they cannot be queried alongside
code, delivery cannot be traced to releases or pull requests, and there is no
first-class way to mark when a behaviour shipped. Moving specs into the graph
as objects makes them queryable, traceable, and connected to the rest of the
project.

## What Changes

- Add a new standalone blueprint repository, `product-specs-blueprint`, that
  installs a `product-specs` schema pack into a Memory project.
- The pack models the OpenSpec grammar as graph objects: `Capability`,
  `Requirement`, `Scenario`, `Change`, `Artifact`, `TaskGroup`, `Task`, plus
  `Release` and `TestSuite` for delivery and verification.
- Model **delivery** as a deliberate act: a first-class `Release` object joined
  by `delivered_in`; `Requirement.status` carries a derived `delivered` mirror.
- Model **verification** separately and orthogonally to delivery: `verified_by`
  from `Requirement` to a thin in-pack `TestSuite`.
- Stage requirements as `proposed` and reconcile them onto the durable
  `Capability` as `active` when the owning `Change` is archived.
- Bundle one authoring skill (`skills/product-specs/SKILL.md`) describing the
  author → implement → deliver → verify → archive loop.
- Document the graph-only operating model and the native history primitives
  (object `created_at`/`version`/`supersedes_id`, object history endpoint,
  journal) that replace git history.
- **Non-goal:** no CLI, import/export tooling, PR ingestion, or archive-guard
  replacement in this change — this change ships the schema pack and skill only.

## Capabilities

### New Capabilities

- `product-specs-blueprint`: the installable blueprint bundle — the
  `product-specs` schema pack (object and relationship contract for
  capabilities, requirements, scenarios, changes, artifacts, task groups,
  tasks, releases, and test suites), the staged `proposed → active`
  requirement lifecycle, delivery and verification modelling, and a bundled
  authoring skill.

### Modified Capabilities

<!-- None: this is a new, standalone blueprint repository with no prior specs. -->

## Impact

- **New repo artifacts:** `schemas/product-specs.yaml`,
  `skills/product-specs/SKILL.md`, `project.yaml`, `README.md`, `LICENSE`.
- **Dependencies:** the Memory CLI (`memory blueprints install|validate`).
  No server or API changes.
- **Compatibility:** the pack declares `Scenario` and `TestSuite` types that
  collide by name with the `code-memory-blueprint` pack; the two packs are not
  intended to be installed into the same project. The bundle targets a
  specs-only graph.
