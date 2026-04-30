# 02 — Architecture Map (PRIMARY FOCUS)

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

This is a Flask SaaS API with 13 blueprints registered behind 13 URL prefixes, backed by SQLAlchemy with 16 model classes (Business, User, UserBusiness, PendingRegistration, LocalDesktopUser, DesktopClientVersion, APAMembership, PharmacyProfile, UserData, WacReference, FulReference, AACReference, ReferenceDatasetVersion, ReportFile, PbmInfo, AuditLog, PasswordResetToken). Three SQLAlchemy ORM event listeners drive activation-key auto-generation and post-commit GHL sync. The system integrates externally with seven services: **Stripe** (billing), **Go High Level / GHL** (CRM + email automation), **Google Cloud Secret Manager** (secrets), **Google Cloud Storage** (reference data), **Cloud SQL** (MySQL or PostgreSQL), **Liberty Software** (prescription source), **Google Sheets** (PBM corrections inbox), and **SMTP** (password reset emails). Two parallel Stripe webhook handlers exist at different URLs. Companion **desktop client** runs out-of-process and consumes a dedicated `/api/desktop/*` surface plus the activation/business endpoints.

This is the migration dependency graph for any Supabase rebuild.

---

## Findings

### Module Topology — Blueprint Registry

**EVIDENCE — All 13 blueprints registered in `create_app()`**
**Source:** `pharmacybooks_api/__init__.py:638-678`

```
auth_bp           → /api/auth        (auth.py — 2731 lines)
business_bp       → /api/business    (business.py — 1863 lines)
pharmacy_bp       → /api              (pharmacy.py — 120 lines, LEGACY)
payments_bp       → /api/payments    (payments.py — 1692 lines)
dashboard_bp      → /api/data        (dashboard.py — 1085 lines)
account_bp        → /api/account    (account.py — 123 lines)
imports_bp        → /api/import     (imports.py — 3227 lines)
admin_bp          → /api/admin      (admin.py — 1372 lines)
integrations_bp   → /api/integrations (integrations.py — 4593 lines)
health_bp         → /api/health     (health.py — 250 lines)
password_resets_bp → /api/password-resets (password_resets.py — 383 lines)
desktop_bp        → /api/desktop    (desktop.py — 841 lines)
web_bp            → (no prefix)     (web.py — 11 lines, serves /reset-password HTML)
```

**INFERENCE — Module sizes correlate with feature scope.** The three giant modules (`integrations.py` 4593 / `imports.py` 3227 / `auth.py` 2731) are unfocused — `integrations.py` mixes GHL CRM operations, GHL webhook handlers, AND a second Stripe webhook handler. `imports.py` mixes upload parsing, GCS data pulls, Liberty integration, and Medicaid rate calculation. `auth.py` mixes registration, OAuth, MFA stubs, pending registrations, multi-pharmacy linking, and verification flows.

---

### Cross-Module Function Calls (control-flow edges)

**EVIDENCE — Inter-blueprint calls** (grep-verified)
- `business.py:8` `from .auth import register` — `business_register()` aliases `register()`. Both `/api/auth/register` and `/api/business/register` execute the same function.
- `business.py:9` `from .models import db, Business, User, AuditLog, APAMembership`
- `auth.py:13` `from .models import db, User, Business, APAMembership, LocalDesktopUser, UserBusiness, PendingRegistration`
- `auth.py:14` `from . import log_stripe_key_info` — package-level shared util.
- `payments.py:1281` (in `admin.py:1281`) `from .payments import _ensure_activation_key` — admin manual-trigger calls into payments helpers.
- `admin.py:1303` `from .integrations import sync_to_ghl` — admin → integrations.
- `payments.py:1359` `from .integrations import sync_to_ghl` — payments → integrations (post-checkout).
- `dashboard.py:944` `from .gcs_loader import load_pbm_info_from_bucket` — dashboard → GCS loader (lazy import).
- `dashboard.py:994` `from .imports import _import_aac_from_gcs` — dashboard → imports (lazy import).
- `desktop.py:35-39` `from .pbm_corrections import …` — desktop → Sheets helper.
- `models.py:1183` `from pharmacybooks_api import integrations` — **models module imports integrations from the package root inside an event-listener callback** to perform GHL sync after commit. This is a back-edge from data layer to integration layer.

**INFERENCE — The dependency graph is a DAG with one back-edge.** `models.py` event listener depends on `integrations.sync_to_ghl`; everything else flows top-down (blueprints → models → utils). The back-edge is intentional but unusual.

---

### SQLAlchemy Schema (full data contract)

**EVIDENCE** — All models defined in `models.py:1-995` plus 3 event listeners at `models.py:998-1271`. Tables and key columns enumerated below. Column types and constraints copied verbatim where decisions matter for migration.

#### `businesses` (`Business`) — `models.py:13-183`

