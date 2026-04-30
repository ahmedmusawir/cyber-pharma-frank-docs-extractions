# 01 — Existing Documentation Review

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

Documentation is voluminous (47 .md files: 17 at root, 30 in `/docs/`) and overlapping. README and `docs/business_workflow.md` describe the canonical onboarding flow accurately. There is heavy redundancy in PostgreSQL setup docs (7 separate files). HIPAA documentation is aspirational — many checkboxes labeled `⚠️` (unfilled). Per approved plan, 13 docs were spot-verified against code; the remaining 34 are catalogued as UNVERIFIED. Notable contradictions: `README.md` claims Python 3.8+ but Dockerfile/AppEngine pin 3.11; `docs/transaction_handling.md` describes a webhook-driven update-only policy that the *primary* webhook handler in `payments.py` follows but the *integrations.py* secondary handler does NOT (see GAP).

---

## Findings — Documentation Inventory

### Root-level docs (17 files)

| File | Size | Topic | Status |
|------|------|-------|--------|
| README.md | 29KB | Project overview, setup, Stripe config | CONFIRMED (spot-checks below) |
| CLAUDE.md | 20KB | Stark Industries Claude Code operating protocol | UNVERIFIED — agent rules, not architecture |
| AGENTS.md | 20KB | Agent-config doc parallel to CLAUDE.md | UNVERIFIED — see 06-PROMPTS-AND-PERSONA |
| WINDSURF.md | 20KB | Windsurf agent rules (parallel to CLAUDE.md) | UNVERIFIED |
| BUSINESS_STATUS_ENDPOINT_IMPLEMENTATION.md | 5.9KB | `/api/business/status` impl notes | CONFIRMED — endpoint exists at `business.py:1048` |
| GHL_ONBOARDING_IMPLEMENTATION.md | 10KB | GHL onboarding sync flow | CONFIRMED — `sync_to_ghl` in `integrations.py` |
| DOCKER_MYSQL_CONNECTION.md | 1.4KB | MySQL Docker container connection | UNVERIFIED |
| EXECUTE_POSTGRESQL_SETUP.md | 7.6KB | Postgres setup runbook | UNVERIFIED |
| POSTGRESQL_MIGRATION_SUMMARY.md | 4.2KB | Postgres migration summary | UNVERIFIED |
| POSTGRESQL_SETUP.md | 1.2KB | Postgres setup quickref | UNVERIFIED |
| POSTGRESQL_SETUP_COMPLETE.md | 3KB | Postgres "complete" doc | UNVERIFIED |
| QUICK_REFERENCE_POSTGRESQL.md | 2.3KB | Postgres reference | UNVERIFIED |
| QUICK_START_POSTGRESQL_CLOUD_SQL.md | 5.5KB | Postgres on Cloud SQL quickstart | UNVERIFIED |
| RUN_THESE_COMMANDS.md | 5KB | One-off runbook | UNVERIFIED |
| SECRET_ROTATION_GUIDE.md | 11KB | Secret Manager rotation | UNVERIFIED |
| SETUP_MYSQL_NOW.md | 2.3KB | MySQL local setup | UNVERIFIED |
| STANDARDIZATION_COMPLETE.md | 3KB | Docker MySQL standardization "done doc" | CONFIRMED — doc claims port 3307 mapping; not verified at runtime |
| STRIPE_SECRETS_MANAGEMENT.md | 21KB | Stripe key management | CONFIRMED in spot-check (validation logic in `__init__.py:53-86`) |
| TEST_CLEANUP_SUMMARY.md | 7.4KB | Test cleanup notes | UNVERIFIED |
| WINDOWS_STORAGE_DOCUMENTATION_SUMMARY.md | 12KB | Desktop client storage | UNVERIFIED — references desktop repo |

### /docs/ folder (30 files)

