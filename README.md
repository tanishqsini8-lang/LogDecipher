# LogDecipher — Explainable Cybersecurity Log Analysis Platform

LogDecipher is a web application that analyzes login-event CSV files, flags suspicious patterns with simple, transparent detection rules, and explains every finding in plain language. It demonstrates basic ideas used in Security Information and Event Management (SIEM) systems.

> **Independent educational project. It is not a production-grade SIEM.** Alerts are leads to investigate, not proof of malicious activity. Use only synthetic or explicitly authorized data, and never put real passwords in a file.

## Features

- **Dashboard** with metrics calculated from the analyzed file: events analyzed, unique alert groups, flagged events (each counted once), severity distribution, and an event timeline.
- **CSV analyzer** with validation: rejects empty, binary, oversized, non-UTF-8 files and files with missing columns; skips and lists individual bad rows (invalid timestamp, IP address, status, username).
- **Four detection rules** (thresholds are configurable in the interface):
  1. Repeated failed logins against one account within a time window.
  2. Successful login after repeated failures on the same account.
  3. One source address failing against many different accounts (password-spraying pattern).
  4. Login outside an expected schedule (runs only when you configure expected hours).
- **Explainable alerts**: unique ID (`LD-0001`…), rule, severity, supporting events with line numbers and timestamps, account and source, why it matched, recommended defensive action, and limitations / possible false positives.
- **Filters** by severity, rule, username, source IP and time range, plus sorting.
- **CSV export** of findings (respects active filters), protected against spreadsheet formula injection.
- **Fictional sample dataset** (`data/sample_logins.csv`, 109 events, invented users and documentation-range IP addresses).

## Installation and running

Requires Python 3.10 or newer (developed and tested on Python 3.13).

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python run.py
```

Open <http://127.0.0.1:5000>, click **Use fictional sample dataset**, or upload your own CSV.

The server listens on `127.0.0.1` only, with debug mode off. Optional environment variables: `LOGDECIPHER_PORT`, `LOGDECIPHER_HOST`, `LOGDECIPHER_SECRET_KEY`.

## Input format

A header row with these columns (any order; extra columns are ignored and never stored):

```csv
timestamp,username,source_ip,status
2026-03-02 09:14:05,aarav,10.0.1.11,success
2026-03-02 09:15:40,meera,203.0.113.45,failure
```

- `timestamp`: ISO 8601 date **and** time. Timestamps with a timezone are converted to UTC; others are used as written.
- `status`: `success` or `failure` (also accepts ok, accepted, failed, fail, denied, rejected).
- Limits: 2 MB and 50,000 rows per file.

## Detection rules and default thresholds

| Rule | Default | Severity |
|---|---|---|
| Repeated failed logins | ≥ 5 failures on one account within 10 minutes | Medium; High at ≥ 2× threshold |
| Success after failures | ≥ 3 failures within the 10 minutes before a success | Medium; High at ≥ 2× threshold |
| Failures across multiple accounts | one source, ≥ 4 accounts within 10 minutes | Medium; High at ≥ 2× threshold |
| Login outside expected hours | off unless start and end hour are set | Low |

The same details, with each rule's limitations, are shown in the application on the **Detection rules** page.

## Testing

```bash
python -m unittest discover -s tests -v
```

150 automated tests cover the detection rules (including boundaries and ordering), CSV parsing and validation, the sample dataset, filters, export safety, CSRF protection, escaping of hostile input, security headers, and error handling.

Optional browser-driven check (needs `pip install -r requirements-dev.txt` and `python -m playwright install chromium`):

```bash
python scripts/capture_screenshots.py
```

This drives the real interface (sample load, filters, export download, alert page, invalid upload, mobile layout) and regenerates the screenshots in `docs/screenshots/`.

## Project structure

```
run.py                     start the server
logdecipher/
  app.py                   Flask routes, security headers, CSRF, error pages
  engine/                  parser.py, detector.py, rules.py, config.py, summary.py
  filters.py exporter.py charts.py store.py analysis.py
  templates/ static/       interface
data/sample_logins.csv     fictional sample data
scripts/                   sample-data generator, screenshot/browser checks
tests/                     automated tests
docs/screenshots/          screenshots of the running application
```

## Security notes

- Uploaded data is kept in server memory only (not written to disk), expires after 2 hours, and can be cleared with **Clear data**.
- All output is HTML-escaped; a strict Content-Security-Policy blocks inline scripts.
- POST requests require a CSRF token; upload size and row limits are enforced; stack traces are never shown to users (they go to the server log).
- Exported CSV cells starting with `=`, `+`, `-`, `@`, tab or carriage return are prefixed with `'`.

## Limitations

- Rules are simple thresholds: slow or distributed attacks below the thresholds are missed, and normal behaviour (mistyped passwords, shared office addresses, shift work) can trigger alerts.
- One schedule applies to all accounts; timestamps are treated as UTC.
- No login or user accounts: designed for a single user on a local machine. Not hardened for public internet deployment.
- No persistence, real-time ingestion, geolocation, threat intelligence or machine learning.
- Tested with the Flask test client and headless Chromium on desktop and phone-sized viewports; not tested on other browsers or real devices.

## Future work

Support for more log formats (SSH, web server), per-account schedules, allow-lists for known addresses, PDF reporting, persistent storage, and user authentication.
