# 03 — Request/Response Lifecycle (Reframed from "Agent Loop")

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

This is a Flask SaaS service, not an LLM agent — there is no plan/act/observe loop. Per the SKILL doc-template guidance, this document captures the **request/response lifecycle** instead. The dominant lifecycle is the multi-stage **business onboarding flow**: a single registration call returns a Stripe checkout URL but writes ZERO rows; payment completion triggers a Stripe webhook that creates the Business record; an SQLAlchemy ORM event listener auto-generates an activation key; another listener schedules a post-commit GHL sync; the user later activates by submitting the key, which rotates the API admin password and consumes the key. Two parallel onboarding entrypoints exist with different DB-write policies (see Notable Asymmetry below). Every authenticated endpoint follows the same pattern: JWT cookie → identity lookup → business_id derivation → business-scoped query → audit log on writes.

---

## Findings

### Lifecycle 1: Business Onboarding (Webhook-Driven Path — `auth.py /register`)

**EVIDENCE** — End-to-end flow
**Source:** `auth.py:71-731` + `payments.py:324-1592` + `models.py:998-1271` + `business.py:659-963` + `integrations.py:_handle_checkout_session_completed`

```
Step 1. CLIENT → POST /api/auth/register (or /api/business/register, alias)
        Body: { ncpdp, npi, pharmacy_name, email, username, password, … }
        ────────────────────────────────────────────────────────────────────
        Server actions (auth.py:71-731):
          a. Generate correlation_id = "reg_{ip}_{ts}"
          b. Validate required fields, username≥3, password≥6
          c. Read-only check: User by username, Business by (ncpdp, npi)
          d. Check trial eligibility (read-only)
          e. APA membership lookup → may auto-apply APA20
          f. Validate promotion_code via stripe.PromotionCode.list
          g. Hash password (Werkzeug PBKDF2)
          h. Create stripe.checkout.Session with ALL data in metadata,
             including the password_hash
          i. Return 201 { checkout_url, session_id, trial_eligible }
        NO db.session.add / commit. Zero rows written.

Step 2. CLIENT → User completes payment in Stripe Checkout (external)

Step 3. STRIPE → POST /api/payments/webhook
        Headers: Stripe-Signature
        Body: { type: "checkout.session.completed", data.object: {…, metadata} }
        ────────────────────────────────────────────────────────────────────
        Server actions (payments.py:324-557 → _handle_checkout_session_completed at :826):
          a. Verify HMAC via stripe.Webhook.construct_event with STRIPE_WEBHOOK_SECRET
             - 403 if signature missing/invalid
             - 500 if webhook secret not configured
          b. STEP 1: Validate metadata has either (ncpdp+npi) OR business_id
          c. STEP 2: Find Business via NCPDP+NPI; if not found:
             - **Update-only policy**: return 404, do NOT create
             - Audit-log the lookup failure
          d. STEP 3: Business found — proceed
          e. STEP 4: Update business.stripe_checkout_session_id
             ↓ TRIGGERS SQLAlchemy event listener (models.py:998):
               _generate_activation_key_on_session_id_change
               → secrets.token_urlsafe(32) (43 chars)
               → raw SQL UPDATE businesses SET activation_key=…
               → raw SQL INSERT into audit_logs (system_event_listener)
          f. STEP 5: Create or fetch Stripe Customer; store stripe_customer_id
          g. STEP 6: Retrieve subscription, set stripe_subscription_id,
             subscription_status, current_period_end, has_used_trial,
             trial_end_date, promotion_code
          h. STEP 6.5: If APA20, mark APAMembership.discount_redeemed (with SELECT FOR UPDATE)
          i. STEP 7: db.session.commit()
             ↓ TRIGGERS event listener _schedule_ghl_sync_after_commit:
               If activation_key was newly set → registers @once after_commit handler:
                 → integrations.sync_to_ghl(business, include_activation_key=True)
                 → Audit-log sync success/failure to table_name='ghl_operations'
          j. STEP 8: Insert audit_logs entries for each field change + webhook event
          k. STEP 9-10: Inline checks; may also call integrations.sync_to_ghl directly
             for redundancy
          l. Return 200 { status: 'success', business_id, changes, activation_key_generated }
        Note: business.status remains 'pending' even after payment.

Step 4. (Out of band) GHL automation sends activation email to business owner.

Step 5. CLIENT → POST /api/business/activate
        Body: { activation_key }
        ────────────────────────────────────────────────────────────────────
        Server actions (business.py:659-963):
          a. Validate key present
          b. Look up Business by activation_key
          c. If business.status == 'suspended' → 403
          d. If business.status == 'active' → 409 (key already used)
          e. Status transitions: pending|trialing → 'active'
          f. Get-or-provision admin User:
             - If no admin user: create User with username = "{ncpdp}_{npi}".lower()
             - If exists: rotate password
             - Generate plaintext password = secrets.token_urlsafe(16)
             - Set password_hash via Werkzeug
          g. Set business.activation_key = NULL (consume key)
          h. db.session.commit()
          i. Audit-log activation
          j. Return 200 { business_id, pharmacy_name, api_username,
             api_password (PLAINTEXT, one-time response), message }
        Now business is active and desktop client has API credentials.

Step 6. (Ongoing) Stripe sends subscription lifecycle events:
          - customer.subscription.updated → update subscription_status,
            current_period_end, cancel_at_period_end
          - customer.subscription.deleted → set subscription_status='canceled'
          - invoice.payment_succeeded → set subscription_status='active';
            also call _ensure_activation_key (defensive idempotent re-generation)
          - invoice.payment_failed → set subscription_status='past_due'
          - customer.subscription.trial_will_end → no-op placeholder
```

