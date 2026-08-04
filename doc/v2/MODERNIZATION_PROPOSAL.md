# v2 — Architecture Review & Modernization Proposal

> **Phase:** Architecture Review & Decision (SDLC step 4 — current).
> **Deliverable:** Review of the existing system and a modernization proposal
> for the v2.0 target architecture.
>
> **Type:** Architectural document only. No implementation code, no Laravel
> files, no database schema changes are included or authorized by this
> document.

**Inputs reviewed:** `../v1/SYSTEM_OVERVIEW.md`, `../v1/SYSTEM_DESIGN.md`
(incl. §13 System Design Analysis Report), `../v1/FILE_REFERENCE.md`,
`../v1/RECOMMENDATIONS.md`, `../v1/DATABASE_SCHEMA.md`; v2 phase documents
`VISION_AND_SCOPE.md`, `REQUIREMENTS_ANALYSIS.md`, `GAP_ANALYSIS.md`,
`MIGRATION_PLAN.md`, `ARCHITECTURE_DECISION.md`.

---

## 1. Executive Summary

The current system ("2D MIS", v1) is a working, production-grade legacy
application built in **procedural PHP** with no framework, a **MySQL/MariaDB**
database containing **live records**, Bootstrap UI, and a monolithic
page-per-file structure. It is **not suitable for long-term maintenance in its
present form**: the System Design Analysis Report rates it 2/5 across
architecture, modularity, maintainability, security, and scalability, and 3/5
for refactoring readiness.

The blockers are structural, not functional:

- The QR-scanning subsystem is **replicated ~16 times**; every change is
  repeated by hand.
- **Two conflicting authorization models** (DB permissions vs hard-coded
  usernames) make access control unpredictable.
- **Security gaps** exist: no CSRF protection, no login throttling, DB
  credentials hard-coded in source, and error disclosure to users.
- **No automated tests, no migrations, no version control** — dev/prod drift
  is already visible (collation mismatch).
- **Monolithic pages** (`view_client.php` ≈ 170 KB, `all_transactions.php`
  ≈ 1,400 lines) resist change and testing.

**Proposal:** Modernize to a **Laravel + Blade + Tailwind CSS** target
architecture while **keeping the existing MySQL database and all live data
untouched**. Migrate **incrementally** — v1 stays in production during
development, each module is built and validated in isolation against a copy of
the data, and a **single freeze-and-cutover** with a tested rollback path
completes the switch with minimal downtime. The migration also replaces the
"navigate-to-view-page" UX with a **right-side sliding details panel** that
keeps the table visible.

The recommendation is **approve**, subject to the decisions and guardrails in
Sections 4–9.

---

## 2. Current Architecture Evaluation

### 2.1 Profile

| Aspect | v1 reality |
|---|---|
| Language | Procedural PHP 7.2-era, no Composer |
| Routing | None — URL maps 1:1 to a PHP file |
| Rendering | Server-rendered HTML + Bootstrap 5 + jQuery + DataTables (server-side JSON) |
| Cross-cutting | `session.php` → `db_connect.php` → `restriction.php` include chain |
| Data access | Raw PDO prepared statements |
| Storage | `main_system`, 31 tables, live records |
| Scanning | 16 near-identical `scanner_*.php` pages |
| Audit | `log_action()` → `tbl_audit_logs` (+ update/photo/password trails) |
| Auth | username + bcrypt; single-device DB `session_token` |

### 2.2 Strengths (preserve these in v2)

1. **Prepared statements everywhere** — SQL injection is handled consistently.
2. **DB-level uniqueness as the real guard** — duplicate prevention for scans
   and payouts is enforced by constraints, not just application checks.
3. **A coherent cross-cutting include chain** — auth, DB, and ACL are the one
   shared architectural element and are invoked consistently on every page.
4. **Audit trail by convention** — every mutation logs who/what/before/after.
5. **Single-device session enforcement** — a genuine, working security feature.
6. **Server-side DataTables processing** — large lists never fully load into
   the browser.
7. **A well-understood, small domain** — clients, households, transactions,
   programs, scholarships, payouts; a tiny team knows it end-to-end.

### 2.3 Weaknesses

1. **No framework/routing** — authorization, validation, and logging must be
   repeated per file; there is no central dispatch.
2. **Copy-paste subsystems** — the scanner family (~16 copies) and payout
   attendance logic (3 copies) make change cost linear in the number of
   programs (anti-pattern A1).