Primary tenant entity. Uniquely identified by composite (`ncpdp`, `npi`).

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | Integer PK | — | |
| `ncpdp` | String(30) | NOT NULL | Immutable after creation |
| `npi` | String(30) | NOT NULL | Immutable after creation |
| `pharmacy_name` | String(255) | NOT NULL | |
| `address`, `address_line2`, `city`, `state`, `zip`, `phone`, `fax`, `email` | String | nullable | Standard contact |
| `pharmacy_license_number`, `pharmacist_license` | String(100) | nullable | CRM-mastered (audit-logged) |
| `contact_person`, `contact_person_last_name` | String(255) | nullable | CRM-mastered |
| `business_email` | String(255) | nullable | Distinct from `email`; preferred for billing |
| `website_url` | String(500) | nullable | |
| `time_zone`, `preferred_contact_method` | String(50) | nullable | CRM-mastered |
| `country`, `pharmacy_software_system`, `role_in_pharmacy`, `mobile_number` | nullable | Onboarding metadata |
| `ghl_contact_id`, `ghl_company_id` | String(255) | nullable | GHL CRM linkage |
| `date_of_registration` | DateTime | nullable | Backend-set on creation |
| `status` | String(50) | NOT NULL, default `'pending'` | Activation lifecycle: `pending` → `active` (suspended is also valid) |
| `stripe_customer_id`, `stripe_subscription_id`, `stripe_checkout_session_id` | String(255) | nullable | |
| `subscription_status` | String(50) | nullable | Stripe billing state: `active` / `trialing` / `past_due` / `canceled` / `unpaid` / `incomplete` |
| `promotion_code` | String(100) | nullable | Audit trail of promo applied |
| `has_used_trial` | Boolean | NOT NULL, default `False` | Single trial per NCPDP+NPI |
| `trial_end_date` | DateTime | nullable | |
| `current_period_end` | DateTime | nullable | |
| `cancel_at_period_end` | Boolean | NOT NULL, default `False` | |
| `activation_key` | String(64) | UNIQUE, indexed, nullable | Set NULL after consumption (`business.py:899`) |
| `created_at`, `updated_at` | DateTime | NOT NULL | UTC, auto-managed |

**Constraints:** `UniqueConstraint('ncpdp', 'npi', name='_ncpdp_npi_uc')`.

**Method:** `is_in_good_standing()` — `True` if `status == 'active'` AND (`subscription_status in ['active', 'trialing']` OR `(subscription_status == 'canceled' AND now < current_period_end)`).

**Relationships:**
- `users` → `User` (1:M, via `User.business_id`, backward-compat single-business linkage)
- `user_businesses` → `UserBusiness` (1:M, multi-pharmacy, `cascade='all, delete-orphan'`)
- `local_desktop_user` → `LocalDesktopUser` (1:1, cascade)
- `desktop_clients` → `DesktopClientVersion` (1:M, cascade)
- `user_data` → `UserData` (backref)
- `report_files` → `ReportFile` (backref)
- `password_reset_tokens` → `PasswordResetToken` (backref)

#### `users` (`User`) — `models.py:185-303`

| Column | Type | Notes |
|---|---|---|
| `id` | PK | |
| `username` | String(255), UNIQUE, NOT NULL | Globally unique (not scoped to business) |
| `password_hash` | String(255), nullable | Nullable for OAuth-only users |
| `email` | String(255), UNIQUE, indexed, nullable | Used as username for new accounts |
| `mfa_enabled` | Boolean, default False | |
| `mfa_secret` | String(255), nullable | **Comment claims "encrypted in production" — no encryption code found in repo** |
| `google_id`, `microsoft_id` | String(255), UNIQUE, indexed, nullable | OAuth provider IDs |
| `oauth_email` | String(255), nullable | |
| `is_admin` | Boolean, NOT NULL | Business-scoped admin |
| `is_super_admin` | Boolean, NOT NULL | Platform admin; MUST have `business_id IS NULL` |
| `business_id` | FK → `businesses.id`, nullable | Backward-compat single-business; deprecated for multi-pharmacy |
| `created_at` | DateTime | |

**Methods:** `get_businesses()`, `get_primary_business()`.

#### `user_businesses` (`UserBusiness`) — `models.py:306-351`

Junction table for multi-pharmacy.

| Column | Notes |
|---|---|
| `id` PK | |
| `user_id` FK→users, NOT NULL, indexed | |
| `business_id` FK→businesses, NOT NULL, indexed | |
| `role` String(50), default `'user'` | `'admin'` or `'user'` (per-business role) |
| `is_primary` Boolean | One primary per user |
| `joined_at` DateTime | |

`UniqueConstraint(user_id, business_id, name='uq_user_business')`.

#### `pending_registrations` (`PendingRegistration`) — `models.py:354-493`

Manual-verification onboarding flow (alternative to Stripe-first flow).

