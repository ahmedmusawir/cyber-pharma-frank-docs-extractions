# 00 — Repo Profile

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

A Flask-based multi-tenant SaaS API for a pharmacy audit/reimbursement platform. Drives subscription onboarding, business activation, prescription data ingestion, and pharmacy reference-data (WAC/AAC/FUL/PBM) lookups for an external desktop client. Backed by Stripe (billing), Google Cloud (Secret Manager, Cloud SQL, GCS, Cloud Functions, Cloud Tasks, App Engine/Cloud Run), Go High Level (CRM/email automation), and Authlib OAuth (Google/Microsoft). Contains 22.5K lines of Python in the main package across 25 modules and 13 Flask blueprints. Author docs claim HIPAA-targeted; encryption/audit controls are partially implemented (see 07-GUARDRAILS).

---

## Findings

### Project Identity

**EVIDENCE** — Project name and purpose
**Source:** `README.md:1-9` — title "Pharmacybooks API"; description "A SaaS pharmacy audit application API with multi-tenant business management, subscription handling, and role-based access control."

**EVIDENCE** — Importable package name
**Source:** `pharmacybooks_api/__init__.py:14, 276` — `from .models import db`; `def create_app(): app = Flask(__name__)`. Top-level entrypoint is `app.py:4` `app = create_app()`.

**EVIDENCE** — Not a git repo at extraction time
**Source:** Harness environment reports `Is a git repository: false`. However `pharmacybooks_api/__init__.py:34-44` calls `git rev-parse HEAD` at startup and stores `GIT_COMMIT_HASH` (returns `'unknown'` if git is unavailable).

**GAP** — No factory docs (APP_BRIEF.md, DATA_CONTRACT.md, FILE_TREE.md, UI_SPEC.md). All design intent is in code, root-level `*.md` files, and `docs/`.

---

### Language and Runtime

**EVIDENCE** — Python runtime
**Source:** `Dockerfile:2` `FROM python:3.11-slim`; `app.yaml:2` `runtime: python311`; `README.md:35` "Python 3.8+" (a contradiction — the Docker/AppEngine targets 3.11, README claims 3.8+).

**EVIDENCE** — Web framework
**Source:** `pharmacybooks_api/__init__.py:1` `from flask import Flask, jsonify, request`. Confirmed by `requirements.txt:1` `Flask`.

**EVIDENCE** — WSGI server
**Source:** `Dockerfile:34` `CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 pharmacybooks_api:app`; `app.yaml:5` `entrypoint: gunicorn -b :$PORT app:app --timeout 600`.

**INFERENCE** — `pharmacybooks_api:app` works because the package surfaces `app` via `pharmacybooks_api/__init__.py` (the create_app function returns app, and the WSGI server calls `pharmacybooks_api:create_app` indirectly via `app.py:app = create_app()`). Two different module targets across deploy methods (`pharmacybooks_api:app` in Dockerfile, `app:app` in app.yaml).

---

### Dependencies (requirements.txt)

**EVIDENCE** — All deps unpinned
**Source:** `requirements.txt:1-20` — 20 packages, none with version constraints:
- Web: `Flask`, `Flask-SQLAlchemy`, `Flask-Migrate`, `Flask-JWT-Extended`, `Flask-Cors`, `Werkzeug`
- DB: `mysql-connector-python`, `psycopg2-binary` (both drivers shipped)
- Payments: `stripe`
- WSGI: `gunicorn`
- Google Cloud: `google-cloud-secret-manager`, `google-cloud-storage`, `google-api-python-client`
- Data parsing: `openpyxl`, `xlrd`, `pandas`
- Misc: `python-dotenv`, `requests`, `cryptography`, `authlib`

**EVIDENCE** — Test deps file exists but contents not inspected
**Source:** `requirements-test.txt` (75 bytes — minimal, pytest-only inferred)

**GAP** — No `pyproject.toml`, no Poetry, no Pipfile, no version pins. Reproducible builds rely on Docker image pin only.

---

### Entrypoints

