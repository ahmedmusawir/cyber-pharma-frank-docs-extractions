# 05 — Context and Memory (Reframed for Flask SaaS)

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

There is no LLM context window or token-budget management. The "memory" of this system is its persistent state in Cloud SQL plus per-request scratch state held in Flask. Three notable mechanisms span across requests: (1) **JWT cookies** (1-hour httpOnly session, server-stateless), (2) **OAuth Flask sessions** (server-side state for OAuth state token), and (3) **Correlation IDs via `contextvars`** for log tracing. Long-lived state lives in 16 SQLAlchemy tables. Reference-data caching strategy uses **content-hash checksums** in `reference_dataset_versions` so desktop clients can skip re-syncing unchanged data. There is **no Redis or external cache**. Module-level globals cache GHL custom-field map and Sheets service account credentials within a single gunicorn worker.

---

## Findings

### Session State (Per-User)

#### JWT Cookie Session

**EVIDENCE** — Cookie configuration
**Source:** `__init__.py:448-459`
```python
app.config['JWT_TOKEN_LOCATION'] = ['cookies']
app.config['JWT_COOKIE_SECURE'] = is_production       # HTTPS only in prod
app.config['JWT_COOKIE_HTTPONLY'] = True              # XSS protection
app.config['JWT_COOKIE_SAMESITE'] = 'Lax'             # CSRF protection
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=1)
app.config['JWT_COOKIE_NAME'] = 'access_token_cookie'
```

**EVIDENCE** — JWT identity payload
**Source:** Multiple `create_access_token(identity=identity)` calls
- `auth.py:1192` activation: `identity=pending_reg.email`
- `auth.py:1269` super admin login: `identity=user.email or user.username`
- `auth.py:1343` regular login: `identity=user.email or user.username`
- `auth.py:2148, 2208, 2621` other paths: same pattern

**INFERENCE** — JWT carries only the identity string (email or username); no roles, no business_id. Authorization is re-derived on every request via DB lookup `User.query.filter_by(...)`. This is stateless from the JWT perspective but adds a DB read to every authenticated request.

#### OAuth Server-Side Session

