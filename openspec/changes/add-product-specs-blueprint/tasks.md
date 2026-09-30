## 1. Schema pack

- [ ] 1.1 Create `schemas/product-specs.yaml`: pack `product-specs`, `version: "0.1.0"`, description/author/license/repositoryUrl. Verify: file parses as YAML and `name`/`version`/non-empty `objectTypes` are present.
- [ ] 1.2 Declare the nine object types — `Capability`, `Requirement`, `Scenario`, `Change`, `Artifact`, `TaskGroup`, `Task`, `Release`, `TestSuite` — each with a `label`, `description`, and typed `properties` exactly as designed. Verify: every type has a name, label, and description; `Requirement.status` and `Change.status` enumerations documented.
- [ ] 1.3 Declare the ten relationship types — `has_requirement`, `has_scenario`, `depends_on`, `proposes_change`, `has_artifact`, `has_task_group`, `contains_task`, `addresses`, `delivered_in` (sourceTypes `[Requirement, Change]`), `verified_by` — with `sourceType`/`targetType` (or plural arrays) naming object types declared in this same pack. Verify: no relationship endpoint names a type absent from the pack.

## 2. Bundled skill

- [ ] 2.1 Create `skills/product-specs/SKILL.md` with YAML frontmatter (`name: product-specs`, a trigger-rich `description`, `metadata.author`/`version`) and a non-empty body. Verify: frontmatter parses and body is non-empty.
- [ ] 2.2 Body documents: the object/relationship model; key prefixes (`cap:`/`req:`/`scen:`/`chg:`/`art:`/`tg:`/`task:`/`rel:`/`ts:`); the author → implement → deliver → verify → archive loop; the `proposed → active` staging rule; delivery via `delivered_in` → `Release`; verification via `verified_by` → `TestSuite`; the graph-only rule; and the native-history note. Verify: each listed element appears.

## 3. Repo metadata and documentation

- [ ] 3.1 Create `project.yaml` with a `projectInfo` blurb (purpose, counts). Verify: file parses.
- [ ] 3.2 Create `README.md` documenting the object/relationship reference, the OpenSpec mapping, the loop and capability construction, the graph-only operating model, the native history primitives, and the caveats (type-name collisions; deferred PR ingestion; archive-guard impact). Verify: README names `created_at`, the version fields, the object history endpoint, and the journal.
- [ ] 3.3 Create `LICENSE` (MIT), `.gitignore`, and keep the `openspec/` change in this same branch. Verify: files present.

## 4. Verification

- [ ] 4.1 Run `memory blueprints validate .` in the repo; verify exit 0 with zero errors.
- [ ] 4.2 Run `memory blueprints install . --dry-run`; verify it reports the `product-specs` pack and the `product-specs` skill with no error.
- [ ] 4.3 Run `openspec validate add-product-specs-blueprint --strict`; verify it passes.
- [ ] 4.4 Confirm all ten relationships resolve in-pack and the pack declares exactly nine object types (cross-check counts). Verify: no endpoint references an undeclared type.
- [ ] 4.5 Open a pull request containing the change directory and the implementation together; verify CI (`validate`) is green.
