# System Design — 2D MIS

> Design-level documentation: how the system is *structured*, the patterns it
> uses, how data flows through each subsystem, and the design decisions
> (explicit and implicit) behind it. Read-only — no source, SQL, or database
> was modified.

---

## 1. Design goals & constraints (as inferred)

The system was built to solve a concrete, operational need rather than as a
platform:

| Goal / constraint | Evidence in the design |
|---|---|
| Fast to build, easy to modify by a small team | Plain PHP, no build step, no dependencies to install |
| Works on shared hosting | File-per-page model, no CLI tooling, standard PHP 7.2 |
| Record people + assistance programs accurately | `tbl_clients`, `tbl_transactions` as the two hubs |
| QR-scan attendance at live events | Camera scanning in-browser (`html5-qrcode`) |
| One operator per account | Single-device session token enforcement |
| Traceability of who did what | `log_action()` audit trail on mutating flows |
| Non-technical users, often on tablets/laptops | Bootstrap 5 + large buttons + sound feedback |

## 2. Architectural style

The system is a **server-rendered, page-per-module web application** with an
**AJAX/JSON endpoint layer** for dynamic tables and scanning.

- **No framework, no routing table.** Each URL maps 1:1 to a PHP file
  (`clients.php`, `scanner_ceap.php`, …). Authorization is per-file via the
  `restriction.php` include, not centralized routing.
- **Thin browser, fat server.** Pages render HTML; DataTables fetch JSON;
  scanners call action handlers. All business rules live in PHP.
- **Database-first design.** The schema is the source of truth; application
  code is written against raw SQL (PDO prepared statements) rather than an ORM.

### Layer view

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation  HTML + Bootstrap 5 + jQuery + DataTables      │
│                + html5-qrcode (scanner pages)                │
├─────────────────────────────────────────────────────────────┤
│  Application  PHP pages (render) + *_action.php / fetch_*.php│
│               (JSON endpoints)  + save_*.php (form posts)    │
├─────────────────────────────────────────────────────────────┤
│  Cross-cutting  session.php  restriction.php  logs.php       │
│                 (auth/ACL/audit)                             │
├─────────────────────────────────────────────────────────────┤
│  Data access   PDO prepared statements (db_connect.php)      │
├─────────────────────────────────────────────────────────────┤
│  Storage       MariaDB (main_system, 31 tables)              │
└─────────────────────────────────────────────────────────────┘
```

## 3. Design patterns in use

### 3.1 Bootstrap-include chain (quasi-front-controller)
Every protected page begins the same way:

```php
include 'session.php';      // session lifecycle + single-device enforcement
include 'db_connect.php';   // $pdo
include 'restriction.php';  // page-level ACL
if (!isset($_SESSION['user_id'])) { header("Location: login.php"); exit; }
```

`session.php` is the closest thing to a front controller — it centralizes
session bootstrap, cookie parameters, token validation, and the
`last_activity` heartbeat. `restriction.php` centralizes page access control.
This include chain is the **only truly shared architectural element**.

### 3.2 Action-handler pattern (command-style endpoints)
State changes are funneled through dedicated handler files that return JSON:

- Scanner: `scanner_*_action.php` handles `action=lookup` / `action=save`.
- Tables: `fetch_*.php` handle DataTables `POST` payloads.
- Forms: `save_*.php` / `*_delete.php` handle form submissions.

The scanner handler is a clear **two-phase command**:
1. `lookup` — read-only query, returns candidate + details.
2. `save` — guarded insert (duplicate check → insert → audit log).

### 3.3 Server-side DataTables contract
List pages delegate paging/search/sorting to the DB. The browser posts
`draw`, `start`, `length`, `search`, `order`, and the endpoint returns the
standard DataTables JSON envelope:

```json
{ "draw": n, "recordsTotal": n, "recordsFiltered": n, "data": [ ... ] }
```

Filters (municipality, barangay, program, date range, status) are sent as
extra params and converted to a dynamic `WHERE` clause built from a `$where[]`
array + named parameters (see `all_transactions.php:19-94`). This
parameter-building pattern keeps SQL injection-safe dynamic filtering.

### 3.4 Observer-style audit logging
`logs.php::log_action()` is called after every mutation
(`ADD_CLIENT`, `SCAN-CEAP`, updates, deletes) and appends to
`tbl_audit_logs` with user, action, target table/id, and before/after JSON
snapshots. It is a coarse-grained observer: call sites invoke it manually.

### 3.5 Gateway/DTO via raw SQL
No ORM or repository layer. Pages query directly. Denormalized columns
(`full_name`, `match_name`, `patient_name`) act as pre-computed DTO fields to
speed up name lookups and matching.

### 3.6 Configuration via constants/arrays in-file
Dropdown lists (programs, statuses) are hard-coded PHP arrays inside pages
(e.g. `scanned_payouts.php:65`, `manage_permissions.php:21`), not centralized
config. This is a design debt (see §11).

## 4. Request lifecycle design

**Typical list page (read):**
```
GET clients.php
  └ session.php   → token OK?
  └ restriction.php → page allowed?
  └ render shell + empty DataTable