| Column | Notes |
|---|---|
| `id` | |
| `ncpdp`, `npi` | NOT NULL |
| `email` | NOT NULL, indexed |
| `pharmacy_name`, `phone`, `address*`, `city`, `state`, `zip`, `fax`, `contact_person*`, `pharmacy_license_number`, `pharmacist_license`, `country`, `pharmacy_software_system`, `role_in_pharmacy`, `mobile_number`, `website_url` | |
| `activation_key`, `desktop_username`, `is_desktop_converter`, `business_id` | Desktop-converter linking (existing business with new admin) |
| `status` String(50), default `'pending_verification'`, indexed | `pending_verification` / `approved` / `rejected` / `expired` / `completed` |
| `verified_by_user_id` FK→users, `verified_at`, `verification_notes` | |
| `activation_token` String(64), UNIQUE, indexed | `secrets.token_urlsafe(32)`, 7-day TTL |
| `activation_token_expires_at`, `activation_link_sent_at` | |
| `created_at`, `updated_at` | |

`UniqueConstraint(email, ncpdp, npi, name='uq_pending_reg_email_business')`.

#### `local_desktop_users` (`LocalDesktopUser`) — `models.py:496-525`

One desktop username per business — used as a verification factor for desktop password resets.

#### `desktop_client_versions` (`DesktopClientVersion`) — `models.py:528-570`

Telemetry from desktop clients: `business_id` + `device_id` + `app_version` + `platform` + `os_version` + `app_channel` + `build_hash` + `metadata_json` + `last_ip_address`.

#### `apa_memberships` (`APAMembership`) — `models.py:573-636`

Drives `APA20` promotional discount eligibility.

| Column | Notes |
|---|---|
| `id` | |
| `license_number` String(64) UNIQUE | Normalized: spaces stripped, uppercased (`@validates('license_number')`) |
| `membership` String(120) NOT NULL | |
| `membership_expires` Date, nullable | |
| `first_name`, `last_name` | NOT NULL |
| `discount_redeemed` Boolean, default False, server_default `'0'` | |
| `discount_redeemed_at` DateTime | |
| `discount_redeemed_business_id` Integer | Not a FK — soft reference |
| `created_at`, `updated_at` | |

#### `pharmacy_profile` (`PharmacyProfile`) — `models.py:638-659`

LEGACY single-tenant table — exposed at PUBLIC `/api/pharmacy_profile` GET/POST in `pharmacy.py:7-27`. **Not multi-tenant; no business_id; no auth.** See 07-GUARDRAILS for the security implication.

#### `user_data` (`UserData`) — `models.py:661-735`

Prescription/dispensing records — the bulk PHI table.

| Column | Notes |
|---|---|
| `id` | |
| `business_id` FK→businesses, NOT NULL, indexed | Multi-tenant; "Never accept business_id from client" comment at line 672 |
| `date_dispensed` Date | |
| `script` String(255) | Rx number, used as dedup key in upsert |
| `drug_name`, `drug_ndc`, `qty`, `medicaid_rate`, `acq`, `acq_net`, `difference`, `insurance`, `source_file`, `status`, `total_paid`, `payment`, `new_paid`, `bin`, `expected_paid`, `new_owed` | Source-file-driven |
| `customer_name`, `first_name`, `last_name`, `customer_id`, `day_supply`, `authorization_number`, `customer_group_number`, `script_pcn`, `compound`, `drug_340b`, `drug_preferred_vendor`, `insurance_rejection_codes`, `primary_network_reimbursement_id`, `gpi`, `nadac`, `awp`, `payer_type` | Desktop-side enrichment |
| `aac_date_used`, `wac_date_used`, `ful_year_used`, `ful_month_used`, `rate_source`, `medicaid_rate_calculated_at`, `medicaid_rate_original` | Rate-recalc audit trail |

**No encryption-at-rest at the column level.** Customer name, prescription, DOB-adjacent fields stored plaintext.

#### `report_files` (`ReportFile`) — `models.py:826-849`

Generated PDF reports per business.

#### Reference data tables

- `wac_reference` (`WacReference`) — `models.py:737-775` — NDC/effective_date unique; contains `wac_by_unit` Computed column with persisted SQL expression.
- `ful_reference` (`FulReference`) — `models.py:778-799` — NDC/year/month unique; `aca_ful` Numeric(12,6).
- `aac_reference` (`AACReference`) — `models.py:865-883` — NDC/aac_date unique.
- `pbm_info` (`PbmInfo`) — `models.py:851-863` — bin/pbm_name/pcn/state lookup.
- `reference_dataset_versions` (`ReferenceDatasetVersion`) — `models.py:802-824` — checksum + row_count + latest_upload_at per dataset (computed in `reference_data_versions.py`).

#### `audit_logs` (`AuditLog`) — `models.py:885-948`

HIPAA audit trail. record_id is `String(128)` to support both numeric and Stripe `cs_test_*` style identifiers.