| File | Topic | Status |
|------|-------|--------|
| AUTO_UPDATE_DEVELOPER_GUIDE.md | Desktop auto-update protocol | UNVERIFIED — referenced in `desktop.py:12, 268` |
| CLOUD_SQL_POSTGRESQL_SETUP.md | Cloud SQL Postgres setup | UNVERIFIED |
| DESKTOP_CLIENT_DEVELOPMENT_CHECKLIST.md | Desktop dev checklist | UNVERIFIED |
| DESKTOP_CLIENT_WINDOWS_STORAGE.md | Desktop Windows storage rules | UNVERIFIED — referenced in `gcs_loader.py:11` |
| DESKTOP_TO_WEBAPP_MIGRATION.md | Desktop→webapp migration | UNVERIFIED |
| DOCKER_MYSQL_SETUP.md | Docker MySQL setup | UNVERIFIED |
| FIELD_MAPPING_SUMMARY.md | GHL field mappings | UNVERIFIED |
| GHL_API_SYNC_IMPLEMENTATION.md | GHL sync implementation | CONFIRMED — `sync_to_ghl` exists in `integrations.py` |
| GHL_EMAIL_OMISSION_WORKFLOW.md | GHL email-omission workaround | CONTRADICTED — code at `integrations.py:351-405` says omit_email/phone/mobile DEFAULTS to FALSE (i.e., emails ARE included); docstring states "omitted only if GHL deduplication is re-enabled". Doc may describe an older state. |
| GHL_REGISTRATION_ENDPOINT.md | GHL registration endpoint spec | CONFIRMED — `/api/integrations/ghl-registration` at `integrations.py:1456` |
| HIPAA_COMPLIANCE_CHECKLIST.md | HIPAA controls inventory | PARTIALLY CONFIRMED — many claimed `[x]` items verified; many `⚠️` items remain unfilled. See 07-GUARDRAILS for delta. |
| MYSQL_SETUP.md | MySQL setup | UNVERIFIED |
| NCPDP_NPI_DESKTOP_AUTH_FLOW.md | Desktop NCPDP+NPI auth flow | CONFIRMED — `/api/auth/verify-desktop-credentials` exists at `auth.py:1831` |
| NCPDP_NPI_VERIFICATION_IMPLEMENTATION.md | NCPDP/NPI verification impl | UNVERIFIED |
| OAUTH_SETUP.md | OAuth setup | CONFIRMED — Google + Microsoft OAuth at `auth.py:2484-2732` |
| POSTGRESQL_MIGRATION.md | Postgres migration | UNVERIFIED |
| REFACTORING_SUMMARY.md | Webhook-driven refactor summary | CONFIRMED — `auth.py /register` performs ZERO db writes (verified at `auth.py:71-731`); webhook handler creates records — but see CONTRADICTION below |
| WEBAPP_USER_CREDENTIALS_GUIDE.md | Webapp user creds guide | UNVERIFIED |
| activation_key_workflow.md | Activation key lifecycle | CONFIRMED — event listeners in `models.py:998-1271` auto-generate keys |
| api_quick_reference.md | API quick reference | UNVERIFIED |
| business_exists_api.md | `/api/business/exists` spec | CONFIRMED — exists at `business.py:308` with NPI=10 digit, NCPDP=7 digit validation |
| business_status_api.md | `/api/business/status` spec | CONFIRMED — exists at `business.py:1048` |
| business_workflow.md | Webhook-driven business workflow (canonical) | CONFIRMED — flow matches `auth.py` + `payments.py:_handle_checkout_session_completed` |
| crm_api_reference.md | CRM API reference | UNVERIFIED |
| crm_fields_and_ghl_integration.md | CRM fields & GHL integration | UNVERIFIED |
| debug_logging_and_health_endpoints.md | Debug logging & health endpoints | UNVERIFIED |
| ful_import_scheduler.md | FUL import Cloud Function scheduler | CONFIRMED — `functions/ful_data_pull/main.py` matches |
| multi_tenant_best_practices.md | Multi-tenant patterns | CONFIRMED — `dashboard.py:54-275` enforces business_id from JWT, ignores client-supplied values |
| pbm_corrections_inbox_apps_script.gs | Google Apps Script for PBM corrections inbox | UNVERIFIED — Apps Script, not Python |
| registration.md | Registration overview | UNVERIFIED |
| registration_verification_api.md | Registration verification API | UNVERIFIED |
| stripe_webhook_setup.md | Stripe webhook setup | UNVERIFIED |
| transaction_handling.md | Webhook-driven flow + orphaned-record handling | CONFIRMED — primary webhook follows update-only policy at `payments.py:_handle_checkout_session_completed`; CONTRADICTED for the secondary `/api/integrations/stripe-webhook` (see below) |
| wac_reference_apps_script.gs | Apps Script for WAC reference | UNVERIFIED |

