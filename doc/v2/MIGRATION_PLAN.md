# v2 — Migration Plan (Planning & Analysis)

This document is the **data-preservation strategy and delivery approach** for
the v2.0 rebuild. It is a plan; nothing here has been executed.

## 1. Non-negotiables

1. **Never drop, truncate, or rebuild `main_system` tables.** No
   `migrate:fresh` / `migrate:refresh`. Ever.
2. **v1 stays live during the whole build.** v2 is developed in parallel and
   switched at a single cutover.
3. **A backup and restore drill must succeed before cutover.**

## 2. Target stack (proposal)

| Layer | Choice | Why |
|---|---|---|
| Language | PHP 8.2+ | Current v1 is PHP; lowest skill/tooling jump |
| Framework | Laravel (recommended) | Auth middleware, migrations, ORM, testing out of the box; CodeIgniter 4 / Slim are lighter alternatives |
| Database | MariaDB/MySQL `main_system` (unchanged) | Data is untouchable |
| Front-end | Bootstrap 5 + jQuery + DataTables (as today) | Preserve staff familiarity, minimize retraining |
| Scanner | `html5-qrcode` (as today) | Already proven in the field |

## 3. Data-preservation approach

### 3.1 Baseline migration, not a redesign
- Generate a **baseline migration** from the current `main_system` schema
  (31 tables) so the schema is versioned going forward.
- The baseline must match the production schema exactly; only safe,
  non-destructive additions are allowed later (new indexes, new tables).

### 3.2 Mapping the legacy schema into the framework
| Concern | Approach |
|---|---|
| Auth by `username`, not `email` | Custom login + user provider keyed on `tbl_users.username` |
| No `remember_token` / `updated_at` on all tables | Define models with explicit timestamps or disable; never require columns that don't exist |
| Session token single-device logic | Port `session.php` contract into auth middleware (FR-1.2) |
| Tables without standard `id` PK shape | Map per-table PK/unique keys explicitly in models |
| Enum columns | Keep values identical in v2; any change is a separate, approved migration |

### 3.3 Read/write parity
- After porting, run v2 against a **copy** of the production data.
- Reconciliation checks: same row counts and sums (e.g. per-program
  transaction counts, payout totals) between v1 and v2 on the same dataset.
- v2 is considered ready only when reads and writes match v1 behavior on the
  copy.

## 4. Delivery phases

| Phase | Work | Exit criteria |
|---|---|---|
| **P0 Analysis** | This folder (vision, requirements, gaps) | Approved by stakeholder |
| P1 Baseline | Framework scaffold, baseline migration, CI, env config, backups | v2 boots against a copy of `main_system` |
| P2 Core | Auth, ACL, client registry, households | Login + CRUD parity on copy |
| P3 Transactions | Transactions, filtering, exports | Parity with v1 `all_transactions.php` |
| P4 Scanner engine | One scanner engine + all 17 programs as config | Each program passes v1-equivalent scan tests |
| P5 Payouts & unpaid | Payout attendance, seats, unpaid verification | Parity with v1 screens |
| P6 Scholars & reports | Scholar flows, all reports/exports | Parity with v1 |
| P7 Hardening | CSRF, rate limiting, audit observers, tests | Security gaps C1–C5 closed |
| P8 Cutover | Final backup, deploy, validate, switch | Reconciliation passes; v1 archived |

## 5. Cutover plan (summary)

1. Freeze v1 writes (a defined maintenance window).
2. Full backup of `main_system` (dump + restore drill already proven).
3. Deploy v2 pointing at the same database.
4. Run reconciliation checks on live data.
5. If reconciliation fails → roll back to v1 (same database, no data loss).
6. On success → disable v1, archive the code, rotate DB credentials.

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Migration baseline drifts from production schema | Generate baseline from the actual prod dump; diff before deploy |
| Framework conventions assume columns v1 lacks | Explicit model definitions; parity tests on copy (3.3) |
| Scanner behavior differences per program | Config-driven rules; per-program acceptance tests |
| Data quality issues in v1 (duplicates, drift) | Documented as separate data-quality effort; not a cutover blocker |
| Staff resistance to new UI | Keep Bootstrap + same flows; pilot users before full rollout |

## 7. Open decisions (for the stakeholder)

1. Framework: Laravel vs CodeIgniter 4 vs Slim (recommend Laravel).
2. Hosting: stay on shared hosting or move to a VPS/cloud (affects PHP 8.2 availability).
3. Whether soft deletes/merge are wanted in v2 (affects P2 scope).
4. Whether new migrations may add indexes (recommended yes; safe and non-destructive).
