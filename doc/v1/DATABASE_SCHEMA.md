# Database Schema — `main_system`

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
