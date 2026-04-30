# 08 — Error Handling and Recovery

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

Errors are handled at three layers: (1) framework-level via Flask error handlers and JWT callbacks; (2) endpoint-level via try/except blocks that translate Python exceptions to HTTP status codes; (3) integration-level via retry-with-backoff and "fail-soft" patterns for GHL/Stripe API calls. The webhook handlers favor returning HTTP 200 even on internal errors to avoid Stripe re-tries. Database operations rollback on exception. The Webhook-Driven onboarding flow eliminates orphaned-record cleanup by design (records only created post-payment); a residual `cleanup_orphaned_registrations.py` script and admin endpoints exist for legacy data. Activation key generation has explicit retry logic (3 attempts × 2-second delay). Stripe `InvalidRequestError` for missing subscriptions is treated as "already canceled" and reconciled gracefully. There is no global exception middleware — every endpoint catches its own.

---

## Findings

### Framework-Level Error Handling

#### JWT error handlers

**EVIDENCE — Three callbacks for missing/expired/invalid tokens**
**Source:** `__init__.py:619-635`
```python
@jwt.expired_token_loader
def expired_token_callback(jwt_header, jwt_payload):
    logger.warning(f"JWT token expired: {jwt_payload}")
    return jsonify({"error": "Token has expired"}), 401

@jwt.invalid_token_loader
def invalid_token_callback(error):
    logger.warning(f"Invalid JWT token: {error}")
    return jsonify({"error": "Invalid token"}), 401

@jwt.unauthorized_loader
def missing_token_callback(error):
    logger.warning(f"Missing JWT token: {error}")
    logger.warning(f"Request cookies: {request.cookies}")
    logger.warning(f"Request headers: {dict(request.headers)}")
    return jsonify({"error": "Authorization required"}), 401
```

**INFERENCE — `missing_token_callback` logs full cookies and headers** for debugging. This dumps potentially sensitive data into application logs (cookie values, Authorization headers). This is fine in dev; in production it could leak session cookies into Cloud Logging.

#### Flask exceptions

**GAP — No global `@app.errorhandler`** for HTTPException, 404, or 500. Default Flask responses apply: text/html 404 page, JSON-less 500. JSON clients hitting an unrouted URL get HTML.

---

### Endpoint-Level Patterns

#### Standard try/except → status code mapping

**EVIDENCE — Consistent pattern**
**Source:** Pattern repeated in nearly every handler:
```python
try:
    # ... business logic ...
    return jsonify({...}), 200
except SomeSpecificError as e:
    db.session.rollback()
    logger.error(...)
    return jsonify({"error": ..., "message": ...}), 4xx_or_5xx
except Exception as e:
    db.session.rollback()
    logger.error(f"...: {str(e)}", exc_info=True)
    return jsonify({"error": "Internal server error"}), 500
```

**EVIDENCE — Exception class handling for Stripe**
**Source:**
- `stripe_error.StripeError` (catch-all for Stripe SDK errors) — handled in `auth.py:674`, `payments.py:108, 199, 265, 316`, etc.
- `stripe_error.SignatureVerificationError` — webhook signature failure (`payments.py:448`, `integrations.py:3162`)
- `stripe.error.InvalidRequestError` — subscription not found, etc. — used for graceful "already canceled" handling in `business.py:109, 1306, 1559`
- `getattr` pattern at `business.py:18-19` to handle either old-style `stripe.error` or new-style `stripe._error`:
  ```python
  StripeError = getattr(getattr(stripe, 'error', None), 'StripeError', Exception)
  InvalidRequestError = getattr(getattr(stripe, 'error', None), 'InvalidRequestError', Exception)
  ```
  (This is a shim for SDK-version compatibility.)

**EVIDENCE — Exception class handling for SQLAlchemy**
**Source:**
- `sqlalchemy.exc.SQLAlchemyError` imported in `imports.py:20` but used only twice
- Most handlers use bare `except Exception` rather than narrower SQLAlchemy classes

