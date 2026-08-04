# 2D MIS — Documentation

> Management Information System for municipal assistance programs
> (province of Ilocos Sur, Philippines).

Documentation is split by system version.

## Version folders

| Folder | Scope |
|---|---|
| [v1/](v1/) | Documentation of the **current** system as built (plain PHP). Reference for how v1 works today. |
| [v2/](v2/) | **Planning & Analysis** for the version 2.0 upgrade. Goals, requirements, gap analysis, and migration approach. |

## Status

- **v1:** analysis/documentation complete (as-is, read-only).
- **v2:** Planning & Analysis phase in progress. No code, files, SQL, or
  database changes have been made to the v1 system.

## Quick facts

- **v1 stack:** plain PHP (no framework, no Composer), PDO + MariaDB/MySQL; Bootstrap 5, jQuery, DataTables, `html5-qrcode`.
- **Database:** `main_system` (31 tables), dump at `u749085076_main_system.sql`. Data must be preserved across the v1 → v2 upgrade.
- **Programs tracked:** AICS, AKAP, MAIP, TUPAD, CEDSSG, CEAP, CEAP_NEW, OTCES, OTEA, CEDSSG_NEW, COFFEE GROWERS, PUSO TI KABABAIHAN, PUSO TI AGTUTUBO, PUSO TI MANNALON, TESDA, GIP, TODA.
