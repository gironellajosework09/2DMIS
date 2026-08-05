# v2 — Engineering Migration Blueprint

> **Phase:** Migration Planning (step 5) — final engineering blueprint.
> Extends `MIGRATION_PLANNING.md` (strategy + runbook) with the component-level
> blueprint developers follow during implementation.
>
> **Scope:** planning only. No Laravel code, no PHP code, no migrations, and
> **no changes to the production `main_system` database** are contained in or
> authorized by this document. The schema is treated as frozen.

**Inputs:** `../v1/FILE_REFERENCE.md`, `../v1/DATABASE_SCHEMA.md`,
`../v1/SYSTEM_DESIGN.md`, `MIGRATION_PLAN.md`, `MIGRATION_PLANNING.md`,
`MODERNIZATION_PROPOSAL.md` (AD-1…12), `ARCHITECTURE_DECISION.md`
(ADR-001…010).

---

## 1. Legacy System Inventory

Inventory of the v1 system (procedural PHP, ~115 files, no framework),
grouped by module. Complexity: **L** low / **M** medium / **H** high.

### 1.1 Shared foundation
| Aspect | Detail |
|---|---|
| Purpose | Bootstrap every page: DB, session, ACL, audit, shell |
| Major files | `db_connect.php`, `session.php`, `restriction.php`, `logs.php`, `navbar.php`, `sidebar.php`, `sidebar.css/js`, `favicon.php`, `check_session.php` |
| Dependencies | none (leaf) |
| Complexity | **M** — session/token logic is subtle |
| Migration priority | **P1** (with Auth) |

### 1.2 Authentication & users
| Aspect | Detail |
|---|---|
| Purpose | Login/logout, user CRUD, force-logout, online users |
| Major files | `login.php`, `logout.php`, `register.php`, `add_user.php`, `manage_php.php`, `force_logout.php`, `currently_logged_users.php`, `fetch_online_users.php`, `verify_mobile.php` |
| Dependencies | Shared foundation |
| Complexity | **H** — bcrypt + single-device `session_token` contract |
| Migration priority | **P1** |

### 1.3 Dashboard & shell
| Aspect | Detail |
|---|---|
| Purpose | Landing page linking to all scanners/modules |
| Major files | `index.php` (`default.php` is dead — do not migrate) |
| Dependencies | Auth + permissions |
| Complexity | **L** |
| Migration priority | **P1/P2** |

### 1.4 Clients module (core registry)
| Aspect | Detail |
|---|---|
| Purpose | Master person registry: CRUD, photos, duplicates, student flows |
| Major files | `clients.php`, `add_client.php`, `edit_client.php`, `view_client.php` (≈170 KB), `delete_client.php`, `fetch_clients.php`, `search_clients.php`, `search_clients_hh.php`, `get_client_hh.php`, `preview_duplicates.php`, `fetch_duplicates.php`, `delete_duplicates.php`, `save_client_photo.php`, `student_photo_upload.php`, `student_update_photo.php`, `student_verify.php` (`client_photo.php` dead — do not migrate) |
| Dependencies | Foundation, municipalities/barangays, households |
| Complexity | **H** — denormalized `full_name`/`match_name`, duplicates, photos |
| Migration priority | **P2** |

### 1.5 Households module
| Aspect | Detail |
|---|---|
| Purpose | Household groups + family-member links |
| Major files | `household.php`, `add_household.php`, `view_household.php`, `delete_household.php`, `add_family_member.php`, `fetch_households.php`, `get_household.php`, `search_households.php` |
| Dependencies | Clients |
| Complexity | **M** |
| Migration priority | **P2** |

### 1.6 Transactions module
| Aspect | Detail |
|---|---|
| Purpose | Central assistance records; filters, CSV export; program-gated |
| Major files | `all_transactions.php` (≈1,400 lines), `add_transaction.php`, `edit_transaction.php`, `view_transaction.php`, `delete_transaction.php`, `update_transaction.php`, `all_transaction_edit.php`, `all_transaction_delete.php`, `fetch_transactions.php`, `transaction_table.php` |
| Dependencies | Clients, program permissions |
| Complexity | **H** |
| Migration priority | **P3** |

### 1.7 Scholars module
| Aspect | Detail |
|---|---|
| Purpose | Scholar enrollment, GIP, grantee updates, scholarship reports, QR viewer |
| Major files | `scholars.php`, `save_scholarship.php`, `fetch_scholars.php`, `update_client_id.php`, `scholarship_reports.php`, `fetch_scholarship_reports.php`, `export_scholarship_reports.php`, `save_gip.php`, `save_grantee_update.php`, `disabled_update_grantee.php`, `update_logs.php`, `fetch_update_logs.php`, `view_qrcode.php` |
| Dependencies | Clients, transactions, exam/results |
| Complexity | **M** |
| Migration priority | **P6** |