| Column | Notes |
|---|---|
| `id` | |
| `user_id` FK→users, nullable | nullable for system actions |
| `username` String(255), NOT NULL | "system_event_listener", "stripe_webhook", etc. for system actions |
| `table_name` | e.g., `'businesses'`, `'webhook_events'`, `'ghl_operations'`, `'business_activation'`, `'activation_verification'`, `'webhook_events'`, `'apa_memberships'`, `'stripe_checkout_sessions'`, `'password_reset_tokens'`, `'users'` |
| `record_id` String(128) | |
| `field_name` | |
| `old_value`, `new_value` Text | Stringified |
| `action` String(50) | `'create'` / `'update'` / `'delete'` / `'activate'` / `'verify'` / `'sync'` / `'process'` / `'error'` |
| `ip_address`, `user_agent` | |
| `created_at` | |

Indexes: `(table_name, record_id)`, `user_id`, `created_at`.

#### `password_reset_tokens` (`PasswordResetToken`) — `models.py:951-994`

Two flavors keyed by `secret_type`:
- `'temporary_password'` — issued by support via `/api/admin/password-resets/temporary` (admin.py:95). Stores both `secret_hash` (Werkzeug PBKDF2) AND `secret_ciphertext` (Fernet of the plaintext) for desktop client poll-and-decrypt.
- `'email_link'` — issued via `/api/password-resets/email-link` (password_resets.py:81). Only `secret_hash` set; `secret_ciphertext` is None.

Other columns: `business_id`, `username`, `source` (default `'support'`), `issued_reason`, `issued_by_user_id`, `issued_by_username`, `expires_at`, `redeemed_at`, `revoked_at`, `revoked_reason`.

---

### SQLAlchemy ORM Event Listeners (control-flow side effects)

**EVIDENCE** — Three listeners on `Business` model
**Source:** `models.py:998-1271`

#### Listener 1: `_generate_activation_key_on_session_id_change` — `models.py:998-1079`

`@event.listens_for(Business, 'after_insert')` AND `'after_update'`

Trigger condition: `target.stripe_checkout_session_id` is non-null AND `target.activation_key` is None.

Action: Generates `secrets.token_urlsafe(32)` (43 chars), executes raw SQL `UPDATE businesses SET activation_key = :key WHERE id = :id` via `connection`, AND inserts an `audit_logs` row (raw SQL) with `username='system_event_listener'`, `new_value='[GENERATED]'` (key is NEVER written to logs).

#### Listener 2: `_detect_activation_key_creation` — `models.py:1082-1145`

`@event.listens_for(Business, 'before_insert' / 'before_update', propagate=True)`

Inspects SQLAlchemy attribute history to detect transition from null→value on `activation_key`. Sets a transient flag `target._trigger_ghl_sync = True`.

#### Listener 3: `_schedule_ghl_sync_after_commit` — `models.py:1148-1271`

`@event.listens_for(Business, 'after_insert' / 'after_update', propagate=True)`

If the transient flag is set, registers a one-time `db.session.after_commit` listener that:
1. Imports `from pharmacybooks_api import integrations` (back-edge)
2. Calls `integrations.sync_to_ghl(business, include_activation_key=True)`
3. Writes audit log entries to `audit_logs.table_name='ghl_operations'` for success / failure / exception
4. **Catches all exceptions** to "prevent transaction failures" — GHL outage cannot break the main transaction.

**INFERENCE** — Activation-key generation and GHL sync are coupled to `Business` ORM events. Any direct SQL UPDATE that bypasses ORM will NOT trigger these. Any future Supabase migration loses this auto-coupling unless replicated via DB triggers + edge functions.

---

### Request/Response Lifecycle (covered in 03-AGENT-LOOP)

See **03-AGENT-LOOP.md** for the canonical onboarding sequence (Registration → Stripe Checkout → Webhook → Business Update → Activation Key → GHL Sync → Account Activation).

---

### External Integration Map (Migration Dependency Graph)

This is the section that drives Supabase migration scoping. Each integration is enumerated with: direction, auth method, payload, retry behavior, and where it lives in code.

#### 1. Stripe (Billing)

