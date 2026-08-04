# 2D MIS — Combined Documentation

> Consolidated single-file reference. Merges the contents of all
> `doc/*.md` documents except `README.md`:
> `SYSTEM_OVERVIEW.md` + `DATABASE_SCHEMA.md` + `FILE_REFERENCE.md` +
> `WORKFLOWS.md` + `RECOMMENDATIONS.md`.
>
> *Automatically generated from the existing documentation files; no source
> code, SQL, or database was modified.*

---

# PART 1 — System Overview

## 1. What the system is

**2D MIS** is a browser-based Management Information System used by a
municipal government to manage social-assistance programs. It keeps a master
registry of people (`tbl_clients`), organizes them into households, records
assistance transactions per program, tracks scholarship grantees, and — most
distinctively — uses **QR codes + webcam scanning** to record attendance and
payouts at on-site events.

It was originally hosted on Hostinger (`u749085076_main_system`) and has been
mirrored locally under XAMPP.

## 2. Technology stack

| Layer | Technology |
|---|---|
| Language | PHP (procedural, no framework, no Composer) |
| Database | MariaDB / MySQL accessed with PDO (prepared statements) |
| Front-end | HTML + Bootstrap 5, jQuery 3.7, DataTables 1.13 |
| QR scanning | `html5-qrcode` library (camera-based, in-browser) |
| QR generation | `api.qrserver.com` (see `view_qrcode.php`) |
| Assets | local `sidebar.css`, `sidebar.js`, `seal_logo.png`, `sounds/*` |

PHP 7.2+ compatible. The local MariaDB is 10.4.32.

## 3. Directory / file layout

```
C:\xampp\htdocs\system\
├── db_connect.php          # PDO connection (single place)
├── session.php             # session lifecycle + single-device enforcement
├── restriction.php         # per-page access control
├── navbar.php, sidebar.php # shared chrome (sidebar.js/css)
├── logs.php                # log_action() helper -> tbl_audit_logs
├── index.php, login.php, logout.php, register.php
├── *.php                   # ~110 pages/endpoints (see FILE_REFERENCE.md)
├── u749085076_main_system.sql   # current schema (31 tables)
├── main_system.sql             # older dev snapshot (7 tables)
├── cache/  uploads/  sounds/  # runtime + media folders
└── doc/                     # this documentation
```

## 4. Request flow (typical page)

1. The browser requests a page, e.g. `clients.php`.
2. The page includes `session.php` → `db_connect.php` → `restriction.php`.
3. `session.php` enforces login, session expiry, and single-device token
   checks; non-admin mismatches redirect to `login.php?session=expired`.
4. `restriction.php` looks up `tbl_permissions` for the current user + page;
   if absent, it alerts "Access Denied" and redirects to `index.php`
   (user id 1 always passes).
5. The page renders an HTML shell (Bootstrap) and its server-side DataTables
   table loads rows via a `fetch_*.php` AJAX endpoint (`POST`).
6. Forms and scan flows post to `*_action.php` / `save_*.php` handlers which
   return JSON; the page's JavaScript updates the UI.

## 5. Authentication

- `login.php` validates with `password_verify()` against `tbl_users.password`
  (bcrypt hashes), then:
  - stores `user_id`, `username`, `session_token` in `$_SESSION`;
  - writes a fresh random `session_token` (64 hex chars) to `tbl_users`.
- `logout.php` nulls the token and destroys the session.
- `register.php` / `add_user.php` create users with `password_hash()`.

## 6. Session control (single-device / forced logout)

`session.php` implements a "one active device" policy:

- **Exempt users** — usernames `super_admin` and `jordi` get a ~10-year
  cookie lifetime and skip token checks.
- **DB-exempt users** — users present in `tbl_multi_device_exemptions` skip
  token checks (managed via `manage_multi_device_exemptions.php`).
- **Everyone else** — every request compares `$_SESSION['session_token']`
  against `tbl_users.session_token`. If they differ (someone else logged in
  as the same user, or an admin force-logged-out), the session is destroyed.
- `force_logout.php` (admin) sets `session_token = NULL` for a target user,
  killing all their sessions.