3. **Two authorization models** — `tbl_permissions`/`restriction.php` plus
   literal-username checks and an implicit `user_id = 1` super-admin (A2, A3).
4. **Monolithic pages** — presentation and business logic are interleaved in
   files of thousands of lines (A4, A5).
5. **Hand-synced denormalized fields** — `full_name`, `match_name`, `age` drift
   across write paths, breaking scans and duplicate detection (A6).
6. **Configuration scattered in files** — program lists and dropdowns are
   hard-coded arrays repeated across pages (A7).
7. **No engineering infrastructure** — no migrations (A8), no automated tests,
   no version control, no backup strategy.
8. **Inconsistent error handling** — `die()`, JSON errors, or silent swallow,
   depending on the file.
9. **Dead code** — `default.php`, `client_photo.php`, and commented blocks (A10).

### 2.4 Architectural risks (if modernized)

| Risk | Description |
|---|---|
| Change amplification | A new program touches ~16 scanner copies + lists in several pages |
| Security exposure | CSRF, brute-force, credential disclosure, error leakage are open |
| Data drift | Denormalized fields silently diverge; duplicates accumulate |
| Upgrade lock-in | No tests/migrations/VCS makes any future change riskier |
| Knowledge silo | The system lives in one person's head; no documentation of decisions existed |
| Scalability ceiling | No caching, no pagination beyond tables; server-side rendering of reports |
| Hosting ceiling | Shared hosting constraints; PHP version must be raised for a framework |

**Verdict:** the architecture served its purpose but has reached its
maintainability ceiling. Long-term maintenance cost and risk are high and
rising. Modernization is justified by **maintainability and security**, not by
feature gaps.

---

## 3. Target Architecture

### 3.1 Stack

| Layer | Choice | Rationale |
|---|---|---|
| Backend | **Laravel** (PHP 8.3+) | Full-stack framework: routing, middleware, auth, ORM, migrations, events, testing |
| Views | **Blade** | Server-rendered templates — same mental model as v1's server-rendered pages; no SPA complexity |
| Styling | **Tailwind CSS** | Design-system-driven, consistent UI; ideal for the slide-over panel interaction |
| Front-end behavior | Minimal vanilla JS / Alpine.js | Progressive enhancement for the sliding panel; no heavy client framework |
| Database | **Existing `main_system` MySQL (unchanged)** | Data preservation is a hard constraint |
| Security | Laravel Auth + middleware + Gates/Policies + CSRF + validation + env config + audit | Framework-native security defaults |
| Deploy | Shared hosting (Hostinger Business+) with SSH/Composer | Continuity with current hosting |

### 3.2 Layered view

```
┌──────────────────────────────────────────────────────────────┐
│  Client      Browser  (Blade HTML + Tailwind + lightweight JS)│
├──────────────────────────────────────────────────────────────┤
│  Web layer   Routes → Middleware (auth, ACL, CSRF, throttle)  │
├──────────────────────────────────────────────────────────────┤
│  Controller  Thin controllers → Form Requests (validation)    │
├──────────────────────────────────────────────────────────────┤
│  Service     Business logic services (clients, transactions,  │
│              scanners, payouts, scholarships)                 │
├──────────────────────────────────────────────────────────────┤
│  Data        Eloquent models mapped to existing tables;       │
│              repositories for legacy-schema access            │
├──────────────────────────────────────────────────────────────┤
│  Storage     Existing MySQL `main_system` (unchanged)         │
└──────────────────────────────────────────────────────────────┘
```

Cross-cutting, everywhere: **audit observer/events**, **centralized error
handling**, **single ACL service**, **env configuration**.

### 3.3 Why Laravel + Blade + Tailwind is appropriate

- **PHP continuity** — the team already writes PHP; no language/tooling
  paradigm shift, and the framework fills exactly the gaps v1 has (routing,
  middleware, validation, migrations, testing).
- **Laravel's security defaults** close v1's known gaps (CSRF, throttling,
  prepared statements, password hashing, session hardening, env config)
  without custom code.
- **Blade keeps the server-rendered model** — v1 renders HTML server-side;
  Blade preserves that flow and staff mental model, avoiding the risk of a
  full client-side SPA rewrite, while enabling clean template composition to
  break apart the monolithic pages.
- **Tailwind delivers the required UX change** — a componentized utility-first
  design system makes the right-side sliding panel and a consistent look
  cheap to build and repeat across every module.
- **The database is a first-class citizen, not a rewrite** — Eloquent maps to
  existing tables; a baseline migration version the schema without altering it.