| Aspect | Detail |
|---|---|
| Direction | Bidirectional — outbound API calls; inbound webhooks |
| Library | `stripe` Python SDK |
| Auth (outbound) | `stripe.api_key = os.environ.get('STRIPE_SECRET_KEY')` set at module import time in `auth.py:56`, `account.py:12`, `payments.py:18`, `integrations.py:30`, `business.py` (lazy `_ensure_stripe_api_key()`) |
| Auth (inbound webhooks) | `stripe.Webhook.construct_event(payload, sig_header, endpoint_secret)` — HMAC verification with `STRIPE_WEBHOOK_SECRET`; rejects with 400/403 on failure |
| Outbound endpoints called | `Customer.create`, `Subscription.retrieve`, `Subscription.modify` (cancel_at_period_end), `checkout.Session.create`, `billing_portal.Session.create`, `PromotionCode.list` |
| Inbound webhook events handled | `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `customer.subscription.trial_will_end`, `invoice.payment_succeeded`, `invoice.payment_failed` |
| Webhook URLs | TWO handlers: `POST /api/payments/webhook` (`payments.py:324`) AND `POST /api/integrations/stripe-webhook` (`integrations.py:3007`). They share the secret. **GAP: Stripe Dashboard configuration determines which one is hit; this is not enforced in code.** |
| Key validation | `__init__.py:53-86` `validate_stripe_api_key` rejects `pk_*` (publishable) and warns on `rk_*` (restricted); requires `sk_test_*` or `sk_live_*` |
| Environment guard | `__init__.py:497-518` raises `ValueError` at startup if test key in production or live key in non-prod (FLASK_ENV-driven) |
| Retry | None at app level — relies on Stripe's own webhook retry. App returns 200 even on internal exceptions (`integrations.py:3308`) to prevent Stripe re-tries from compounding errors. |

**Data flowing OUT to Stripe:** Business identity (NCPDP, NPI, pharmacy_name, email), customer metadata, **password_hash** (PBKDF2 from werkzeug, embedded in checkout session metadata for the auth.py path), promotion codes.

**Data flowing IN from Stripe webhooks:** customer_id, subscription_id, subscription status, period_end timestamps, trial timestamps, invoice IDs.

#### 2. Go High Level / GHL (CRM + Email Automation)

| Aspect | Detail |
|---|---|
| Direction | Bidirectional |
| Library | `requests` (no official SDK) |
| Base URL | `https://rest.gohighlevel.com/v1` |
| Auth (outbound) | `Authorization: Bearer {GHL_API_KEY}` from env / Secret Manager |
| Auth (inbound webhooks) | HMAC-SHA256 via `X-GHL-Signature` (preferred) OR static `X-Webhook-Token` (fallback), both compared against `GHL_WEBHOOK_SECRET` using `hmac.compare_digest` |
| Endpoints called | `GET /contacts/?email=…`, `POST /contacts/` (create), `PATCH /contacts/{id}` (update), `DELETE /contacts/{id}` |
| Inbound webhook URLs | `POST /api/integrations/ghl-registration` (creates Business pending+Stripe checkout — different policy from `/api/auth/register`); `POST /api/integrations/ghl-contact-updated`; `POST /api/integrations/ghl-status-sync` |
| Custom field map | Loaded from env var `GHL_CONTACT_CUSTOM_FIELD_ID_MAP` (JSON) — embedded in `app.yaml:43` (24 fields, e.g., `activation_key`, `ncpdp`, `npi`, `stripe_customer_id`, `pharmacist_license`); cached after first read |
| Retry | Exponential backoff (2/4/8s) for 429/500/502/503/504 in `create_ghl_contact` (max 3 attempts) and `sync_to_ghl` (variable). 10s default request timeout. |
| Behavior on missing key | `GHL_API_KEY` not set → ALL GHL calls return `{success: False}` with a warning log; app continues without GHL sync |
| Behavior on missing webhook secret | `GHL_WEBHOOK_SECRET` not set → **inbound webhook authentication is DISABLED with only a warning log** (`integrations.py:1574-1577`). Security risk in dev/test if production env var is also missing. |

**Data flowing OUT to GHL:** Business contact + custom fields including `activation_key`, `ncpdp`, `npi`, `stripe_customer_id`, `stripe_subscription_id`, `subscription_status`, `pharmacist_license`, `database_id`, `promotion_code`, `has_used_trial`, `trial_end_date`, `current_period_end`, `cancel_at_period_end`, `created_at`, `updated_at`, `address1`, `address_line2`, `city`, `state`, `postal_code`, `phone`, `mobile`, `email`, contact_person.

**Data flowing IN from GHL:** Contact updates (full profile incl. PHI-adjacent fields), registration form submissions (browser-form OR JSON, 302 redirect or 201 JSON response based on `Accept`/`Content-Type`).

#### 3. Google Cloud Secret Manager

| Aspect | Detail |
|---|---|
| Direction | Inbound (read-only) |
| Library | `google.cloud.secretmanager.SecretManagerServiceClient` |
| Auth | Application Default Credentials (ADC) on Cloud Run / App Engine |
| Function | `__init__.py:157-211` `access_secret(secret_id, project_id)` — falls back to env var first, then Secret Manager, then dev defaults |
| Secrets read | `JWT_SECRET_KEY`, `DB_USER`/`DB_PASSWORD` (or `_POSTGRES` variants), `DB_NAME`, `STRIPE_SECRET_KEY` / `STRIPE_SECRET_KEY_TEST`, `STRIPE_WEBHOOK_SECRET` / `STRIPE_WEBHOOK_SECRET_TEST`, `STRIPE_MONTHLY_PRICE_ID`, `STRIPE_PROMO_CODE_ID_APA20`, `GHL_API_KEY`, `SMTP_CONFIG_JSON` |
| Project ID | `os.environ['GOOGLE_CLOUD_PROJECT']`, defaults to `"pharmacy-books-desktop"` (`__init__.py:315`) |

