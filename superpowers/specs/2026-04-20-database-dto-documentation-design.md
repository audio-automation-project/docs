# Database and DTO Documentation Design (Evidence-Driven)

Date: 2026-04-20  
Status: Approved design  
Audience: Team developers

## 1) Goal

Produce a reliable documentation set that explains:

- what database tables/entities and DTOs/models exist,
- how they are connected,
- why those connections are required for system behavior,
- and which connections are currently logical-only vs physically enforced.

The result must enable developers to implement features and migrations confidently without deep code archaeology.

## 2) Scope

This design covers the following documentation outcomes:

1. Update `docs/architecture/audio-platform-overview.md` (high-level architecture only).
2. Add `docs/architecture/database-and-relations.md`.
3. Add `docs/features/dto-catalog-and-mapping.md`.
4. Add `docs/reports/database-dto-gap-report.md`.

DTO/model coverage is exhaustive (`Option C`): include active, legacy, and unverified items with explicit status labels.

## 3) Constraints and Principles

- Evidence-driven first: prefer verifiable code/config facts over assumptions.
- Use two relationship layers:
  - logical/business links,
  - physically confirmed schema constraints.
- Mark every important statement with evidence confidence:
  - `Verified in code`
  - `Verified in config`
  - `Inferred from docs`
  - `Missing in current workspace`
- Use Mermaid diagrams in each document where relationships or flow are explained.
- Keep docs implementation-oriented for team use (not onboarding tutorial style).

## 4) Information Model for Documentation

### 4.1 Database relationship model

Relationships are documented with both meaning and enforcement state:

- **Logical relationship**: required by workflow/business lifecycle.
- **Physical relationship**: confirmed FK/unique/index/constraint in schema.
- **Missing constraint**: relationship expected logically but not confirmed physically.

Each key relation includes:

- why it exists,
- what can fail if missing,
- proposed follow-up action if not physically enforced.

### 4.2 DTO/model catalog model

Each DTO/model entry includes:

- purpose,
- producer(s),
- consumer(s),
- source/sink,
- constraints/validation (if any),
- stability label (`Stable`, `Evolving`, `Deprecated candidate`),
- evidence level.

Status classes:

- `Active`
- `Legacy`
- `Unknown/Unverified`

## 5) Document-by-Document Design

## 5.1 `docs/architecture/audio-platform-overview.md` (update)

Role:
- Keep cluster/pipeline-level view only.

Changes:
- Preserve high-level boundaries and data ownership narrative.
- Add explicit links to the new database and DTO docs.
- Keep details that belong to schema/DTO mapping out of this file.

Mermaid:
- One top-level flowchart for cluster pipeline and ownership boundaries.

## 5.2 `docs/architecture/database-and-relations.md` (new)

Role:
- Primary source of truth for table/entity connections and rationale.

Sections:
- Domain map (catalog, processing, distribution, projection).
- Relationship matrix (logical vs physical vs missing).
- Rationale and impact by key relation.
- Constraint/index backlog references.

Mermaid:
- `erDiagram` for entities and relationships.
- `flowchart` for cross-domain data flow.
- confidence legend for relation labels.

## 5.3 `docs/features/dto-catalog-and-mapping.md` (new)

Role:
- Exhaustive DTO/model reference across modules.

Sections:
- Inventory by status (`Active`, `Legacy`, `Unknown/Unverified`).
- Producer/consumer mapping.
- Boundary guidance (where a DTO should and should not be reused).
- Stability/deprecation guidance.

Mermaid:
- request/response and internal mapping flow (`flowchart`).
- inter-module transfer map (`graph TD`).

## 5.4 `docs/reports/database-dto-gap-report.md` (new)

Role:
- Actionable report for remediation and risk tracking.

Sections:
- Verified findings.
- Gap matrix (`impact`, `risk type`, `owner domain`, `priority`, `suggested fix`).
- Execution order (`quick wins`, `medium`, `structural`).
- Definition of done for this pass.

Mermaid:
- priority flow and dependency graph for fixes.

## 6) Success Criteria

This pass is complete when:

1. The team can understand data and DTO relationships without reading large source areas.
2. Logical vs physical links are clearly distinguished.
3. Missing constraints/uncertainties are explicitly listed and prioritized.
4. Architecture, feature, and report docs are cross-linked and internally consistent.

## 7) Risks and Mitigations

- **Risk:** Source gaps in current workspace can force assumptions.  
  **Mitigation:** mark all uncertain points as `Missing in current workspace`.

- **Risk:** Existing architecture doc becomes overloaded again.  
  **Mitigation:** keep details in dedicated docs and enforce cross-linking.

- **Risk:** DTO inventory drifts after code changes.  
  **Mitigation:** include update checklist and ownership in report doc.

## 8) Out of Scope

- Refactoring runtime code or schema in this documentation pass.
- Enforcing new DB constraints in code/migrations.
- API behavior changes.

## 9) Transition to Planning

After the docs are created and reviewed, the next step is a focused implementation plan for:

- concrete edits to the 4 documentation files,
- evidence extraction method and annotation format,
- review/validation checklist for consistency and maintenance.

## Implementation Plan

- `superpowers/plans/2026-04-20-database-dto-documentation-implementation-plan.md`