---

## 4. Architectural Decisions

Format per decision: **Problem → Decision → Justification → Expected Benefits →
Risks.**

### AD-1 Modernize on Laravel
- **Problem:** v1 has no framework; cross-cutting concerns are repeated per file, and maintenance is linear in the number of pages/programs.
- **Decision:** Adopt Laravel as the application framework (see ADR-001).
- **Justification:** Mature routing/middleware/auth/migrations/testing; PHP continuity; largest ecosystem; directly absorbs v1's include-chain patterns.
- **Expected benefits:** Centralized auth/ACL/validation/logging; testability; reduced change cost.
- **Risks:** Requires PHP 8.3+; hosting tier must support SSH/Composer (mitigate: verify Hostinger Business plan and PHP version first).

### AD-2 Server-rendered Blade views
- **Problem:** v1 mixes presentation and logic in monolithic pages; an SPA would be overkill and a staff-retraining risk.
- **Decision:** Render all screens with Blade templates; controllers stay thin; partials/components used for repeated UI (incl. the slide-over panel).
- **Justification:** Matches v1's server-rendered nature; simplifies accessibility and backward compatibility; breaks up monoliths into composable templates.
- **Expected benefits:** Consistent UI composition; the panel becomes one reusable component; pages are readable.
- **Risks:** Server round-trips per action (acceptable at this scale); needs discipline to keep templates presentation-only.

### AD-3 Tailwind CSS styling
- **Problem:** v1's Bootstrap styling is hand-copied and inconsistent; the required slide-over UX needs a coherent component system.
- **Decision:** Adopt Tailwind CSS for all new UI, including a shared component pattern for tables, forms, and the details panel.
- **Justification:** Utility-first design system produces consistent, maintainable styling; perfect fit for a reusable sliding panel; supersedes the provisional "keep Bootstrap" stance of ADR-006 per this proposal.
- **Expected benefits:** Visual consistency across modules; faster UI iteration; the panel component is built once.
- **Risks:** Visual regression vs v1 (mitigate: replicate v1 layouts/labels; pilot users before cutover); build tooling required at deploy time.

### AD-4 Keep the existing database unchanged
- **Problem:** 31 tables with live records; any schema change risks data loss and breaks v1 during the build.
- **Decision:** The `main_system` schema is **read-only for v2**. A **baseline migration** is generated to match production exactly; all later migrations are additive only. `migrate:fresh`/`refresh` are forbidden.
- **Justification:** Data preservation is the top constraint; v1 must keep working against the same DB until cutover.
- **Expected benefits:** No data migration, no renumbering; rollback is a re-point of the app.
- **Risks:** Legacy column shapes need explicit model mapping (custom PKs, absent `updated_at`); enum columns limit extensibility (mitigate: map explicitly, keep enum values identical).

### AD-5 Incremental build, single cutover
- **Problem:** True rolling dual-write migration (both apps writing the same live DB simultaneously) is dangerous without idempotency guarantees.
- **Decision:** **Incremental development against a database copy; one freeze-and-cutover.** Modules are built and validated one at a time on the copy (in the Section 8 order); v1 remains the production source of truth; at cutover v1 writes are frozen, v2 is deployed to the same DB, reconciliation runs, then v1 is retired (see MIGRATION_PLAN.md §5).
- **Justification:** Eliminates dual-write conflicts; keeps downtime to a single maintenance window; every module gets isolated parity testing.
- **Expected benefits:** Minimal downtime (one window), reversible, low data-integrity risk.
- **Risks:** The cutover window concentrates risk (mitigate: freeze + full backup + tested rollback; reconcile counts/sums before enabling writes).

### AD-6 Port the v1 authentication contract
- **Problem:** Framework auth defaults assume email/`remember_token`; v1's username + single-device `session_token` must keep working.
- **Decision:** Custom username-based login; single-device enforcement moved into middleware comparing the session token against the DB token; multi-device exemption becomes a permission.
- **Justification:** Preserves the working security behavior staff rely on (force-logout, single device) with zero data change.
- **Expected benefits:** Same behavior, now testable and centralized.
- **Risks:** Custom middleware needs maintenance across Laravel upgrades (mitigate: keep it thin, unit-tested).

### AD-7 Single authorization service (RBAC)
- **Problem:** v1 has two conflicting ACL models and an implicit super-admin.
- **Decision:** One ACL service backed by the existing `tbl_permissions` /
  `tbl_program_permissions` data, exposed to controllers via middleware and
  Gates/Policies. Username-based checks are eliminated.
