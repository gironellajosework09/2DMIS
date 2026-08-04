# v2 — Requirements Analysis

Functional and non-functional requirements for v2.0, derived from the v1
analysis (`../v1/SYSTEM_OVERVIEW.md`, `../v1/SYSTEM_DESIGN.md`). Each
requirement is traceable to a v1 behavior that must be preserved or replaced.

## 1. Functional requirements

### FR-1 Authentication & sessions
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-1.1 | Log in with username + password (bcrypt verified) | `login.php`, `tbl_users` |
| FR-1.2 | Enforce single-device sessions via a session token; re-login invalidates other devices | `session.php`, `tbl_users.session_token` |
| FR-1.3 | Admin can force-logout a user remotely | `restriction.php` token check |
| FR-1.4 | Keep a configurable idle timeout (1 day normal; 10 years for exempt users) | `session.php` |
| FR-1.5 | User can change password; changes must be audited | `password_resets`, `change_password.php` |

### FR-2 Access control
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-2.1 | One unified permission model: users → permissions → pages/programs | `tbl_permissions`, `tbl_program_permissions`, `restriction.php` |
| FR-2.2 | Remove username-based gating; grant admin rights through permissions only | `sidebar.php`, `audit_logs.php`, `manage_php.php`, `clients.php` (v1 anti-pattern A2) |
| FR-2.3 | Multi-device exemption for select accounts, still centrally manageable | `tbl_multi_device_exemptions` |

### FR-3 Client registry
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-3.1 | Add a client with ID type/fields, contact, demographics, address (municipality/barangay), program category | `add_client.php` |
| FR-3.2 | Derive `full_name` / `match_name` in one central code path (no manual sync) | `tbl_clients`, v1 anti-pattern A6 |
| FR-3.3 | View a client profile aggregating photos, family, household, transactions, scholarship, GIP | `view_client.php` |
| FR-3.4 | Search, filter, and server-side paginate the client list | `fetch_clients.php` |
| FR-3.5 | Detect potential duplicates (name + municipality/barangay) and manage merges/deletes with operator review | `fetch_duplicates.php` |

### FR-4 Households & family
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-4.1 | Create households (code + head), assign clients as members | `tbl_household`, `household.php` |
| FR-4.2 | Record family member relationships (kinship edges) | `tbl_family_members` |

### FR-5 Transactions & assistance programs
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-5.1 | Record one transaction per assistance event: program, amount, status (PENDING PAYOUT / PAID), dates | `tbl_transactions`, `all_transactions.php` |
| FR-5.2 | Filter/search/export transactions by program, municipality, barangay, date range, status | `all_transactions.php` |
| FR-5.3 | Keep the existing set of programs as data (config-driven), not scattered arrays | v1 anti-pattern A7 |
| FR-5.4 | Support program-specific duplicate rules (fixed remark key, monthly guard, exam-derived program, etc.) | `scanner_*_action.php` variants |

### FR-6 QR scanning & payout attendance
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-6.1 | One scanner engine; a program is a config entry, not a copied file | v1 anti-pattern A1 (16 copies) |
| FR-6.2 | Camera scan with `html5-qrcode`, lookup by name, then guarded save | `scanner_*.php`, `scanner_*_action.php` |
| FR-6.3 | Audio/visual feedback (success / not found) | `sounds/*.mp3` |
| FR-6.4 | One attendance row per transaction per payout event (DB unique = belt, app check = suspenders) | `tbl_payout_scans`, `_2`, `_unpaid` |
| FR-6.5 | Support seat-aware payout scanning | `tbl_seats2`, `scanner_payout_action.php` |
| FR-6.6 | Capture unpaid-grantee verification incl. optional proxy receiver | `tbl_unpaid_verifications` |

### FR-7 Scholarship management
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-7.1 | Manage exam/results (admission gate), scholar enrollment info, and GIP applicants | `tbl_exam`, `tbl_results`, `tbl_scholar_info`, `tbl_gip_info` |
| FR-7.2 | Scholar reports joining transactions × scholar info | `export_scholarship_reports.php` |

### FR-8 Reporting & exports
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-8.1 | All existing reports remain available | `reports/` pages |
| FR-8.2 | CSV exports with UTF-8 BOM for Excel compatibility | `all_transactions.php:98-108` |

### FR-9 Audit & logging
| ID | Requirement | Source (v1) |
|---|---|---|
| FR-9.1 | Audit every mutation: who, what (action), target table/id, before/after snapshot | `logs.php`, `tbl_audit_logs` |
| FR-9.2 | Keep update/photo/password audit trails | `tbl_update_logs`, `tbl_photo_logs`, `password_resets` |

## 2. Non-functional requirements

| ID | Requirement | Notes |
|---|---|---|
| NFR-1 | **Data integrity** — cutover must not lose, alter, or renumber any record | Hard requirement |
| NFR-2 | **Security** — CSRF tokens, login rate limiting, no credentials in code, no error disclosure to users, prepared statements throughout | v1 gaps C1–C4 |
| NFR-3 | **Maintainability** — framework, migrations, service layer, tests | v1 anti-patterns A4–A6 |
| NFR-4 | **Performance** — current volume is fine; design must scale without rewrites (indexing, query hygiene, optional caching) | |
| NFR-5 | **Usability** — staff keep the same flows; no retraining beyond what the framework imposes | v1 UX preserved |
| NFR-6 | **Deployability** — simple deploy/rollback on the hosting target | avoid complex tooling |
| NFR-7 | **Auditability** — all FR-9 records kept for the required period | |

## 3. Requirements not carried from v1

These v1 behaviors are intentionally **not** reproduced in v2:

| Dropped | Reason |
|---|---|
| `manage_php.php` (edit PHP at runtime) | Dangerous; superseded by version control |
| Dead code (`default.php`, `client_photo.php`) | Removed with v1 |
| Hard-coded admin usernames in application logic | Replaced by permissions (FR-2.2) |
| In-file program lists/arrays | Replaced by configuration (FR-5.3) |