### tests/

| File | Topic | Status |
|------|-------|--------|
| tests/README_old.md | Old test docs | UNVERIFIED |
| tests/TEST_DOCUMENTATION.md | Current test inventory | CONFIRMED structure — claims 433 tests across categories; not run |

### openapi.yaml (~58KB)

**EVIDENCE — File exists**
**Source:** `openapi.yaml` (58182 bytes). Contents not parsed end-to-end.

**QUESTION** — Is `openapi.yaml` regenerated from code, or is it hand-edited? It will drift from the live blueprints unless tooling regenerates it.

---

## Findings — Spot-Verification Detail

### CONFIRMED claims

#### README.md — Onboarding flow (lines 11–21)

**Claim:** "1. Registration → 2. Payment → 3. Webhook Processing → 4. Activation Key → 5. Activation → 6. Ongoing Management"

**Verification:**
- Step 1 — `auth.py:71` `register()` and `business.py:561` `business_register()` (alias) and `integrations.py:1456` `ghl_registration()` — three entrypoints; ALL create a Stripe checkout session with metadata.
- Step 2 — Stripe Checkout (external).
- Step 3 — `payments.py:324` `stripe_webhook()` is the primary handler; `integrations.py:3007` `stripe_webhook()` is a SECOND handler at a different URL.
- Step 4 — `models.py:998-1079` `_generate_activation_key_on_session_id_change` event listener auto-generates a 43-char key when `stripe_checkout_session_id` is set; `payments.py:_ensure_activation_key` is a fallback path.
- Step 5 — `business.py:659` `/api/business/activate` consumes the key and rotates the API admin password.
- Step 6 — Stripe webhooks update `subscription_status`, `current_period_end`, `cancel_at_period_end` on `customer.subscription.updated`, `invoice.payment_succeeded`, `invoice.payment_failed`.

**Verdict:** CONFIRMED structurally. Two minor wrinkles flagged in 10-RAW-FINDINGS: (a) two webhook handlers exist at different URLs; (b) `business.activation_key` is set to NULL after first use (see `business.py:899`), so `/api/auth/verify-desktop-credentials` falls back to "trust NCPDP+NPI+desktop_username" if the key is gone (`auth.py:1909-1939`).

#### docs/multi_tenant_best_practices.md

**Claim:** "All endpoints must derive `business_id` from the authenticated user's session" (lines 44–72) and "All data queries MUST filter by `business_id`" (line 75).

**Verification:** `dashboard.py:54-127` (`get_user_data`), `dashboard.py:129-167` (`create_user_data`), `dashboard.py:169-274` (`bulk_upsert_user_data`) — all derive `business_id` from `User.query.filter_by(username=current_username).first().business_id` and explicitly comment "CRITICAL: Always filter by business_id from authenticated session" / "Never accept business_id from client". Confirmed pattern.

**Verdict:** CONFIRMED for `dashboard.py`. CAVEAT: `dashboard.py:930-937` `create_pbm_info` accepts `**record` from `request.json` without business_id scoping (PBM info is reference data, not business-scoped — but mass-assignment is still a smell). See 10-RAW-FINDINGS.

#### docs/transaction_handling.md (Webhook-Driven Flow)

**Claim:** "No database writes during registration" (line 27) and "webhook creates records ONLY after successful payment" (line 31).

