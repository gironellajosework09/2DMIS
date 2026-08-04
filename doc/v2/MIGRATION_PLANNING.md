# v2 — Migration Planning (Detailed)

> **Phase:** Migration Planning (SDLC step 5).
> Operational elaboration of the strategy in `MIGRATION_PLAN.md`. This document
> turns the strategy into executable detail: environments, data artifacts,
> baseline migration workflow, backup/restore drill, reconciliation framework,
> per-module gates, cutover runbook, and rollback.
>
> Planning & Analysis only — nothing here has been executed; no schema or data
> changes are made by this document.

**Related:** `MIGRATION_PLAN.md` (strategy), `ARCHITECTURE_DECISION.md`
(decisions AD-4/AD-5), `MODERNIZATION_PROPOSAL.md` (roadmap P0–P8),
`REQUIREMENTS_ANALYSIS.md` (parity targets).

---

## 1. Objectives of the migration

| Objective | Measure |
|---|---|
| Zero data loss | Row counts, IDs, and sums identical before vs after cutover |
| Schema unchanged | `SHOW CREATE TABLE` diffs empty; no migration alters existing tables |
| Minimal downtime | Single maintenance window; v1 runs until the freeze |
| Reversible | Rollback = re-point to v1; no data restore required in the happy path |
| Backward compatible | Login, audit format, CSV export, permission keys, program names unchanged |

## 2. Environments & data artifacts

| Environment | Purpose | Data |
|---|---|---|
| Local dev (XAMPP) | Build & test modules | Copy of prod dump, `utf8mb4_unicode_ci` for MariaDB 10.4 |
| Staging (Hostinger) | Parity validation + rehearsal cutover | Copy of prod dump |
| Production (Hostinger) | Live system | Live `main_system` (untouched until cutover) |

**Artifacts:**
- `u749085076_main_system.sql` — production dump (source of truth for baseline).
- Staging copy — refreshed before each parity gate.
- `.env` per environment — no secrets in the repository.

## 3. Baseline migration workflow (AD-4)

1. Export a fresh production dump on a schedule **before** building the
   baseline (freeze the reference).
2. Generate a **baseline migration** from that dump's exact DDL.
3. **Verify**: apply the baseline to an empty database and diff
   `SHOW CREATE TABLE` for all 31 tables against the dump — must be identical.
4. Confirm collation handling: if prod supports `utf8mb4_uca1400_ai_ci`, the
   baseline keeps it; local MariaDB 10.4 uses `utf8mb4_unicode_ci` — this is an
   environment difference, not a schema change.
5. Commit the baseline. All future migrations are **additive only** and require
   stakeholder approval (ADR-005).

## 4. Backup & restore drill (gate before P0 completes)

Run this **before any module work** and **again before cutover**:

1. Full `mysqldump` of `main_system`.
2. Restore into an empty database (e.g. `main_system_drill`).
3. Verify: row counts per table, checksums of key tables, a sample of recent
   records present.
4. Document timings (dump size, restore time) to size the cutover window.
5. Exit criterion: restore succeeds and reconciles; drill is signed off.

## 5. Reconciliation framework

Applied after every parity gate and at cutover. Compare **v1 output vs v2
output on the same dataset (a copy)**.

| Check | What it validates |
|---|---|
| Table row counts | No records lost or duplicated |
| SUM of `amount` per program/status | Monetary totals match |
| MAX(ID) per table | Latest inserts visible; no renumbering |
| Sample record fields | Values identical (no silent transforms) |
| Audit rows count + recent entries | Audit contract preserved (AD-9) |
| Permission keys | ACL data intact (AD-7) |

Automate as a script: run once per module gate and at cutover.

## 6. Module migration gates (roadmap P0–P8)

Each module gate requires: entry criteria met, development on staging copy,
parity check passes, then merge to the v2 codebase. The module is **not**
shipped to production until cutover.

| Gate | Module | Parity checks (beyond §5) |
|---|---|---|
| P0 | Foundations | Baseline diff empty; boot on copy; restore drill passes |
| P1 | Auth + RBAC | Login works by username; single-device force-logout; every v1 page-permission row enforced |
| P2 | Clients + households | Client CRUD, household assignment, duplicate detection match v1 |
| P3 | Transactions + reports/exports | Filters/sorts match; CSV export byte-comparable to v1 |
| P4 | Scanner engine + 17 programs | Per-program matrix: every duplicate-rule variant + scan path reproduces v1 |
| P5 | Payout attendance + unpaid | Unique-key behavior, seats, proxy capture match v1 |
| P6 | Scholars/GIP/exam | Reports totals match v1 exports |
| P7 | Admin + audit viewer | Permission screens; audit viewer reads v1 history unchanged |
| P8 | Hardening + regression + cutover | Full test suite green; security findings closed; rehearsal cutover passes |

**Definition of done per gate:** automated tests green + parity script green +
manual spot-check by a user familiar with the v1 module.

## 7. Cutover runbook (single maintenance window)

Sequence, with owner and timing logged per step:

1. **Announce & freeze** — notify staff; begin window; block v1 writes
   (e.g. maintenance notice on v1 login).
2. **Final backup** — full dump; verify immediately (restore drill already
   proven; §4).
3. **Deploy v2** — deploy code to production pointing at `main_system`;
   run migrations (baseline already applied; only additive migrations run).
4. **Sanity check** — app boots; config validated; `APP_DEBUG=false`;
   `.env` protected.
5. **Reconcile** — run §5 checks against pre-cutover v1 numbers (captured at
   step 2).
6. **Switch** — if green: update DNS/document root, retire v1 (keep a
   read-only archive), rotate DB credentials.
7. **Post-window** — verify live writes, audit logging, first login;
   confirm with staff.

Window size target: the freeze window should fit the sum of steps 2–5; backup
timing from the drill (municipal data volume is small, so this is realistically
a short window).

## 8. Rollback procedure

| Trigger | Action | Data impact |
|---|---|---|
| Reconciliation fails at step 5 | Point document root back to v1; v2 is disabled | None (v1 never stopped serving) |
| v2 fails sanity check | Same as above | None |
| Post-switch failure within 24 h | Re-point to v1; v2 removed from production root | None in happy path; worst case restore from step-2 backup (drill-proven) |

Rollback is exercised **once on staging** (rehearsal cutover) before the real
cutover.

## 9. Communication & change windows

- A **change window** is reserved (evening/weekend, staff-aware) for cutover.
- Staff get a short briefing: timed logins, generic error messages, new
  slide-over panel (no behavior change to core tasks).
- Every gate (P0–P8) has a sign-off; the migration log records who/where/when.

## 10. Acceptance criteria (definition of done for the migration)

1. All P0–P8 exit criteria met on staging.
2. Rehearsal cutover (staging) passes reconciliation and rollback.
3. Real cutover window completes with reconciliation green.
4. Live writes verified; audit trail appending; staff confirm workflows.
5. v1 archived (read-only); DB credentials rotated; backup schedule live.

---

*End of Migration Planning (Detailed).*