### 1.8 Scanner subsystem (16 pages + 16 handlers)
| Aspect | Detail |
|---|---|
| Purpose | Camera QR/name scanning → guarded transaction insert / attendance |
| Major files | `scanner_ceap(.php/_action.php)`, `scanner_ceap_new`, `scanner_cedssg`, `scanner_cedssg_new`, `scanner_cedssg_update`, `scanner_tupad`, `scanner_toda`, `scanner_otces`, `scanner_otea`, `scanner_new_scholars`, `scanner_ongoing_scholars`, `scanner_payout`, `scanner_payout_unpaid`, `scanner_generic` |
| Dependencies | Clients, transactions, seats, exam/results |
| Complexity | **H** — per-program duplicate rules; biggest copy-paste risk |
| Migration priority | **P4** (after core stable) |

### 1.9 Payout attendance
| Aspect | Detail |
|---|---|
| Purpose | Three identical one-scan-per-transaction attendance screens |
| Major files | `scanned_payouts.php`, `scanned_payouts2.php`, `scanned_payouts_unpaid.php`, `fetch_scanned_payouts.php`, `fetch_scanned_payouts2.php`, `fetch_scanned_payouts_unpaid.php` |
| Dependencies | Transactions, users |
| Complexity | **M** |
| Migration priority | **P5** |

### 1.10 Unpaid verification
| Aspect | Detail |
|---|---|
| Purpose | Record grantees who couldn't be paid; optional proxy receiver |
| Major files | `unpaid_verifications.php`, `disabled_unpaid.php`, `unpaid_save.php`, `fetch_unpaid_verifications.php`, `export_unpaid_verifications.php`, `search_grantee.php`, `search_unpaid_grantee.php` |
| Dependencies | Clients, transactions, municipalities |
| Complexity | **M** |
| Migration priority | **P5** |

### 1.11 Administration subsystem
| Aspect | Detail |
|---|---|
| Purpose | Permissions, program permissions, exemptions, audit viewer/leaderboard |
| Major files | `manage_permissions.php`, `manage_program_permissions.php`, `manage_multi_device_exemptions.php`, `audit_logs.php`, `fetch_logs.php`, `fetch_leaderboard.php` |
| Dependencies | Users |
| Complexity | **M** |
| Migration priority | **P7** |

### 1.12 Utilities & assets
| Aspect | Detail |
|---|---|
| Purpose | Geography cascade, assets, storage |
| Major files | `get_barangays.php`; assets `seal_logo.png`, `sounds/*.mp3`, `uploads/`, `cache/`; dumps `u749085076_main_system.sql` (schema only), `main_system.sql` (dev, contains sample data) |
| Dependencies | none |
| Complexity | **L** |
| Migration priority | **P0** (assets) / throughout |

### 1.13 Cross-cutting observations
- **Business logic** lives inside pages (validation, normalization, audit calls).
- **Authorization** is split: `restriction.php` (DB) + username checks (sidebar, audit, clients) + implicit `user_id = 1`.
- **Reports** are server-rendered queries with CSV (UTF-8 BOM) exports.
- Dead files to exclude: `default.php`, `client_photo.php`.

---

## 2. Legacy-to-Modern Transformation Matrix