**EVIDENCE** — Main HTTP entrypoint
**Source:** `app.py:1-7`:
```python
from pharmacybooks_api import create_app
app = create_app()
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

**EVIDENCE** — Cloud Functions entrypoints (separate deploy targets)
**Source:**
- `functions/aac_import_trigger/main.py` — AAC import scheduler trigger
- `functions/ful_data_pull/main.py:71` `@functions_framework.http def trigger_ful_import(request)` — Cloud Scheduler → Cloud Tasks
- `functions/ful_import_worker/main.py` — Cloud Tasks worker that calls back into the API
- `functions/wac_import_trigger/main.py` — WAC import scheduler trigger

**EVIDENCE** — Standalone scripts (run via `python -m` or directly)
**Source:**
- `pharmacybooks_api/scripts/cleanup_orphaned_registrations.py`
- `pharmacybooks_api/scripts/normalize_existing_ndcs.py`
- `pharmacybooks_api/scripts/run_ful_import.py`
- `pharmacybooks_api/scripts/verify_transaction_handling.py`
- Root-level `check_*.py`, `create_*.py`, `diagnose_preview.py`, `fix_alembic_version.py`, `issue_temp_password.py`, `list_super_admins.py`, `update_test_user_password.py`, `verify_*.py`, `test_cloud_sql_connection.py`, `test_db_connection_simple.py`, `test_email_query.py`, `test_postgres_password.py` — operational tooling not part of the test suite

---

### Tech Stack Summary

| Layer | Technology |
|---|---|
| Language | Python 3.11 |
| Web framework | Flask + Flask-SQLAlchemy + Flask-Migrate (Alembic) + Flask-JWT-Extended + Flask-CORS |
| WSGI server | gunicorn (1 worker × 8 threads on Cloud Run) |
| Database | Dual-driver: MySQL (mysql-connector-python) and PostgreSQL (psycopg2-binary), runtime-switchable via `DB_TYPE` env var; SQLite for tests |
| Migrations | Alembic (62 versions in `migrations/versions/`) |
| Auth | JWT cookies (httpOnly, SameSite=Lax) via Flask-JWT-Extended; Authlib OAuth (Google, Microsoft) |
| Payments | Stripe (Customers, Checkout Sessions, Subscriptions, Promotion Codes, Billing Portal, Webhooks) |
| Email | SMTP (host/port/TLS/SSL configurable via env or consolidated `SMTP_CONFIG_JSON` secret) |
| CRM | Go High Level (GHL) via REST `https://rest.gohighlevel.com/v1` with bearer-token auth |
| Cloud | Google Cloud: Secret Manager, GCS, Cloud SQL (Unix socket on App Engine; TCP on Cloud Run), Cloud Run, App Engine Standard, Cloud Functions, Cloud Tasks |
| Sheets | Google Sheets API (PBM Corrections inbox) via service account |
| Crypto | `cryptography.fernet` (key = SHA256 of `TEMP_PASSWORD_ENCRYPTION_KEY` / `PASSWORD_RESET_SECRET_KEY` / `SECRET_KEY` / hardcoded fallback) |
| Logging | Standard `logging` + `contextvars` correlation IDs |
| Reference data | WAC, AAC, FUL, PBM info, APA membership tables; signature checksums via `reference_data_versions.py` |
| External APIs | Liberty Software (`liberty_client.py`) for prescription data |
| Frontend | None — this is a JSON API plus one HTML template (`reset_password.html`); webapp client lives elsewhere (`WEBAPP_URL` env var) |
| Desktop client | Companion app outside this repo; consumes `/api/desktop/*` endpoints |
| Testing | pytest (`pytest.ini`), markers: `integration`, `unit`; ~38 test files; tests use SQLite in-memory and JWT manager fixtures |

---

### File Tree (top 2 levels — actual)

**EVIDENCE** — Listed via `ls -la` and `Glob`
**Source:** Repo root listing performed during Phase 1.

