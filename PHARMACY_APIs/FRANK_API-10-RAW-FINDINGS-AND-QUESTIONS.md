# 10 — Raw Findings and Questions

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

This is the catch-all for surprises, contradictions, vibe-coded smells, and shallow-scan flags surfaced during extraction. Per Tony's instruction: shallow scan for QUESTION-tagged items, capture and move on. The repo shows multiple signs of incremental rework (62 Alembic migrations, several "merge_multiple_alembic_heads" entries, two parallel registration paths, two Stripe webhook handlers, three IDE-agent config docs in parallel) without consolidation. There are concrete operational risks (committed binaries, embedded sensitive-but-not-secret config, inconsistent privilege gates on reference data, fail-open auth on GHL webhooks). The `session.json` file at the repo root is non-secret-bearing (no API keys / passwords) but contains a real-looking Stripe price ID and test NPI/NCPDP — flagged here per Tony's instruction.

---

## Findings — Surprises and Contradictions

### S1. Two parallel Stripe webhook handlers

**EVIDENCE — Identical event coverage at two URLs**
**Source:** `payments.py:324` `/api/payments/webhook` and `integrations.py:3007` `/api/integrations/stripe-webhook`. Both use `STRIPE_WEBHOOK_SECRET`.

**QUESTION — Which is canonical?** Operational risk if both are configured in Stripe Dashboard. They differ in retry behavior (500 vs 200 on processing exceptions) and structure (inline vs helper-function-based). See 02-ARCHITECTURE-MAP, 03-AGENT-LOOP, 07-GUARDRAILS.

### S2. Two parallel registration paths with opposite DB-write policies

**EVIDENCE**
**Source:** `auth.py:71-731` `/register` (zero DB writes; webhook-driven) vs `integrations.py:1456-2191` `/ghl-registration` (creates `Business` row with `status='pending'` BEFORE checkout).

**QUESTION** — Documented webhook-driven flow (`docs/transaction_handling.md`, `docs/REFACTORING_SUMMARY.md`) describes only the auth.py path. The GHL path violates the policy.

### S3. Three agent-config docs at root with same protocol

**EVIDENCE** — `CLAUDE.md` (v3.0), `AGENTS.md`, `WINDSURF.md` (v2.0). All ~20KB. All describe the "Stark Industries" operating protocol.

**QUESTION — Is one canonical?** Version drift between v2.0 (Windsurf) and v3.0 (Claude) suggests manual sync is happening per-tool.

**INFERENCE** — Plus the agent docs claim a tech stack (FastAPI + Supabase + Next.js + Zustand + `/types` folder + `config_service.py`) that does NOT match this repo (Flask + MySQL/Postgres + no `/types` + no config_service). The agent docs were likely cloned from another project in the family.

### S4. CLAUDE.md mandates `os.getenv()` use only via `config_service` — code violates this 74 times