**EVIDENCE — `/api/business/activate` rotates the password and returns plaintext**
**Source:** `business.py:862-940` — `generated_password = secrets.token_urlsafe(16)` … `'api_password': generated_password` in response.

**INFERENCE — The activation key acts as a single-use bearer credential.** Once consumed, `business.activation_key` is NULL. From this point, any verification path that needs to cross-check the key falls back to NCPDP+NPI+desktop_username (`auth.py:1909-1939`).

---

### Lifecycle 2: Business Onboarding (GHL-Driven Path — `integrations.py /ghl-registration`)

**EVIDENCE** — Different policy, same outcome
**Source:** `integrations.py:1456-2191`

This entrypoint is invoked by GHL form submissions or GHL webhook automation. **It violates the webhook-driven "no DB writes during registration" policy.**

```
Step 1. GHL → POST /api/integrations/ghl-registration
        Headers: X-GHL-Signature (HMAC) OR X-Webhook-Token
        Body: { ncpdp, npi, pharmacy_name, address_*, business_*, contact_*, … }
        ────────────────────────────────────────────────────────────────────
          a. Auth: HMAC-SHA256 via X-GHL-Signature OR static token
             via X-Webhook-Token. If GHL_WEBHOOK_SECRET not set,
             auth is DISABLED (warning only).
          b. Field validation (required: 15 GHL field names mapped to model fields)
          c. Read-only check: Business by (ncpdp, npi) → 409 if exists
          d. **STEP 1: CREATE GHL CONTACT IMMEDIATELY** (best-effort, retry-once)
             - POST {GHL}/contacts/ with email/phone/mobile/customFields
             - On failure, ghl_contact_id = None and continue
          e. **STEP 2: INSERT Business with status='pending'** (DB WRITE)
             - db.session.add(new_business); db.session.flush()
             - log_business_creation → audit log entries for all CRM fields
          f. APA membership lookup, promotion_code validation
          g. plan_type validation ('standard' or 'concierge')
          h. Create stripe.checkout.Session with metadata including business_id
          i. db.session.commit()
          j. If browser request → 302 redirect to checkout URL
             If JSON request → 201 { redirect_url, session_id }

Step 2-onward: Same as Lifecycle 1 from Step 2. Webhook then UPDATES the
              already-created Business row.
```

**INFERENCE** — The GHL path creates the Business row pre-payment so GHL can have the `business_id` available for its own automations. This deviates from `docs/transaction_handling.md` and means abandoned GHL registrations leave orphan `status='pending'` rows. The orphan-cleanup endpoints (`/api/admin/orphaned-registrations`) and the `cleanup_orphaned_registrations.py` script exist to manage this.

---