GET/POST fetch_clients.php  (DataTables params)
  └ build WHERE from filters → PDO query → JSON envelope → DataTable render
```

**Typical mutation (scan save):**
```
scanner_ceap.php (camera) ──POST action=lookup──▶ scanner_ceap_action.php
   ▲                                              └ match full_name → JSON
   │ POST action=save&id=N
   └────────────────────────────────────────────▶ duplicate guard → INSERT
                                                  → log_action → JSON ok
```

**Lifecycle rules**
- No caching layer; every request hits the DB (acceptable for the data volume).
- Timezone is explicitly set to `Asia/Manila` on pages that print dates
  (`all_transactions.php:4`, `audit_logs.php:4`), but not globally — a
  latent inconsistency.
- Mutations use `try/catch (PDOException)` and return JSON error messages,
  surfacing raw DB text to the UI in some handlers.

## 5. Subsystem design

### 5.1 Authentication & session subsystem
- **Design:** `tbl_users` holds `username`, bcrypt `password`, plus
  `session_token` and `last_activity`. Login mints a fresh token and stores it
  both in the session and the DB row.
- **Single-device design:** for non-exempt users, every request compares the
  session token to the DB token. Any divergence (re-login elsewhere, admin
  force-logout) invalidates the session. Exemptions are a union of
  hard-coded usernames (`super_admin`, `jordi`) and
  `tbl_multi_device_exemptions`.
- **Keep-alive design:** `navbar.php` polls `check_session.php` every 2 s to
  detect remote invalidation client-side.

### 5.2 Access control subsystem
Two mechanisms (design conflict — see §11):
1. **Page ACL** — `tbl_permissions(user_id, page_name)`, enforced by
   `restriction.php`; `manage_permissions.php` rebuilds rows per user.
2. **Username gating** — `sidebar.php`, `audit_logs.php`, `manage_php.php`,
   `clients.php` check `$_SESSION['username']` against literal names.

### 5.3 Client registry subsystem
- Single write path in `add_client.php`: uppercase normalization → insert →
  derive `full_name`/`match_name` → optional aff. orgs → audit log.
- Reads are served via `fetch_clients.php` (server-side) and
  `view_client.php` (aggregate profile joining photos, family, household,
  transactions, scholar, GIP).
- Duplicate detection is a batch compare over `full_name`/`match_name`
  within municipality/barangay (`fetch_duplicates.php`), with operator review
  before delete.

### 5.4 Household subsystem
`tbl_household` (code + head) ↔ `tbl_clients.household_id` (FK SET NULL);
`tbl_family_members` (client_id, relative_id, relationship) models kinship
edges independent of households.

### 5.5 Transactions subsystem
The program-agnostic hub: one `tbl_transactions` row per assistance event,
with `program` enum, `status` (PENDING PAYOUT / PAID), amount and dates.
Pages filter/sort/export this table; scanners write to it; payout
attendance references it by `transaction_id`.

### 5.6 QR scanner subsystem (replicated design)
Each program has a copy of the same design:
- **Page** — `html5-qrcode` camera + result area + save/cancel buttons.
- **Handler** — `lookup` (name match) + `save` (duplicate-guarded insert +
  audit log).
- **Variants** by duplicate rule: fixed remark key (CEAP/OTEA/OTCES/new
  variants), per-month guard (TUPAD), program derived from exam results
  (`scanner_new_scholars_action.php`), update-in-place
  (`scanner_cedssg_update_action.php`), validate-existing-scholar
  (`scanner_ongoing_scholars_action.php`), and seat-aware payout scan
  (`scanner_payout_action.php` joining `tbl_seats2`).
- **Feedback design:** Bootstrap modal + `sounds/success.mp3` /
  `sounds/not_found.mp3`.

### 5.7 Payout attendance subsystem
`tbl_payout_scans` / `_2` / `_unpaid` are three identical one-to-one logs:
`transaction_id` UNIQUE enforces "scan once per transaction" at the DB level
(belt), and the handler pre-checks "already scanned" (suspenders). Each table
backs a separate attendance screen.

### 5.8 Unpaid verification subsystem
`tbl_unpaid_verifications` captures the grantee plus an optional proxy
receiver (full proxy identity block). This allows reporting who actually
received a payout when the grantee could not appear.

### 5.9 Scholarship subsystem
`tbl_scholar_info` (enrollment) + `tbl_transactions` (financial/payout
status) + `tbl_exam`/`tbl_results` (admission gate) + `tbl_gip_info` (GIP
applicant data). Reports join transactions × scholar_info.

### 5.10 Reporting/export subsystem
Pages render reports server-side; exports stream CSV with a UTF-8 BOM for
Excel compatibility (`all_transactions.php:98-108`,
`export_scholarship_reports.php`).

## 6. Data design

### Entity hub model
`tbl_clients` is the primary hub; `tbl_transactions` is the activity hub;
`tbl_users` is the actor hub. Everything else attaches to one of these.

```
              ┌────────────┐
              │ tbl_users  │
              └──┬──────┬──┘
        audits / │      │ scanned_by
             ┌───▼──────▼────┐
             │  tbl_transactions  │◄──── tbl_payout_scans*
             └───▲──────────┘
                 │ client_id
   aff_orgs ─┐   │
   photos ───┤   │
   family ───┤   │
   scholar ──┤   │
   gip ──────┤   │
   unpaid ───┘   │
        ┌────────┴────────┐
        │   tbl_clients    │──► tbl_household (head + member)
        └─────────────────┘
