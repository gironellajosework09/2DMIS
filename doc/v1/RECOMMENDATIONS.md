# Recommendations & Improvement Plan

> Read-only assessment produced from static analysis of the source and schema.
> Nothing was changed. Items are grouped by priority
> (🔴 Critical → 🟠 High → 🟡 Medium → 🟢 Low) and category.
> Suggested approaches are design-level only.

---

## 🔴 Critical — Security

### C1. Production database credentials are hard-coded in source
- **Finding:** `db_connect.php` shipped with live Hostinger credentials
  (`u749085076_main_system` / `&lt;redacted&gt;`) in plaintext. Anyone with repo/file
  access had full DB credentials, including external-host access.
- **Why it matters:** Credential leak → full data exfiltration/modification
  of the production DB.
- **Suggestion:** Move connection settings to environment variables or a
  `.env` file excluded from source control. Rotate the leaked password.
  Treat the exposed password as compromised regardless of local switch-over.

### C2. No CSRF protection anywhere
- **Finding:** No form in the app uses a CSRF token; every state-changing
  action (`manage_permissions.php`, `force_logout.php`, `unpaid_save.php`,
  scanners, deletes, password reset) is a plain POST that an attacker can
  trigger cross-site.
- **Suggestion:** Add a per-session CSRF token generated in `session.php`,
  embedded in all forms, and verified by all POST handlers.

### C3. No login rate-limiting / brute-force protection
- **Finding:** `login.php` has unlimited attempts; no account lockout, no
  delay, no failed-login audit.
- **Suggestion:** Track failed attempts (DB or session), throttle after N
  failures, add exponential backoff, and optionally record failed attempts
  for review. Consider a password policy (min length already exists in
  `manage_php.php`; extend to registration).

### C4. Super-admin bypass and username-based gating are fragile
- **Finding:**
  - `restriction.php:22` lets `user_id == 1` access everything unconditionally.
  - `sidebar.php`, `audit_logs.php`, `manage_php.php`, and `clients.php`
    hard-code the usernames `super_admin`, `god_admin`, `jordi`.
- **Why it matters:** Changing a username silently changes privileges;
  `user_id 1` privilege is implicit and undocumented; two parallel auth
  models (username vs. permission) are easy to misconfigure.
- **Suggestion:** Introduce a real `role` column (e.g. `admin`, `user`) on
  `tbl_users`, keep the permission tables, and replace all username
  hard-coding with role/permission checks in a single shared function.

### C5. `display_errors` enabled on a live page
- **Finding:** `manage_php.php:2-3` sets `ini_set('display_errors', 1)` and
  `error_reporting(E_ALL)`, which can leak paths, queries, and schema details
  on failure.
- **Suggestion:** Only enable display in local/dev; log to a file on
  production. Remove the directive or gate it behind an environment flag.

### C6. Third-party QR generation dependency
- **Finding:** `view_qrcode.php` generates QR codes through the external
  `api.qrserver.com` service.
- **Why it matters:** Sends names over the network to a third party and
  breaks offline; availability depends on the service.
- **Suggestion:** Generate QR codes locally (e.g. bundled `phpqrcode`
  library) so scanning keys never leave the server.

---

## 🟠 High — Maintainability & Architecture

### M1. Massive code duplication, especially the 16 scanners
- **Finding:** Each scanner (`scanner_*.php` + `scanner_*_action.php`) is a
  near-identical copy differing mainly by the program constant and duplicate
  rule. `fetch_*.php` and page boilerplate are similarly duplicated.
- **Why it matters:** Every program change means editing ~30 files; bugs get
  fixed in some scanners but not others.
- **Suggestion:** Extract a single reusable scanner page + handler
  parameterized by program and duplicate rule (config-driven), then make each
  program's page a thin wrapper. Do the same for list/fetch pages.

### M2. Monolithic pages
- **Finding:** `view_client.php` ≈ 170 KB, `all_transactions.php` ≈ 1,400
  lines, `disabled_update_grantee.php` ≈ 72 KB. HTML, CSS, PHP, and JS are
  interleaved.
- **Suggestion:** Split into partials (query layer, view layer, JS layer);
  move shared head/CSS/DataTables setup into a common layout include.

### M3. Two competing access-control models
- **Finding:** Permission rows (`tbl_permissions`) + hard-coded username
  checks coexist (`restriction.php` vs `sidebar.php`/`audit_logs.php`).
- **Suggestion:** Single source of truth (roles + permission tables), one
  `can_access(page)` / `is_admin()` helper used everywhere.

### M4. No error-handling or logging strategy
- **Finding:** Some pages `die()` on DB failure; `session.php` swallows
  exceptions silently; `manage_php.php` turns display_errors on; no
  application-level logger beyond the audit tables.
- **Suggestion:** Central exception handler + file-based error log, JSON
  error responses for endpoints, and structured logging of failures.

### M5. Schema managed by hand, no migrations
- **Finding:** `tbl_transactions.program` is an `enum` (adding a program
  requires an ALTER), `tbl_scholar_info.program` is a different enum, and the
  dev vs. production dumps diverge.
- **Suggestion:** Introduce a migration file per change; avoid `enum` for
  extensible value lists (use lookup tables + FK), or centralize program
  definitions in one config used by both PHP and DB.

### M6. Dead / orphaned code
- **Finding:** `default.php` (Hostinger welcome page); `client_photo.php`
  queries a `client_photos` table with `photo`/`mime_type` that does not
  exist (schema uses `tbl_client_photos.photo_path`); large commented blocks
  in `clients.php`; duplicate register vs. add_user pages.