### Lifecycle 3: Manual Verification Onboarding (`auth.py /register-pending`)

**EVIDENCE** — Phone-call verification flow
**Source:** `auth.py:734-1210` (PendingRegistration → approve → activate)

```
Step 1. CLIENT → POST /api/auth/register-pending
        Body: { ncpdp, npi, email, … (optional desktop_username + activation_key for converters) }
        ───────────────────────────────────────────────────────────────────
          a. Read-only checks (no payment)
          b. Insert PendingRegistration row with status='pending_verification'
          c. Return 201 { pending_id }

Step 2. SUPER ADMIN → GET /api/auth/admin/pending-registrations?status=pending_verification
        Returns the queue.

Step 3. (Out of band) Admin calls pharmacy on the published phone number
        to verify identity, then:

Step 4. SUPER ADMIN → POST /api/auth/admin/pending-registrations/{id}/approve
        ──────────────────────────────────────────────────────────────────
          a. Generate activation_token = secrets.token_urlsafe(32) (7-day TTL)
          b. Update status='approved', verified_by_user_id, verified_at
          c. Set activation_link_sent_at = now
          d. **TODO comment** — email actually being sent is "not yet implemented"
             (auth.py:990 — comment + dead code-fragment). The endpoint returns
             the activation_link in the response so admin can copy/paste manually.
          e. Return { activation_link }

Step 5. USER (browser) → GET /api/auth/activate?token=...
        ──────────────────────────────────────────────────────────────────
          Returns 200 { email, ncpdp, npi, token } (or 400/404)

Step 6. USER → POST /api/auth/activate (body: { token, password })
        ──────────────────────────────────────────────────────────────────
          a. Find PendingRegistration; check token validity (TTL + status)
          b. Find or create Business; create User with email as username
          c. Create UserBusiness row, role='admin'
          d. Mark pending_reg.status='completed'
          e. db.session.commit()
          f. create_access_token + set_access_cookies
          g. Return 200 { user, access_token } (cookie also set)
```

---

### Lifecycle 4: Authenticated Request (general pattern)

**EVIDENCE** — Identical pattern across `business.py`, `dashboard.py`, `account.py`, `desktop.py`, etc.

```
1. CLIENT → request with JWT cookie (or Authorization header — both supported)
2. @jwt_required() decorator parses token; rejects expired/invalid
3. Handler: identity = get_jwt_identity()  # email or username
4. user = User.query.filter_by(username=identity).first()
   (account.py and a few endpoints try email first, then username)
5. Auth checks:
   - If not user → 404
   - If endpoint is super-admin only and user.is_super_admin == False → 403
   - If endpoint is admin only and user.is_admin == False → 403
   - If endpoint is business-scoped and user.is_super_admin → 403
     ("Super admins must use admin endpoints")
6. business_id = user.business_id   # NEVER from request body
7. Build query with .filter_by(business_id=user.business_id)
8. For mutations:
   - record['business_id'] = user.business_id   # override any client value
   - db.session.add / db.session.commit
   - audit_utils.log_field_change for CRM-mastered fields
9. Return jsonify(...) with appropriate status code
10. Errors → db.session.rollback(); generic error response (no PHI in messages)
```

---

### Lifecycle 5: Stripe Subscription Cancellation/Renewal

**EVIDENCE** — `business.py:1238-1494` (cancel) and `business.py:1497-1744` (renew)

Cancel:
1. JWT-required, admin-required (NOT super-admin)
2. `_ensure_stripe_api_key()` ensures stripe.api_key set
3. `stripe.Subscription.retrieve` → handle 3 cases:
   - InvalidRequestError (subscription not found in Stripe): treat as already canceled, clear local state, audit-log
   - status == 'canceled': sync local fields, return
   - cancel_at_period_end already True: confirm, return
4. Otherwise: `stripe.Subscription.modify(id, cancel_at_period_end=True)`
5. Update local Business fields, audit-log changes, commit
6. Return billing-period info

Renew:
1. Similar pre-checks
2. If existing subscription has `cancel_at_period_end=True` → modify back to False
3. Otherwise create new Customer (if needed) + new checkout.Session
4. Persist `business.stripe_checkout_session_id`, audit-log, return checkout URL