- `navbar.php` polls `check_session.php` every 2 seconds; the client-side
  JSON statuses are `ok`, `logged_out`, or `another_device`.

## 7. Access control model

Two mechanisms coexist:

1. **Per-page permissions** — `tbl_permissions(user_id, page_name, can_access)`
   enforced by `restriction.php`. `manage_permissions.php` rebuilds the rows
   for a user. User id 1 is always allowed.
2. **Username-based gating** — several pages/menus hard-code usernames:
   - `sidebar.php` shows admin links only to `super_admin`, `god_admin`, `jordi`.
   - `audit_logs.php` allows only `super_admin`, `god_admin`, `jordi`.
   - `clients.php` shows the "Remove Duplicates" button to `super_admin`/`jordi`.

   This mix (DB-permission vs. hard-coded username) is intentional to document —
   it is a maintenance risk if usernames change.

## 8. Auditing

- `logs.php` exposes `log_action($pdo, $user_id, $action, $target_table,
  $target_id, $old_value, $new_value)` → `tbl_audit_logs`.
- Used for add/edit/delete of clients, transactions, and every QR "save".
- `tbl_update_logs` tracks grantee-profile updates (`save_grantee_update.php`).
- `tbl_photo_logs` tracks client-photo changes.

## 9. Module map

```
Login / Users / Admin ....... login, logout, register, add_user, manage_php,
                             manage_permissions, manage_program_permissions,
                             manage_multi_device_exemptions, currently_logged_users,
                             audit_logs
Clients ..................... clients, add/edit/view/delete_client, search_clients,
                             get_client_hh, preview_duplicates, client photos
Households .................. household, add/view/delete_household, add_family_member,
                             search_households, fetch_households, get_household
Transactions ................ all_transactions, add/edit/view/delete_transaction,
                             update_transaction, fetch_transactions
Scholars .................... scholars, save_scholarship, fetch_scholars,
                             scholarship_reports + export, save_gip, update_client_id,
                             view_qrcode
Scanners (16) ............... scanner_*.php + scanner_*_action.php
Payout attendance ........... scanned_payouts, scanned_payouts2, scanned_payouts_unpaid
Unpaid grantees ............. unpaid_verifications, unpaid_save, disabled_unpaid,
                             search_grantee/search_unpaid_grantee, export
```
See [FILE_REFERENCE.md](FILE_REFERENCE.md) for the complete file-by-file map.

## 10. Known observations (read-only analysis)

- `db_connect.php` credentials: currently set to the **local** XAMPP database
  (`main_system` / `root` / empty password). The original file pointed at the
  Hostinger remote database (`u749085076_main_system` / `&lt;redacted&gt;`).
- `default.php` is the Hostinger welcome page left in the document root — it is
  not part of the application.
- `client_photo.php` queries a `client_photos` table with `photo` / `mime_type`
  columns that do not exist in the current schema (`tbl_client_photos` uses
  `photo_path`). It appears to be dead/orphaned legacy code.
- `transaction_table.php` is a shared renderer that expects `$aics_transactions`
  to be set by the including page.
- Multiple scanners share near-identical logic; the payout scanner also matches
  against `tbl_seats2` (seat assignments) and allows a `lookup_ignore_scan` mode.
- CSV exports emit an Excel-friendly UTF-8 BOM.

---

# PART 2 — Database Schema

> Derived from `u749085076_main_system.sql` (phpMyAdmin dump, MariaDB 11.8.8)
> and the older dev dump `main_system.sql`. 31 tables total.

## Entity relationships (high level)

```
tbl_municipalities 1 ── n tbl_barangays
tbl_clients n ── 1 tbl_household (household_id; head_household fk back to clients)
tbl_clients 1 ── n tbl_client_aff_orgs
tbl_clients 1 ── n tbl_client_photos
tbl_clients 1 ── n tbl_family_members (client_id / relative_id)
tbl_clients 1 ── n tbl_transactions
tbl_clients 1 ── n tbl_scholar_info
tbl_clients 1 ── n tbl_gip_info
tbl_clients 1 ── n tbl_unpaid_verifications
tbl_transactions 1 ── 1 tbl_payout_scans / tbl_payout_scans2 / tbl_payout_scans_unpaid
tbl_users 1 ── n tbl_permissions / tbl_program_permissions / tbl_multi_device_exemptions / tbl_audit_logs
tbl_exam 1 ── n tbl_results (exam_no)
```