| Legacy component | Target component (Laravel) | Phase | Remarks |
|---|---|---|---|
| `db_connect.php` ($pdo) | `config/database.php` + `.env`; Eloquent connection `mysql` | P0 | Credentials via env; same DB |
| `session.php` | `config/session.php` + `Authenticate` middleware + `EnsureSingleDevice` middleware | P1 | Port token contract (ADR-002) |
| `restriction.php` | `Gate`/`Policies` + `AuthorizePage` middleware; `EnsureSingleDevice` | P1 | One ACL service (ADR-003) |
| `logs.php::log_action()` | `AuditService` + model events/observers | P1 | Same `tbl_audit_logs` contract (AD-9) |
| `navbar.php`, `sidebar.php` | Blade layout partials + components | P1 | Role-driven menu |
| `check_session.php` | JSON route returning session status | P1 | Polled by front-end |
| `login.php` / `logout.php` | `AuthController` + `LoginRequest` + routes | P1 | Username-based provider (AD-6) |
| `register.php`, `add_user.php`, `manage_php.php` | `UserController` + `FormRequest` + views | P7 | Admin-only |
| `force_logout.php` | `AdminController@forceLogout` (revoke token) | P1 | Same behavior |
| `currently_logged_users.php`, `fetch_online_users.php` | `SessionController` + DataTables route | P1 | `last_activity` |
| `index.php` | `DashboardController` + view | P1 | Menu + stats |
| `clients.php`, `fetch_clients.php` | `ClientController@index` + DataTables route | P2 | Server-side contract kept |
| `add_client.php` | `ClientController@store` + `ClientService` + `ClientRequest` | P2 | Central `full_name`/`match_name` derivation (AD-6 in service) |
| `edit_client.php` | `ClientController@edit/update` | P2 | |
| `view_client.php` | `ClientController@show` (data) + **slide-over panel** | P2 | Replaces navigation (AD-10) |
| `delete_client.php` | `ClientController@destroy` + Policy | P2 | Keep v1 guard rules |
| `search_clients.php`, `search_clients_hh.php`, `get_client_hh.php` | Search routes / `ClientService` | P2 | |
| `preview_duplicates.php`, `fetch_duplicates.php`, `delete_duplicates.php` | `DuplicateController` + `DuplicateService` | P2 | Batch compare preserved |
| `save_client_photo.php`, `student_*` | `PhotoController` + storage; `StudentController` | P2 | `tbl_client_photos` |
| `household.php` + helpers | `HouseholdController` + views | P2 | |
| `add_family_member.php` | `FamilyMemberController` | P2 | Unique `(client_id, relative_id)` |
| `all_transactions.php` + `fetch_transactions.php` | `TransactionController@index` + DataTables route + CSV export | P3 | Program-gated |
| `add/edit/view/delete/update_transaction.php`, `all_transaction_edit/delete.php` | `TransactionController` CRUD + `TransactionService` + Policies | P3 | |
| `transaction_table.php` | Blade partial component | P3 | |
| `scholars.php`, `save_scholarship.php`, `fetch_scholars.php`, `update_client_id.php` | `ScholarController` + `ScholarService` + DataTables route | P6 | |
| `scholarship_reports.php` + feeds + export | `ReportController` + `ReportService` | P6 | CSV BOM kept |
| `save_gip.php` | `GipController` | P6 | |
| `save_grantee_update.php`, `disabled_update_grantee.php`, `update_logs.php` + feed | `GranteeUpdateController` | P6 | `tbl_update_logs` |
| `view_qrcode.php` | `QrController` (QR API or package) | P6 | |
| `scanner_*.php` + `scanner_*_action.php` (16+16) | `ScannerController` + `ScannerService` + **program config** + one scanner Blade view; routes `lookup`/`save` | P4 | ADR-004 |
| `scanned_payouts*.php` + feeds | `PayoutAttendanceController` + views + DataTables routes | P5 | 3 screens, DB-unique kept |
| `unpaid_verifications.php`, `unpaid_save.php`, feeds, export, searches | `UnpaidVerificationController` + `UnpaidService` + DataTables/export routes | P5 | |
| `manage_permissions.php`, `manage_program_permissions.php`, `manage_multi_device_exemptions.php` | `AdminPermissionController` (one ACL service) | P7 | ADR-003 |
| `audit_logs.php`, `fetch_logs.php`, `fetch_leaderboard.php` | `AuditController` + DataTables routes | P7 | Permission-based, not username |
| `get_barangays.php` | `GeographyController@barangays` JSON | P2 | Cascade dropdown |
| `sidebar.css`, `sidebar.js` | Tailwind utilities + Vite asset | P1 | ADR-003/AD-3 |
| `sounds/*.mp3`, `seal_logo.png`, `uploads/` | `public/` assets + `storage` | P0 | |
| `default.php`, `client_photo.php` | **—** (dead code, not migrated) | — | A10 cleanup |

---

## 3. Database Mapping (no schema changes)

Every major table of `main_system` → proposed Eloquent model. All models set
`$table = 'tbl_*'` and `$timestamps = false` unless stated otherwise.

| Table | Proposed model | PK | Relationships | Special handling / legacy constraint |
|---|---|---|---|---|
| `tbl_users` | `User` | id | hasMany permissions, program_permissions, exemptions, audit_logs | **No `email` column** → configure auth provider keyed on `username`; disable default password-reset flow (see below); `password` bcrypt; `session_token` + `last_activity` |
| `tbl_permissions` | `Permission` | id | belongsTo User | `page_name` keys match v1 page names — keep identical |
| `tbl_program_permissions` | `ProgramPermission` | id | belongsTo User | `program_name` values = v1 program enum strings |
| `tbl_multi_device_exemptions` | `MultiDeviceExemption` | id | belongsTo User | `user_id` UNIQUE |
| `tbl_audit_logs` | `AuditLog` | id | belongsTo User | Keep `action`/`target_table`/`target_id`/`old_value`/`new_value` format exactly (AD-9) |
| `tbl_update_logs` | `UpdateLog` | id | — | append-only |
| `tbl_photo_logs` | `PhotoLog` | id | — | append-only |
| `password_resets` | `PasswordReset` | id | — | **Column conflict with Laravel defaults** — legacy columns are `changed_by/changed_for/changed_at`. Keep as a v1 audit table; do NOT use Laravel's `password_resets`/`password_reset_tokens` for framework reset (Laravel 9+ default table is `password_reset_tokens`, so disable framework reset to avoid confusion) |
| `tbl_clients` | `Client` | id | belongsTo Household; hasMany aff_orgs, photos, family_members, transactions, scholar_info, gip_info, unpaid_verifications | `city_municipality`/`barangay` store **IDs in varchar** (unconstrained FKs — keep values as-is); `full_name`/`match_name` denormalized → derived in `ClientService` on write; enum `sex/civil_status/pwd/ip/category`; no `updated_at` |
| `tbl_household` | `Household` | id | hasMany clients; `head_household` FK→clients | `household_id` (code) UNIQUE; `head_household` nullable |
| `tbl_family_members` | `FamilyMember` | id | belongsTo Client x2 | UNIQUE `(client_id, relative_id)` — keep constraint as the belt |
| `tbl_municipalities` | `Municipality` | id | hasMany barangays | read-mostly |
| `tbl_barangays` | `Barangay` | id | belongsTo Municipality | FK CASCADE |
| `tbl_transactions` | `Transaction` | id | belongsTo Client; hasOne payout_scan(s) | `program` enum → keep strings; `remarks` carries scanner keys; no `updated_at`; existing indexes on program/date — reuse, do not add in schema |
| `tbl_payout_scans` / `_2` / `_unpaid` | `PayoutScan` / `PayoutScan2` / `PayoutScanUnpaid` | id | belongsTo Transaction, User | `transaction_id` **UNIQUE** — the anti-duplicate belt (preserve, never relax) |
| `tbl_scholar_info` | `ScholarInfo` | id | belongsTo Client | has `updated_at`; `normalized_name`/`match_name` generated |
| `tbl_gip_info` | `GipInfo` | id | belongsTo Client | has `updated_at`; normalized fields |
| `tbl_unpaid_verifications` | `UnpaidVerification` | id | belongsTo Client, Municipality | `is_proxy` + proxy identity block |
| `tbl_exam` | `Exam` | id | hasMany results (exam_no) | `exam_no` link key |
| `tbl_results` | `ExamResult` | id | belongsTo Exam | `approved` drives auto program |
| `tbl_seats` / `tbl_seats2` | `Seat` / `Seat2` | id | — | payout scanner join on `name` vs `clients.full_name` — keep name-matching semantics |
| `tbl_absent`, `tbl_kababaihan`, `gender`, `tbl_details`, `temp_details` | read-only reference/import (Query Builder or plain models) | id | — | import/compare data; no FKs; do not model as domain entities |

