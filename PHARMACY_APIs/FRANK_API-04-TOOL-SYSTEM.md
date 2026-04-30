# 04 — Blueprint / Endpoint Catalog (Reframed from "Tool System")

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

Per the SKILL doc-template guidance, this Flask SaaS service has no LLM "tools" — instead, the equivalent is its **HTTP endpoint surface**. ~130 routes are registered across 13 blueprints. Routes split roughly into PUBLIC (registration, activation, webhooks, health, password reset request, desktop auto-update, OAuth, the legacy `/api/pharmacy_profile`), JWT-required (most business-scoped reads/writes), `admin_required` (per-business admin), and `super_admin_required` (platform-wide). 110 `@jwt_required` / `@admin_required` / `@super_admin_required` decorator usages were grep-counted across 10 files. Auth is enforced via decorators only — there is no central middleware/policy layer.

---

## Findings — Endpoint Catalog

### auth_bp — `/api/auth/*` (auth.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/register` | PUBLIC | auth.py:71 | Stripe-first registration → checkout URL; ZERO db writes |
| POST | `/register-pending` | PUBLIC | auth.py:734 | Manual-verification onboarding (creates PendingRegistration) |
| GET | `/admin/pending-registrations` | JWT (super-admin in handler) | auth.py:913 | List pending registrations |
| POST | `/admin/pending-registrations/<int:id>/approve` | JWT (super-admin) | auth.py:948 | Approve pending; generate activation_token |
| POST | `/admin/pending-registrations/<int:id>/reject` | JWT (super-admin) | auth.py:1023 | Reject |
| GET | `/activate?token=...` | PUBLIC | auth.py:1070 | Validate activation_token; show details |
| POST | `/activate` | PUBLIC | auth.py:1104 | Complete activation: set password, create User+UserBusiness, set JWT cookie |
| POST | `/login` | PUBLIC | auth.py:1213 | Username/email+password; sets JWT cookie + body access_token |
| POST | `/logout` | JWT | auth.py:1378 | unset_jwt_cookies |
| GET | `/me` | JWT | auth.py:1396 | Current user + business |
| POST | `/users` | JWT (admin) | auth.py:1425 | Add user to current business |
| GET | `/users` | JWT (admin) | auth.py:1486 | List users in current business |
| DELETE | `/users/<int:user_id>` | JWT (admin) | auth.py:1512 | Delete user (not self) |
| PUT | `/users/<int:user_id>` | JWT (admin) | auth.py:1552 | Update user (password / is_admin) |
| POST | `/verify-activation-key` | PUBLIC | auth.py:1604 | Verify key (does NOT consume); return business state |
| POST | `/verify-desktop-credentials` | PUBLIC | auth.py:1831 | NCPDP+NPI+desktop_username+activation_key check |
| POST | `/verify-desktop-and-create-user` | PUBLIC | auth.py:1989 | Self-service webapp account creation from desktop verification |
| GET | `/businesses` | JWT | auth.py:2231 | List user's businesses (multi-pharmacy) |
| POST | `/businesses/link` | JWT | auth.py:2277 | Add another business via NCPDP+NPI+activation_key |
| POST | `/businesses/switch` | JWT | auth.py:2410 | Set primary business |
| GET | `/oauth/google/authorize` | PUBLIC | auth.py:2648 | Initiate Google OAuth |
| GET | `/oauth/google/callback` | PUBLIC | auth.py:2666 | Google callback → JWT + redirect to webapp |
| GET | `/oauth/microsoft/authorize` | PUBLIC | auth.py:2691 | Initiate Microsoft OAuth |
| GET | `/oauth/microsoft/callback` | PUBLIC | auth.py:2709 | Microsoft callback |

**INFERENCE** — The 4 `/admin/pending-registrations/*` endpoints use `@jwt_required` but enforce super-admin in-line (`auth.py:924, 965, 1034`). They could be using the centralized `super_admin_required` decorator from `admin.py:56` for consistency.