#### 4. Cloud SQL (MySQL or PostgreSQL)

| Aspect | Detail |
|---|---|
| Driver | `mysql-connector-python` OR `psycopg2-binary` (selected at runtime by `DB_TYPE` env var; default `'mysql'`) |
| Connection types | App Engine Unix socket (`/cloudsql/{CLOUD_SQL_CONNECTION_NAME}`) OR TCP (`DB_HOST:DB_PORT`) OR SQLite via `SQLALCHEMY_DATABASE_URI` (tests/dev) |
| URI construction | `__init__.py:404-443` builds the URI with URL-encoded user/password |
| Cloud SQL instance | `pharmacy-books-desktop:us-east1:userdata` (hardcoded in `app.yaml:31, 47`) |

#### 5. Google Cloud Storage (GCS)

| Aspect | Detail |
|---|---|
| Library | `google.cloud.storage` |
| Bucket | `inclusion_lists` (default; overridable via `GCS_BUCKET_NAME` env) — also `pharmacy-books-data` referenced in `imports.py` |
| Auth | ADC |
| Modules using GCS | `gcs_loader.py` (PBM info, AAC), `imports.py` (WAC, AAC, FUL CSV/Excel/zip pulls) |
| Triggered by | API endpoints `POST /api/data/pbm_info/load`, `POST /api/data/aac_reference/load`, `POST /api/import/wac_reference_from_gcs`, `POST /api/import/aac_reference_from_gcs`, `POST /api/import/ful_reference_from_csv`, `POST /api/import/liberty/user_data`. Also Cloud Functions in `functions/` orchestrate scheduled imports. |

#### 6. SMTP (Outbound Email)

| Aspect | Detail |
|---|---|
| Library | stdlib `smtplib`, `email.message.EmailMessage` |
| Module | `email_utils.py:37-94` `send_email`; `email_utils.py:97-130` `send_password_reset_email` |
| Config | Env: `SMTP_HOST`, `SMTP_PORT` (default 587), `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM`, `SMTP_USE_TLS` (default True), `SMTP_USE_SSL` (default False). Consolidated `SMTP_CONFIG_JSON` secret can populate these (`__init__.py:214-274`). |
| Behavior on missing host | Logs the email body (with masked recipient) and returns False — no email sent. |
| Currently sent | Password reset emails from `password_resets.py:244` |
| Not sent | Activation emails — `auth.py:990` has a `# TODO: Send email with activation link` comment; activation emails are routed via GHL automation instead |

#### 7. Liberty Software API

| Aspect | Detail |
|---|---|
| Direction | Outbound |
| Module | `liberty_client.py:18-73` |
| Auth | HTTP Basic with `LIBERTY_USERNAME`/`LIBERTY_PASSWORD` + custom `Customer:` header (base64 of `{npi}:{location_key}` or pre-set `LIBERTY_CUSTOMER`) |
| Base URL | `LIBERTY_BASE_URL` env var |
| Used by | `imports.py:3006` `POST /api/import/liberty/user_data` and `pharmacybooks_api/scripts/run_ful_import.py` |

#### 8. Google Sheets API (PBM Corrections Inbox)

| Aspect | Detail |
|---|---|
| Library | `googleapiclient.discovery.build('sheets', 'v4', …)` |
| Auth | Service account from env `PBM_CORRECTIONS_SERVICE_ACCOUNT_INFO` (JSON or base64-JSON) OR `PBM_CORRECTIONS_SERVICE_ACCOUNT_JSON` (file path) OR ADC fallback |
| Spreadsheet | `PBM_CORRECTIONS_SPREADSHEET_ID` + sheet `PBM_CORRECTIONS_SHEET_NAME` (default `"PBM Corrections Inbox"`) |
| Operation | `values().append` only |
| Used by | Desktop endpoint `/api/desktop/pbm-corrections` (`desktop.py:745`) |

#### 9. Authlib OAuth (Google + Microsoft)

| Aspect | Detail |
|---|---|
| Library | `authlib.integrations.flask_client.OAuth` |
| Endpoints | `/api/auth/oauth/{google|microsoft}/authorize` and `/api/auth/oauth/{google|microsoft}/callback` |
| Discovery | OpenID Connect via `server_metadata_url` for both providers |
| Scopes | `openid email profile` |
| Auth env | `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`; `MICROSOFT_CLIENT_ID`/`MICROSOFT_CLIENT_SECRET` |
| Behavior on missing | OAuth disabled with warning; endpoints return 503 |
| State storage | Flask session (server-side; `SECRET_KEY` from env or JWT secret as fallback) |
| Frontend redirect | `WEBAPP_URL/dashboard` (default `http://localhost:5173/dashboard`) |