**Framework defaults to disable/override (no schema impact):**
- Auth user provider: `username` key, not `email`.
- Timestamps: off per legacy model; `ScholarInfo`/`GipInfo` keep theirs.
- Enum values: keep as string casts; never introduce PHP enums that rename values.
- Password reset: disabled (v1 has no email-based reset; reset is admin-driven).

---

## 4. Technical Transformation Strategy

For each transformation: **what changes and why.**

| Legacy | Target | Why |
|---|---|---|
| Procedural PHP pages | **Laravel controllers** (route → controller → view) | Gives central dispatch, testable logic, single entry points; replaces the page-per-file model (A4/A5) |
| `include 'session.php'` etc. | **Blade layouts + middleware** | Layout inheritance replaces the copy-pasted header/sidebar include chain; middleware replaces per-file bootstrapping (A1) |
| Raw SQL (PDO) | **Eloquent / Query Builder** | Safe parameter binding in one code path; models map to existing tables without schema change (AD-4) |
| Mixed HTML/PHP | **Blade templates** | Separates presentation from logic; components reuse UI (panel, tables); breaks monoliths (A4) |
| `$_SESSION['user_id']` checks per page | **Auth middleware + guards** | Central, testable authentication; same single-device token via `EnsureSingleDevice` (AD-6) |
| Manual validation in pages | **Laravel validation (Form Requests)** | Consistent, centralized input rules; closes ad-hoc validation drift |
| `restriction.php` + username checks | **Gates / Policies + one ACL service** | One predictable authorization model; kills the dual-ACL and `user_id=1` special case (A2/A3, AD-7) |
| Bootstrap pages | **Tailwind components** | Design-system-consistent UI; enables the slide-over panel cheaply (AD-3) |
| Separate View pages | **Right-side sliding details panel** | Removes a navigation round-trip per record; table stays visible; panel is one shared component (AD-10) |
| `log_action()` manual calls | **Model events / observers → AuditService** | Audit becomes structural; cannot be forgotten (AD-9) |
| Copied scanners | **One ScannerService + program config** | Programs become data; per-program duplicate rules encoded in config (ADR-004) |
| Server-side DataTables feed scripts | **DataTables JSON routes** | Same contract (draw/recordsTotal/recordsFiltered/data), framework-served (ADR-006 heritage) |
| CSV export (manual fputcsv) | **ExportService (UTF-8 BOM)** | Same byte-level output for backward compatibility |
| Hard-coded env/DB config | **`.env` + config files** | Secrets out of code (ADR-007) |

Every transformation preserves observable behavior (same data, same rules, same
exports) while changing only the internal structure.

---

## 5. Module Deliverables (P0–P8)

Each phase lists deliverables, acceptance criteria, dependencies, risks, and
completion criteria. Gates build on `MIGRATION_PLANNING.md` §6.

### P0 — Foundations
- **Deliverables:** Laravel scaffold; `.env` per environment; baseline migration generated from prod dump; DB copy for staging; Git + CI stub; backup/restore drill; asset migration (`seal_logo.png`, `sounds/`, `uploads/` wiring); `README` runbook.
- **Acceptance criteria:** app boots against a copy of `main_system`; baseline `SHOW CREATE TABLE` diff empty; restore drill passes.
- **Dependencies:** Architecture decisions signed off; Hostinger Premium+ (SSH tier) / PHP 8.3+ confirmed.
- **Risks:** baseline drift (mitigate: generate from fresh dump + diff); PHP/hosting mismatch.
- **Completion criteria:** P0 gate signed off; CI green.