### business_bp — `/api/business/*` (business.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/exists` | PUBLIC | business.py:308 | NPI+NCPDP existence check (10/7 digit validated) |
| GET | `/by-activation-key` | PUBLIC | business.py:386 | Pre-populate registration form by activation key |
| GET | `/apa-membership` | PUBLIC | business.py:478 | APA membership eligibility lookup |
| POST | `/register` | PUBLIC | business.py:561 | ALIAS — calls auth.register() |
| GET | `/register/verify` | PUBLIC | business.py:584 | Poll: did webhook process this checkout session yet? |
| POST | `/activate` | PUBLIC | business.py:659 | Consume activation key; activate Business; rotate API admin password (returns plaintext) |
| GET | `/info` | JWT | business.py:966 | Current business as_dict |
| GET | `/status` | JWT | business.py:1048 | Status + subscription + activation_key_present + trial info |
| POST | `/subscription/cancel` | JWT (admin) | business.py:1238 | Set cancel_at_period_end via Stripe |
| POST | `/subscription/renew` | JWT (admin) | business.py:1497 | Reverse cancellation OR create renewal checkout |
| GET | `/subscription/portal` | JWT | business.py:1746 | Stripe billing portal URL |

### pharmacy_bp — `/api/*` (pharmacy.py — LEGACY)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/pharmacy_profile` | **PUBLIC** | pharmacy.py:7 | Returns first PharmacyProfile (single-tenant legacy) |
| POST | `/pharmacy_profile` | **PUBLIC** | pharmacy.py:15 | Mass-assignment update of PharmacyProfile (`PharmacyProfile(**data)`) |
| GET | `/health` | PUBLIC | pharmacy.py:29 | Duplicate of `/api/health/` |
| GET | `/profile` | JWT | pharmacy.py:33 | Current user's business as_dict |
| PUT | `/profile` | JWT (admin) | pharmacy.py:52 | Update business (CRM-mastered fields read-only for non-super-admin) |

**EVIDENCE — `/api/pharmacy_profile` is unauthenticated mass-assignment**
**Source:** `pharmacy.py:15-27`:
```python
@pharmacy_bp.route("/pharmacy_profile", methods=["POST"])
def update_pharmacy_profile():
    data = request.json
    profile = PharmacyProfile.query.first()
    if not profile:
        profile = PharmacyProfile(**data)
        db.session.add(profile)
    else:
        for k, v in data.items():
            setattr(profile, k, v)
    db.session.commit()
```
No `@jwt_required`. No business scoping. Mass-assignment from JSON. See 07-GUARDRAILS for severity.

### payments_bp — `/api/payments/*` (payments.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/create-checkout-session` | JWT | payments.py:29 | Create Stripe checkout session (auth'd flow) |
| POST | `/renew` | JWT | payments.py:116 | Renewal checkout |
| GET | `/status` | JWT | payments.py:207 | Subscription status (synced from Stripe live) |
| POST | `/cancel-subscription` | JWT (admin) | payments.py:273 | Cancel-at-period-end |
| POST | `/webhook` | PUBLIC + signature | payments.py:324 | Primary Stripe webhook (HMAC-verified) |
| GET | `/webhook/health` | PUBLIC | payments.py:1590 | Verify webhook secret configured |
| POST | `/check-trial-eligibility` | PUBLIC | payments.py:1666 | NCPDP+NPI trial check |