- **Suggestion:** Audit and remove dead files, or clearly archive them;
  remove commented-out code once replaced.

---

## 🟡 Medium — Data Quality & Integrity

### D1. Referential integrity not enforced on geography fields
- **Finding:** `tbl_clients.city_municipality` and `.barangay` store IDs in
  varchar columns with **no FK constraint**, unlike `tbl_barangays` which
  references `tbl_municipalities` with a real FK.
- **Suggestion:** Add FKs (or validated lookup joins) and index the columns;
  enforce at the app layer until schema FKs are possible.

### D2. Stale denormalized fields
- **Finding:** `full_name`, `match_name`, `patient_name`, and `age` are
  stored and updated manually at various call sites (`add_client.php`,
  `save_grantee_update.php`, `update_client_id.php`). `age` will drift from
  `birthdate` over time.
- **Why it matters:** If one writer forgets to update them, scans and
  duplicate detection silently fail (scanners match `full_name`).
- **Suggestion:** Compute `age` from `birthdate` on read; generate
  `full_name`/`match_name` via a DB trigger or a single centralized helper;
  add a consistency-check report to find drift.

### D3. Duplicate detection relies on fuzzy name matching only
- **Finding:** Duplicates are found by comparing `match_name`/`full_name`
  within municipality/barangay. No birthdate/mobile cross-check.
- **Suggestion:** Include birthdate + mobile number in the scoring to reduce
  false positives/negatives; let operators review before delete (already the
  pattern in `preview_duplicates.php`).

### D4. No soft-delete / audit of deletions
- **Finding:** `delete_client.php`, `delete_transaction.php`,
  `delete_household.php` hard-delete rows; FK cascades remove related data.
- **Suggestion:** Add `deleted_at` / `is_deleted` flags and suppress in
  queries; or ensure deletion is always audit-logged with the full snapshot.

### D5. Static `enum` program lists
- **Finding:** Adding a program (e.g. a new scholarship) requires a schema
  change in `tbl_transactions` and `tbl_scholar_info` plus edits in
  `manage_permissions.php`, `scanned_payouts.php`, and scanner whitelists.
- **Suggestion:** Centralize program definitions (lookup table or a single
  PHP config) and generate dropdowns from it.

### D6. Multiple payout-scan tables
- **Finding:** `tbl_payout_scans`, `tbl_payout_scans2`,
  `tbl_payout_scans_unpaid` are identical shapes.
- **Suggestion:** One table with a `channel`/`type` column; keeps attendance
  queryable in a single place.

### D7. Staging/reference tables without owners
- **Finding:** `gender`, `tbl_absent`, `tbl_kababaihan`, `tbl_details`,
  `temp_details` are import/compare leftovers with no FK relationships and
  some non-idiomatic columns (`Count` on `tbl_absent`).
- **Suggestion:** Document their purpose or drop them; rename reserved-word
  columns; restrict app access to these tables.

---

## 🟢 Low — UX, Performance & Ops

### O1. Session & cookie hardening is mostly done but inconsistent
- **Finding:** Good practices are present: `httponly`, `SameSite=Lax`,
  token-enforced single device. Inconsistency: `check_session.php` starts a
  session directly (skipping `session.php`) so cookie params differ between
  the poll endpoint and regular pages.
- **Suggestion:** Route `check_session.php` through the same session
  bootstrap; enable `secure` cookies once HTTPS is guaranteed.

### O2. Local vs. production config drift
- **Finding:** The dump uses `utf8mb4_uca1400_ai_ci` (MariaDB 10.6+ /
  MySQL 8) which local MariaDB 10.4 rejects — schema/engine version drift
  between environments.
- **Suggestion:** Standardize collations; add a small install/migration
  check that verifies DB version and collations before running.

### O3. Environment / backup / ops
- **Finding:** Production PHP is 7.2.34 (EOL); no visible automated backup,
  health check, or deployment pipeline.
- **Suggestion:** Upgrade PHP; automate DB dumps (host-side cron + local
  copies); keep `seal_logo.png`/uploads in backups; add a smoke test after
  each deploy (login + one scan lookup).

### O4. Data-entry robustness
- **Finding:** `age` is taken from the form (trusted input); `mobile_no` is a
  loose varchar; no birthdate-format validation beyond the date input.
- **Suggestion:** Compute age server-side from birthdate; validate/normalize
  phone and email; enforce name/date formats with reusable validators.

### O5. Missing user-facing conveniences
- **Finding:** No CSV export for the clients list (transactions and unpaid
  verifications already have export); no unified global search; no
  password-reset flow for regular users (only super_admin resets via
  `manage_php.php`).
- **Suggestion:** Add client export; add a search box that queries clients,
  transactions, scholars, and households; add self-service password reset
  backed by the existing `password_resets` table.

---

## Suggested priority order

1. **C1–C4** — rotate credentials, add CSRF + rate limiting, unify access
   control and remove username hard-coding.
2. **M1–M4** — collapse the 16 scanners into one parameterized flow;
   centralize auth/permission helpers; add logging.
3. **D1–D4** — enforce geography FKs, stop age/full-name drift, strengthen
   duplicate detection, add soft delete.
4. **M5/M6, D5–D7** — migrations, remove dead code, centralize program lists.
5. **O1–O5** — session hardening, environment checks, ops/backups, UX polish.

> All recommendations are design-level. Implementing any of them should be a
> separate, reviewed change.