```

### Design choices worth noting
- **Denormalization:** `full_name`, `match_name`, `patient_name` are stored
  for fast scan/lookup matching. Cost: must be kept in sync at every write.
- **Geography as IDs-in-varchar:** `clients.city_municipality` /
  `.barangay` hold FK values but without constraints — a deliberate shortcut
  that trades integrity for flexibility.
- **Enums for extensible lists:** `program` and `sex`/`civil_status`/`category`
  are enum columns; adding a value requires a DDL change.
- **No soft deletes:** deletions are physical; FK cascades remove dependents.

## 7. State & concurrency design

- **Session state:** PHP `$_SESSION` (file-based, 1-day default; 10-year for
  exempt users), cookie `httponly` + `SameSite=Lax`, `secure` only when HTTPS.
- **Idempotency at the DB:** unique keys stop double-scans
  (`tbl_payout_scans*.transaction_id`) and duplicate family links.
- **Duplicate guards at the app:** program+remark existence checks before
  scanner inserts; TUPAD monthly check.
- **Race caveat:** check-then-insert is not transactional, so two rapid scans
  can both pass the app-level check; only DB unique constraints fully prevent
  duplicates. Payout scans are safe (UNIQUE); scanner transaction inserts rely
  on the app-level check.

## 8. Security design (as built)

| Control | Present | Notes |
|---|---|---|
| Password hashing | ✅ | bcrypt via `password_hash` / `password_verify` |
| Prepared statements | ✅ | PDO everywhere |
| Login required | ✅ | `session.php` + per-page check |
| Session hardening | ✅ | httponly, SameSite=Lax, token-enforced single device |
| CSRF tokens | ❌ | none |
| Login rate limiting | ❌ | none |
| Output escaping | ⚠️ | `htmlspecialchars` used in many pages; not uniformly |
| Error disclosure control | ❌ | `manage_php.php` enables `display_errors`; some handlers echo DB errors |
| Credential storage | ❌ | DB credentials hard-coded in `db_connect.php` |

## 9. Error handling & logging design

- **DB connect:** single point (`db_connect.php`) that `die()`s with the
  exception message.
- **Query errors:** mixed — some pages `die()`, some `try/catch` and return
  JSON, some swallow exceptions silently (`session.php:90-94`).
- **Audit logging:** `tbl_audit_logs` (who/what/when), `tbl_update_logs`
  (grantee updates), `tbl_photo_logs` (photo changes), `password_resets`
  (who changed whose password).
- **Application error log:** none beyond the above tables.

## 10. Performance design

- Indexed hot paths: name/fulltext on `tbl_clients`, composite
  `(lastname,firstname,middlename)`, program/date indexes on
  `tbl_transactions`, unique scan keys.
- Server-side processing keeps DataTables from loading full tables into the
  browser.
- No caching, no pagination beyond DataTables; acceptable at current scale,
  a risk as records grow.

## 11. Design anti-patterns & debt (what to fix in v2)

| # | Anti-pattern | Where | Impact |
|---|---|---|---|
| A1 | Copy-paste subsystem (scanners) | `scanner_*.php` × 16 | Bug fixes don't propagate; change cost is linear |
| A2 | Two ACL models (DB vs username) | `restriction.php` vs `sidebar.php`/`audit_logs.php`/`manage_php.php` | Confusing, fragile if usernames change |
| A3 | Implicit super-user | `restriction.php` user_id 1 | Undocumented privilege |
| A4 | Monolithic pages | `view_client.php` (~170 KB), `all_transactions.php` (~1,400 lines) | Hard to read/test/maintain |
| A5 | Raw SQL with no service layer | all pages | No reuse, testing is hard |
| A6 | Denormalized fields synced by hand | `full_name`, `match_name`, `age`, `patient_name` | Drift breaks scans & duplicates |
| A7 | In-file config duplication | program lists, dropdowns | Adding a program touches many files |
| A8 | No migrations | schema dumps | Dev/prod drift (collation mismatch) |
| A9 | No CSRF/rate-limit | all forms | Security gap |
| A10 | Dead code | `default.php`, `client_photo.php`, commented blocks | Confusion, maintenance burden |

## 12. Design evolution path (v2.0)

Recommended sequence if the system is rebuilt on a framework (e.g. Laravel)
while keeping the same database:

1. **Keep the schema & data**; introduce migrations only as baselines.
2. **Centralize cross-cutting concerns**: single auth middleware (port the
   single-device token logic), single ACL service (merge A2/A3), CSRF + rate
   limit middleware (A9).
3. **Collapse the scanner family** into one controller + one reusable view
   driven by a program config (A1) — the scanner contract (`lookup`/`save`)
   maps cleanly to two routes.
4. **Introduce a service layer** for clients/transactions/scholars to make
   `full_name`/`match_name`/`age` derivation a single code path (A6).
5. **Refactor the monoliths** into partial views + presenters (A4).
6. **Add migrations + deploy pipeline + backups** (A8, ops).

The existing design already contains the blueprint for this: the include chain,
the `lookup`/`save` contract, the DataTables envelope, and the audit-log
interface translate directly into middleware, routes, resources, and observers
in a framework.

## 13. System Design Analysis Report — quality evaluation

Ratings use a 1–5 scale (1 = critical/blocking, 5 = excellent), judged against
the system's actual operating context: shared hosting, a small team, and
modest data volume. Evidence references are to sections above and to
`../v1/RECOMMENDATIONS.md`.

| Dimension | Rating | Verdict |
|---|---|---|
| Architecture quality | **2 / 5** | Coherent layering, but no routing/dispatch or service layer |
| Modularity | **2 / 5** | Good cross-cutting includes; heavily duplicated subsystems |
| Maintainability | **2 / 5** | Readable and consistent, but untested and drift-prone |
| Security design | **2 / 5** | Good fundamentals; critical gaps (CSRF, rate limit, credentials) |
| Scalability | **2 / 5** | Fine now; will degrade without caching/pagination changes |
| Refactoring readiness | **3 / 5** | Strong blueprint for v2; no tests and dead code raise risk |

### 13.1 Architecture quality — 2/5
**Strengths**
- Clear conceptual layering (presentation / application / cross-cutting /
  data access) and a single, consistent include chain for auth/ACL/audit (§3.1).
- Read vs write separation via `fetch_*.php` / `*_action.php` endpoints.
- Database-first with prepared statements everywhere — safe and predictable.

**Weaknesses**
- No router or front controller: URL → file mapping is implicit, so
  authorization, logging, and validation must be repeated per file.
- Business logic lives inside presentation pages (`view_client.php`,
  `all_transactions.php`) — A4, A5.
- Raw SQL everywhere with no service/repository layer → no reuse, no unit
  testing (§3.5, A5).

### 13.2 Modularity — 2/5
**Strengths**
- Cross-cutting concerns are centralized in `session.php`,
  `restriction.php`, `logs.php` — the only real shared modules.
- Audit logging is a consistent observer-style call site (§3.4).

**Weaknesses**
- The scanner subsystem is copied ~16 times (A1); payout-attendance logic and
  tables are likewise replicated threefold (§5.6–5.7).
- Program lists/dropdowns are hard-coded arrays repeated across files (A7).
- Duplicate screens (`fetch_*` tables per module) diverge in details.

### 13.3 Maintainability — 2/5
**Strengths**
- Plain, consistent PHP style; every page follows the same skeleton — easy for
  the current team to navigate.
- Audit trails are defined by convention and consistently populated.

**Weaknesses**
- No automated tests anywhere; no migration tooling (A8); not under version
  control at the time of this analysis.
- Denormalized fields (`full_name`, `match_name`, `age`) are synced by hand in
  multiple write paths → drift (A6, D-items).
- Error handling is inconsistent: `die()`, JSON errors, or silent swallow
  depending on file (§9).
- Dead code (`default.php`, `client_photo.php`) confuses readers (A10).

### 13.4 Security design — 2/5
**Strengths**
- bcrypt password hashing, PDO prepared statements, httponly + SameSite=Lax
  sessions, and DB-token single-device enforcement are all done properly (§8).
- Duplicate prevention is backed by DB unique constraints, not just app checks.

**Weaknesses (critical)**
- No CSRF tokens (A9), no login rate limiting (C2).
- DB credentials hard-coded in `db_connect.php` (C3).
- Error disclosure: `manage_php.php` enables `display_errors`; some handlers
  echo raw DB messages (C4).
- Two parallel ACL models, one of which keys on literal usernames and an
  undocumented `user_id = 1` super-user (A2, A3).

### 13.5 Scalability — 2/5
**Strengths**
- Hot paths are indexed (name/fulltext on `tbl_clients`, program/date on
  `tbl_transactions`, unique scan keys); server-side DataTables keeps large
  lists off the browser (§10, §3.3).
- DB-enforced idempotency keeps concurrent scans safe on payout tables (§7).

**Weaknesses**
- No caching layer; every request hits the DB (§4).
- Pagination only exists inside DataTables endpoints; reports render
  full-server-side.
- Check-then-insert scanning is non-transactional — a genuine (if rare) race
  (§7).

### 13.6 Refactoring readiness — 3/5
**Strengths**
- The design already encodes the v2 blueprint: the include chain maps to
  middleware, the `lookup`/`save` contract maps to routes, the DataTables
  envelope and audit interface map to framework resources/observers (§12).
- The domain is small and well understood; a config-driven scanner engine
  (one engine, 17 program entries) is a well-bounded refactor.
- The data layer is raw SQL with clear tables — portable to a framework
  without a schema rewrite.

**Weaknesses**
- No tests means the refactor has no safety net; monolithic pages are the
  riskiest split (A4); dead code must be removed first (A10).
- Hand-synced denormalized fields must be unified into one write path before
  or during the rebuild (A6).
- No migration baseline yet → dev/prod drift must be frozen first (A8).

### 13.7 Overall verdict
At **2/5 overall**, the system is fit for purpose today but structurally
constrained: it can be maintained, but each change is slow, repetitive, and
risky. The upgrade to v2 is justified primarily by **maintainability and
security**, not by raw feature gaps. Because the data layer is clean and the
design already contains a clear migration blueprint, the refactor is feasible
with moderate risk — provided migrations, tests, and a data-preservation
baseline are established before any code is rewritten (see
`../v2/MIGRATION_PLAN.md`).

---

*End of design documentation.*