**EVIDENCE — Exception class handling for Google APIs**
**Source:**
- `google.api_core.exceptions.GoogleAPIError` — caught in `imports.py:1844, 1903, 2079` (returns 502 with details)
- `googleapiclient.errors.HttpError` — caught in `pbm_corrections.py:118` (raised as `PbmCorrectionsAppendError`)
- `zipfile.BadZipFile` — `imports.py:2070` (FUL CSV unzip)

**EVIDENCE — Exception class handling for HTTP**
**Source:**
- `requests.exceptions.RequestException`, `requests.exceptions.Timeout` in `integrations.py:153, 588, 571` for GHL calls

#### Database rollback

**EVIDENCE — Almost every mutating endpoint rolls back on exception**
**Source:** `db.session.rollback()` appears 67+ times. Pattern: caught exception → rollback → log → return error JSON.

**EVIDENCE — Rollback in webhook handlers**
**Source:** `payments.py:1521-1556` — wraps the post-checkout work in try/except; on failure rolls back, then attempts to write a fresh AuditLog entry in a new transaction with its own try/except.

#### Sanitized errors for HIPAA

**EVIDENCE — Generic client messages**
**Source:** `dashboard.py:127, 167, 273`, `auth.py:1370-1376`, etc.:
```python
# HIPAA Compliance: Sanitize error messages to prevent PHI exposure
logger.error(f"Login error: {str(e)}", exc_info=True)
return jsonify({
    'error': 'Login failed',
    'message': 'An error occurred during login. Please try again.'
}), 500
```

**GAP — Inconsistent sanitization**
**Source:** `auth.py:731` `return jsonify({'error': f'Registration failed: {str(e)}'}), 500` — leaks raw exception text. Same pattern at `imports.py` for various 500 responses including `details: str(exc)`.

---

### Integration-Level Patterns

#### Stripe webhook handler — return 200 to prevent retry storms

**EVIDENCE — Catch-all returns 200 in `/api/integrations/stripe-webhook`**
**Source:** `integrations.py:3261-3308` — internal exceptions during event processing are caught, audit-logged, and the response is **still 200** with text comment:
> "Note: We still return 200 because the webhook was received and signature was valid. The error is in our processing logic, not the webhook itself."

**INFERENCE — This prevents Stripe from retrying webhooks that would fail repeatedly**, but it also means Stripe never sees the failure. The audit_logs table is the only place the failure is recorded.

**CONTRAST — `/api/payments/webhook` returns 500**
**Source:** `payments.py:1521-1555` returns 500 on processing exceptions. Stripe will retry. This is the older handler.

**INFERENCE — Inconsistency between the two webhook handlers** in retry behavior. Operational risk: depending on which URL Stripe is configured to hit, a transient bug either gets retried (with all the side effects of retry) or silently dropped.

#### Stripe `InvalidRequestError` reconciliation

**EVIDENCE — Treats missing-subscription as already-canceled**
**Source:** `business.py:1306-1338` (cancel) and `business.py:1559-1576` (renew) — when Stripe returns `InvalidRequestError` for `Subscription.retrieve`, the code:
1. Clears local `stripe_subscription_id` to None
2. Sets `subscription_status='canceled'`, `cancel_at_period_end=False`
3. Audit-logs the changes
4. Returns 400 "Subscription not found" with a friendly message

This is a **self-healing** pattern — local DB drift from Stripe state is reconciled at next interaction.

#### Activation key generation retry

**EVIDENCE — Three attempts × 2-second delay**
**Source:** `payments.py:651-786` `_ensure_activation_key`:
- Generates `secrets.token_urlsafe(32)`
- Attempts commit; on failure, rolls back, logs, sleeps 2s, retries
- Before each retry attempt, refreshes the business object and **checks for idempotency** (another concurrent process may have set the key)
- After all 3 attempts fail, writes a critical-level log entry, audit-logs the failure, returns `(False, None)` — "manual_intervention_required=True"

**INFERENCE — Robust against transient DB blips during webhook processing**, but a hard DB outage exhausts retries quickly (6s total). For Cloud SQL, transient errors are usually rapid recovery.

#### GHL retry with exponential backoff

