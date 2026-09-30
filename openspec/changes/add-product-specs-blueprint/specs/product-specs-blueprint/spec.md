## Purpose

Defines the installable `product-specs` blueprint: a Memory schema pack that
models product specs — capabilities, requirements, scenarios, changes,
artifacts, task groups, tasks, releases, and test suites — as knowledge-graph
objects, together with a bundled authoring skill. The blueprint moves product
specs out of repository files and into the graph while preserving a queryable
product timeline and delivery/verification traceability.

## ADDED Requirements

### Requirement: Pack models product-spec objects

The blueprint SHALL install a schema pack named `product-specs` declaring these
object types: `Capability`, `Requirement`, `Scenario`, `Change`, `Artifact`,
`TaskGroup`, `Task`, `Release`, and `TestSuite`.

#### Scenario: Pack validates
- **WHEN** a user runs `memory blueprints validate` against this blueprint
- **THEN** validation exits 0 with no errors

#### Scenario: Object types are installed
- **WHEN** the pack is installed into a project
- **THEN** all nine object types exist in the project schema

### Requirement: Pack declares the relationship contract

The pack SHALL declare the relationship types `has_requirement`, `has_scenario`,
`depends_on`, `proposes_change`, `has_artifact`, `has_task_group`,
`contains_task`, `addresses`, `delivered_in`, and `verified_by`, and every
relationship source and target type SHALL name an object type declared in the
same pack.

#### Scenario: Relationship endpoints resolve in-pack
- **WHEN** validation runs
- **THEN** every relationship `sourceType`/`targetType` names an object type declared in the pack

#### Scenario: A change proposes a capability with an operation
- **WHEN** a `Change` proposes a `Capability`
- **THEN** the `proposes_change` relationship carries an `operation` of `added`, `modified`, or `removed`

### Requirement: Requirements stage as proposed

A `Requirement` SHALL be linked to its `Capability` when authored and SHALL
carry `status: proposed` until the owning `Change` is archived, at which point a
requirement whose `origin` is `added` or `modified` SHALL become `status:
active`.

#### Scenario: New requirement starts proposed
- **WHEN** a requirement is authored for a new or modified capability
- **THEN** it carries `status: proposed` and an `origin` of `added`, `modified`, or `removed`

#### Scenario: Archive activates the delta
- **WHEN** the owning change is archived
- **THEN** requirements with origin `added` or `modified` become `status: active`

#### Scenario: Removed requirement is deprecated not deleted
- **WHEN** a change removes a requirement and is archived
- **THEN** the requirement's status becomes `deprecated` and the object is retained

### Requirement: Delivery is modelled explicitly

The blueprint SHALL provide a `Release` object type and a `delivered_in`
relationship so a delivered `Requirement` or `Change` is linked to the `Release`
that shipped it, and a `Requirement` SHALL carry a derived `delivered` status
value when so linked.

#### Scenario: Mark a requirement delivered
- **WHEN** a user links a requirement to a release via `delivered_in`
- **THEN** the requirement is queryable as delivered in that release

#### Scenario: What shipped in a release
- **WHEN** a user traverses a release's incoming `delivered_in` relationships
- **THEN** the requirements and changes delivered in that release are returned

### Requirement: Verification is separate from delivery

The blueprint SHALL provide a `TestSuite` object type and a `verified_by`
relationship so a `Requirement` can be linked to the test suite that verifies
it, independently of its delivery state.

#### Scenario: Verify a requirement
- **WHEN** a requirement is linked to a test suite via `verified_by`
- **THEN** the requirement is queryable as verified separately from its delivery state

### Requirement: Blueprint bundles an authoring skill

The blueprint SHALL include a skill at `skills/product-specs/SKILL.md` with a
non-empty `name` and `description` and a body describing the author → implement
→ deliver → verify → archive loop.

#### Scenario: Skill passes blueprint validation
- **WHEN** validation runs
- **THEN** the skill has a name, a description, and a non-empty content body

### Requirement: Bundle is installable and documents graph-only operation

The blueprint SHALL be installable with `memory blueprints install`, and its
README SHALL describe the graph-only operating model and the native history
primitives that replace git history.

#### Scenario: Dry-run install succeeds
- **WHEN** a user runs `memory blueprints install <dir> --dry-run`
- **THEN** the run reports the pack and skill it would create without error

#### Scenario: History primitives documented
- **WHEN** a reader opens the README
- **THEN** it names object `created_at`, the object version fields, the object history endpoint, and the project journal as the history source