## Core registry tables

### `tbl_clients` — master person registry
| Column | Type | Notes |
|---|---|---|
| id | int(11) PK | |
| family_id | int(11) | legacy |
| household_id | int(11) | FK → `tbl_household.id` (ON DELETE SET NULL) |
| lastname / firstname | varchar(100) | |
| middlename | varchar(100) | |
| extensionname | varchar(20) | e.g. JR |
| region | varchar(100) | default `Region I` |
| province | varchar(100) | default `Ilocos Sur` |
| city_municipality | varchar(100) | stores municipality **id** |
| barangay | varchar(100) | stores barangay **id** |
| house_no | varchar(50) | |
| mobile_no | varchar(15) | |
| email | varchar(255) | |
| birthdate | date | |
| age | int(11) | |
| sex | enum(MALE,FEMALE) | |
| civil_status | enum(SINGLE,MARRIED,WIDOWED) | |
| pwd | enum(YES,NO) | persons with disability |
| ip | enum(YES,NO) | Indigenous People |
| ip_group | varchar(255) | |
| occupation | varchar(100) | |
| monthly_income | decimal(10,2) | |
| category | enum(MINOR (0-17),YOUTH (18-29),ADULT (30-59),SENIOR CITIZEN (60 AND ABOVE)) | |
| aff_org | varchar(255) | |
| precinct_no / voter_id | varchar(50) | |
| created_at | timestamp | |
| full_name | varchar(255) | denormalized `LASTNAME, FIRSTNAME MIDDLE` (set at insert) |
| match_name | varchar(255) | duplicate-matching helper |
| *Indexes* | | fulltext on (lastname,firstname,middlename); composite name+location indexes |

### `tbl_household` — household groups
| Column | Type | Notes |
|---|---|---|
| id | int(11) PK | |
| household_id | varchar(20) UNIQUE | human-readable code |
| head_household | int(11) | FK → `tbl_clients.id` (the head) |
| created_at | timestamp | |

### `tbl_family_members` — client-to-relative links
`id` PK, `client_id` FK, `relative_id` FK, `relationship` varchar(50).
Unique on `(client_id, relative_id)`.

### `tbl_municipalities` / `tbl_barangays` — geography reference data
- `tbl_municipalities`: `id`, `name`, `code`.
- `tbl_barangays`: `id`, `municipality_id` FK → municipalities (CASCADE), `name`.

## Program / assistance tables

### `tbl_transactions` — the central assistance record
| Column | Type | Notes |
|---|---|---|
| id | int(11) PK | |
| client_id | int(11) | FK → `tbl_clients.id` |
| program | enum | AICS, AKAP, MAIP, TUPAD, CEDSSG, CEAP, CEAP_NEW, OTCES, OTEA, CEDSSG_NEW, COFFEE GROWERS, PUSO TI KABABAIHAN, PUSO TI AGTUTUBO, PUSO TI MANNALON, TESDA, GIP, TODA |
| patient_name | varchar(255) | denormalized beneficiary name |
| date_applied | date | |
| type | varchar(255) | e.g. MEDICAL, BURIAL, SCHOLARSHIP, FOOD SUBSIDY |
| remarks | text | used by scanners for semester/session keys (e.g. `1ST SEM SY2025-2026 DOCS SUBMITTED`) |
| comments | text | |
| suggested_amount | decimal(10,2) | |
| status | varchar(50) | e.g. `PENDING PAYOUT`, `PAID` |
| amount_paid | decimal(10,2) | |
| payout_date | date | |
| date_paid | date | |
| gwa / units | varchar(255) | scholarship grade data |
| created_at | timestamp | |
| *Indexes* | | program, client_id, date_applied, payout_date, date_paid |

