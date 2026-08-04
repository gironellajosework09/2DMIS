# File Reference

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