**EVIDENCE — `create_ghl_contact` retries on 429/500/502/503/504**
**Source:** `integrations.py:506-595`:
- 3 attempts (configurable)
- Exponential delay: 2, 4, 8 seconds
- Catches `requests.exceptions.Timeout` separately with retry
- Catches `requests.exceptions.RequestException` non-retryable
- Default request_timeout=10s

**EVIDENCE — Smaller retry budget for inline registration calls**
**Source:** `integrations.py:1751-1755` — when GHL contact creation runs *inline* during registration (latency-sensitive), `max_retries=1, request_timeout=6`. If it fails, registration continues without ghl_contact_id (warning logged).

**EVIDENCE — sync_to_ghl retries are present in inner functions**
**Source:** `integrations.py:1157-1240` (around contact PATCH retries) — similar exponential backoff pattern.

**EVIDENCE — Idempotent contact creation pattern**
**Source:** `integrations.py:143-179` `_find_contact_id_by_email` — best-effort lookup before creating to avoid duplicate contacts.

#### GHL fail-soft

**EVIDENCE — GHL API key missing → all sync calls return `{success: False}` with warning, app continues**
**Source:** `integrations.py:410-423`. **The webhook does NOT fail if GHL is down.** This is intentional — activation key is generated regardless.

**EVIDENCE — sync_to_ghl exceptions caught in models event listener**
**Source:** `models.py:1243-1270` — wraps the entire post-commit GHL call in try/except, audit-logs the exception, NEVER re-raises. The main transaction is unaffected by GHL outages.

#### SMTP fail-soft

**EVIDENCE — Missing SMTP host → log instead of send**
**Source:** `email_utils.py:47-57`:
```python
if not host:
    logger.warning("SMTP_HOST not configured; email send skipped", ...)
    logger.info("Email content (not sent): %s", text_body, ...)
    return False
```
This allows local development without configuring SMTP.

**EVIDENCE — Send exceptions logged, email_sent flag returned**
**Source:** `email_utils.py:85-94` — catches `Exception`, logs with reason, returns False. Caller can decide whether to fail the request.

**EVIDENCE — Password reset endpoint logs but does not block on email failure**
**Source:** `password_resets.py:244-263` — calls `send_password_reset_email`, logs whether sent, returns 202 either way.

---

### Specific Recovery Tooling

#### `cleanup_orphaned_registrations.py` (script)

**EVIDENCE — Standalone script for legacy orphan cleanup**
**Source:** `pharmacybooks_api/scripts/cleanup_orphaned_registrations.py` (filename, not read in full)

**Per `docs/transaction_handling.md:248-258`:**
- `--dry-run` lists orphans without deleting
- Default mode requires explicit confirmation
- `--force` skips confirmation
- `--hours-old N` configures age threshold (default 24)
- Identifies orphans by: `stripe_customer_id IS NOT NULL AND stripe_subscription_id IS NULL AND subscription_status IS NULL AND created_at < cutoff`

**INFERENCE — This script is residual from the pre-webhook-driven era.** With the current Stripe-first flow in `auth.py /register`, no orphans should be created. With the GHL flow in `integrations.py /ghl-registration`, orphans (Business with status='pending', no subscription) CAN still be created — but the cleanup criteria in the script (`stripe_customer_id IS NOT NULL`) would NOT match these GHL-created orphans (because no Stripe customer was created during the GHL flow). The script is partially obsolete relative to the GHL flow.

#### Admin orphan cleanup endpoints

**EVIDENCE — REST equivalents of the script**
**Source:** `admin.py:734` `GET /api/admin/orphaned-registrations?hours_old=24` and `admin.py:792` `DELETE /api/admin/orphaned-registrations/<int:business_id>`. Both super-admin-required. The DELETE endpoint includes a safety check (`admin.py:818-828`) refusing to delete if `stripe_subscription_id` or `subscription_status` is set.

#### `fix_alembic_version.py` (root-level script)

**EVIDENCE — File exists at the repo root**
**Source:** `fix_alembic_version.py` (3.5KB, not read in full)

**INFERENCE** — Suggests Alembic migration head conflicts have occurred. The `migrations/versions/` folder shows multiple `merge_multiple_alembic_heads.py` files — see 10-RAW-FINDINGS.