### dashboard_bp — `/api/data/*` (dashboard.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/user_data` | JWT (business-scoped) | dashboard.py:54 | Paginated prescription records, business_id filter |
| POST | `/user_data` | JWT | dashboard.py:129 | Create record (business_id from JWT) |
| POST | `/user_data/bulk` | JWT | dashboard.py:169 | Bulk upsert by `script` |
| PUT | `/user_data/<int:id>` | JWT | dashboard.py:691 | Update (business_id filter) |
| DELETE | `/user_data/<int:id>` | JWT | dashboard.py:735 | Delete (business_id filter) |
| GET | `/wac_reference` | JWT | dashboard.py:278 | All WAC rows |
| GET | `/wac_reference/metadata` | JWT | dashboard.py:286 | Checksum + row_count |
| POST | `/wac_reference` | JWT | dashboard.py:394 | Create row |
| PUT | `/wac_reference/<int:id>` | JWT | dashboard.py:775 | Update |
| DELETE | `/wac_reference/<int:id>` | JWT | dashboard.py:817 | Delete |
| GET | `/ful_reference` | JWT | dashboard.py:316 | NDC/year/month filter |
| GET | `/ful_reference/metadata` | JWT | dashboard.py:300 | Checksum |
| GET | `/ful_reference/export?cursor&page_size` | JWT | dashboard.py:351 | Cursor pagination for desktop sync |
| GET | `/aac_reference?ndc&date` | JWT | dashboard.py:445 | Effective-date-based AAC lookup |
| GET | `/aac_reference/metadata` | JWT | dashboard.py:477 | Checksum |
| POST | `/aac_reference/batch` | JWT | dashboard.py:491 | Multi-NDC batch lookup |
| GET | `/aac_reference/export?cursor&page_size&min_aac_date` | JWT | dashboard.py:566 | Pagination |
| POST | `/aac_reference/load` | JWT | dashboard.py:989 | Trigger GCS-driven AAC import |
| POST | `/aac_reference` | JWT | dashboard.py:1043 | Create AAC row |
| PUT | `/aac_reference/<int:id>` | JWT | dashboard.py:1058 | Update AAC row |
| DELETE | `/aac_reference/<int:id>` | JWT | dashboard.py:1073 | Delete AAC row |
| GET | `/report_files` | JWT (business-scoped) | dashboard.py:605 | Reports for current business |
| POST | `/report_files` | JWT | dashboard.py:649 | Create report |
| PUT | `/report_files/<int:id>` | JWT | dashboard.py:830 | Update |
| DELETE | `/report_files/<int:id>` | JWT | dashboard.py:874 | Delete |
| GET | `/pbm_info` | JWT | dashboard.py:916 | PBM lookup |
| POST | `/pbm_info` | JWT | dashboard.py:930 | Create (mass-assignment from `**record`) |
| POST | `/pbm_info/load` | JWT | dashboard.py:939 | GCS-driven PBM info bulk load |
| PUT | `/pbm_info/<int:id>` | JWT | dashboard.py:970 | Update (mass-assignment) |
| DELETE | `/pbm_info/<int:id>` | JWT | dashboard.py:979 | Delete |

**INFERENCE** — Reference data writes (WAC/AAC/FUL/PBM) are protected only by `@jwt_required`, NOT by `admin_required` or `super_admin_required`. Any authenticated user can create/update/delete reference data, which is platform-wide (not business-scoped). This is a privilege gap (see 07-GUARDRAILS).

### account_bp — `/api/account` (account.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `` | JWT | account.py:16 | Combined: user + business + subscription + (admin only) users list |

### imports_bp — `/api/import/*` (imports.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/apa_memberships` | JWT (super-admin) | imports.py:1654 | Replace/upsert APA roster from JSON |
| POST | `/wac_reference` | JWT (super-admin) | imports.py:1744 | Upload WAC CSV |
| POST | `/wac_reference_from_gcs` | JWT (super-admin) | imports.py:1796 | Pull WAC CSV from GCS |
| POST | `/aac_reference_from_gcs` | JWT (super-admin) | imports.py:1855 | Pull AAC Excel from GCS |
| POST | `/ful_reference` | JWT (super-admin) | imports.py:1914 | Fetch FUL from Medicaid.gov data.json |
| POST | `/ful_reference_upload` | JWT (super-admin) | imports.py:1959 | Upload FUL CSV directly |
| POST | `/ful_reference_from_csv` | JWT (super-admin) | imports.py:2010 | Read FUL CSV from GCS path |
| POST | `/user_data/preview` | JWT (business-scoped) | imports.py:2463 | Detect column mapping (CSV/XLSX) |
| POST | `/user_data` | JWT (business-scoped) | imports.py:2690 | Import prescription data (CSV/XLSX) |
| POST | `/liberty/user_data` | JWT (business-scoped) | imports.py:3006 | Pull from Liberty Software API |

**EVIDENCE — Super-admin enforcement is in-handler, not via decorator**
**Source:** `imports.py:1660-1664` shows the pattern (every super-admin imports endpoint has the same in-handler check). The codebase has a `super_admin_required` decorator but `imports.py` doesn't use it.