### P1 — Auth + RBAC
- **Deliverables:** username auth provider; `EnsureSingleDevice` middleware (token contract); login/logout/force-logout; user provider mapping; ACL service + page middleware + Gates; permission seeding; navbar/sidebar Blade layout + Tailwind shell; session status route.
- **Acceptance criteria:** login by username works; token mismatch forces logout; every v1 page-permission row enforced; exempt accounts still exempt; audit writes to `tbl_audit_logs`.
- **Dependencies:** P0.
- **Risks:** framework auth defaults vs legacy `tbl_users` (mitigate: no `email`; disable reset).
- **Completion criteria:** P1 parity + tests green.

### P2 — Clients + households (+ slide-over)
- **Deliverables:** `ClientService` (central `full_name`/`match_name`/`age` derivation); Client CRUD; DataTables route; duplicate detection + review/delete; photo upload; households + family members; geography cascade; **slide-over details panel** component.
- **Acceptance criteria:** client CRUD matches v1; duplicate candidates match v1 results; row-click opens panel with full profile; Edit/Delete work from panel; CSV of v1 sample rows identical.
- **Dependencies:** P1; municipalities/barangays data.
- **Risks:** denormalized drift (mitigate: single service path); view-page monolith (mitigate: panel component first).
- **Completion criteria:** P2 parity + tests green; staff pilot OK on panel.

### P3 — Transactions + reports/exports
- **Deliverables:** `TransactionService`; CRUD + DataTables + filters; program-permission gating; CSV export with BOM; report list parity.
- **Acceptance criteria:** filters/sorts match v1; per-program sums match; export byte-comparable to v1.
- **Dependencies:** P1, P2.
- **Risks:** enum/remarks edge cases (mitigate: keep strings byte-for-byte).
- **Completion criteria:** P3 parity + tests green.

### P4 — Scanner engine (all programs)
- **Deliverables:** one `ScannerService`; program config (duplicate rules, target tables); one scanner Blade view + `lookup`/`save` routes; per-program acceptance matrix for all 17 programs; sound/modal feedback.
- **Acceptance criteria:** each program variant (fixed remark key, monthly guard, exam-derived, update-in-place, validate-existing, seat-aware, generic) reproduces v1 exactly; duplicates blocked at DB + app.
- **Dependencies:** P2 (clients), P3 (transactions).
- **Risks:** highest — variant drift (mitigate: matrix + test each before sign-off).
- **Completion criteria:** 17/17 matrix entries pass.

### P5 — Payout attendance + unpaid verification
- **Deliverables:** `PayoutAttendanceController` (3 screens); `UnpaidVerificationController` + service (proxy block); seat info join; DataTables + CSV.
- **Acceptance criteria:** one-scan-per-transaction enforced; seats match; unpaid proxy capture matches v1.
- **Dependencies:** P3.
- **Risks:** duplicate-race (mitigate: keep UNIQUE + app pre-check).
- **Completion criteria:** P5 parity + tests green.

### P6 — Scholars / GIP / exam
- **Deliverables:** `ScholarController`/`GipController`/exam handling; grantee updates (`tbl_update_logs`); scholarship reports + export; QR viewer.
- **Acceptance criteria:** report totals match v1 exports; update-log format preserved.
- **Dependencies:** P2, P3.
- **Risks:** low.
- **Completion criteria:** P6 parity + tests green.

### P7 — Administration
- **Deliverables:** permission management (pages + programs + exemptions) on the single ACL service; audit viewer + leaderboard (permission-based); **remove `manage_php.php`** concept (version control replaces runtime PHP editing).
- **Acceptance criteria:** admins grant/revoke permissions identically to v1; audit viewer reads v1 history unchanged.
- **Dependencies:** P1 (ACL), all modules.
- **Risks:** permission-screen UX change (mitigate: pilot with admin).
- **Completion criteria:** P7 parity + tests green.

### P8 — Hardening + regression + cutover
- **Deliverables:** full test suite; security review vs C1–C5; reconciliation script run on staging; rehearsal cutover + rollback; cutover runbook execution; credential rotation; v1 archive.
- **Acceptance criteria:** all gates green; rehearsal cutover reconciles; rollback exercised; live cutover green.
- **Dependencies:** P0–P7.
- **Risks:** cutover window (mitigate: freeze + backup + rollback, per `MIGRATION_PLANNING.md` §7–8).
- **Completion criteria:** acceptance criteria of `MIGRATION_PLANNING.md` §10 met.

---

## 6. Dependency Matrix

```
P0 Foundations
   │
   ▼
P1 Auth ──► Permissions (ACL) ──► (menu/sidebar)
   │
   ├──────────► P2 Clients ──► P3 Transactions ──► P4 Scanner engine
   │                │                │                  │
   │                ▼                ▼                  ▼
   │         Households       (seats)          P5 Payout + Unpaid
   │                                              │
   │                                              ▼
   │                                          P6 Scholars / GIP / exam
   │                                              │
   └────────────► P7 Administration ◄─────────────┘
                              │
                              ▼
                        P8 Hardening + Cutover
```

