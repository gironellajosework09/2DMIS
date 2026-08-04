# System Overview

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