### `tbl_payout_scans` / `tbl_payout_scans2` / `tbl_payout_scans_unpaid` — payout attendance
- `id` PK, `transaction_id` FK → `tbl_transactions.id` (**UNIQUE** — one scan per transaction), `scanned_text`, `scanned_by` FK → `tbl_users.id`, `scanned_at`.
- The three tables correspond to three separate payout-attendance screens.

### `tbl_scholar_info` — scholarship enrollment
`id`, `client_id` FK (CASCADE), `full_name`, `program`
enum(CEDSSG, CEAP, CEDSSG_NEW, CEAP_NEW, OTEA, OTCES), `school`, `school_type`,
`campus`, `college_department`, `course`, `year_level`, `is_regular`, `year_started`,
`landbank_no`, `created_at`, `updated_at`, `normalized_name`
(generated `lcase(trim(full_name))`), `match_name`.

### `tbl_gip_info` — GIP program applicant info
`id`, `client_id` FK, `full_name`, `valid_govt_id`, `id_number`,
`insurance_beneficiary`, `emergency_contact`, `ecp_contact_number`, `ecp_address`,
`college`, `course`, `year_graduated`, `high_school`, `elementary_school`,
`latest_work_experience`, `position`, `period_of_engagement`, `special_skills`,
`achievements`, `created_at`, `updated_at`, `normalized_name`, `match_name`.

### `tbl_unpaid_verifications` — grantees who could not be paid
`id`, `client_id` FK, `municipality_id` FK, `is_proxy` tinyint, proxy identity
fields (`proxy_lastname/firstname/middlename`, `proxy_relationship`,
`proxy_phone`, `proxy_birthdate`, `proxy_gender`, `proxy_occupation`,
`proxy_monthlyincome`), `created_at`.

## Exam / scholarship-gate tables

### `tbl_exam` — exam registration
`id`, `exam_no`, `fullname`, `barangay`, `town`, `email_address`, `contact`,
`school`, `course`, `year`, `scholarship`, `exam_date`, `exam_time`,
`permit_confirmed`, `score`, `normalized_name` (generated).

### `tbl_results` — exam results
`id`, `exam_no`, `score`, `approved`. Used by
`scanner_new_scholars_action.php` to auto-derive the program.

### `tbl_seats` / `tbl_seats2` — event seating assignments
`id`, `program`, `name`, `town`, `section`, `box`, `row`, `seat`.
Payout scanner joins `tbl_seats2.name` against `tbl_clients.full_name` to
return seat info during a scan.

### `tbl_absent`, `tbl_kababaihan`, `gender`, `tbl_details`, `temp_details` — reference/import tables
Standalone data tables (no FKs) used for import/compare of name lists and
reports. `temp_details` is a staging table for CSV-style imports.

## Users & permissions

### `tbl_users`
`id` PK, `username` UNIQUE, `password` (bcrypt hash), `created_at`,
`last_activity` (drives "currently logged in"), `session_token`
(single-device enforcement).

### `tbl_permissions`
`id` PK, `user_id`, `page_name` (e.g. `scanner_ceap.php`), `can_access` tinyint.

### `tbl_program_permissions`
`id` PK, `user_id`, `program_name`. Restricts which programs a user can
process (checked by `fetch_transactions.php`, `add_transaction.php`,
`edit_transaction.php`).

### `tbl_multi_device_exemptions`
`id` PK, `user_id` UNIQUE, `created_at`. Exempt users skip the token check.

## Logs

### `tbl_audit_logs`
`id` PK, `user_id`, `action` (e.g. `ADD_CLIENT`, `SCAN-CEAP`), `target_table`,
`target_id`, `old_value` text, `new_value` text, `created_at`.

### `tbl_update_logs`
`id` PK, `client_id`, `full_name`, `ip_address`, `action`, `created_at`.

### `tbl_photo_logs`
`id` PK, `client_id` FK, `action`, `remarks`, `updated_by`, `log_date`,
`before_photo`, `after_photo`, `program`, `ip_address`.

### `password_resets`
`id` PK, `changed_by`, `changed_for`, `changed_at`.

## Data tables (per client module)