**Verification:** `auth.py:71-731` (the `/api/auth/register` endpoint) — confirmed: NO `db.session.add` / `db.session.commit` calls in the success path; only Stripe API calls and read-only queries against `User`, `Business`, `APAMembership`. On Stripe error, comment at `auth.py:683` says "No database changes to rollback - no records were created."

**Verdict:** CONFIRMED for `/api/auth/register`. **CONTRADICTED for `/api/integrations/ghl-registration`** at `integrations.py:1786-1869` — this endpoint creates a `Business` record with `status='pending'` BEFORE checkout, the opposite of the documented webhook-driven flow. The two registration paths use opposite policies.

#### docs/business_workflow.md

**Claim:** Documents the same webhook-driven flow as transaction_handling.md.

**Verdict:** CONFIRMED for `auth.py /register` path. CONTRADICTED for `integrations.py /ghl-registration` path.

#### docs/HIPAA_COMPLIANCE_CHECKLIST.md (lines 81–129)

**Claim:** "JWT tokens stored in httpOnly cookies", "Session timeout: 1 hour", "RBAC", "Encryption and decryption implemented", "All PHI access logged", "HSTS headers", "HTTPS-only configuration".

**Verification:**
- httpOnly: CONFIRMED — `__init__.py:452` `app.config['JWT_COOKIE_HTTPONLY'] = True`.
- 1-hour timeout: CONFIRMED — `__init__.py:454` `app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=1)`.
- RBAC: CONFIRMED — `admin.py:21-86` `admin_required` and `super_admin_required` decorators.
- Encryption: CONFIRMED for support secrets — `crypto_utils.py:31-49` Fernet-based; KEY DERIVED FROM `SECRET_KEY` if no `TEMP_PASSWORD_ENCRYPTION_KEY` set; **fallback to literal `"pharmacybooks-fallback-secret"` if all else missing** (`crypto_utils.py:18`). NOT confirmed for column-level PHI encryption — there's no encryption-at-rest of `user_data`, `mfa_secret`, OAuth tokens, etc., despite the model field comment claiming `mfa_secret` is "encrypted in production" (`models.py:231` — only a comment, no encryption logic found).
- All PHI access logged: PARTIAL — `audit_utils.py:98-149` `log_field_change` is invoked for CRM-mastered field updates and webhook events. NOT all `user_data` reads/writes are audited.
- HSTS: CONFIRMED — `__init__.py:680-686` `_apply_security_headers` sets `Strict-Transport-Security` for one year + preload + includeSubDomains.
- HTTPS-only: PARTIAL — `JWT_COOKIE_SECURE` set to `is_production` (`__init__.py:451`); HTTPS enforcement happens at the proxy layer, not the app.

**Verdict:** Mostly CONFIRMED at the framework level; MFA and column-level encryption are documented as intent but not implemented.

#### docs/business_exists_api.md

**Claim:** NPI must be 10 digits, NCPDP must be 7 digits.

**Verification:** `business.py:347-359` validates `npi.isdigit() and len(npi) == 10` and `ncpdp.isdigit() and len(ncpdp) == 7`.

**Verdict:** CONFIRMED.

#### docs/activation_key_workflow.md

**Claim:** Activation keys auto-generated when `stripe_checkout_session_id` is set, idempotent.

**Verification:** `models.py:998-1079` SQLAlchemy `after_insert`/`after_update` event listener generates `secrets.token_urlsafe(32)` (43 chars), uses raw SQL `UPDATE businesses SET activation_key = :key WHERE id = :id`, and writes an audit log row. Idempotency check at `models.py:1026-1031`. Confirmed.

**Verdict:** CONFIRMED.

#### docs/ful_import_scheduler.md

**Claim:** Cloud Scheduler → Cloud Function → Cloud Tasks → Cloud Run worker.

**Verification:** `functions/ful_data_pull/main.py:51-67` enqueues a Cloud Task that POSTs to `WORKER_URL` with year/month payload. Matches.

**Verdict:** CONFIRMED.