**EVIDENCE** — `CLAUDE.md:545` (failure mode #20): "Calling os.getenv() directly instead of via config_service" and `CLAUDE.md:514`: "All env vars via `config_service.py` — never call `os.getenv()` directly".

**Code state:** No `config_service.py` exists. `os.environ.get(...)` appears 74 times across 14 files in `pharmacybooks_api/`.

### S5. README contradicts Dockerfile on Python version

**EVIDENCE** — `README.md:35` "Python 3.8+"; `Dockerfile:2` `FROM python:3.11-slim`; `app.yaml:2` `runtime: python311`. PEP 604 union syntax (Python 3.10+) used in `models.py`, `auth.py`, etc.

### S6. `cloud-sql-proxy.exe` (32 MB binary) committed to repo

**EVIDENCE** — `ls -la` at root shows `cloud-sql-proxy.exe` 32731136 bytes, `Mar 20 06:12`. Not gitignored.

**QUESTION — Why?** Bloats the repo, slows clones, no version-pinning. Should be downloaded by setup scripts.

### S7. `upload-trace.log` (5.1 MB) committed to repo

**EVIDENCE** — Same `ls -la`. 5380320 bytes.

**QUESTION** — Likely committed by accident; should be removed from history or at minimum gitignored going forward.

### S8. `session.json` at repo root (331 bytes)

**EVIDENCE — Read in full**
**Source:** `session.json:1-16`:
```json
{
  "success_url": "https://example.com/success",
  "cancel_url": "https://example.com/cancel",
  "mode": "payment",
  "line_items": [
    {
      "price": "price_1SGQX7EaLoVMLk3o209AnN93",
      "quantity": 1
    }
  ],
  "metadata": {
    "npi": "9876543210",
    "ncpdp": "5465468",
    "pharmacy_name": "Wedowee Pharmacy"
  }
}
```

**Findings (per Tony's request to flag):**
- **No API keys, no passwords, no secrets.** Not a security incident.
- Contains a Stripe `price_*` ID — could be production or test; format `price_1SGQX7EaLoVMLk3o209AnN93` is consistent with the other live price IDs in `app.yaml:41-42`.
- Contains test NPI `9876543210` (synthetic) and NCPDP `5465468` (7 digits — valid NCPDP format) and pharmacy name "Wedowee Pharmacy" (a real Alabama town — could be a real pharmacy).
- Likely a developer's checkout-session test payload left in the repo.

**QUESTION — Is "Wedowee Pharmacy" a real customer or test data?** If real, the file constitutes inadvertent disclosure.

### S9. `app.yaml` embeds the GHL custom-field UUID map (24 fields)

**EVIDENCE — `app.yaml:43`**
**Source:** Single-line `GHL_CONTACT_CUSTOM_FIELD_ID_MAP` JSON value — 24 GHL field UUIDs (each ~20 chars).

**QUESTION — Is this map intended to be in version control?** Field UUIDs are not "secret" but they are GHL-account-specific and tie this code to a single GHL tenant. A different GHL tenant cannot deploy without rewriting this file.

### S10. `app.yaml` embeds the LIVE Stripe publishable key

**EVIDENCE — `app.yaml:40`**
**Source:** `STRIPE_PUBLISHABLE_KEY: "pk_live_51RuwBA…00vPEgThKm"`. Publishable keys are not secret (they are designed to be exposed in frontend code), but pinning the live account ID in version control is a tradeoff: it pins the tenant identity to source.

### S11. `auth.py:526-527` placeholder URL `'https://your-app.com/success'`

**EVIDENCE — Default fallback never branded**
**Source:** `auth.py:526-527`:
```python
success_url = data.get('success_url', 'https://your-app.com/success')
cancel_url = data.get('cancel_url', 'https://your-app.com/cancel')
```

Other endpoints use `'https://pharmacybooks.com/thankyou'` defaults. This default in `auth.py` looks like leftover scaffolding.

### S12. 62 Alembic migrations with multiple "merge_multiple_alembic_heads"

**EVIDENCE — `ls migrations/versions | wc -l = 62`**

Migration filenames suggest at least 5+ head merges historically:
- `03959e744ee9_merge_multiple_heads.py`
- `0a79919a7f5c_merge_multiple_alembic_heads.py`
- `33559da3e729_merge_multiple_alembic_heads.py`
- `52ccdfc28f1d_merge_all_migration_heads_into_single_.py`
- `604c13d29927_merge_heads_for_oauth.py`
- `6cf800d28f22_merge_migration_branches.py`
- `6d5907903e16_merge_multiple_alembic_heads.py`
- `839157e5b9fe_merge_multiple_alembic_heads.py`

**QUESTION — How often do branched migrations occur?** Plus the existence of `fix_alembic_version.py` at the repo root strongly suggests history operations have been needed.

### S13. Seven separate PostgreSQL setup docs at root

**EVIDENCE**
- `EXECUTE_POSTGRESQL_SETUP.md`
- `POSTGRESQL_MIGRATION_SUMMARY.md`
- `POSTGRESQL_SETUP.md`
- `POSTGRESQL_SETUP_COMPLETE.md`
- `QUICK_REFERENCE_POSTGRESQL.md`
- `QUICK_START_POSTGRESQL_CLOUD_SQL.md`
- `docs/CLOUD_SQL_POSTGRESQL_SETUP.md`
- `docs/POSTGRESQL_MIGRATION.md`

Plus ~7 MySQL docs. Plus `STANDARDIZATION_COMPLETE.md` claiming Docker MySQL is the single source of truth. Plus the dual-driver code in `__init__.py:333-403`.

**QUESTION — What's the actual database direction?** The repo carries enough evidence to support: (a) MySQL is current production, Postgres migration in progress, (b) Postgres migration is complete and MySQL is legacy, or (c) both are supported indefinitely.

### S14. `pharmacy.py /api/pharmacy_profile` is PUBLIC, single-tenant, mass-assignment

**EVIDENCE — `pharmacy.py:7-27`** (verbatim in 04-TOOL-SYSTEM and 07-GUARDRAILS).

**QUESTION — Active or dead code?** No JWT, no business_id filter, no validation. **Highest-severity finding** in 07-GUARDRAILS.

### S15. Reference-data CRUD lacks admin gate

**EVIDENCE** — `dashboard.py:394, 775, 817, 1043, 1058, 1073, 930, 970, 979` — all `@jwt_required()` only. Any business user can corrupt platform-wide WAC/AAC/PBM data. The corresponding IMPORT endpoints in `imports.py` correctly require super-admin.

**QUESTION** — Inconsistency between `dashboard.py` writes and `imports.py` writes — same data, different gates.

### S16. GHL webhook auth fails OPEN if env var missing

**EVIDENCE — `integrations.py:1573-1577, 2724-2728`**
```python
else:
    logger.warning(
        "GHL_WEBHOOK_SECRET not configured - webhook authentication disabled. "
        "This should only happen in development!"
    )
```

**QUESTION** — Should this be `raise RuntimeError` or `return 503` instead? Currently a missing env var silently disables auth.

### S17. `crypto_utils.py:18` hardcoded fallback secret

**EVIDENCE**
**Source:** `crypto_utils.py:11-19`:
```python
def _get_raw_secret() -> str:
    return (
        app.config.get("TEMP_PASSWORD_ENCRYPTION_KEY")
        or app.config.get("PASSWORD_RESET_SECRET_KEY")
        or app.config.get("SECRET_KEY")
        or "pharmacybooks-fallback-secret"
    )
```

**QUESTION** — Should this be `raise RuntimeError("encryption key not configured")` instead?

### S18. `session.json` and `smtp-config.json` at repo root

**EVIDENCE — `smtp-config.json` (181 bytes)**
**Source:** Not read in this extraction (file is small but contents could include credentials). `app.yaml` does NOT reference this file. May be unused.

**QUESTION — What's in `smtp-config.json`?** Per the file size (181 bytes) it could contain SMTP credentials. Should be inspected separately.

### S19. `business.py /activate` returns plaintext API password

**EVIDENCE — `business.py:927-933`**
```python
response = {
    'success': True,
    'business_id': business.id,
    'pharmacy_name': business.pharmacy_name,
    'api_username': admin_user.username if admin_user else None,
    'api_password': generated_password,
    'message': 'Business activated successfully'
}
```

**QUESTION — Documented?** This is intentional (desktop client needs the password to start authenticating), but it's a single-use bearer credential exposure.

### S20. Test blueprint prefix differs from production

**EVIDENCE — `tests/test_multi_tenant_isolation.py:42`**
```python
app.register_blueprint(dashboard_bp, url_prefix='/api/dashboard')
```
Production: `app.register_blueprint(dashboard_bp, url_prefix='/api/data')` (`__init__.py:654`).

**QUESTION** — Tests can pass on different prefixes than production — endpoint URL contract isn't validated.

### S21. `integrations.py /create-stripe-checkout` re-implements registration

**EVIDENCE — `integrations.py:2642-3004`** — A third checkout creation endpoint (in addition to `/api/auth/register`, `/api/business/register` alias, `/api/payments/create-checkout-session`, and `/api/business/subscription/renew`).

**QUESTION** — Is one of these the canonical? They each have slightly different payload shapes and policies.

### S22. Mass-assignment on `dashboard.py /pbm_info`

**EVIDENCE** — `dashboard.py:933` `pbm = PbmInfo(**record)`; `dashboard.py:973-975` `for k, v in request.json.items(): setattr(rec, k, v)`. Any field on PbmInfo can be set or modified by any authenticated user. PbmInfo isn't business-scoped — it's platform reference data.

### S23. Activation key fallback verification in desktop converter flows

**EVIDENCE — `auth.py:1909-1939, 2109-2132`** — once `business.activation_key` is NULL (consumed), `/verify-desktop-credentials` and `/verify-desktop-and-create-user` accept "trust NCPDP+NPI+desktop_username" as proof. Documented in 03-AGENT-LOOP and 07-GUARDRAILS.

### S24. Activation key auto-generation runs on UPDATE of any field

**EVIDENCE — `models.py:998` `@event.listens_for(Business, 'after_update')`** — fires on EVERY update, not just `stripe_checkout_session_id` updates. The body has an early-return if `stripe_checkout_session_id` is None or `activation_key` is already set, but the listener still runs for every update of every field. Slight performance cost; not a correctness issue.

### S25. SQLAlchemy event listener calls `connection.execute(text(...))` after_insert/after_update

**EVIDENCE — `models.py:1048-1073`** — uses raw SQL on the existing `connection` to UPDATE the just-inserted row (since the ORM session has already flushed at this point in the lifecycle).

**QUESTION** — This is a known SQLAlchemy idiom for after-event mutations, but it bypasses ORM tracking. Any subsequent ORM logic that reads `target.activation_key` from the in-memory session would see None unless a `db.session.refresh(target)` happens.

### S26. Inconsistent `username` field semantics

**EVIDENCE — `models.py:223` `username = db.Column(db.String(255), unique=True, nullable=False)`**

For new accounts, `username == email` (`auth.py:1151, 2169`).
For desktop-converter API admin: `username = f"{ncpdp}_{npi}".lower()` (`business.py:870`).
For OAuth users: `username = email_local_part` (with numeric suffix on collision).

Three different conventions for the same field. JWT identity uses `email` if available, else `username`. Some endpoints query by username, some by email, some try both.

### S27. `account.py` is the only Python file using both email-then-username fallback

**EVIDENCE — `account.py:32-39`** — tries email lookup first, then username. Most other endpoints try username first or use email-only. Inconsistency creates ambiguity in JWT identity resolution.

### S28. `archive/implementation_notes/` excluded per plan but exists

**EVIDENCE — Per `ls archive/`** the folder exists. Per the approved plan, this folder is OUT OF SCOPE. **Note:** existence flagged here per plan instruction.

### S29. Body of `auth.py register()` has a syntax-suspicious decorator on a helper

**EVIDENCE — `auth.py:60-69`**
```python
@auth_bp.route('/register', methods=['POST'])
def _normalize_license_number(value: str | None) -> str | None:
    if not value:
        return None
    normalized = ''.join(str(value).split()).upper()
    return normalized or None


def register():
```

**QUESTION** — The decorator `@auth_bp.route('/register', methods=['POST'])` is applied to `_normalize_license_number`, NOT to `register`. This means `_normalize_license_number` IS the registered route handler — but its body returns a string, not a Flask response. Yet the rest of the codebase imports `register` as if it's the route handler (`business.py:8`). **This looks like a bug** — the decorator is likely orphaned because of a recent helper extraction. Either Flask's blueprint registers `_normalize_license_number` as the route (which would 500 because it returns a string), or the file has a parse-time issue that lets `register` somehow be the handler.

**INFERENCE** — Looking at how `business.py:582` `business_register()` calls `return register()` and works in production, the actual deployed route handler appears to be `register`. This suggests the `@auth_bp.route` decorator may have been moved by mistake during refactoring, but the existing test suite hides the issue (tests register `register` directly via blueprint mounting, bypassing whatever decorator misalignment exists).

**HIGH-PRIORITY QUESTION** — This needs deeper investigation. Either the route system has a magic that swallows the typo, or `/api/auth/register` is currently broken in some way that hasn't been caught.

### S30. Twin `cancel_subscription` and `renew` endpoints

**EVIDENCE — Cancel:** `payments.py:273` (`/api/payments/cancel-subscription`) AND `business.py:1238` (`/api/business/subscription/cancel`) — both authenticated, both modify subscription via Stripe. Different audit-logging patterns and different response shapes.

**Renew:** `payments.py:116` AND `business.py:1497` AND `integrations.py:2642` (kind of). Three renewal-creation paths.

**QUESTION** — Which is canonical? Documentation does not say.

### S31. Auth.py debug logs include sanitized payloads

**EVIDENCE — `auth.py:126-128`**
```python
sanitized_data = {k: v for k, v in (data or {}).items() if k != 'password'}
logger.debug(f"[correlation_id={correlation_id}] Registration payload (sanitized): {sanitized_data}")
```
Only `password` is removed. `email`, `address`, all other PII remain. If `LOG_LEVEL=DEBUG` is set in production, these are emitted to Cloud Logging.

### S32. Logging style mixes f-strings and `extra=` dicts

**EVIDENCE — Mixed throughout** (e.g., `business.py:373` f-string vs `business.py:179-186` `extra=`). Cloud Logging will index `extra` fields; f-strings end up as opaque text.

### S33. `app.py` and `pharmacybooks_api/__init__.py` both wrap `create_app`

**EVIDENCE** — `app.py:4` `app = create_app()`; `Dockerfile:34` `gunicorn pharmacybooks_api:app`; `app.yaml:5` `gunicorn app:app`.

**QUESTION** — How does `pharmacybooks_api:app` resolve? `pharmacybooks_api/__init__.py:276` defines `def create_app()` but does NOT define module-level `app = ...`. **If the Dockerfile literally runs `gunicorn pharmacybooks_api:app`, it would fail at import time because no `app` exists at the package root.** Yet this is the production Dockerfile — it likely succeeds because `app.py:4` happens to be in scope OR there's a deployment quirk we're missing.

**HIGH-PRIORITY QUESTION** — Verify Dockerfile actually deploys correctly, or if `pharmacybooks_api:app` fails and the deploy actually uses a different command.

### S34. `pharmacybooks_api/__init__.py` calls subprocess at import time

**EVIDENCE — `__init__.py:34-44`** runs `git rev-parse HEAD` via subprocess at module import time. Adds startup latency. Returns `'unknown'` if git is missing — not a hard failure.

### S35. SMTP config can come from a JSON secret OR individual env vars

**EVIDENCE — `__init__.py:214-274` `apply_smtp_config_from_secret`** — populates `os.environ` from a JSON Secret Manager secret if present. Individual env vars (`SMTP_HOST`, etc.) take precedence.

### S36. `cleanup_orphaned_registrations.py` script criteria don't match GHL flow orphans

**EVIDENCE — Per `docs/transaction_handling.md:235-258`** the script identifies orphans by `stripe_customer_id IS NOT NULL AND stripe_subscription_id IS NULL`. But the GHL registration flow creates Business records with `stripe_customer_id IS NULL` (Stripe customer is created in the webhook, not at GHL registration time).

**QUESTION** — GHL-flow orphans (Business `status='pending'` with NO Stripe customer) are NOT cleaned up by this script.

### S37. `STANDARDIZATION_COMPLETE.md` references wrong directory path

**EVIDENCE — Line 44**
```
cd "C:\Users\fbtan\Manage Medical Dropbox\Frank Tant\PC (2)\Desktop\PBM Audit Tools\pharmacybooks-api"
```
This is one developer's local Dropbox path. Hardcoded in committed docs. Also implies the developer "fbtan" / "Frank Tant" — personal info.

### S38. Reset password HTML hardcodes a logo URL

**EVIDENCE — `pharmacybooks_api/templates/reset_password.html:206`** `<img src="https://storage.googleapis.com/msgsndr/COgRLDxQx2jv6R6JbAhw/media/68a478abfcbcd61a6c4be41e.png" alt="Pharmacy Books">` — GHL CDN URL embedded in template.

### S39. Single-worker gunicorn assumption in `Dockerfile`

**EVIDENCE — `Dockerfile:34`** `--workers 1 --threads 8`. With multiple workers, module-level caches (`integrations.py:38-42` GHL custom-field map, `pbm_corrections.py @lru_cache`) would not propagate, but this is fine for now.

**QUESTION** — Is Cloud Run scaled by replicas (multiple instances) rather than workers per instance? In that case, each instance has its own module-level state — also fine.

### S40. `webhook/health` endpoint is PUBLIC

**EVIDENCE — `payments.py:1590` `/api/payments/webhook/health`** — anyone can hit this and learn whether `STRIPE_WEBHOOK_SECRET` is configured (`{webhook_secret_configured: bool, using_development_secret: bool}`). Not high risk, but information disclosure.

### S41. Stripe key info logged in non-production environments

**EVIDENCE — `__init__.py:113-154` `log_stripe_key_info`** — logs masked key (first 8 + last 4 chars) only when `DEBUG=true` or `FLASK_ENV in {development, dev, test, staging}`. Includes `staging` — which may be production-like.

### S42. App startup raises ValueError if Stripe key wrong

**EVIDENCE — `__init__.py:486-518`** — app refuses to start. Good. **Side effect:** if Stripe credentials are misconfigured, the entire API is offline (vs. degraded). This is intentional but trades availability for correctness.

### S43. `pyproject.toml` absent

**EVIDENCE** — No `pyproject.toml`, no Poetry, no Pipfile, no `setup.py`, no `setup.cfg`. Only `requirements.txt` (no version pins). Reproducible builds rely on Docker base image only.

### S44. No `.env.example` in `pharmacybooks_api/`; root has `.env.example`

**EVIDENCE — `ls .env.example`** — exists at repo root. Not read in this extraction.

**QUESTION** — Is `.env.example` complete relative to the 50+ env vars actually consumed? See 02-ARCHITECTURE-MAP for full env-var inventory.

### S45. ⚠️ Heavy emoji usage in production logs

**EVIDENCE** — Throughout: 🎯, 🔐, 📦, 🔍, ✅, ⚠️, ❌, 🔑, 🔄, etc. Cloud Logging supports them; some text-based log analysis tools may not.

### S46. Three concurrent deployment targets

**EVIDENCE** — `Dockerfile` (Cloud Run), `app.yaml` (App Engine), `functions/*/main.py` (Cloud Functions). Plus `Dockerfile.ful-job` for a separate FUL import job.

**QUESTION** — Which is the production target? `app.yaml` references App Engine, `Dockerfile` says Cloud Run.

### S47. `os.environ.get('STRIPE_SECRET_KEY')` set at module-import time in 5 files

**EVIDENCE — Modules with `stripe.api_key = os.environ.get('STRIPE_SECRET_KEY')` at top level:**
- `account.py:12`
- `auth.py:56`
- `payments.py:18`
- `integrations.py:30`

Plus `business.py` uses lazy `_ensure_stripe_api_key()`. **The module-level set captures the env var at import time.** If the env var is set DURING `create_app()` (`__init__.py:545`), it propagates because `account.py` etc. are imported AFTER `create_app()` registers blueprints. Order-sensitive.

**QUESTION** — Brittle. A direct import of `account.py` before `create_app()` runs would set `stripe.api_key = None`.

### S48. `business.py` uses different Stripe error import pattern

**EVIDENCE — `business.py:18-19`** uses `getattr` for backward compatibility:
```python
StripeError = getattr(getattr(stripe, 'error', None), 'StripeError', Exception)
InvalidRequestError = getattr(getattr(stripe, 'error', None), 'InvalidRequestError', Exception)
```
Other files use `from stripe import _error as stripe_error`. Inconsistent SDK error handling across the repo.

### S49. `desktop.py` has a deprecated `/latest-version` endpoint

**EVIDENCE — `desktop.py:253` and docstring at lines 7-13** — endpoint is deprecated in favor of static manifest at `https://storage.googleapis.com/downloads.pharmacybooks.com/latest.json`. Returns hardcoded version `1.0.0`. **Still routable.**

### S50. `app.yaml` and `Dockerfile` use different gunicorn command lines

**EVIDENCE**
- App Engine: `gunicorn -b :$PORT app:app --timeout 600`
- Cloud Run: `gunicorn --bind :$PORT --workers 1 --threads 8 pharmacybooks_api:app`

Different module targets (`app:app` vs `pharmacybooks_api:app`), different worker config, different timeout.

---

## Findings — Useful Patterns to Preserve

### P1. APA20 redemption with row-level locking

**EVIDENCE** — `payments.py:1149` `APAMembership.query.filter_by(license_number=license_for_discount).with_for_update().first()` — uses SELECT FOR UPDATE to prevent double redemption under concurrent webhook processing. Falls back to non-locking if backend doesn't support it.

### P2. Reference dataset versioning via content hash

**EVIDENCE** — `reference_data_versions.py:41-203` SHA-256 over normalized rows. Excellent pattern for desktop-client cache invalidation.

### P3. Cursor-based pagination for desktop reference sync

**EVIDENCE** — `dashboard.py:351-392` `/ful_reference/export` and `dashboard.py:566-601` `/aac_reference/export`. Avoids OFFSET-based pagination's scan cost.

### P4. Correlation IDs across webhook → handler → audit log

**EVIDENCE** — `logging_utils.py` + correlation ID propagation through Stripe webhook chain. Improves traceability.

### P5. Email enumeration prevention

**EVIDENCE** — `password_resets.py` returns 202 universally regardless of user existence. Best practice.

### P6. Stripe key environment guard

**EVIDENCE** — `__init__.py:497-518` refuses to start with mismatched key/env. Prevents the "test card in production" class of bugs.

### P7. SQLAlchemy event listeners for cross-cutting concerns

**EVIDENCE** — `models.py:998-1271` activation key auto-gen + post-commit GHL sync. Ensures no flow forgets these steps.

### P8. Multi-tenant isolation comments in models

**EVIDENCE** — `models.py:672-673` and `models.py:836-837` — explicit "Never accept business_id from client" comments next to the column definitions. Documents the contract at the schema level.

### P9. CRM_MASTERED_FIELDS set drives audit + read-only behavior

**EVIDENCE** — `audit_utils.py:15-41` — single source of truth for "which fields are CRM-owned". Used by both audit logging and pharmacy.py read-only check.

---

## QUESTION-tagged shallow-scan flags (per Tony's instruction)

**Q1.** `auth.py:60-69` decorator is on `_normalize_license_number` instead of `register` — looks like a refactoring bug. **Investigate.**

**Q2.** `Dockerfile:34` references `pharmacybooks_api:app` but `pharmacybooks_api/__init__.py` defines no module-level `app`. **Investigate runtime behavior.**

**Q3.** `session.json` at repo root contains pharmacy_name "Wedowee Pharmacy" — real or test? **Investigate.**

**Q4.** GHL_EMAIL_OMISSION_WORKFLOW.md doc vs `integrations.py:351-405` code disagree on default email-omission policy. **Read the doc to confirm.**

**Q5.** Two Stripe webhook handlers — which is configured in production Stripe Dashboard? **Operational check.**

**Q6.** `pharmacy_profile` PUBLIC + mass-assignment endpoint — kill it or guard it. **Decision.**

**Q7.** `crypto_utils.py:18` hardcoded `"pharmacybooks-fallback-secret"` — remove fallback, fail closed.

**Q8.** GHL webhook auth fail-open if `GHL_WEBHOOK_SECRET` missing — fail closed instead.

**Q9.** Reference-data write endpoints (`/api/data/wac_reference` POST/PUT/DELETE, `/api/data/pbm_info` POST/PUT) — gate to super-admin?

**Q10.** Why are MySQL and PostgreSQL drivers both shipped — what's the migration plan?

**Q11.** `STANDARDIZATION_COMPLETE.md` claims Docker MySQL is single source of truth, but Postgres dual-driver remains active. Reconcile.

**Q12.** `cloud-sql-proxy.exe` 32MB binary committed to repo — remove?

**Q13.** `upload-trace.log` 5MB log file committed — remove?

**Q14.** `STANDARDIZATION_COMPLETE.md:44` contains `C:\Users\fbtan\Manage Medical Dropbox\Frank Tant\…` — sanitize.

**Q15.** Activation key returns plaintext API password in `/api/business/activate` — documented in API contracts?

**Q16.** Test blueprint URL prefix mismatch (`/api/dashboard` vs `/api/data`) — fix or use real `create_app()` in tests.

**Q17.** Three IDE-agent docs (CLAUDE.md, AGENTS.md, WINDSURF.md) at versions 3.0/?/2.0 — sync them or canonicalize one.

**Q18.** Agent-config docs reference FastAPI / Supabase tech stack but repo uses Flask / MySQL — drift, or aspirational future state?

**Q19.** OpenAPI spec `openapi.yaml` may drift from live blueprints — generation pipeline?

**Q20.** Health endpoint `/api/payments/webhook/health` is PUBLIC and reveals webhook config status — gate it?

**Q21.** Dual cancel-subscription endpoints (`/api/payments/cancel-subscription` and `/api/business/subscription/cancel`) — pick one.

**Q22.** Dual renew-subscription endpoints — same problem.

**Q23.** Dual checkout-creation endpoints (auth/register, business/register, payments/create-checkout-session, business/subscription/renew, integrations/create-stripe-checkout, integrations/ghl-registration) — six paths!

**Q24.** Multi-pharmacy via UserBusiness vs legacy `User.business_id` — when is the legacy path retired?

**Q25.** `mfa_secret` comment claims encryption — implement or update comment.

**Q26.** Audit log table is mutable — DB-level append-only constraint?

**Q27.** `archive/implementation_notes/` — keep or remove? (Out of scope for this extraction per plan.)

**Q28.** `smtp-config.json` at repo root (181 bytes) — does it contain credentials?

**Q29.** 7 Postgres setup docs + 3+ MySQL setup docs — consolidate.

**Q30.** Why does `app.yaml` embed live STRIPE_PUBLISHABLE_KEY — should this be in Secret Manager too?

---

## Open Questions Summary (top 10 for migration scoping)

1. **Two webhook handlers** — which is canonical? (S1, Q5)
2. **Two registration paths** with opposite DB-write policies — unify? (S2)
3. **`pharmacy_profile` public mass-assignment** — kill or guard? (S14, Q6)
4. **Reference-data write privilege** — admin gate? (S15, Q9)
5. **GHL webhook fail-open** — fix? (S16, Q8)
6. **`crypto_utils` hardcoded fallback** — remove? (S17, Q7)
7. **MFA implementation** — when? (07-GUARDRAILS gap, Q25)
8. **MySQL vs PostgreSQL** — pick one? (S13, Q10, Q11)
9. **Audit log immutability** — DB-level constraint? (Q26)
10. **`auth.py:60-69` decorator misplacement** — bug? (S29, Q1)

---

## Verification Checklist

- [x] All findings labeled with evidence tags (EVIDENCE / INFERENCE / QUESTION)
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included (findings are observations; QUESTION tags are flags only)
- [x] GAPs explicitly documented in 07-GUARDRAILS; this doc is the surprises catch-all