- **Justification:** One predictable model; permission data is seeded unchanged.
- **Expected benefits:** Predictable authorization; admin rights become data, not code.
- **Risks:** Rebuilding the permission-management screen; mapping v1 page names to v2 routes (mitigate: keep the same permission keys).

### AD-8 One scanner engine, programs as configuration
- **Problem:** 16 copied scanner files; every fix must be repeated.
- **Decision:** A single scanner controller + reusable view; per-program
  configuration encodes duplicate rules (fixed remark key, monthly guard,
  exam-derived, update-in-place, validate-existing) and target tables.
- **Justification:** Kills the copy-paste subsystem; a new program becomes config.
- **Expected benefits:** One code path; fixes propagate everywhere.
- **Risks:** Config must faithfully reproduce all v1 variants (mitigate: per-program acceptance matrix before cutover).

### AD-9 Audit logging via events
- **Problem:** v1 audits by manual `log_action()` call sites; easy to miss a mutation.
- **Decision:** Keep the existing audit tables and content contract; fire audits from framework events/observers on the same write paths v1 logged.
- **Justification:** Audit becomes structural; history stays queryable in v2 unchanged.
- **Expected benefits:** No mutation is added without its audit.
- **Risks:** Must not double-log or change the record format (mitigate: parity tests against v1 log content).

### AD-10 Right-side sliding details panel (UI/UX)
- **Problem:** v1 opens a separate View page per record — a full navigation per inspection, losing table context and scroll position.
- **Decision:** Replace the navigate-to-view pattern with a **right-side sliding panel** triggered by clicking a table row; the table remains visible behind it. See Section 6.
- **Justification:** Directly requested UX goal; reduces navigation, keeps context, faster record review.
- **Expected benefits:** Fewer round trips, no context loss, consistent across all modules.
- **Risks:** Accessibility/complexity (mitigate: keyboard/Esc handling, focus management, and a no-JS fallback that still renders details).

### AD-11 Framework-native security stack
- **Problem:** v1 lacks CSRF, throttling, credential separation, and error suppression.
- **Decision:** Global CSRF on state-changing routes, login throttling, env-based secrets, centralized error handling with `APP_DEBUG=false`, credential rotation at cutover.
- **Justification:** Closes the C1–C5 findings with framework defaults plus a thin policy layer.
- **Expected benefits:** Known gaps closed with minimal custom code.
- **Risks:** Behavior changes for staff (timed logins, generic errors) — brief awareness needed.

### AD-12 Engineering foundations (tests, migrations, CI, VCS)
- **Problem:** v1 has no automated tests, migrations, version control, or backups.
- **Decision:** Git from day one; baseline migration (AD-4); automated tests for auth, ACL, scanners, and transactions; scheduled backups with a proven restore drill before cutover.
- **Justification:** Without these, any modernization is untestable and unrepeatable.
- **Expected benefits:** A safety net for the incremental migration and future change.
- **Risks:** New operational discipline for a small team (mitigate: keep CI simple, run locally if needed).

---

## 5. Security Architecture

The target security architecture fixes v1's measured weaknesses (§8 of the
System Design Analysis) using framework-native mechanisms:

| Control | v1 | v2 |
|---|---|---|
| Authentication | username + bcrypt, single-device token | Same contract, moved into **auth middleware** (AD-6) |
| Authorization | Two models (DB + usernames), implicit super-admin | **One ACL service + Gates/Policies** (AD-7) |
| CSRF | ❌ none | **Global CSRF middleware** on all state-changing routes |
| Brute-force | ❌ none | **Login throttling / rate limiting** |
| Input validation | Ad-hoc checks in pages | **Centralized validation** (form request objects / validators) before controllers |
| Sessions | httponly, SameSite=Lax; secure only on HTTPS | Same, plus hardened framework defaults; cookie `secure` enforced under HTTPS |
| Secrets | Hard-coded DB credentials in `db_connect.php` | **`.env` / environment variables**; no secrets in the repository; credentials rotated at cutover |
| SQL safety | PDO prepared statements | Eloquent/query-builder with parameter binding (same guarantee, one code path) |
| Error disclosure | `display_errors` + raw DB messages to users | **Centralized exception handling**; `APP_DEBUG=false` in production; logged server-side |
| Audit | Manual `log_action()` call sites | **Events/observers** on all mutations (AD-9) |
| Permissions data | `tbl_permissions`, `tbl_program_permissions` | Same tables, seeded unchanged, enforced centrally |