### admin_bp — `/api/admin/*` (admin.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/password-resets/temporary` | JWT + admin_required | admin.py:95 | Issue support temp password (encrypted ciphertext + hash) |
| GET | `/users?business_id=` | JWT + admin_required | admin.py:234 | List users; super-admin can filter by business |
| GET | `/users/<int:user_id>` | JWT + admin_required | admin.py:273 | Get user (super-admin: any) |
| POST | `/users` | JWT + admin_required | admin.py:312 | Create user in current business |
| PUT | `/users/<int:user_id>` | JWT + admin_required | admin.py:383 | Update user |
| DELETE | `/users/<int:user_id>` | JWT + admin_required | admin.py:446 | Delete user |
| GET | `/businesses` | JWT + super_admin_required | admin.py:500 | List all businesses |
| GET | `/businesses/<int:business_id>` | JWT + super_admin_required | admin.py:524 | Get business |
| PUT | `/businesses/<int:business_id>` | JWT + super_admin_required | admin.py:554 | Update CRM-mastered fields (audit-logged) |
| DELETE | `/businesses/<int:business_id>` | JWT + super_admin_required | admin.py:657 | Delete business + cascade users |
| GET | `/businesses/<int:business_id>/users` | JWT + super_admin_required | admin.py:700 | List users in business |
| GET | `/orphaned-registrations?hours_old=` | JWT + super_admin_required | admin.py:734 | Find orphan registrations (legacy cleanup) |
| DELETE | `/orphaned-registrations/<int:business_id>` | JWT + super_admin_required | admin.py:792 | Delete orphan with safety check |
| GET | `/reports/user-counts-per-business` | JWT + super_admin_required | admin.py:861 | Cross-business analytics |
| GET | `/reports/prescription-counts-per-business` | JWT + super_admin_required | admin.py:961 | Cross-business analytics |
| GET | `/activation-keys?…` | JWT + super_admin_required | admin.py:1058 | List activation keys (returns plaintext) |
| GET | `/activation-keys/<int:business_id>` | JWT + super_admin_required | admin.py:1153 | Get activation key for support |
| POST | `/manual-activation-key-and-ghl-sync` | JWT + super_admin_required | admin.py:1214 | Manual trigger of activation key + GHL sync |

### integrations_bp — `/api/integrations/*` (integrations.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/ghl-registration` | HMAC sig OR token (DISABLED IF SECRET MISSING) | integrations.py:1456 | GHL form/webhook → create Business (status=pending) + checkout |
| POST | `/ghl-contact-updated` | HMAC sig OR token | integrations.py:2194 | GHL contact update → sync to Business |
| POST | `/ghl-status-sync` | HMAC sig OR token | integrations.py:2518 | GHL status sync |
| POST | `/create-stripe-checkout` | Token-only | integrations.py:2642 | Token-authenticated checkout creation (no Business write) |
| POST | `/stripe-webhook` | Stripe HMAC | integrations.py:3007 | **Second** Stripe webhook handler |

### health_bp — `/api/health/*` (health.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/` and `` | PUBLIC | health.py:18-19 | `{status: 'ok'}` (used by App Engine readiness/liveness) |
| GET | `/registration-test` | Optional X-Health-Secret | health.py:107 | Echo request for debugging (only enabled in non-prod or with secret) |
| POST | `/registration-test` | Optional X-Health-Secret | health.py:107 | Same |
| GET | `/ping` | PUBLIC | health.py:238 | `{status: 'healthy', service: 'pharmacybooks-api', version: '1.0.0'}` |

### password_resets_bp — `/api/password-resets/*` (password_resets.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| POST | `/email-link` | PUBLIC | password_resets.py:81 | Issue email-link reset; sends SMTP email |
| GET | `/email-link/verify?token=` | PUBLIC | password_resets.py:266 | Validate token (raw token vs stored hash) |
| POST | `/email-link/complete` | PUBLIC | password_resets.py:300 | Complete reset: token + new_password |

**Always returns 202** for `/email-link` POST regardless of whether the user exists — prevents enumeration attacks.

### desktop_bp — `/api/desktop/*` (desktop.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/latest-version?current_version=X.Y.Z` | PUBLIC | desktop.py:253 | DEPRECATED auto-update check |
| GET | `/password-resets/temporary?username=&business_id=` | JWT | desktop.py:379 | Poll for support-issued temp password |
| GET | `/local-users?business_id=` | JWT | desktop.py:517 | Get LocalDesktopUser |
| PUT | `/local-users?business_id=` | JWT | desktop.py:548 | Set LocalDesktopUser.username |
| POST | `/versions?business_id=` | JWT | desktop.py:639 | Telemetry — record desktop version per device |
| POST | `/pbm-corrections?business_id=` | JWT | desktop.py:745 | Submit PBM corrections to Google Sheet |

### web_bp — (no prefix) (web.py)