#### 10. Cloud Tasks + Cloud Functions (FUL Import Pipeline)

| Aspect | Detail |
|---|---|
| Trigger | Cloud Scheduler → `functions/ful_data_pull/main.py:trigger_ful_import` |
| Queue | `FUL_TASK_QUEUE` (default `'ful-import-queue'`) at `FUL_TASK_LOCATION` (default `'us-east1'`) |
| Worker | `functions/ful_import_worker/main.py` (Cloud Function), POST to `WORKER_URL` |
| Auth | Optional OIDC with `FUL_WORKER_SERVICE_ACCOUNT` |

---

### Configuration Surface

**EVIDENCE** — All `os.environ.get(...)` calls (50+ unique env vars used)

**Core / runtime:**
- `FLASK_ENV` (production/dev/test/staging) — drives many decisions
- `APP_ENV` (alt to FLASK_ENV)
- `DEBUG`, `DEBUG_ENDPOINTS`, `LOG_LEVEL`
- `PORT`
- `SQLALCHEMY_DATABASE_URI` (test override)
- `SECRET_KEY` (Flask sessions; falls back to JWT secret)
- `FLASK_SECRET_KEY`

**Database:**
- `DB_TYPE` (mysql / postgresql)
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`
- `DB_SOCKET_DIR` (default `/cloudsql`)
- `CLOUD_SQL_CONNECTION_NAME`

**GCP:**
- `GOOGLE_CLOUD_PROJECT`, `GCP_PROJECT`, `GCLOUD_PROJECT`, `PROJECT_ID`, `FUL_TASK_PROJECT`
- `GCS_BUCKET_NAME` (default `inclusion_lists` or `pharmacy-books-data`)

**Stripe:**
- `STRIPE_SECRET_KEY`, `STRIPE_SECRET_KEY_TEST`
- `STRIPE_WEBHOOK_SECRET`, `STRIPE_WEBHOOK_SECRET_TEST`
- `STRIPE_MONTHLY_PRICE_ID`
- `STRIPE_MONTHLY_CONCIERGE`
- `STRIPE_PROMO_CODE_ID_APA20`
- `STRIPE_PUBLISHABLE_KEY` (in app.yaml — used by frontend)
- `STRIPE_CHECKOUT_SUCCESS_URL`, `STRIPE_CHECKOUT_CANCEL_URL`
- `STRIPE_BILLING_PORTAL_RETURN_URL`

**GHL:**
- `GHL_API_KEY`
- `GHL_WEBHOOK_SECRET`
- `GHL_CONTACT_CUSTOM_FIELD_ID_MAP` (JSON of 24 field UUIDs)

**OAuth:**
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
- `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`
- `WEBAPP_URL` (default `http://localhost:5173`)

**SMTP:**
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM`, `SMTP_USE_TLS`, `SMTP_USE_SSL`
- `PASSWORD_RESET_LINK_BASE_URL`, `PASSWORD_RESET_LINK_EXPIRY_HOURS`, `PASSWORD_RESET_EMAIL_SUBJECT`
- Or consolidated: `SMTP_CONFIG_JSON` (Secret Manager)

**Encryption:**
- `TEMP_PASSWORD_ENCRYPTION_KEY`, `PASSWORD_RESET_SECRET_KEY` (alternates for crypto_utils)

**Liberty:**
- `LIBERTY_BASE_URL`, `LIBERTY_USERNAME`, `LIBERTY_PASSWORD`
- `LIBERTY_CUSTOMER` OR (`LIBERTY_NPI` + `LIBERTY_LOCATION_KEY`/`LIBERTY_API_KEY`)

**Sheets / PBM corrections:**
- `PBM_CORRECTIONS_SERVICE_ACCOUNT_INFO`, `PBM_CORRECTIONS_SERVICE_ACCOUNT_JSON`
- `PBM_CORRECTIONS_SPREADSHEET_ID`, `PBM_CORRECTIONS_SHEET_NAME`

**Cloud Tasks (FUL pipeline):**
- `FUL_TASK_QUEUE`, `FUL_TASK_LOCATION`, `FUL_WORKER_URL`, `FUL_WORKER_SERVICE_ACCOUNT`, `PBM_FUL_MONTH_LAG`
- `FUL_DOWNLOAD_URL` (override)

**Health endpoints:**
- `HEALTH_ENDPOINT_SECRET`

---

### Companion Desktop Client (Out-of-process, contracts only)

**EVIDENCE** — Endpoints consumed by the desktop client (visible from this repo's side):

| Endpoint | Purpose |
|---|---|
| `GET /api/desktop/latest-version?current_version=X.Y.Z` | DEPRECATED — desktop should use `https://storage.googleapis.com/downloads.pharmacybooks.com/latest.json` manifest instead |
| `GET /api/desktop/password-resets/temporary?username=…&business_id=…` | Poll for support-issued temporary password (returns `secret_ciphertext` for client to decrypt) |
| `GET /api/desktop/local-users?business_id=…` | Get desktop username configured for business |
| `PUT /api/desktop/local-users?business_id=…` | Set desktop username |
| `POST /api/desktop/versions?business_id=…` | Telemetry — record desktop version per device_id |
| `POST /api/desktop/pbm-corrections?business_id=…` | Submit PBM correction candidates → Google Sheet |
| `POST /api/auth/login` | JWT cookie + `access_token` in body |
| `POST /api/auth/verify-activation-key` | Verify key + report subscription state |
| `POST /api/auth/verify-desktop-credentials` | NCPDP+NPI+desktop_username+activation_key verification |
| `POST /api/auth/verify-desktop-and-create-user` | Self-service webapp account creation from desktop |
| `POST /api/business/activate` | Consume activation key + receive `api_username`/`api_password` (rotated) |
| `GET /api/business/info`, `GET /api/business/status`, `GET /api/business/by-activation-key` | Status checks |
| `POST /api/business/subscription/cancel`, `POST /api/business/subscription/renew`, `GET /api/business/subscription/portal` | Subscription mgmt |
| `GET /api/data/wac_reference/metadata`, `GET /api/data/ful_reference/export?cursor=…`, `GET /api/data/aac_reference/batch` | Reference data sync (cursor-based pagination) |
| `GET /api/account` | Combined account dashboard |