**EVIDENCE** — Flask session used for OAuth state
**Source:** `auth.py:2659, 2701` — `session['oauth_redirect_uri'] = redirect_uri` (stored in Flask's signed-cookie session backend); `auth.py:2681, 2723` — `session.pop('oauth_redirect_uri', None)`.

**EVIDENCE** — Flask SECRET_KEY for session signing
**Source:** `__init__.py:461` — `app.config['SECRET_KEY'] = os.environ.get('FLASK_SECRET_KEY', jwt_secret)` (falls back to JWT secret).

#### Cookie / session security headers

**EVIDENCE** — Session cookies parallel JWT settings
**Source:** `__init__.py:462-464`
```python
app.config['SESSION_COOKIE_SECURE'] = is_production
app.config['SESSION_COOKIE_HTTPONLY'] = True
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'
```

---

### Per-Request Context

#### Correlation IDs via `contextvars`

**EVIDENCE** — Thread/coroutine-safe correlation context
**Source:** `logging_utils.py:22-54`
```python
_correlation_id_context: ContextVar[Optional[str]] = ContextVar('correlation_id', default=None)

def set_correlation_id(correlation_id: str) -> None: ...
def get_correlation_id() -> Optional[str]: ...
def clear_correlation_id() -> None: ...
```

**INFERENCE** — Correlation IDs are typically request-scoped: registration uses `f"reg_{ip}_{ts}"` (`auth.py:102`); webhooks use `f"webhook_{ip}_{ts}"` then upgrade to `f"webhook_{event_id}"` and finally `f"{correlation_id}|session_{session_id}"` for chained tracing. The webhook correlation IDs propagate across event handlers and audit log entries.

#### Flask `request` context

**EVIDENCE** — Standard Flask request-scoped context
**Source:** Used throughout for `request.json`, `request.headers`, `request.remote_addr`, `request.cookies`. No custom request-context state added.

#### Audit log requester capture

**EVIDENCE** — IP and user agent extracted at request time
**Source:** `audit_utils.py:113-119`
```python
if request:
    ip_address = request.headers.get('X-Forwarded-For', request.remote_addr)
    user_agent = request.headers.get('User-Agent', '')[:500]
```

---

### Long-Lived State (Cloud SQL — see 02-ARCHITECTURE-MAP for full schema)

#### Multi-tenant isolation

**EVIDENCE** — `business_id` is the tenant key
**Source:** `models.py:673` (UserData) and `models.py:838` (ReportFile) both have `business_id` NOT NULL with `index=True` and a comment "Never accept business_id from client; always derive from authenticated session".

**EVIDENCE** — Filter pattern
**Source:** `dashboard.py:90, 107, 158, 240` — all queries filter by `user.business_id`.

#### Audit log

**EVIDENCE** — `audit_logs` table is the immutable history of mutations
**Source:** `models.py:885-948`. Indexes on `(table_name, record_id)`, `user_id`, `created_at`.

**INFERENCE** — `audit_logs` is append-only by convention but there is no DB-level constraint preventing UPDATE or DELETE; super-admins with DB access could modify it. No log-shipping or external SIEM observed in code.

#### Pending registrations

**EVIDENCE** — Out-of-band manual verification state
**Source:** `models.py:354-493`. Status machine: `pending_verification` → `approved` → `completed` (or `rejected` / `expired`).

#### Password reset tokens

**EVIDENCE** — Two flavors with different storage strategies
**Source:** `models.py:951-994`
- `secret_type='temporary_password'`: stores both `secret_hash` (Werkzeug PBKDF2) for verification AND `secret_ciphertext` (Fernet) for desktop client to decrypt and use.
- `secret_type='email_link'`: stores only `secret_hash`; the raw token only travels via the email link.

#### Reference dataset versioning

**EVIDENCE** — Content-hash + row-count cache
**Source:** `reference_data_versions.py:41-203` computes SHA-256 over normalized rows for WAC/FUL/AAC/PBM, and `_update_dataset_version` upserts into `reference_dataset_versions`. Used by `dashboard.py:286-311` `/wac_reference/metadata` and `/ful_reference/metadata` endpoints so desktop clients can skip re-fetching unchanged datasets.

---

### Module-Level Caches (Single-Worker Memory)

**EVIDENCE — GHL custom field map cache**
**Source:** `integrations.py:38-42`
```python
_cached_custom_field_map_raw = None
_cached_custom_field_map = {}
_custom_field_missing_env_logged = False
_custom_field_map_error_logged = False
_custom_field_missing_key_logged = set()
```
Re-parses only when the env-var value changes (`integrations.py:182-240`).

**EVIDENCE — Sheets service cache**
**Source:** `pbm_corrections.py:60-63` — `@lru_cache()` on `_get_sheets_service`.

**EVIDENCE — OAuth client cache**
**Source:** `auth.py:2482-2491` — global `oauth = OAuth(app)` initialized once at app boot.

**EVIDENCE — GIT_COMMIT_HASH constant**
**Source:** `__init__.py:48` — captured at module import time via subprocess.

**INFERENCE — Single-worker assumption.** Gunicorn runs with `--workers 1 --threads 8` (`Dockerfile:34`), so module-level caches are effectively per-process and don't need cross-worker invalidation. If worker count is ever increased, GHL custom-field map updates would not propagate across workers without a restart.

---

### What Is NOT Cached / Memorized

**GAP — No Redis / Memcached / external cache.** Reference data, business profiles, and Stripe subscription state are all re-queried from Cloud SQL on each request.

**GAP — No request-level memoization** for repeated `User.query.filter_by(username=...)` lookups. Many endpoints look up the user 1+ times per request.

**GAP — No conversation history or LLM context.** This is a Flask SaaS API; "agent loop" doesn't apply.

---

### Lifecycle / TTL Summary

| State | Duration | Storage |
|---|---|---|
| JWT access token | 1 hour | httpOnly cookie |
| OAuth redirect URI | Until callback consumes | Flask session cookie |
| Activation key | Until consumed via `/api/business/activate` (then NULL'd) | `businesses.activation_key` |
| PendingRegistration activation_token | 7 days | `pending_registrations.activation_token_expires_at` |
| Email-link reset token | `PASSWORD_RESET_LINK_EXPIRY_HOURS` (default 4, clamped 1-72) | `password_reset_tokens.expires_at` |
| Support temp password | 5 min – 24h (admin-controlled) | `password_reset_tokens.expires_at` |
| Stripe checkout session | Stripe-controlled (24h default) | `businesses.stripe_checkout_session_id` |
| Trial period | 14 days (`auth.py:613`) | `businesses.trial_end_date` (Stripe-managed) |
| Audit log entries | Indefinite | `audit_logs` table |
| Reference dataset version | Recomputed on import | `reference_dataset_versions` |
| Desktop version telemetry | Indefinite | `desktop_client_versions` (last_seen_at refresh) |
| GHL custom field map cache | Until env var changes | Module-level dict |
| Correlation ID | Per request | `contextvars.ContextVar` |

---

### State Transitions (key flows)

**Business.status:**
- `pending` (initial) → `active` (via `/api/business/activate`) → `suspended` (manual super-admin)
- Note: business.status is intentionally distinct from `subscription_status` (Stripe-managed). The webhook updates subscription_status only.

**Business.subscription_status (Stripe-driven):**
- `incomplete` → `trialing` → `active` (after trial conversion or first invoice paid)
- `active` → `past_due` (failed invoice) → `unpaid` (after retries) → `canceled`
- `active` ↔ `canceled` (with `cancel_at_period_end=True/False`)

**PendingRegistration.status:**
- `pending_verification` → `approved` (super-admin approve) OR `rejected` (super-admin reject) OR `expired`
- `approved` → `completed` (user activates via email link)

**PasswordResetToken (email-link):**
- created (active) → `redeemed_at` set (used)
- created (active) → `revoked_at` set (superseded)
- created → expired (TTL elapsed; not actively flagged)

---

### HTTP-Level Caching

**GAP — No HTTP cache headers** (no `Cache-Control`, `ETag`, `Last-Modified`) emitted by routes. JSON responses are explicitly fresh on every request.

**EVIDENCE — Security headers only**
**Source:** `__init__.py:680-686` — `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`.

---

## Open Questions

1. Should reference-data endpoints (large WAC/FUL/AAC reads) gain HTTP cache headers + ETag based on `reference_dataset_versions.checksum`? The infrastructure exists.
2. Is single-worker gunicorn (`--workers 1`) intentional, or a Cloud Run cold-start optimization? Multi-worker would invalidate module-level caches.
3. Why does the system avoid Redis given the volume of repeated user/business lookups per request?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (no external cache; no HTTP caching; audit log not append-only at DB level)