**Why this order minimizes migration risk:**
1. **Auth first (P1)** — nothing functions without login/authorization; it proves the riskiest port (username provider + single-device token) early, before any business module is built.
2. **Clients before transactions (P2 → P3)** — transactions are the activity hub that references clients; building the registry first gives the slide-over pattern and the name-derivation service that everything downstream (scanners, payouts, reports) depends on.
3. **Scanner last-but-not-least (P4)** — the highest-risk rewrite (16→1 engine) is deliberately deferred until the core registry and transaction paths are proven, so its per-program matrix is validated against stable foundations.
4. **Payouts depend on transactions (P5)** — attendance rows key on `transaction_id`.
5. **Scholars depend on clients + transactions (P6)** — reports join both.
6. **Administration last (P7)** — it governs all modules; rebuilding its ACL screen on the new single service only makes sense once the modules it guards exist.
7. **Hardening/cutover final (P8)** — by definition closes the loop.
8. Any upstream failure blocks only its dependents, and each gate is independently testable against a DB copy — so a defect is contained to one module.

---

## 7. Legacy Compatibility Matrix

Every guarantee that must survive the migration, and how it is preserved.

| Guarantee | Preservation strategy |
|---|---|
| **Database schema** | Frozen; baseline migration matches prod exactly; additive-only after (AD-4) |
| **Primary keys** | Never renumbered; new rows continue existing auto-increment; MAX(ID) reconciliation per gate |
| **Foreign keys / uniqueness** | Constraints untouched; models map to them; `transaction_id` UNIQUE stays the anti-duplicate belt |
| **Audit logs** | Same tables and field format; appended via events/observers (AD-9); history readable in v2 |
| **CSV exports** | UTF-8 BOM + same columns/order; byte-comparable tests (P3/P6) |
| **Business rules** | Encoded in services (duplicate rules, name matching, one-scan-per-transaction) and validated by per-module parity tests |
| **Permission keys** | `page_name` values identical to v1; seeded unchanged (ADR-003) |
| **Program names** | 17 enum strings byte-for-byte; scanner config uses the same keys (ADR-004) |
| **Login behavior** | username + bcrypt; same hashes validate; no re-hash needed (AD-6) |
| **Session behavior** | single-device token + idle timeout + exempt accounts preserved (ADR-002) |
| **Denormalized fields** | `full_name`/`match_name`/`age` continue to be stored; now derived in one service path (AD-6 fix) |
| **Report formats** | Same queries → same numbers; CSV unchanged |
| **Deep links / print** | Detail routes remain reachable; slide-over is primary, not replacement of capability (AD-10) |

---

## 8. File Migration Matrix

Implementation checklist. **Status:** all `Planned` until its phase starts.
Dead files (`default.php`, `client_photo.php`) are deliberately excluded.