### `tbl_client_aff_orgs`
`id` PK, `client_id` FK (CASCADE), `organization`.

### `tbl_client_photos`
`id` PK, `client_id` FK (CASCADE), `photo_path`, `captured_from`
enum(UPLOAD, CAMERA), `created_at`.

> Note: legacy `client_photo.php` reads an old `client_photos` table
> (`photo`, `mime_type`) that does not exist in this schema.

## Older dev schema (`main_system.sql`) — for reference
The dev dump contains `tbl_aics` (AICS assistance records), a simpler
`tbl_clients` (no household_id, pwd, ip, category, aff_org, email, full_name),
`tbl_users`, `tbl_municipalities`, `tbl_barangays`, `tbl_family_members`.
This file is superseded by `u749085076_main_system.sql`.

---

# PART 3 — File Reference

Complete catalog of the `C:\xampp\htdocs\system` application files, grouped by
module. *(Inventory as of the time of writing; ~115 PHP files.)*

## Shared foundation (included by nearly every page)

| File | Purpose |
|---|---|
| `db_connect.php` | PDO connection (`$pdo`); currently local `main_system`. |
| `session.php` | Session bootstrap, lifetime rules, single-device token enforcement, `last_activity` heartbeat. |
| `restriction.php` | Per-page permission check against `tbl_permissions` (user id 1 always allowed). |
| `logs.php` | `log_action()` helper → `tbl_audit_logs`. |
| `navbar.php` | Top bar; polls `check_session.php` every 2 s for forced-logout detection. |
| `sidebar.php` | Left navigation; admin links gated by username. |
| `sidebar.css` / `sidebar.js` | Sidebar styling + toggle. |
| `favicon.php` | Emits the `seal_logo.png` favicon link. |
| `check_session.php` | JSON endpoint: `ok` / `logged_out` / `another_device`. |

## Auth & users

| File | Purpose |
|---|---|
| `login.php` | Login form; bcrypt verify; creates session + token. |
| `logout.php` | Nulls token, destroys session. |
| `register.php` | Admin "Create User" form (`password_hash`). |
| `add_user.php` | Create User page. |
| `manage_php.php` | User management (list users). |
| `force_logout.php` | Admin action: sets target user's `session_token = NULL`. |
| `currently_logged_users.php` | Online-users screen. |
| `fetch_online_users.php` | AJAX feed of users (last activity). |
| `verify_mobile.php` | Client mobile verification. |

## Dashboard & shell

| File | Purpose |
|---|---|
| `index.php` | Dashboard with links to all 13 QR scanners. |
| `default.php` | Hostinger welcome page — **leftover, not part of the app**. |

## Clients module

| File | Purpose |
|---|---|
| `clients.php` | Client list (DataTables server-side) with municipality/barangay filters + duplicate-removal link. |
| `add_client.php` | Create client; writes `full_name`/`match_name`; stores aff. orgs; audit log. |
| `edit_client.php` | Edit client incl. household, aff. orgs. |
| `view_client.php` | Client profile: photos, family, household, transactions, scholar, GIP. |
| `delete_client.php` | Delete a client. |
| `fetch_clients.php` | AJAX feed for the clients table. |
| `search_clients.php` | Name search (also joins family members). |
| `search_clients_hh.php` | Name search for household assignment. |
| `get_client_hh.php` | Returns client + aff. orgs for household dropdown. |
| `preview_duplicates.php` | Duplicate preview screen. |
| `fetch_duplicates.php` | Duplicate candidates feed. |
| `delete_duplicates.php` | Bulk-delete confirmed duplicates. |
| `client_photo.php` | **Legacy/dead** photo stream (queries non-existent `client_photos`). |
| `save_client_photo.php` | Upload client photo → `tbl_client_photos` (+ photo log). |
| `student_photo_upload.php` / `student_update_photo.php` | Student-facing photo update screens. |
| `student_verify.php` | Identity verification by name/birthdate. |

## Households module

| File | Purpose |
|---|---|
| `household.php` | Household list with filters. |
| `add_household.php` | Create household (select head). |
| `view_household.php` | Household detail + members. |
| `delete_household.php` | Delete household. |
| `add_family_member.php` | Link a client as relative to another client. |
| `fetch_households.php` | Household table feed. |
| `get_household.php` | Single household lookup. |
| `search_households.php` | Household search. |

## Transactions module

| File | Purpose |
|---|---|
| `all_transactions.php` | Full transaction list: program/date/municipality/status filters + CSV export. |
| `add_transaction.php` | New transaction (programs gated by `tbl_program_permissions`). |
| `edit_transaction.php` | Edit a transaction. |
| `view_transaction.php` | Transaction detail. |
| `delete_transaction.php` | Delete a transaction. |
| `update_transaction.php` | Save edited transaction. |
| `all_transaction_edit.php` | Edit flow used from the All Transactions list. |
| `all_transaction_delete.php` | Delete flow from the list. |
| `fetch_transactions.php` | AJAX feed (respects program permissions). |
| `transaction_table.php` | Shared renderer expecting `$aics_transactions` + `$client`. |

## Scholars module

| File | Purpose |
|---|---|
| `scholars.php` | Manage scholar records (server-side DataTable). |
| `save_scholarship.php` | Insert/update `tbl_scholar_info`. |
| `fetch_scholars.php` | Scholar table feed. |
| `update_client_id.php` | Re-link a scholar record to a different client id. |
| `scholarship_reports.php` | Scholarship report screen. |
| `fetch_scholarship_reports.php` | Report table feed (transactions + scholar_info). |
| `export_scholarship_reports.php` | Report CSV export. |
| `save_gip.php` | Save GIP applicant info (`tbl_gip_info`). |
| `save_grantee_update.php` | Grantee profile update; writes `tbl_update_logs`; touches clients/transactions/scholar_info. |
| `disabled_update_grantee.php` | "Scholarship Grantee Update" page. |
| `update_logs.php` | Grantee update logs screen. |
| `fetch_update_logs.php` | Update-logs feed. |
| `view_qrcode.php` | Scholar QR viewer (generates QR via `api.qrserver.com`). |

## QR Scanners (webcam)

Each scanner page (`.php`) = camera UI; matching `scanner_*_action.php` =
JSON handler (actions `lookup` / `save`).

| Scanner page | Action | Program / behavior |
|---|---|---|
| `scanner_ceap.php` | `scanner_ceap_action.php` | CEAP; inserts scholarship transaction (`1ST SEM SY2025-2026 DOCS SUBMITTED`, 5000); duplicate check by remark. |
| `scanner_ceap_new.php` | `scanner_ceap_new_action.php` | CEAP_NEW; same pattern. |
| `scanner_cedssg.php` | `scanner_cedssg_action.php` | CEDSSG. |
| `scanner_cedssg_new.php` | `scanner_cedssg_new_action.php` | CEDSSG_NEW. |
| `scanner_cedssg_update.php` | `scanner_cedssg_update_action.php` | Updates an existing CEDSSG transaction. |
| `scanner_tupad.php` | `scanner_tupad_action.php` | TUPAD; duplicate guard per month. |
| `scanner_toda.php` | `scanner_toda_action.php` | TODA. |
| `scanner_otces.php` | `scanner_otces_action.php` | OTCES. |
| `scanner_otea.php` | `scanner_otea_action.php` | OTEA. |
| `scanner_new_scholars.php` | `scanner_new_scholars_action.php` | Auto program from `tbl_results.approved`. |
| `scanner_ongoing_scholars.php` | `scanner_ongoing_scholars_action.php` | Validates against an existing scholar transaction. |
| `scanner_payout.php` | `scanner_payout_action.php` | Payout attendance → `tbl_payout_scans2`; joins `tbl_seats2` for seat info; `lookup_ignore_scan` mode. |
| `scanner_payout_unpaid.php` | `scanner_payout_unpaid_action.php` | Unpaid-payout attendance → `tbl_payout_scans_unpaid`. |
| `scanner_generic.php` | `scanner_generic_action.php` | Generic scanner; program chosen in form → `tbl_transactions`. |

## Payout attendance

