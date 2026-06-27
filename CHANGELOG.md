# Changelog

## v3.0 — Multi-DBMS + automatic technique selection

> Renamed: `pg-time-blind` → **blindfold** (script `pg_time_blind.py` → `blindfold.py`).

### Added
- **Automatic DBMS detection** (fingerprinting) for **PostgreSQL, MySQL, MSSQL, Oracle**;
  pin manually with `--dbms`.
- **Error-based extraction** — forces a reflected DB error and dumps the value in a single
  request (PostgreSQL `CAST`, MySQL `extractvalue` with automatic chunking, MSSQL `CAST/convert`).
  Tried first because it's the fastest path; `--no-error` to skip.
- **Technique auto-selection**: error-based → boolean-based → time-based.
- **Safe mode (default)**: risky `OR` contexts that can change app state are skipped unless
  `--allow-or` is given.
- **Threaded boolean extraction** via `--threads N` (time-based stays serial for clean timing).
- **Robust calibration**: 3 samples per side and digit-stripped token matching to survive
  dynamic content (CSRF tokens, timestamps).
- Resume checkpoint now also caches the detected DBMS/technique/context.

### Changed
- Detection reorganised into explicit **DBMS / TECHNIQUE / CONTEXT** reporting.
- Per-character primitives are now DBMS-aware (`substr` vs `substring`, `length` vs `len`).

### Notes
- Oracle uses boolean extraction (inline time/error primitives omitted for reliability).
- Existing PostgreSQL commands keep working; output now reports the detected backend + technique.

## v2.0 — Structured detection (boolean + time-based)

- Two-phase workflow: detect injection (boolean preferred, time fallback) then extract.
- Auto-calibrated TRUE/FALSE signal; automatic context discovery; resume support.
