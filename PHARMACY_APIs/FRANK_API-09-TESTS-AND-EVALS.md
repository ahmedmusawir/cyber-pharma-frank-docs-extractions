# 09 — Tests and Evals

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

The test suite is integration-heavy and pytest-based. `tests/TEST_DOCUMENTATION.md` claims **433 tests** across **38 test files** organized into Core Workflow / Activation Key / GHL Integration / Webhook & Stripe / and "Additional" categories. Tests use SQLite in-memory and Flask test client; multi-tenant isolation is explicitly tested. There are NO unit tests for individual utility functions (e.g., `ndc_utils.normalize_ndc`, `crypto_utils`); the bulk of coverage is endpoint-level integration tests that touch JWT, DB, blueprint registration, and mocked external APIs. Several `manual_test_*.py` files exist for ad-hoc operational testing. **GAPs:** no load/performance tests, no security/authn fuzzing tests, no end-to-end tests against a real Stripe sandbox, no contract tests against the OpenAPI spec, no migration replay tests, no cross-DB-driver test (MySQL vs PostgreSQL parity).

---

## Findings

### Test Infrastructure

**EVIDENCE — pytest configuration**
**Source:** `pytest.ini:1-13`
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

**EVIDENCE — Test deps file exists**
**Source:** `requirements-test.txt` (75 bytes — minimal, contents not enumerated)

**EVIDENCE — Tests directory layout**
**Source:** `ls tests/`:
- `__init__.py` (file exists but length 0 — empty package marker)
- `README_old.md`, `TEST_DOCUMENTATION.md`
- `manual_test_business_status.py` (manual / ad-hoc)
- 38 `test_*.py` files
- No conftest.py at the tests root (per ls)

**EVIDENCE — Test app construction pattern**
**Source:** `tests/test_multi_tenant_isolation.py:23-54` (representative)
```python
@pytest.fixture
def app():
    app = Flask(__name__)
    app.config['TESTING'] = True
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
    app.config['JWT_SECRET_KEY'] = 'test-secret-key'
    jwt = JWTManager(app)
    db.init_app(app)
    app.register_blueprint(auth_bp, url_prefix='/api/auth')
    app.register_blueprint(admin_bp, url_prefix='/api/admin')
    app.register_blueprint(dashboard_bp, url_prefix='/api/dashboard')
    with app.app_context():
        db.create_all()
        if 'sqlite' in app.config['SQLALCHEMY_DATABASE_URI']:
            db.session.execute(db.text('PRAGMA foreign_keys=ON'))
            db.session.commit()
        yield app
```

**INFERENCE** — Tests construct their own Flask app per fixture rather than using `create_app()`. This decouples tests from Secret Manager / Stripe key validation / startup checks but means each test file may construct slightly different app configurations. The blueprint URL prefix is `/api/dashboard` here vs the production prefix `/api/data` (`__init__.py:654`) — **these are inconsistent**.

---

### Test File Inventory (38 files)

**EVIDENCE — File names per `ls tests/`**

#### Core Workflow Tests (~36 tests claimed)

- `test_business_activation_and_health.py`
- `test_business_exists.py`
- `test_business_info_endpoint.py`
- `test_business_status_endpoint.py`
- `test_complete_activation_workflow.py`
- `test_payments_status_endpoint.py`

#### Activation Key Tests (~46 tests claimed)

- `test_activation_key_workflow.py`
- `test_activation_key_event_listener.py`
- `test_activation_key_event_listener_ghl_sync.py`
- `test_activation_key_retry_logic.py`
- `test_activation_key_edge_cases.py`
- `test_enhanced_activation_key_logging.py`
- `test_trial_activation_key_immediate.py`
- `test_manual_activation_key_ghl_sync.py`

#### GHL Integration Tests (~90 tests claimed)

- `test_ghl_activation_key_sync.py`
- `test_ghl_contact_creation_workflow.py`
- `test_ghl_onboarding_sync.py`
- `test_ghl_registration_checkout.py`
- `test_ghl_registration_debug_logging.py`
- `test_ghl_registration_immediate_insert.py`
- `test_ghl_registration_required_fields.py`

#### Webhook & Stripe Tests (~89 tests claimed)

- `test_create_stripe_checkout.py`
- `test_environment_based_stripe_loading.py`
- `test_stripe_integrations_webhook.py`
- `test_stripe_key_logging.py`
- `test_stripe_key_validation.py`
- `test_stripe_webhook_missing_fields.py`
- `test_webhook_audit_logging.py`
- `test_webhook_enhancements.py`
- `test_webhook_event_handling.py`
- `test_webhook_logging_enhancements.py`

#### Other tests

- `test_audit_log_string_record_id.py`
- `test_crm_fields_and_integration.py`
- `test_desktop_app_activation_verification.py`
- `test_desktop_update_endpoint.py`
- `test_desktop_version_tracking.py`
- `test_health_endpoints.py`
- `test_local_desktop_user_endpoints.py`
- `test_logging_utils.py`
- `test_multi_tenant_isolation.py`
- `test_password_reset_email_flow.py`
- `test_pbm_corrections_sheet.py`
- `test_plan_type_support.py`
- `test_registration_transaction_handling.py`
- `test_registration_verification.py`

---

### Coverage Claims

**CLAIM — 433 tests, integration-heavy**
**Source:** `tests/TEST_DOCUMENTATION.md:5` — "comprehensive coverage for the PharmacyBooks API, with **433 tests** organized into functional categories".

**CLAIM — Coverage by workflow**
**Source:** `tests/TEST_DOCUMENTATION.md:74-100`:
- Registration: standard, GHL, field validation, duplicate detection, transaction integrity
- Payment: checkout creation, webhook handling, subscription status, trial handling, error recovery
- Activation: key generation, activation via key, retry logic, event listeners, GHL sync
- Account status: business status, subscription status, payment standing

**INFERENCE — Verification of "433"**: Not run during extraction. The file count (38) and claimed test counts per file (5–24 tests each) are consistent with 433 total.

---

### Patterns Observed

#### Multi-tenant isolation testing

**EVIDENCE — Explicit cross-business test fixtures**
**Source:** `tests/test_multi_tenant_isolation.py:64-120` creates two complete businesses with users + data, then asserts:
- User from Business 1 cannot see Business 2's data
- Client-supplied business_id is ignored
- Foreign key constraints prevent orphaned records
- Indexes exist on business_id columns

**INFERENCE** — Multi-tenant security is well-tested at the framework level. Whether every endpoint is covered for this property is not verifiable from the test names alone.

#### SQLite in-memory database

**EVIDENCE — Tests use `sqlite:///:memory:`**
**Source:** `tests/test_multi_tenant_isolation.py:29`. No use of MySQL/PostgreSQL in tests.

**GAP — No tests against MySQL or PostgreSQL drivers.** The dual-driver design (`__init__.py:404-443`) is untested at runtime. SQL syntax differences (e.g., `Computed` columns in `wac_reference`, `ON CONFLICT`/`ON DUPLICATE KEY` upserts in `gcs_loader.py:22 mysql_insert`) may not work identically across MySQL and PostgreSQL.

#### Mocking external services

**INFERENCE** — From `tests/TEST_DOCUMENTATION.md` and the test names (e.g., `test_stripe_integrations_webhook.py`, `test_ghl_*.py`), tests mock Stripe and GHL via `unittest.mock`. Not verified at line level.

---

### Test Helper Files

**EVIDENCE — `manual_test_business_status.py`**
**Source:** Filename present in `tests/` but starts with `manual_test_` (NOT picked up by `pytest` per `pytest.ini:python_files = test_*.py`). This is operational tooling, not part of the suite.

**EVIDENCE — `test_documentation.md` claims `payments_status_endpoint` is "NEW"**
**Source:** `tests/TEST_DOCUMENTATION.md:32, 87` — "test_payments_status_endpoint.py (8 tests) ⭐ NEW". Suggests recent additions.

---

### Operational Tooling (Root-Level, NOT pytest)