Billing portal (`business.py:1746-1862`):
- Creates Stripe Customer if missing
- `stripe.billing_portal.Session.create` returns portal URL
- 502 on Stripe errors

---

### Lifecycle 6: Reference Data Pull (Cloud Functions chain)

**EVIDENCE** — `functions/ful_data_pull/main.py:71-109` + `functions/ful_import_worker/main.py`

```
Cloud Scheduler → trigger_ful_import (HTTP Cloud Function)
                ↓ enqueue Cloud Task
                tasks_v2.create_task(...) with OIDC (optional)
                ↓
Cloud Tasks  → POST WORKER_URL with { year, month }
              ↓
Cloud Function ful_import_worker:
              ↓ POST /api/import/ful_reference {year, month}
              (back into the API; uses internal auth/service-account)
              ↓
imports.py _fetch_ful_records (Medicaid.gov data.json)
              ↓
DB upsert into ful_reference, refresh checksum
```

A similar chain exists for AAC and WAC imports (`functions/aac_import_trigger/`, `functions/wac_import_trigger/`).

---

### Notable Asymmetry: Two Webhook Handlers, Two Registration Policies

**EVIDENCE — Two Stripe webhook handlers**
**Source:** `payments.py:324` (`/api/payments/webhook`) and `integrations.py:3007` (`/api/integrations/stripe-webhook`).

Both:
- Read `STRIPE_WEBHOOK_SECRET`
- Verify HMAC via `stripe.Webhook.construct_event`
- Handle `checkout.session.completed` + 5 other event types
- Look up Business via metadata or customer_id
- Update Stripe IDs and subscription state
- Write audit logs

Differences:
- `payments.py` flow uses inline functions and includes APA20-redemption logic
- `integrations.py` flow uses smaller helper functions (`_find_business_from_*`, `_log_audit_changes`) and is the more recent/refactored version
- `integrations.py` returns 200 even on processing exceptions (to avoid Stripe retries); `payments.py` returns 500

**GAP** — Stripe Dashboard configuration (single endpoint) determines which one is hit. There is no code-level coordination ensuring only one runs. If both URLs are configured in Stripe, every event would be processed twice (with idempotency safety due to NCPDP+NPI lookup, but doubled audit-log entries and possible APA20 double-counting risk).

---

### Notable Asymmetry: Activation-key consumption

**EVIDENCE** — Different endpoints handle activation key differently
- `/api/auth/verify-activation-key` (`auth.py:1604`) — verification only, does NOT consume the key
- `/api/business/activate` (`business.py:659`) — consumes (sets to NULL)
- `/api/auth/verify-desktop-credentials` (`auth.py:1831`) — gracefully accepts if key already consumed
- `/api/auth/verify-desktop-and-create-user` (`auth.py:1989`) — same graceful fallback

After consumption, the activation key is gone forever (no audit of the original value, only `[GENERATED]` in audit_logs). To re-issue, support uses `/api/admin/manual-activation-key-and-ghl-sync` (super-admin) which calls `_ensure_activation_key` to generate a new one.

---

### Notable Asymmetry: Cancel-subscription endpoints

**EVIDENCE** — Two endpoints, both authenticated, different behavior
- `POST /api/payments/cancel-subscription` (`payments.py:273`) — sets cancel_at_period_end; 4 happy-path branches; returns Stripe object fields
- `POST /api/business/subscription/cancel` (`business.py:1238`) — same outcome but with audit logging, "subscription not found" graceful handling, billing_portal-aware response

Same pattern for renew (`/api/payments/renew` vs `/api/business/subscription/renew`). Documentation does not indicate which is canonical.

---

## Open Questions

1. Which webhook URL is currently configured in Stripe Dashboard? Both? (Operational risk.)
2. Why is the `/api/auth/admin/pending-registrations/.../approve` endpoint missing the actual email send? (TODO at `auth.py:990`.)
3. Is the manual phone-verification onboarding flow (`/api/auth/register-pending`) actually used in production, or is it experimental?
4. After activation key consumption, how is support expected to re-issue if the user lost the desktop client and never created a webapp account? `/admin/manual-activation-key-and-ghl-sync` is the implied path.

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented
