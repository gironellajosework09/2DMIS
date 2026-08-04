# v2 — Vision & Scope

## 1. Why v2 exists

The v1 system works and is in production use, but its structure limits
maintenance, security, and growth. Analysis of v1 (see
`../v1/SYSTEM_DESIGN.md` and `../v1/RECOMMENDATIONS.md`) shows the main drivers:

1. **Maintainability ceiling.** The scanner subsystem is copied ~16 times,
   so every bug fix or program change must be repeated by hand.
2. **Security gaps.** No CSRF protection, no login rate limiting, hard-coded
   DB credentials, and error disclosure (`../v1/SYSTEM_DESIGN.md` §8).
3. **Data-integrity risk.** Denormalized fields (`full_name`, `match_name`,
   `age`) are synced by hand and drift over time.
4. **Two conflicting access-control models** (DB permissions vs hard-coded
   usernames) make authorization hard to reason about.
5. **Growth.** As records grow, server-side rendering without caching or a
   service layer will strain performance.

## 2. Objectives

| # | Objective | Success measure |
|---|---|---|
| O1 | Preserve all v1 data exactly | Same records, IDs, and history after cutover |
| O2 | Remove the copy-paste scanner subsystem | One scanner engine driven by config; programs become data, not code |
| O3 | Close the known security gaps | CSRF, rate limiting, no credentials in code, no error disclosure |
| O4 | Single, predictable access-control model | One ACL service; no username-based checks |
| O5 | Keep the same workflow for end users | Staff do their jobs the same way; minimal retraining |
| O6 | Make the codebase maintainable | Framework with tests, migrations, and a service layer |

## 3. Scope

### In scope
- Rebuild the application on a modern PHP framework.
- Port all existing modules: login/session, clients, households, transactions,
  all assistance programs, QR scanning, payout attendance, unpaid verification,
  scholarship records, reporting/exports, audit logging, user & permission
  management.
- Backfill a migration baseline that matches the existing `main_system` schema
  so the data can be versioned going forward.
- Introduce the security controls from §O3 and the maintainability fixes
  from `../v1/SYSTEM_DESIGN.md` §12.

### Out of scope (for now)
- Redesigning the data model / changing table structures.
- New assistance programs beyond what exists today.
- Mobile apps, offline mode, or biometric/ID hardware integration.
- Rewriting the historical data or cleaning existing records (a separate
  data-quality effort, see `../v1/RECOMMENDATIONS.md` D-items).

## 4. Key constraints

| Constraint | Implication |
|---|---|
| Keep `main_system` database as-is | No `migrate:fresh` / `refresh`; migrations only add a baseline |
| Keep v1 running during development | v2 is built alongside; single cutover at the end |
| Staff familiarity | Keep UI flows and naming consistent with v1 |
| Shared hosting heritage | Choose a stack that still fits the deployment target, or move hosting deliberately |
