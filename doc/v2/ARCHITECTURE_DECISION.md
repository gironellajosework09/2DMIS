# v2 — Architecture Decision Record (ADR) Collection

> Outcome of the **Architectural Review & Decision** phase for the v2.0
> rebuild. Each ADR records a context (why), a decision (what), and the
> consequences (impact), so choices are explicit and reviewable.
>
> Planning & Analysis only — no code, SQL, or database changes are implied
> or have been made.

**Reference inputs:** `../v1/SYSTEM_DESIGN.md` (design & anti-patterns),
`../v1/RECOMMENDATIONS.md` (priorities), `VISION_AND_SCOPE.md`,
`GAP_ANALYSIS.md`, `MIGRATION_PLAN.md`.

## Decision summary

| ADR | Topic | Decision |
|---|---|---|
| ADR-001 | Framework | **Laravel** (PHP 8.2+), unless hosting blocks it → fallback CodeIgniter 4 |
| ADR-002 | Authentication & sessions | Port v1 contract: username login, bcrypt, DB `session_token` single-device middleware |
| ADR-003 | Access control | One ACL service (permissions + roles); no username-based checks |
| ADR-004 | Scanner engine | One controller + one reusable view, driven by program configuration |
| ADR-005 | Data layer & migrations | Baseline migration matching `main_system`; no redesign, no destructive migrations |
| ADR-006 | Front-end & request contract | Keep Bootstrap 5 + jQuery + server-side DataTables + `html5-qrcode` |
| ADR-007 | Security hardening | Global CSRF, login throttling, env-based secrets, no error disclosure |
| ADR-008 | Audit & logging | Framework events/observers firing the same audit contract as v1 |
| ADR-009 | Reporting & exports | Port v1 report queries; reuse CSV-with-BOM export contract |
| ADR-010 | Environment & deploy | `.env` config, migration-based deploys, version control, scheduled backups |

---

## ADR-001 — Framework selection

**Status:** Proposed (recommended: Laravel).

**Context**
- v1 is plain PHP with no framework; the team already writes PHP.
- The rebuild must preserve the existing database and staff workflows
  (VISION_AND_SCOPE §O1, §O5) and close the v1 gaps in `GAP_ANALYSIS.md`.
- The hosting target is currently shared hosting; PHP version availability is
  an open question (`MIGRATION_PLAN.md` §7).

**Alternatives considered**
| Option | Strengths | Weaknesses | Fit |
|---|---|---|---|
| Laravel | Full-stack: auth middleware, migrations, ORM (Eloquent), queues, testing, large ecosystem | PHP 8.2+ required; heavier than the app strictly needs | **High** |
| CodeIgniter 4 | Light, PHP 7.4+/8.x friendly, simple deployment, small learning curve | Smaller ecosystem, weaker ORM/testing story than Laravel | Medium (fallback) |
| Slim (micro) | Minimal, fast | Too little structure — we would rebuild the framework we just critiqued | Low |

**Decision**
Adopt **Laravel** as the v2 framework. If the hosting target cannot provide
PHP 8.2+ without a deliberate migration, revisit and select **CodeIgniter 4**
instead (it can run on older PHP and keeps the same guardrails).

**Consequences**
- (+) Middleware, routes, migrations, and an event system directly absorb the
  v1 include-chain patterns (ADR-002/003/008).
- (+) Eloquent models can map to existing tables without schema changes
  (ADR-005).
- (−) Requires PHP 8.2+; hosting may need to change (`MIGRATION_PLAN.md` §7).
- (−) Framework conventions (timestamps, auth scaffolds) need explicit
  overrides to fit the legacy schema.

---

## ADR-002 — Authentication & sessions

**Status:** Proposed.

**Context**
- v1 authenticates by **username** (not email) with bcrypt, then enforces
  single-device via a DB `session_token` compared on every request
  (`../v1/SYSTEM_DESIGN.md` §5.1, FR-1.2/1.3).
- Framework defaults assume email + optional `remember_token` columns that
  the legacy tables may not have (`MIGRATION_PLAN.md` §3.2).

**Decision**
- Custom login keyed on `tbl_users.username` with `password_verify` (bcrypt).
- Port the single-device contract into **auth middleware**: on every
  authenticated request, compare the session token against the DB token;
  mismatch → force logout (mirrors v1 `restriction.php` behavior).