```
pharmacybooks-api-main/
├── app.py                           # WSGI entry — calls create_app()
├── app.yaml                         # App Engine Standard config (live Stripe pub key + GHL field map embedded)
├── Dockerfile                       # Cloud Run image
├── Dockerfile.ful-job               # Separate image for FUL import job
├── docker-compose.yml               # Local dev composition
├── cloudbuild-ful-job.yaml          # Cloud Build for FUL job
├── requirements.txt, requirements-test.txt
├── pytest.ini
├── openapi.yaml                     # 58KB OpenAPI spec
├── session.json                     # 331-byte sample Stripe checkout session payload (no secrets, but contains a real-looking Stripe price ID and test NPI/NCPDP)
├── smtp-config.json                 # 181 bytes
├── .env.backup, .env.example, .gcloudignore, .gitignore
├── cloud-sql-proxy.exe              # 32MB binary committed
├── upload-trace.log                 # 5MB log file committed
├── pharmacybooks_api/               # Main Flask package
│   ├── __init__.py (688)            # create_app, secrets, Stripe key validation, blueprint registration
│   ├── account.py (123)             # /api/account combined account endpoint
│   ├── admin.py (1372)              # /api/admin/* — user mgmt, business mgmt, orphan cleanup, reports, activation keys
│   ├── apa_membership_loader.py (223)
│   ├── audit_utils.py (210)         # AuditLog helpers + sensitive-field masking + CRM_MASTERED_FIELDS list
│   ├── auth.py (2731)               # /api/auth/* — register, login, OAuth (Google/Microsoft), pending registrations, activation, multi-pharmacy linking
│   ├── business.py (1863)           # /api/business/* — exists, by-activation-key, register alias, activate, status, info, subscription/cancel/renew/portal
│   ├── crypto_utils.py (49)         # Fernet encrypt/decrypt for support secrets
│   ├── dashboard.py (1085)          # /api/data/* — user_data CRUD, WAC/FUL/AAC/PBM reference reads + writes, GCS-backed loads
│   ├── desktop.py (841)             # /api/desktop/* — auto-update, local-users, versions, pbm-corrections, password-resets/temporary
│   ├── email_utils.py (130)         # SMTP sender + masking
│   ├── gcs_loader.py (701)          # GCS readers for PBM/AAC bulk data
│   ├── health.py (250)              # /api/health/* — health, ping, registration-test echo
│   ├── imports.py (3227)            # /api/import/* — APA, WAC, AAC, FUL, user_data uploads (CSV/XLSX/GCS); Liberty integration
│   ├── integrations.py (4593)       # /api/integrations/* — GHL contact CRUD, GHL webhooks, second Stripe webhook handler, GHL custom-field map
│   ├── liberty_client.py (73)       # Liberty Software HTTP client
│   ├── logging_utils.py (372)       # contextvars correlation_id + structured logging helpers
│   ├── models.py (1270)             # SQLAlchemy schema (16 models) + 3 SQLAlchemy event listeners
│   ├── ndc_utils.py (39)            # NDC normalization (11-digit zero-padded)
│   ├── password_resets.py (383)     # /api/password-resets/* — email-link request/verify/complete
│   ├── payments.py (1692)           # /api/payments/* — checkout, renew, status, cancel, webhook (primary), webhook/health, check-trial-eligibility
│   ├── pbm_corrections.py (129)     # Google Sheets append helper
│   ├── pharmacy.py (120)            # Legacy /api/pharmacy_profile (PUBLIC, single-tenant) + /api/health + /api/profile
│   ├── reference_data_versions.py (305)  # Checksum/version tracking for WAC/AAC/FUL/PBM
│   ├── scripts/
│   │   ├── cleanup_orphaned_registrations.py
│   │   ├── normalize_existing_ndcs.py
│   │   ├── run_ful_import.py
│   │   └── verify_transaction_handling.py
│   ├── templates/reset_password.html  # 311-line HTML page; calls /api/password-resets/email-link/verify
│   └── web.py (11)                  # Public route serving reset_password.html
├── tests/                           # ~38 test files (claimed 433 tests in TEST_DOCUMENTATION.md)
│   ├── README_old.md, TEST_DOCUMENTATION.md
│   └── test_*.py, manual_test_*.py
├── docs/                            # ~30 .md files + 2 .gs (Apps Script) files
├── migrations/                      # Alembic
│   ├── README, alembic.ini, env.py, script.py.mako
│   └── versions/                    # 62 migration files (multiple "merge_multiple_alembic_heads" entries)
├── functions/                       # 4 Cloud Functions (aac_import_trigger, ful_data_pull, ful_import_worker, wac_import_trigger), each with main.py + requirements.txt
├── cloud_tasks/ful-import-task*.json
├── archive/implementation_notes/    # OUT OF SCOPE per approved plan
├── brain-drain/                     # This skill (SKILL.md + references/)
├── scripts/                         # Cloud SQL setup PowerShell/bash, APA membership import, Liberty import, FUL test
├── _EXTRACTIONS/                    # This extraction (output)
├── ~17 root-level .md docs          # README, CLAUDE, AGENTS, WINDSURF, POSTGRESQL_*×7, STRIPE_*, GHL_*, etc.
├── ~15 root-level helper .py        # Operational tools (not part of test suite)
├── session_2026-04-29.md, RECOVERY.md  # Created by Claude Code session
└── CLAUDE.md                        # Stark Industries operating protocol (vibe-coded)
```