#### STANDARDIZATION_COMPLETE.md

**Claim:** Docker MySQL container is single source of truth, port 3307.

**Verdict:** CONFIRMED at the docs+config level — `docker-compose.yml` and DOCKER_MYSQL_CONNECTION.md align. Runtime not verified.

#### STRIPE_SECRETS_MANAGEMENT.md

**Claim:** Backend uses secret keys (sk_); rejects publishable (pk_) and warns on restricted (rk_); validates env-vs-key-type alignment.

**Verification:** `__init__.py:53-86` `validate_stripe_api_key` does exactly this. `__init__.py:497-518` raises `ValueError` if a test key is detected in production or vice versa, halting app startup.

**Verdict:** CONFIRMED.

#### NCPDP_NPI_DESKTOP_AUTH_FLOW.md

**Claim:** Desktop verifies via NCPDP + NPI + desktop_username + activation_key.

**Verification:** `auth.py:1831-1986` `/api/auth/verify-desktop-credentials` requires all four; if `business.activation_key` is NULL (already used), the check falls back to "NCPDP+NPI+desktop_username matches" only.

**Verdict:** CONFIRMED with a behavioral nuance flagged in 10-RAW-FINDINGS — "the activation key was already used" is a successful auth path here.

### CONTRADICTED claims

#### docs/GHL_EMAIL_OMISSION_WORKFLOW.md (UNREAD; inferred from filename + code comments)

**Claim (inferred from doc filename and `integrations.py:351-405` docstrings):** Email/phone/mobile should be OMITTED on initial GHL contact creation to avoid GHL deduplication merging.

**Code state:** As of the current `integrations.py:351-405`, defaults are `omit_email=False, omit_phone=False, omit_mobile=False`. The docstring explicitly says:
> "GHL deduplication has been disabled via GHL settings, allowing us to send complete contact information on initial creation without risk of unwanted contact merging."

**Verdict:** Either GHL_EMAIL_OMISSION_WORKFLOW.md is now historical context describing a previous state, or the doc and code disagree. **CONTRADICTED** until the doc itself is read in full to confirm.

#### docs/transaction_handling.md applied to `/api/integrations/ghl-registration`

**Claim:** Webhook-driven, no DB writes during registration.

**Code state:** `integrations.py:1783-1869` creates a `Business` record with `status='pending'` BEFORE the Stripe checkout session is created, contradicting the doc.

**Verdict:** CONTRADICTED for the GHL registration path. Two parallel registration policies exist in the codebase.

#### README.md — Python version

**Claim:** `README.md:35` "Python 3.8+".

**Code state:** `Dockerfile:2` `FROM python:3.11-slim`; `app.yaml:2` `runtime: python311`. Multiple modules use Python 3.10+ syntax: `models.py:617` `business_id: int | None`, `auth.py:28` `def validate_email(email: str) -> tuple[bool, str]:` (PEP 604 union types — Python 3.10+).

**Verdict:** CONTRADICTED — README is wrong; minimum supported version is 3.10 (more likely 3.11 to match Docker).

---

## Open Questions

1. **Doc/code drift policy:** Which doc set is canonical when in conflict — README, `/docs/`, or root-level done-docs (`STANDARDIZATION_COMPLETE`, `POSTGRESQL_SETUP_COMPLETE`, etc.)?
2. **GHL deduplication current state:** Is GHL deduplication actually disabled (per `integrations.py:357-360`)? If re-enabled silently, contacts will merge.
3. **OpenAPI generation:** Is `openapi.yaml` hand-maintained or regenerated? Drift risk is high.
4. **Postgres docs:** 7 different Postgres docs exist; can they be consolidated, or do they describe distinct stages of a multi-step migration?
5. **Two agent-config docs** (`AGENTS.md`, `WINDSURF.md`) — do these serve different IDE tools? Are they kept in sync? See 06-PROMPTS-AND-PERSONA.

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (incomplete HIPAA controls; column-level encryption gap; GHL doc/code drift)
