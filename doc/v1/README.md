# 2D MIS — v1 System Documentation

> Version 1 (the current plain-PHP system). Management Information System for
> municipal assistance programs (province of Ilocos Sur, Philippines).

This folder is the read-only reference documentation for the current
`C:\xampp\htdocs\system` application **as it exists today** (the "v1" system).
No source code, files, SQL, or database were modified to produce it.

See [../README.md](../README.md) for the root index (v1 vs v2).

## Documentation index

| Document | Contents |
|---|---|
| [SYSTEM_OVERVIEW.md](SYSTEM_OVERVIEW.md) | Architecture, stack, request flow, authentication, session control, access control, security notes |
| [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) | Design analysis: patterns, layers, subsystem design, data model, state/concurrency, security & error handling design, anti-patterns, v2.0 evolution path |
| [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md) | Every table in `main_system` (columns, types, keys, relationships) |
| [FILE_REFERENCE.md](FILE_REFERENCE.md) | Catalog of all 115 PHP files grouped by module, with one-line purpose each |
| [WORKFLOWS.md](WORKFLOWS.md) | Step-by-step end-to-end flows: login, add client, QR scanning, payouts, reporting |
| [RECOMMENDATIONS.md](RECOMMENDATIONS.md) | Improvement plan by priority: security, maintainability, data quality, UX/ops |
| [COMPLETE_DOCUMENTATION.md](COMPLETE_DOCUMENTATION.md) | Single-file merge of all of the above except this README |

## Quick facts (v1)

- **Language / stack:** plain PHP (no framework, no Composer), PDO + MariaDB/MySQL.
- **Front-end:** Bootstrap 5, jQuery, DataTables (server-side), `html5-qrcode`.
- **Database:** `main_system` (31 tables), SQL dump at `u749085076_main_system.sql`.
- **Auth:** username/password (bcrypt via `password_hash`), PHP sessions, single-device token enforcement.
- **Access control:** per-page `tbl_permissions`, per-program `tbl_program_permissions`, plus username-based admin checks.
- **Core domain:** clients, households, transactions (assistance programs), scholarship records, QR-code attendance/payout scanning, unpaid-grantee verification, audit logging.
- **Programs tracked:** AICS, AKAP, MAIP, TUPAD, CEDSSG, CEAP, CEAP_NEW, OTCES, OTEA, CEDSSG_NEW, COFFEE GROWERS, PUSO TI KABABAIHAN, PUSO TI AGTUTUBO, PUSO TI MANNALON, TESDA, GIP, TODA.

## How to run v1 locally (XAMPP)

1. Import `u749085076_main_system.sql` into a local database named `main_system`
   (MariaDB 10.4 requires replacing the collation `utf8mb4_uca1400_ai_ci`
   with `utf8mb4_unicode_ci` before import).
2. Ensure `db_connect.php` points at the local database
   (host `localhost`, db `main_system`, user `root`, empty password).
3. Serve the folder from Apache and open `login.php`.