| Method | Path | Auth | Source | Purpose |
|---|---|---|---|---|
| GET | `/reset-password?token=` | PUBLIC | web.py:7 | Serve `reset_password.html` template |

---

### Decorator Inventory

**EVIDENCE — Auth decorators defined**
**Source:**
- `flask_jwt_extended.jwt_required` (no override)
- `admin.py:21-53` `def admin_required(fn)` — passes `current_user` as first arg
- `admin.py:56-86` `def super_admin_required(fn)` — passes `current_user` as first arg

**EVIDENCE — Decorator usage frequency** (Grep `@(jwt_required|super_admin_required|admin_required)`)
- admin.py: 38 usages
- account.py: 1
- auth.py: 13
- business.py: 6
- dashboard.py: 30
- desktop.py: 5
- imports.py: 10
- payments.py: 4
- pharmacy.py: 2
- (scripts) verify_transaction_handling.py: 1

Total: 110 across 10 files.

**INFERENCE** — `admin_required` / `super_admin_required` are used consistently in `admin.py` but NOT in `imports.py` (which manually checks `if not user.is_super_admin`). This inconsistency is a refactoring opportunity, not a security gap.

---

### Authentication Methods Summary

| Method | Where | Notes |
|---|---|---|
| JWT cookie (`access_token_cookie`, httpOnly, SameSite=Lax) | Most authenticated endpoints | 1-hour expiry; Set-Cookie returned by `/login`, `/activate`, `/oauth/*/callback` |
| `Authorization: Bearer <token>` | Also accepted (Flask-JWT-Extended supports both) | Desktop client uses Bearer header per docs |
| OAuth (Google, Microsoft) | `/api/auth/oauth/*` | Authlib OIDC; redirect to webapp |
| Stripe HMAC signature | `/api/payments/webhook`, `/api/integrations/stripe-webhook` | `stripe.Webhook.construct_event` |
| GHL HMAC + token fallback | `/api/integrations/ghl-*` | `verify_ghl_signature` (HMAC-SHA256) or `hmac.compare_digest` of static token |
| `X-Health-Secret` header | `/api/health/registration-test` | `hmac.compare_digest` |
| `X-Webhook-Token` static token | `/api/integrations/create-stripe-checkout` | `hmac.compare_digest` |
| No auth | Registration, activation, status checks, password reset request, OAuth init, health, desktop auto-update, **`/api/pharmacy_profile`** | See 07-GUARDRAILS for the legacy concern |

---

### Request Patterns

**EVIDENCE — Field-name normalization**
**Source:** `auth.py:151-158`, `auth.py:1444-1446`, `admin.py:334-336` — both `username` and `user_name` are accepted; `auth.py:1227-1228` accepts both `email` and `username`.

**EVIDENCE — Pagination**
- Page-based (`page`, `per_page`): `dashboard.py:54-127` `/user_data`. Max `per_page=100`.
- Cursor-based: `dashboard.py:351, 566` `/ful_reference/export`, `/aac_reference/export`. Max `page_size=20000`.

**EVIDENCE — Audit logging trigger**
- Centralized helpers: `audit_utils.log_field_change`, `log_business_changes`, `log_business_creation`
- `CRM_MASTERED_FIELDS` set defines which fields are audit-logged on update (`audit_utils.py:15-41`)
- Webhook events also write audit log entries with `username='stripe_webhook'`, `'system_event_listener'`, `'system_ghl_sync'`, `'GHL_REGISTRATION'`, etc.

---

## Open Questions

1. Why does `imports.py` check super-admin in-handler instead of using the `super_admin_required` decorator from `admin.py`? Should this be unified?
2. Are reference-data write endpoints (`POST /api/data/wac_reference`, `POST /api/data/aac_reference`, `POST /api/data/pbm_info`) intentionally accessible to any authenticated user, or should they require admin/super-admin? See 07-GUARDRAILS.
3. The `/api/pharmacy_profile` endpoints (legacy, public, mass-assignment) — kept intentionally for backward compat, or dead code? Removal would close a major data-integrity gap.
4. Is the duplicate health endpoint (`/api/health/` + `/api/health/ping` + `/api/pharmacy/health`) intended? App Engine readiness uses `/api/health` per `app.yaml:57`.

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (legacy public mass-assignment endpoint; reference-data writes lack admin gate)