#### Activation key re-issuance

**EVIDENCE — Super-admin manual trigger**
**Source:** `admin.py:1214` `POST /api/admin/manual-activation-key-and-ghl-sync` — calls `_ensure_activation_key` + `sync_to_ghl`. This re-generates and re-syncs an activation key for a specific business.

#### Subscription state desync recovery

**EVIDENCE — `_refresh_subscription_from_stripe` rebuilds local state**
**Source:** `business.py:99-200`. Used in `/api/business/status` (`business.py:1148-1158`) when local state is missing fields. Detects field changes, persists them, logs them, returns the freshened subscription.

---

### Recovery via Audit Log

**EVIDENCE — Audit log functions as a forensic record**
**Source:** Examples of recovery-relevant audit entries:
- Webhook errors: `username='stripe_webhook'`, `field_name='database_commit'`, `new_value=ERROR: ...`
- Activation key commit failures: `field_name='activation_key_commit'`, `new_value=COMMIT_FAILED_AFTER_3_RETRIES: ...`
- GHL sync failures: `table_name='ghl_operations'`, `field_name='onboarding_sync_failed'`
- APA20 redemption: `table_name='apa_memberships'`, `field_name='discount_redeemed'`

**INFERENCE** — The audit log is the source of truth for "what failed and when". Recovery procedures rely on querying it. There is no separate retry-queue table.

---

### Logging Levels and Cloud Logging Integration

**EVIDENCE — `logger.critical` reserved for catastrophic failures**
**Source:** `payments.py:747` — exhausted retry attempts on activation key commit.

**EVIDENCE — `logger.error` with `exc_info=True` for stack traces**
**Source:** All major exception handlers. Standard pattern.

**EVIDENCE — Structured logging via `extra={...}` dict**
**Source:** `email_utils.py:50-56`, `password_resets.py:107-110, 252-261`, `business.py:181-186`, etc. — uses `extra` to attach structured fields that Cloud Logging picks up as searchable.

**INFERENCE — Mixed logging style.** Some logs use f-strings (free-form text), others use `extra=` (structured). Cloud Logging will index the `extra` fields but not the f-string contents. Search ergonomics differ.

---

### Idempotency Guarantees

**EVIDENCE — Activation key generation is idempotent**
**Source:** `models.py:1026-1031` and `payments.py:606-805` both check for existing key before generating; `payments.py:660-672` re-checks on each retry attempt.

**EVIDENCE — Webhook business lookups are idempotent**
**Source:** `payments.py:826-1597` — looks up business by NCPDP+NPI; updates only changed fields; running twice produces no additional change.

**EVIDENCE — GHL contact creation has best-effort lookup-before-create**
**Source:** `integrations.py:143-179` — searches by email first.

**EVIDENCE — APA20 discount redemption is one-shot**
**Source:** `payments.py:1167-1182` — checks `discount_redeemed` before marking; uses `with_for_update()` (with fallback to plain `first()` if backend doesn't support it).

**GAP — No event_id deduplication.** Stripe's webhook event_id is logged but not used for deduplication. If the same event is delivered twice (Stripe is at-least-once), the audit log gets duplicated entries. Only the data mutations are idempotent due to "compare-then-set" patterns.

---

## Open Questions

1. Is `cleanup_orphaned_registrations.py` script still relevant for the GHL flow? Its match criteria (`stripe_customer_id IS NOT NULL`) don't apply to GHL orphans.
2. Why do the two Stripe webhook handlers differ in retry behavior (`/api/payments/webhook` returns 500; `/api/integrations/stripe-webhook` returns 200)?
3. Is there a runbook for "activation key generation failed after 3 retries"? The code logs `manual_intervention_required=True` but no admin endpoint exists for ad-hoc key generation by business_id alone (only `manual-activation-key-and-ghl-sync` which combines both).
4. Should event_id-based deduplication be added to webhook handlers to prevent duplicate audit log entries?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (inconsistent webhook retry; no event_id dedup; legacy cleanup script obsolescence)