| # | Legacy file | Target component | Phase | Status |
|---|---|---|---|---|
| 1 | `db_connect.php` | `config/database.php`, `.env`, `AppServiceProvider` DB boot | P0 | Planned |
| 2 | `.sql` dumps | baseline migration (generated, not authored) | P0 | Planned |
| 3 | assets (`seal_logo.png`, `sounds/`, `uploads/`) | `public/` + `storage/` wiring | P0 | Planned |
| 4 | `session.php` | `config/session.php`; `EnsureSingleDevice` middleware | P1 | Planned |
| 5 | `restriction.php` | ACL service; `AuthorizePage` middleware; Gates/Policies | P1 | Planned |
| 6 | `logs.php` | `AuditService`; model events/observers | P1 | Planned |
| 7 | `login.php` | `AuthController@login` + `LoginRequest` | P1 | Planned |
| 8 | `logout.php` | `AuthController@logout` (route) | P1 | Planned |
| 9 | `check_session.php` | `SessionController@status` (JSON) | P1 | Planned |
| 10 | `force_logout.php` | `AdminController@forceLogout` | P1 | Planned |
| 11 | `currently_logged_users.php` | `SessionController@online` + view | P1 | Planned |
| 12 | `fetch_online_users.php` | DataTables route (online) | P1 | Planned |
| 13 | `verify_mobile.php` | `ClientController@verifyMobile` | P2 | Planned |
| 14 | `navbar.php` | Blade layout partial | P1 | Planned |
| 15 | `sidebar.php` | Blade component (role-driven) | P1 | Planned |
| 16 | `sidebar.css/js` | Tailwind + Vite asset | P1 | Planned |
| 17 | `favicon.php` | layout `<link>` | P1 | Planned |
| 18 | `index.php` | `DashboardController@index` + view | P1 | Planned |
| 19 | `clients.php` | `ClientController@index` + view | P2 | Planned |
| 20 | `fetch_clients.php` | DataTables route (clients) | P2 | Planned |
| 21 | `add_client.php` | `ClientController@store` + `ClientService` | P2 | Planned |
| 22 | `edit_client.php` | `ClientController@edit/update` | P2 | Planned |
| 23 | `view_client.php` | `ClientController@show` + slide-over panel | P2 | Planned |
| 24 | `delete_client.php` | `ClientController@destroy` + Policy | P2 | Planned |
| 25 | `search_clients.php` | `ClientController@search` | P2 | Planned |
| 26 | `search_clients_hh.php` | `ClientController@searchForHousehold` | P2 | Planned |
| 27 | `get_client_hh.php` | `ClientController@hhOptions` (JSON) | P2 | Planned |
| 28 | `preview_duplicates.php` | `DuplicateController@preview` + view | P2 | Planned |
| 29 | `fetch_duplicates.php` | DataTables route (duplicates) | P2 | Planned |
| 30 | `delete_duplicates.php` | `DuplicateController@destroyMany` | P2 | Planned |
| 31 | `save_client_photo.php` | `PhotoController@store` | P2 | Planned |
| 32 | `student_photo_upload.php` | `StudentController@uploadPhoto` | P2 | Planned |
| 33 | `student_update_photo.php` | `StudentController@updatePhoto` | P2 | Planned |
| 34 | `student_verify.php` | `StudentController@verify` | P2 | Planned |
| 35 | `get_barangays.php` | `GeographyController@barangays` | P2 | Planned |
| 36 | `household.php` | `HouseholdController@index` + view | P2 | Planned |
| 37 | `add_household.php` | `HouseholdController@store` | P2 | Planned |
| 38 | `view_household.php` | `HouseholdController@show` (+ panel) | P2 | Planned |
| 39 | `delete_household.php` | `HouseholdController@destroy` + Policy | P2 | Planned |
| 40 | `add_family_member.php` | `FamilyMemberController@store` | P2 | Planned |
| 41 | `fetch_households.php` | DataTables route (households) | P2 | Planned |
| 42 | `get_household.php` | `HouseholdController@show` (JSON) | P2 | Planned |
| 43 | `search_households.php` | `HouseholdController@search` | P2 | Planned |
| 44 | `all_transactions.php` | `TransactionController@index` + view | P3 | Planned |
| 45 | `fetch_transactions.php` | DataTables route (transactions) | P3 | Planned |
| 46 | `add_transaction.php` | `TransactionController@create/store` | P3 | Planned |
| 47 | `edit_transaction.php` | `TransactionController@edit/update` | P3 | Planned |
| 48 | `view_transaction.php` | `TransactionController@show` (+ panel) | P3 | Planned |
| 49 | `delete_transaction.php` | `TransactionController@destroy` + Policy | P3 | Planned |
| 50 | `update_transaction.php` | merged into edit flow | P3 | Planned |
| 51 | `all_transaction_edit.php` | `TransactionController@editFromList` | P3 | Planned |
| 52 | `all_transaction_delete.php` | `TransactionController@destroyFromList` | P3 | Planned |
| 53 | `transaction_table.php` | Blade partial component | P3 | Planned |
| 54 | `scanner_ceap(.php/_action.php)` | Scanner engine config entry + routes | P4 | Planned |
| 55 | `scanner_ceap_new` | config entry | P4 | Planned |
| 56 | `scanner_cedssg` | config entry | P4 | Planned |
| 57 | `scanner_cedssg_new` | config entry | P4 | Planned |
| 58 | `scanner_cedssg_update` | config entry (update-in-place rule) | P4 | Planned |
| 59 | `scanner_tupad` | config entry (monthly guard) | P4 | Planned |
| 60 | `scanner_toda` | config entry | P4 | Planned |
| 61 | `scanner_otces` | config entry | P4 | Planned |
| 62 | `scanner_otea` | config entry | P4 | Planned |
| 63 | `scanner_new_scholars` | config entry (exam-derived) | P4 | Planned |
| 64 | `scanner_ongoing_scholars` | config entry (validate-existing) | P4 | Planned |
| 65 | `scanner_payout` | config entry (seat-aware, `lookup_ignore_scan`) | P4 | Planned |
| 66 | `scanner_payout_unpaid` | config entry | P4 | Planned |
| 67 | `scanner_generic` | config entry (program chosen in form) | P4 | Planned |
| 68 | `scanned_payouts.php` | `PayoutAttendanceController` (screen 1) | P5 | Planned |
| 69 | `scanned_payouts2.php` | screen 2 | P5 | Planned |
| 70 | `scanned_payouts_unpaid.php` | screen 3 | P5 | Planned |
| 71 | `fetch_scanned_payouts.php` | DataTables route (scans 1) | P5 | Planned |
| 72 | `fetch_scanned_payouts2.php` | DataTables route (scans 2) | P5 | Planned |
| 73 | `fetch_scanned_payouts_unpaid.php` | DataTables route (scans unpaid) | P5 | Planned |
| 74 | `unpaid_verifications.php` | `UnpaidVerificationController@index` + view | P5 | Planned |
| 75 | `disabled_unpaid.php` | merged into 74 | P5 | Planned |
| 76 | `unpaid_save.php` | `UnpaidVerificationController@store` + service | P5 | Planned |
| 77 | `fetch_unpaid_verifications.php` | DataTables route (unpaid) | P5 | Planned |
| 78 | `export_unpaid_verifications.php` | ExportService route | P5 | Planned |
| 79 | `search_grantee.php` | `SearchController@grantee` | P5 | Planned |
| 80 | `search_unpaid_grantee.php` | `SearchController@unpaidGrantee` | P5 | Planned |
| 81 | `scholars.php` | `ScholarController@index` + view | P6 | Planned |
| 82 | `save_scholarship.php` | `ScholarController@store/update` | P6 | Planned |
| 83 | `fetch_scholars.php` | DataTables route (scholars) | P6 | Planned |
| 84 | `update_client_id.php` | `ScholarController@relink` | P6 | Planned |
| 85 | `scholarship_reports.php` | `ReportController@scholarship` + view | P6 | Planned |
| 86 | `fetch_scholarship_reports.php` | DataTables route (scholarship report) | P6 | Planned |
| 87 | `export_scholarship_reports.php` | ExportService route | P6 | Planned |
| 88 | `save_gip.php` | `GipController@store` | P6 | Planned |
| 89 | `save_grantee_update.php` | `GranteeUpdateController@store` | P6 | Planned |
| 90 | `disabled_update_grantee.php` | `GranteeUpdateController@edit` + view | P6 | Planned |
| 91 | `update_logs.php` | `GranteeUpdateController@logs` + view | P6 | Planned |
| 92 | `fetch_update_logs.php` | DataTables route (update logs) | P6 | Planned |
| 93 | `view_qrcode.php` | `QrController@show` | P6 | Planned |
| 94 | `manage_permissions.php` | `AdminPermissionController@pages` | P7 | Planned |
| 95 | `manage_program_permissions.php` | `AdminPermissionController@programs` | P7 | Planned |
| 96 | `manage_multi_device_exemptions.php` | `AdminPermissionController@exemptions` | P7 | Planned |
| 97 | `audit_logs.php` | `AuditController@index` + view | P7 | Planned |
| 98 | `fetch_logs.php` | DataTables route (audit) | P7 | Planned |
| 99 | `fetch_leaderboard.php` | DataTables route (leaderboard) | P7 | Planned |
| 100 | `register.php`, `add_user.php`, `manage_php.php` | `UserController` (admin) | P7 | Planned |