Net improvement: v1's open findings (C1–C5 and A2/A3/A9) are closed by
defaults the framework enforces, not by discipline.

---

## 6. UI/UX Architecture

### 6.1 Problem with the current interaction

v1 uses a *navigate-to-view* pattern:

```
Table → click "View" → separate detail page → Back → table reloads/scroll lost
```

Each record inspection is a full page round trip: context is lost, scroll
position is lost, and the operator must navigate back for the next record.

### 6.2 Proposed interaction model

```
Table (server-side paginated)
   │
   └─ click anywhere on a row ──▶ Right-side sliding panel opens
        ├── Close button (top)
        ├── Full record details
        ├── Edit button
        └── Delete button
   The table remains visible (and usable) behind the panel.
```

### 6.3 Behavior design

- **Trigger:** a click on any row (not just a View button). The clicked row is
  highlighted while the panel is open.
- **Content:** the panel is populated from the row's loaded data plus a
  lightweight details fetch (profile aggregates: photos, family, household,
  transactions, scholarship, GIP — the current `view_client.php` content).
- **Close:** top-right close button, plus **Esc** key; closing returns visual
  focus to the table without reloading it.
- **Staying in context:** the table's page, filters, and scroll position are
  untouched, so operators can review many records quickly.
- **Actions:** Edit opens the same edit form used by the existing edit page
  (rendered into the panel or navigated on submit); Delete triggers the
  guarded delete flow with the same duplicate/impact checks v1 uses.
- **Reuse:** the panel is **one shared component** used by clients,
  transactions, scholars, households, and payouts — not a per-module copy.
- **Progressive enhancement:** with JavaScript disabled, the panel degrades to
  the existing detail view route (deep-linkable, printable) — preserving v1
  backward compatibility.
- **Accessibility:** focus is trapped and managed in the panel, `aria`
  attributes describe the drawer, and Esc/close semantics are explicit.

### 6.4 Architectural impact

- A **panel component** (Blade partial) + a tiny shared state controller
  (open/close/load) replaces the per-module view pages for quick inspection.
- Detail *routes* still exist (for deep links, printing, and no-JS fallback),
  but the primary path is the panel — reducing page navigations and load
  without removing any capability.

---

## 7. Migration Strategy

**Recommended: Incremental build + single freeze-and-cutover** (AD-5),
consistent with `MIGRATION_PLAN.md`.

### 7.1 Guardrails (non-negotiable)
1. The `main_system` schema is **read-only for v2**; baseline + additive
   migrations only.
2. **No `migrate:fresh` / `refresh` / truncate** — ever.
3. v1 **stays live** and remains the production source of truth until cutover.
4. A **backup and restore drill** must succeed before any cutover.
5. **Backward compatibility** is preserved (see §7.3).

### 7.2 How the incremental strategy works
1. **Develop against a copy.** v2 is built and tested module-by-module against
   a full copy of the production data, so live data is never at risk.
2. **Validate per module.** Each migrated module must pass parity checks
   (same row counts, same sums, same behavior) before the next is started —
   this is where "incremental" is honored, in *development*.
3. **One cutover.** In a maintenance window: freeze v1 writes → full backup →
   deploy v2 against the same DB → run reconciliation → if green, retire v1;
   if not, roll back v2 (v1 and data are untouched).
4. **Why not dual-write in production?** Running two stacks writing the same
   live tables without idempotency guarantees risks duplicates and diverging
   state. A single freeze window is the safer way to honor "minimize downtime"
   and "preserve data" simultaneously.

### 7.3 Backward compatibility commitments
- Login remains **username + password**; permission keys keep v1 names.
- Audit tables and their record format are unchanged (v2 appends to them).
- CSV export format (UTF-8 BOM) is unchanged.
- Detail routes remain reachable by URL for deep links/printing (panel is the
  primary path).
- Existing program names and transaction status values are preserved
  byte-for-byte.

---

## 8. Migration Roadmap

Ordered by **dependency first, then risk exposure and user value**, each with
its exit criterion.