---

### Build / Run Commands

**EVIDENCE** — Cloud Run / Docker
**Source:** `Dockerfile:34` — `gunicorn --bind :$PORT --workers 1 --threads 8 pharmacybooks_api:app`

**EVIDENCE** — App Engine
**Source:** `app.yaml:5` — `gunicorn -b :$PORT app:app --timeout 600`

**EVIDENCE** — Local development
**Source:** `app.py:6-7` — `app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))`

**EVIDENCE** — Tests
**Source:** `pytest.ini:1-13`:
```
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short --strict-markers
markers =
    integration: Integration tests that test multiple components together
    unit: Unit tests that test individual functions
```

**CLAIM** — Test count is 433
**Source:** `tests/TEST_DOCUMENTATION.md:5` — "comprehensive coverage for the PharmacyBooks API, with **433 tests**". Not verified by running pytest.

---

### Project Purpose (per README)

**CLAIM** — Six-step onboarding flow
**Source:** `README.md:11-21` — Registration → Stripe Checkout → Webhook creates business+user → Activation key emitted → User unlocks desktop app → Stripe webhooks manage subscription. Verified at code level in `auth.py`, `payments.py`, `integrations.py`, and `models.py` event listeners (see 03-AGENT-LOOP).

**CLAIM** — Multi-tenant via NCPDP+NPI uniqueness
**Source:** `README.md:25` "Businesses uniquely identified by NCPDP+NPI". Verified at `models.py:85-87` `__table_args__ = (db.UniqueConstraint('ncpdp', 'npi', name='_ncpdp_npi_uc'),)`.

**CLAIM** — RBAC (admin / regular / super admin)
**Source:** `README.md:26` and verified in `models.py:185-220, 241-245`, `admin.py:21-86` decorators `admin_required` / `super_admin_required`.

---

### Deployment Targets

**EVIDENCE** — Three concurrent deploy paths
**Source:**
1. **Cloud Run** — `Dockerfile`, gunicorn, env vars and Cloud SQL via TCP
2. **App Engine Standard** — `app.yaml`, runtime python311, Cloud SQL via Unix socket at `/cloudsql/`, automatic_scaling 1–10 instances, readiness/liveness probes hit `/api/health`
3. **Cloud Functions** — `functions/*/main.py` (separate deploys, FUL import scheduler chain)

**EVIDENCE** — Sensitive deploy-config items in plain YAML
**Source:** `app.yaml:31-43` embeds:
- `CLOUD_SQL_CONNECTION_NAME: "pharmacy-books-desktop:us-east1:userdata"`
- `STRIPE_PUBLISHABLE_KEY: "pk_live_51RuwBA…00vPEgThKm"` (live key — publishable is technically safe to expose, but it pins the live Stripe account in version control)
- `STRIPE_MONTHLY_PRICE_ID: "price_1S97QEEaLoVMLk3oQd2lKsZ0"`
- `STRIPE_MONTHLY_CONCIERGE: "price_1SGN4QEaLoVMLk3oFbZFzgyc"`
- `GHL_CONTACT_CUSTOM_FIELD_ID_MAP: '{...}'` — full JSON of GHL custom field UUIDs (24 fields)
- GCS bucket `inclusion_lists`, PBM Corrections spreadsheet ID

**INFERENCE** — Sensitive-but-not-secret config values are committed to the repo (see 10-RAW-FINDINGS for surfaced concerns).

---

## Open Questions

1. Is App Engine + Cloud Run intended to coexist long-term, or is one the migration target?
2. Why are both MySQL and PostgreSQL drivers shipped — is the project mid-migration, or is multi-DB a runtime requirement?
3. Why is `cloud-sql-proxy.exe` (32MB) and `upload-trace.log` (5MB) committed to the repo?
4. Why is `session.json` committed at the repo root?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist (paths confirmed via Read/Glob)
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented
