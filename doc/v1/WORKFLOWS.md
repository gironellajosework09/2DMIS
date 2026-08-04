# End-to-End Workflows

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