| Step | Module | Why first/this order | Exit criterion |
|---|---|---|---|
| P0 | **Foundations**: scaffolding, baseline migration, env/CI, backups, DB copy | Everything depends on it; freezes schema parity | v2 boots against a copy of `main_system`; restore drill passes |
| P1 | **Auth + RBAC** | No module is usable without login/authorization; also validates AD-6/7 | Login, single-device, and permission checks pass on the copy |
| P2 | **Client registry + households** | Core data; most-used screens; establishes the row-click slide-over pattern | Client CRUD + household flows match v1; panel parity confirmed |
| P3 | **Transactions + reports/exports** | Highest daily usage after clients; validates server-side tables and CSV | Transaction filters/sums/exports match v1 on the copy |
| P4 | **Scanner engine + all 17 programs** | Biggest win (kills 16 copies) but highest risk; done after core is stable | Per-program acceptance matrix passes (scan variants, duplicate rules) |
| P5 | **Payout attendance + unpaid verification** | Depends on transactions; DB-unique constraints give a safety net | Payout scan/unpaid flows match v1 (incl. seats) |
| P6 | **Scholarship / GIP / exam** | Self-contained, lower traffic | Scholar reports match v1 exports |
| P7 | **Admin**: users/permissions, audit viewer; remove `manage_php.php` | Last; only admins see it; replaces v1's runtime-PHP-editing risk | Permission management works; audit viewer reads v1 history |
| P8 | **Hardening + regression + cutover** | Full test pass, security review, freeze window, reconciliation, retire v1 | Reconciliation green; rollback exercised once on a copy |

**Rationale:** P0/P1 first because nothing else functions without them and
they prove the risky parts of AD-4/6/7 early. High-traffic core modules (P2/P3)
next so real staff workflows validate the UX early. The scanner engine (P4) is
deferred until the core is stable because it is the largest risk-to-reward
item; payouts/scholars/admin follow in dependency order. Hardening and cutover
are last, by definition.

---

## 9. Risks and Mitigation

| # | Risk | Likelihood / Impact | Mitigation |
|---|---|---|---|
| R1 | **Data loss / corruption at cutover** | Low / Critical | Freeze writes; full backup; restore drill before cutover; reconciliation counts/sums; rollback = re-point v1 |
| R2 | **Baseline migration drifts from production** | Medium / High | Generate baseline from the actual prod dump; diff before deploy; additive migrations only |
| R3 | **Dual-write conflict avoided wrongly** | Low / High | Single cutover (AD-5); v1 is sole writer until the window |
| R4 | **Scanner variants behave differently in v2** | Medium / High | Per-program acceptance matrix (P4) on the copy; test every duplicate-rule variant |
| R5 | **Legacy schema fights framework conventions** | Medium / Medium | Explicit model mapping (custom PKs, no `updated_at`); parity tests on the copy |
| R6 | **UI regression vs v1 / staff pushback** | Medium / Medium | Replicate layouts/labels; slide-over panel piloted with users before cutover; no-JS fallback retained |
| R7 | **Hosting cannot run PHP 8.3+ or lacks SSH** | Medium / High | Confirm Hostinger Business+ plan and PHP 8.3+ before building; fallback decision documented (ADR-001) |
| R8 | **Performance on shared hosting** | Low-Medium / Medium | Sync queue / cron for background work; server-side paging retained; chunk long exports; upgrade path to Cloud/VPS if needed |
| R9 | **Scope creep during migration** | Medium / Medium | Modules are acceptance-gated (Section 8); no new programs/features until cutover |
| R10 | **Knowledge loss (single-maintainer system)** | Medium / Medium | This ADR + docs set; v2 code is self-documenting via framework conventions |

---

## 10. Final Recommendation

**Approve the modernization** of 2D MIS v1 → v2 (Laravel + Blade + Tailwind
CSS) on the existing, unchanged `main_system` database, executed as an
**incremental build with a single freeze-and-cutover**, per Sections 3–8.

**Go / no-go criteria before building begins (P0):**
1. Hostinger account confirmed **Business+** with **PHP 8.3+** in both hPanel
   and SSH CLI.
2. Baseline migration generated from the production dump and verified byte-for-
   byte against it.
3. Full backup + restore drill passes.
4. The Architecture Decision records (ADR-001…010) and AD-1…12 above are
   reviewed and accepted; the four open questions in
   `ARCHITECTURE_DECISION.md` are answered (framework confirm, hosting,
   soft-deletes, index additions).

**Cutover approval requires:** P0–P7 exit criteria met, reconciliation green
on a rehearsal cutover (on a copy), and a signed-off rollback plan.

The modernization is justified and low-risk **only if** the data-preservation
and schema-unchanged guardrails are treated as absolute — they are the reason
this proposal is safe to approve.

---

*End of Architecture Review & Modernization Proposal.*
