# 2D MIS — v2 Planning & Analysis

> Version 2.0 upgrade of the municipal assistance MIS.
> This folder holds the **Planning & Analysis phase** only — no code has been
> written and no changes have been made to the v1 system, its database, or its
> data.

## Documentation index

| Document | Contents |
|---|---|
| [VISION_AND_SCOPE.md](VISION_AND_SCOPE.md) | Why v2 exists, objectives, what is in / out of scope |
| [REQUIREMENTS_ANALYSIS.md](REQUIREMENTS_ANALYSIS.md) | Functional & non-functional requirements derived from v1 analysis |
| [GAP_ANALYSIS.md](GAP_ANALYSIS.md) | What v1 lacks vs what v2 must provide (mapped to v1 anti-patterns & recommendations) |
| [MIGRATION_PLAN.md](MIGRATION_PLAN.md) | Data-preservation strategy, target stack, phased delivery, and rollout |
| [ARCHITECTURE_DECISION.md](ARCHITECTURE_DECISION.md) | ADR collection: framework, auth, ACL, scanner engine, data layer, security, logging, deploy |
| [MODERNIZATION_PROPOSAL.md](MODERNIZATION_PROPOSAL.md) | Architecture review & modernization proposal (Laravel + Blade + Tailwind; evaluation, decisions, security, UI/UX panel, migration strategy/roadmap, risks) |
| [MIGRATION_PLANNING.md](MIGRATION_PLANNING.md) | Migration Planning phase: baseline workflow, backup/restore drill, reconciliation framework, P0–P8 gates, cutover runbook, rollback |

## Phase status

| Phase | Status |
|---|---|
| Planning & Analysis (this folder) | **In progress** |
| Design | Not started |
| Development | Not started |
| Testing | Not started |
| Rollout | Not started |

## Guardrails

- **Data is untouchable.** All records in `main_system` must survive the
  upgrade. Any plan that drops, truncates, or rebuilds tables is rejected.
- **No changes to v1.** v1 keeps running and keeps accepting data until v2 is
  ready to take over.
- **Documentation only.** Nothing in this folder implies changes to source
  code, files, SQL, or the database.