| File | Purpose |
|---|---|
| `scanned_payouts.php` | Payout attendance 1 (`tbl_payout_scans`). |
| `scanned_payouts2.php` | Payout attendance 2 (`tbl_payout_scans2`). |
| `scanned_payouts_unpaid.php` | Unpaid attendance (`tbl_payout_scans_unpaid`). |
| `fetch_scanned_payouts.php` | Feed for attendance 1 (+ `tbl_seats`). |
| `fetch_scanned_payouts2.php` | Feed for attendance 2 (+ `tbl_seats2`). |
| `fetch_scanned_payouts_unpaid.php` | Feed for unpaid attendance. |

## Unpaid grantees / verification

| File | Purpose |
|---|---|
| `unpaid_verifications.php` | Unpaid grantees screen. |
| `disabled_unpaid.php` | "Unpaid Verification" page. |
| `unpaid_save.php` | Save verification (incl. proxy data) → `tbl_unpaid_verifications`. |
| `fetch_unpaid_verifications.php` | Feed. |
| `export_unpaid_verifications.php` | CSV export. |
| `search_grantee.php` | Grantee search (municipalities + clients + transactions + scholar). |
| `search_unpaid_grantee.php` | Same, for unpaid flow. |

## Admin / management

| File | Purpose |
|---|---|
| `manage_permissions.php` | Rebuild per-user page access in `tbl_permissions`. |
| `manage_program_permissions.php` | Per-user program access in `tbl_program_permissions`. |
| `manage_multi_device_exemptions.php` | Manage users exempt from single-device rule. |
| `audit_logs.php` | Activity logs (username-gated) + leaderboard. |
| `fetch_logs.php` | Audit-log feed. |
| `fetch_leaderboard.php` | Leaderboard feed (top users by actions). |

## Location helper

| File | Purpose |
|---|---|
| `get_barangays.php` | JSON list of barangays for a `municipality_id` (cascading dropdowns). |

## Data & assets (not code)

| Path | Purpose |
|---|---|
| `u749085076_main_system.sql` | Current schema dump (31 tables). |
| `main_system.sql` | Older dev dump (7 tables). |
| `seal_logo.png` | Logo / favicon. |
| `sounds/` | `success.mp3`, `not_found.mp3` for scanner feedback. |
| `uploads/` | `client_photos/`, `profile_photos/` storage folders. |
| `cache/` | Runtime cache folder. |

---

# PART 4 — End-to-End Workflows

## 1. User logs in

1. `login.php` posts `username` / `password`.
2. `session.php` starts a session with a 1-day cookie (10 years for
   `super_admin` / `jordi`).
3. `password_verify()` matches the bcrypt hash in `tbl_users`.
4. A fresh 64-hex `session_token` is generated, stored in the session **and**
   written to `tbl_users.session_token`.
5. Redirect to `index.php` (dashboard with 13 scanner shortcuts).

**Enforcement loop:** on every request, `session.php` compares the session
token with the DB token for non-exempt users. `navbar.php` polls
`check_session.php` every 2 s; an "another_device" response triggers a client-side
alert + redirect to `login.php`.

## 2. Admin creates a user

- `register.php` / `add_user.php` → `password_hash()` → `tbl_users`.
- `manage_permissions.php` selects a user and checks page checkboxes →
  deletes + reinserts `tbl_permissions` rows for that user.
- `manage_program_permissions.php` does the same for `tbl_program_permissions`
  (which programs the user may process).
- `manage_multi_device_exemptions.php` can add the user to the exemption table.

## 3. Encode a client

1. `add_client.php` (or `add_household.php` first to create a household).
2. Fields are uppercased; municipality/barangay stored as IDs
   (dropdowns fed by `get_barangays.php`).
3. Insert into `tbl_clients`; then update `full_name = "LASTNAME, FIRSTNAME MIDDLE"`
   and `match_name`.
4. Optional affiliated organizations → `tbl_client_aff_orgs`.
5. `log_action(..., 'ADD_CLIENT', ...)` writes to `tbl_audit_logs`.
6. Photo may be attached via `save_client_photo.php` → `tbl_client_photos`
   (+ `tbl_photo_logs`).