**Desktop is told to use ADC (Application Default Credentials) through the API only — desktop must NOT hit GCS or Cloud SQL directly.** `gcs_loader.py:7-11` reinforces this.

---

### Text-Based Architecture Diagram

```
                    ┌────────────────────────────────────┐
                    │     External Browser / Webapp       │
                    │     (CORS: localhost:3000/5173)     │
                    └───────────────┬─────────────────────┘
                                    │ HTTPS, JWT cookie
       ┌────────────────────────────┴────────────────────────────┐
       │                                                          │
┌──────▼──────┐    ┌──────────────────────────────────┐    ┌─────▼──────┐
│ Desktop App │    │     pharmacybooks-api (Flask)     │    │ GHL Webhook│
│  (separate  │───▶│   gunicorn 1×8 on Cloud Run/AE   │◀───│   Sender   │
│    repo)    │    │                                   │    └────────────┘
└─────────────┘    │  13 Blueprints (~22.5K LOC):       │
                   │   auth, business, payments,        │
                   │   dashboard, account, admin,       │
                   │   imports, integrations, desktop,  │
                   │   password_resets, health, web,    │
                   │   pharmacy (legacy)                │
                   │                                   │
                   │  ┌─────────────────────────────┐   │
                   │  │  models.py (16 models +     │   │
                   │  │   3 ORM event listeners)    │   │
                   │  │                             │   │
                   │  │   activation key auto-gen   │   │
                   │  │   GHL post-commit sync      │   │
                   │  └──────────┬──────────────────┘   │
                   └─────┬───────┼─────────┬────────────┘
              JWT cookie │       │ SQLAlchemy
                         │       ▼              │
                         │  ┌────────────┐       │
                         │  │  Cloud SQL │       │
                         │  │ MySQL OR   │       │
                         │  │ PostgreSQL │       │
                         │  └────────────┘       │
                         │                       │
                         │   Outbound REST/SDK   ▼
        ┌────────────────┴───────────┐  ┌─────────────────┐
        │                            │  │  Google Cloud:  │
   ┌────▼────┐  ┌────────┐  ┌────────┴┐ │  - Secret Mgr   │
   │ Stripe  │  │  GHL   │  │ Liberty │ │  - GCS          │
   │  API    │  │ API    │  │  API    │ │  - Cloud Run    │
   └────┬────┘  └───┬────┘  └─────────┘ │  - App Engine   │
        │          │                     │  - Functions    │
        ▼          ▼                     │  - Cloud Tasks  │
   Webhooks   Webhooks                   │  - Sheets API   │
   (×2 URLs)                             └─────────────────┘
                          ┌──────────┐
                          │   SMTP   │
                          │ outbound │
                          └──────────┘
```

---

## Open Questions

1. Why are there two Stripe webhook handlers (`/api/payments/webhook` and `/api/integrations/stripe-webhook`)? Which is currently configured in Stripe Dashboard? See 10-RAW-FINDINGS.
2. Why are there two registration paths with opposing DB-write policies (`/api/auth/register` writes ZERO rows; `/api/integrations/ghl-registration` creates a `Business` row with `status='pending'` BEFORE checkout)?
3. Is `mfa_secret` actually encrypted somewhere (claim in `models.py:231`), or is the comment aspirational?
4. Is the `pharmacy_profile` table (PUBLIC, single-tenant) still in active use, or is it dead code? See 07.
5. The PostgreSQL+MySQL dual-driver: is this active migration support, or vestigial? How is it tested?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (column-level encryption gap; webhook duplication; dual-policy registration; missing webhook secret = disabled auth)