**EVIDENCE — 11 root-level `.py` scripts that look test-like but are operational**
**Source:** `ls` repo root:
- `check_migration_status.py`, `check_super_admin.py`, `check_testuser.py`
- `create_super_admin.py`, `create_test_user.py`, `create_checkout_session.py`
- `diagnose_preview.py`, `fix_alembic_version.py`
- `issue_temp_password.py`, `list_super_admins.py`
- `update_test_user_password.py`
- `verify_docker_mysql.py`, `verify_postgresql.py`
- `test_cloud_sql_connection.py`, `test_db_connection_simple.py`, `test_email_query.py`, `test_postgres_password.py`

**INFERENCE — Operational utilities, not tests.** Their names start with `test_` but they live at the repo root, not in `tests/`. They will not be picked up by pytest unless invoked manually.

---

### What is NOT tested

**GAP — No load / performance tests.** The `dashboard.py:107` paginates with `per_page` capped at 100, but there is no test demonstrating performance at high row counts.

**GAP — No security fuzzing.** Authentication bypass attempts, SQL injection vectors, mass-assignment exploits (especially against `pharmacy.py /pharmacy_profile` and `dashboard.py /pbm_info` POST/PUT) are not tested.

**GAP — No contract tests against `openapi.yaml`.** Drift between live blueprints and the spec is not caught automatically.

**GAP — No migration replay tests.** With 62 Alembic migrations and multiple `merge_multiple_alembic_heads.py` files, idempotency of the migration chain is not verified by tests.

**GAP — No tests for individual utility modules:**
- `ndc_utils.normalize_ndc` — no direct unit test
- `crypto_utils.encrypt_secret` / `decrypt_secret` — no direct unit test
- `audit_utils.mask_sensitive_value` / `mask_sensitive_dict` — no direct unit test (referenced in some integration tests)
- `email_utils._mask_email` / `send_email` — partial via password_reset_email_flow

**GAP — No tests for HTML template rendering.** `reset_password.html` (311 lines, JS-driven) is not tested.

**GAP — No end-to-end / Stripe sandbox tests.** All Stripe interactions are mocked; the live Stripe sandbox is not exercised in CI.

**GAP — No cross-DB parity tests.** Schema and queries are tested only on SQLite.

**GAP — No load-tested webhook handler.** The activation-key generation retry path (3 attempts) is unit-tested but not under realistic concurrent load.

**GAP — No tests for the legacy `pharmacy_profile` endpoints.** Their existence in production poses risk; no test asserts they are restricted (because they're not).

**GAP — No tests verifying the `manual_test_*` files are aligned with current API.** They could be silently broken.

**GAP — No test of the desktop client API contracts** as a black-box (the desktop repo presumably has its own tests).

---

### Test Marker Usage

**EVIDENCE — `pytest.ini` defines `integration` and `unit` markers**
**Source:** `pytest.ini:11-13`.

**INFERENCE — Marker usage actual rate not verified.** Most tests probably are `integration`-flavored (they touch the DB and full app). Without grepping all 38 files, the breakdown is unclear.

---

### Test Documentation Quality

**EVIDENCE — `tests/TEST_DOCUMENTATION.md` is comprehensive**
**Source:** `tests/TEST_DOCUMENTATION.md:1-100+` (read first 100 lines)

The doc enumerates per-file test counts, lists workflow-coverage status, suggests `pytest tests/`, `pytest tests/ --cov=pharmacybooks_api --cov-report=html` for coverage. **Coverage report itself was not run during extraction.**

**EVIDENCE — `tests/README_old.md` exists** — suggests the doc has been versioned (the new doc supersedes the old).

---

## Open Questions

1. What is the actual passing rate of the 433 claimed tests? (Not run during extraction.)
2. Why are blueprint URL prefixes inconsistent between tests (`/api/dashboard`) and production (`/api/data`)?
3. Is coverage measured in CI? `tests/TEST_DOCUMENTATION.md` mentions `--cov` but no CI config (e.g., `.github/workflows/`) is present in the repo.
4. Are there cross-DB-driver tests planned for the MySQL ↔ PostgreSQL transition?
5. Should `pharmacy_profile` legacy endpoints get explicit "removal-pending" tests (e.g., expect 410 Gone)?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (no load tests; no fuzzing; no contract tests; no cross-DB parity; no util-level unit tests; no migration replay)
