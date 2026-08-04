# v2 — Gap Analysis (v1 → v2)

Maps every known v1 weakness to the v2 decision that resolves it. Sources:
`../v1/RECOMMENDATIONS.md` (C/M/D/O priorities) and
`../v1/SYSTEM_DESIGN.md` §11 (anti-patterns A1–A10).

## 1. Critical gaps (security) — must close first

| v1 issue | Ref | v2 resolution |
|---|---|---|
| No CSRF tokens on any form/post | C1 | Global CSRF middleware on all state-changing routes |
| No login rate limiting / brute-force protection | C2 | Login throttling (attempt + lockout) |
| DB credentials hard-coded in `db_connect.php` | C3 | Environment variables / `.env`, secrets out of repo |
| `display_errors` + raw DB errors shown to users | C4 | Log errors server-side; show generic messages |
| Exposed password considered compromised | C5 | Rotate DB credentials on cutover |
| Single-device token logic only in v1, must be ported safely | — | Dedicated auth middleware implementing the same contract (FR-1.2) |

## 2. High-impact maintainability gaps

| v1 issue | Ref | v2 resolution |
|---|---|---|
| Scanner subsystem copied ~16 times | A1 | One scanner controller + one view; program = config entry (FR-6.1) |
| Two ACL models (DB permissions vs usernames) | A2 | Single ACL service (FR-2.1, FR-2.2) |
| Implicit super-user (user_id 1) | A3 | Explicit role/permission definition, seeded but documented |
| Monolithic pages (`view_client.php`, `all_transactions.php`) | A4 | Route + controller + partial views + presenters |
| Raw SQL scattered with no service layer | A5 | Repository/service layer; models per hub entity |
| Denormalized fields synced by hand (`full_name`, `match_name`, `age`) | A6 | Central derivation service called by one write path (FR-3.2) |
| In-file config duplication (programs, dropdowns) | A7 | Config/store for programs & enums (FR-5.3) |
| No migrations → dev/prod drift (collation mismatch seen locally) | A8 | Migration baseline matching `main_system` + up-to-date deploy pipeline |
| No tests | — | Automated tests for auth, ACL, scanners, transactions |

## 3. Medium data-quality gaps

| v1 issue | Ref | v2 resolution |
|---|---|---|
| `age` / `full_name` drift on client edits | D-items | Derived fields computed centrally at write time; optional periodic recompute job |
| Enum `program` columns block adding programs | D-items | Store program as config; only the values that are truly fixed stay enums |
| Geography stored as unconstrained varchar FKs | D-items | Keep v1 values (data is untouchable); new lookups optional later |
| No soft deletes; physical cascades | D-items | Evaluate soft-delete + restore for v2 if staff need it; otherwise keep v1 semantics |
| Duplicate clients persist | D-items | Improve duplicate detection UX; operator-guided merge workflow (FR-3.5) |

## 4. Low/ops gaps

| v1 issue | v2 resolution |
|---|---|
| No backup strategy | Scheduled DB backups + restore drills before cutover |
| No version control | Git repo from day one of v2 |
| Collation `utf8mb4_uca1400_ai_ci` unsupported on local MariaDB | Standardize on `utf8mb4_unicode_ci` baseline for all new migrations |
| Timezone set per-page (`Asia/Manila`) | Set once in app config |

## 5. What v1 does well — preserve as-is

- PDO prepared statements everywhere (keep in v2).
- DB-level unique constraints as the real anti-duplicate guarantee (keep).
- Audit trail tables and call points (port directly to observers/events).
- The `lookup`/`save` scanner contract (maps cleanly to v2 routes).
- Server-side DataTables contract (reuse as-is).
- CSV export with UTF-8 BOM (reuse as-is).