7. `view_client.php` renders the full profile.

## 4. Duplicate handling

- `clients.php` (admin button) → `preview_duplicates.php` → `fetch_duplicates.php`
  compares `match_name`/`full_name` across municipality/barangay.
- `delete_duplicates.php` removes confirmed duplicate rows.

## 5. QR scanning — scholarship intake (e.g. CEAP)

1. Staff open `scanner_ceap.php`; camera scans the student's QR.
2. Page POSTs `action=lookup&scanned=<text>` to `scanner_ceap_action.php`.
3. Handler matches the scanned text against `tbl_clients.full_name`
   (case-insensitive, trimmed); returns `{success, data:{id, full_name}}`.
4. Staff clicks **Save** → POST `action=save&id=<client_id>`.
5. Handler:
   - rejects if a CEAP transaction with remark `1ST SEM SY2025-2026 DOCS SUBMITTED`
     already exists for that client (duplicate guard);
   - inserts `tbl_transactions` (program `CEAP`, amount 5000, status
     `PENDING PAYOUT`);
   - `log_action(..., 'SCAN-CEAP', ...)`.
6. Success/error modal + beep sound (`sounds/success.mp3`,
   `sounds/not_found.mp3`).

Other program scanners follow the same pattern with their own program constant
and duplicate checks (TUPAD guards by month; `scanner_new_scholars_action.php`
reads the program from `tbl_results.approved`; `scanner_ongoing_scholars_action.php`
validates the person has an existing scholar transaction; `scanner_cedssg_update_action.php`
updates an existing CEDSSG record).

## 6. QR scanning — payout attendance

1. `scanner_payout.php` → `scanner_payout_action.php`.
2. `lookup`: scans text, normalizes spaces, then matches
   `tbl_seats2.name` ↔ `tbl_clients.full_name` (exact, then partial LIKE),
   joined to `tbl_transactions` (programs in a whitelist). Returns
   transaction + seat/town/box/row/seating info.
3. `lookup_ignore_scan`: same lookup but skips the "already scanned" check.
4. `save`: inserts into `tbl_payout_scans2` (unique on `transaction_id`),
   so each transaction can be scanned **once**; duplicate scan returns an error.
5. `scanned_payouts*.php` list the recorded attendance; `tbl_seats`/`tbl_seats2`
   back the seat layouts.

## 7. Unpaid grantees

- `unpaid_verifications.php` lists grantees whose payout is still pending.
- `unpaid_save.php` records the verification; if a proxy received the payout,
  proxy details are stored (name, relationship, phone, birthdate, gender,
  occupation, income) in `tbl_unpaid_verifications`.
- `export_unpaid_verifications.php` produces a CSV report.
- `scanner_payout_unpaid.php` records attendance for these unpaid grantees in
  `tbl_payout_scans_unpaid`.

## 8. Scholarship reports

- `scholarship_reports.php` → `fetch_scholarship_reports.php` joins
  `tbl_transactions` + `tbl_scholar_info`.
- `export_scholarship_reports.php` streams a CSV with UTF-8 BOM for Excel.
- `scholars.php` → `fetch_scholars.php` manages `tbl_scholar_info`
  (server-side DataTable, edit client-id inline).

## 9. GIP applicants

- `save_gip.php` stores GIP application details (IDs, emergency contact,
  education history, work experience, skills) in `tbl_gip_info`.

## 10. Admin monitoring

- `audit_logs.php` (username-gated to super_admin/god_admin/jordi) shows
  `tbl_audit_logs`; `fetch_leaderboard.php` ranks users by number of actions.
- `currently_logged_users.php` + `fetch_online_users.php` use
  `tbl_users.last_activity`.
- `force_logout.php` nulls a user's token to kick them out everywhere.

---

# PART 5 — Recommendations & Improvement Plan

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

---

# PART 6 — System Design

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
modest data volume. Evidence references point to the design sections above
and to the recommendations (Part 5).

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
baseline are established before any code is rewritten (see the v2 migration
plan).

---

*End of combined documentation.*