- Keep exempt accounts centrally manageable via a permission/config flag
  (v1's multi-device exemption becomes a permission, not a hard-coded name).
- Do **not** require `remember_token` or `updated_at`; disable or map these
  explicitly per model.

**Consequences**
- (+) Same security behavior staff already expect; remote force-logout works.
- (−) Custom middleware must be maintained against framework auth upgrades.
- (−) Every request pays one extra DB read for the token check (acceptable at
  current scale).

---

## ADR-003 — Access control (single ACL service)

**Status:** Proposed.

**Context**
- v1 has **two** conflicting models: `tbl_permissions`/`tbl_program_permissions`
  enforced by `restriction.php`, plus literal-username checks in
  `sidebar.php`, `audit_logs.php`, `manage_php.php`, `clients.php`
  (`../v1/SYSTEM_DESIGN.md` §5.2, anti-pattern A2; GAP_ANALYSIS §2).

**Decision**
- Implement **one ACL service**: users → permissions (pages) + program
  permissions, with admin/super-admin expressed as a role or permission —
  never as a hard-coded username or an implicit `user_id = 1` (v1 A3).
- Enforce via framework middleware (route/program level), mirroring v1's
  per-page checks but centralized.
- All existing permission data in `tbl_permissions` /
  `tbl_program_permissions` is seeded into the same tables on cutover (no data
  loss).

**Consequences**
- (+) Authorization becomes predictable and testable.
- (+) Changing a username no longer changes who can do what (v1 A2 fixed).
- (−) `manage_permissions` screen needs rebuilding on the new model.

---

## ADR-004 — Scanner engine (one engine, programs as config)

**Status:** Proposed.

**Context**
- v1 replicates the scanner ~16 times with per-program duplicate rules
  (`../v1/SYSTEM_DESIGN.md` §5.6, A1; FR-6.1; GAP_ANALYSIS §2). Every fix or
  new program means editing copies.

**Decision**
- Build **one scanner controller + one reusable view** (camera via
  `html5-qrcode`, lookup → confirm → save, sound/modal feedback).
- A **program configuration** drives behavior: duplicate rule (fixed remark
  key / monthly guard / exam-derived / update-in-place / validate-existing),
  and which tables to write (transactions, payout scans, unpaid
  verifications).
- The v1 `lookup`/`save` two-phase contract maps directly to two routes.

**Consequences**
- (+) One code path; adding a program = adding config, not code.
- (+) Bug fixes propagate everywhere automatically.
- (−) The config must faithfully encode all 17 v1 variants — requires a
  per-program acceptance matrix before cutover.

---

## ADR-005 — Data layer & migrations

**Status:** Proposed (non-negotiable guardrails).

**Context**
- The database `main_system` (31 tables) is the source of truth and must not
  lose or renumber records (`MIGRATION_PLAN.md` §1; NFR-1).
- v1 has no migrations → dev/prod drift (collation mismatch seen locally) (A8).

**Decision**
- Generate a **baseline migration** from the production schema; it must match
  the dump exactly (`utf8mb4_unicode_ci` standard).
- Migrations after baseline are **additive only** (new indexes, new tables)
  unless explicitly approved; `migrate:fresh`/`refresh` are forbidden.
- Eloquent models map existing tables explicitly; disable timestamps or map
  columns that exist; never require columns the DB lacks.
- Enum values (`program`, `status`, `sex`, etc.) keep v1 values byte-for-byte.

**Consequences**
- (+) Schema becomes versioned; dev/prod drift stops.
- (+) Data and IDs preserved; rollback is a re-point of the app, not a restore.
- (−) Eloquent/Laravel defaults need configuration for legacy shapes (custom
  primary keys, no `updated_at`).

---

## ADR-006 — Front-end & request contract

**Status:** **Superseded** by `MODERNIZATION_PROPOSAL.md` (AD-3, AD-10). The
proposal's approved direction replaces "keep Bootstrap 5 + jQuery +
DataTables" with **Blade + Tailwind CSS** and the **right-side sliding details
panel** (row-click → panel, table stays visible). The server-side table
contract and CSV-with-BOM export behavior are retained conceptually.

**Original decision (retained as history):**

**Context**
- v1 uses Bootstrap 5 + jQuery + server-side DataTables + `html5-qrcode`
  (`../v1/SYSTEM_DESIGN.md` §2). Staff are trained on these screens; the
  goal is minimal retraining (O5, NFR-5).

**Decision**
- Keep the same front-end stack and the **server-side DataTables JSON
  contract** (draw/recordsTotal/recordsFiltered/data) for list pages.
- Reuse the CSV-with-UTF-8-BOM export convention for Excel compatibility.
- Keep screens named and laid out consistently with v1 where practical.

**Consequences**
- (+) Near-zero staff retraining; existing table/report expectations hold.
- (−) DataTables/jQuery are older tooling; fine to keep, but flagged as a
  future refresh candidate (not a v2 blocker).

---

## ADR-007 — Security hardening

**Status:** Proposed (must close v1 C1–C5).

**Context**
- v1 gaps: no CSRF, no login throttling, hard-coded DB credentials, error
  disclosure (`../v1/SYSTEM_DESIGN.md` §8; GAP_ANALYSIS §1).

**Decision**
- **CSRF:** global middleware on all state-changing routes.
- **Login throttling:** attempt + lockout on the login route.
- **Secrets:** DB credentials and keys via `.env` / environment; never in code.
- **Errors:** log server-side; show generic messages to users; never echo DB
  errors or enable `display_errors` in production.
- **Sessions:** keep httponly, `SameSite=Lax`, `secure` on HTTPS, and the
  single-device token check (ADR-002).
- Rotate the exposed DB credentials as part of cutover.

**Consequences**
- (+) C1–C5 closed.
- (−) Small behavioral change: timed logins and generic error messages; staff
  need brief awareness.

---

## ADR-008 — Audit & logging

**Status:** Proposed.

**Context**
- v1 audits every mutation to `tbl_audit_logs` (+ `tbl_update_logs`,
  `tbl_photo_logs`, `password_resets`) via `log_action()`
  (`../v1/SYSTEM_DESIGN.md` §3.4, FR-9.1/9.2).

**Decision**
- Keep the same audit tables and content contract (who / action / target /
  before-after JSON / timestamp).
- Trigger audits via **framework events/observers** fired on the same write
  paths v1 logged (clients, scans, payouts, updates, password changes), so no
  mutation can be added without its audit.

**Consequences**
- (+) Audit becomes structural, not a manual call site.
- (+) Historical audit data remains queryable in v2 unchanged.

---

## ADR-009 — Reporting & exports

**Status:** Proposed.

**Context**
- v1 reports are server-rendered queries with CSV export (`../v1/SYSTEM_DESIGN.md`
  §5.10, FR-8.1/8.2).

**Decision**
- Port each v1 report query into the v2 reporting layer (repository/service),
- Keep the CSV-with-BOM contract; add framework-native filtering/sorting where
  it already matches v1 behavior.

**Consequences**
- (+) Reports behave identically; easier to test.
- (−) Report SQL must be re-verified against the legacy schema (no schema
  redesign, ADR-005).

---

## ADR-010 — Environment & deployment

**Status:** Proposed.

**Context**
- v1 has no version control, no deploy pipeline, no backup strategy
  (GAP_ANALYSIS §4).

**Decision**
- Version control (Git) from day one; branch per phase.
- All configuration via `.env`; no credentials in the repo.
- Deploys run migrations (additive only); rollback = point v2 back to v1.
- Scheduled database backups + a proven restore drill before any cutover.

**Consequences**
- (+) Repeatable, reversible deployments.
- (−) New operational discipline for the team (brief process change).

---

## Trade-off summary

| Area | v1 way | v2 decision | Main gain |
|---|---|---|---|
| Structure | Page-per-file, includes | Framework (routes/middleware/controllers) | Maintainability |
| Scanners | 16 copies | 1 engine + config | Change cost drops to ~zero per program |
| Auth | Username + token in PHP | Same contract as middleware | Same security, testable |
| ACL | Two models | One service | Predictability |
| Data | No migrations | Baseline + additive | Dev/prod parity |
| Security | Gaps C1–C5 | Global hardening | Risk reduction |

## Open decisions (for stakeholder)

1. Confirm **Laravel** (ADR-001) or fall back to CodeIgniter 4 — depends on
   whether hosting can run PHP 8.2+.
2. Hosting: ✅ **RESOLVED** — Hostinger **Premium** plan (owner-confirmed); includes SSH (Premium+ tier) and PHP 8.3+; stay on shared hosting (affects ADR-001 and ADR-010).
3. Soft deletes / client merge in v2 (affects ADR-005 scope).
4. May v2 add new indexes to existing tables? (recommended: yes; additive and
   safe — GAP_ANALYSIS §4).

## How to use this document

- Each ADR is **proposed** until the stakeholder approves it; record approval
  by flipping Status to **Accepted**.
- When a decision changes later, add a new ADR that supersedes the old one
  rather than editing history.
- These decisions feed the Design phase; the Design phase must not contradict
  an accepted ADR without a superseding record.