---

## 9. Implementation Readiness Assessment

### 9.1 Remaining documentation (before P0)
- **Environment runbook** for Hostinger (PHP pin in hPanel + SSH CLI, deploy steps) — partially covered in `MIGRATION_PLAN.md`; formalize.
- **Per-program scanner matrix** (all 17 programs × duplicate rule) — must be written before P4.
- **Reconciliation script spec** — counts/sums/MAX(ID) queries per table (from `MIGRATION_PLANNING.md` §5).
- **Test strategy** — what parity means per module (section 5 acceptance criteria are the basis).

### 9.2 Remaining risks
| Risk | Open question |
|---|---|
| Framework/auth port (no email, token contract) | Low-technical, but must be proven first (P1 gate) |
| Scanner variant fidelity | Highest; needs the 17-program matrix |
| Denormalized-field drift | Mitigated by single `ClientService`; historical drift remains a data-quality item (out of scope) |
| Hosting PHP 8.3+ / plan tier (Premium+ for SSH) | **Owner action** — team lacks hPanel access; owner must confirm SSH enabled + PHP 8.3+ before staging/cutover |

### 9.3 Missing decisions (must be made before coding)
1. **Framework confirmation** — Laravel (default) vs CodeIgniter 4 fallback (ADR-001).
2. **Hosting** — ✅ **RESOLVED**: Hostinger **Premium** plan confirmed (owner) — includes SSH (Premium+ tier) and PHP 8.3+; stay on shared hosting, no VPS needed at this scale (affects ADR-001/010).
3. **Soft deletes / client merge** — in or out of v2 scope (affects P2).
4. **Index additions** — approve additive-only new indexes (recommended yes).
5. **ADRs sign-off** — flip ADR-001…010 from Proposed → Accepted.

### 9.4 Prerequisites
- Production dump captured for baseline (fresh, not the old one).
- Backup + restore drill passed.
- `doc/v2` blueprint (this document) reviewed by the team.
- `.gitignore` for v2 (Laravel defaults — track `.php`, ignore `.env`/`vendor`/`node_modules`).
- **Hosting owner actions** (team has no hPanel access): confirm **SSH enabled** (Advanced → SSH Access), **PHP 8.3+ selectable**, and provide SSH/staging credentials to the team. Local XAMPP development can proceed without these; they gate staging and cutover only.

### 9.5 Recommendations before coding begins
1. Get the five missing decisions (9.3) answered and ADRs accepted.
2. Run P0 first and stop: proving baseline parity + boot on a DB copy is the cheapest validation of the whole strategy.
3. Freeze the v1 production dump used for baseline; treat it as the parity reference.
4. Keep v1 fully operational until P8; the blueprint changes nothing in production.

---

*End of Engineering Migration Blueprint.*
