# 06 — Prompts and Persona

**Repo:** pharmacybooks-api-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

There are no LLM system prompts — this is a Flask SaaS API, not an AI agent. The "instructional content" in the repo falls into three categories: (1) **agent-config files for IDE tooling** (`CLAUDE.md`, `AGENTS.md`, `WINDSURF.md`) describing how AI assistants should work in this codebase; (2) **email/HTML templates** sent to users (password reset email body, password reset HTML page); (3) **error and informational message strings** returned in API responses. The three agent-config docs share a "Stark Industries" operational protocol and overlap heavily — they are characterized once below per the SKILL guidance, with deltas flagged.

---

## Findings

### Category 1: Agent Configuration Documents

#### Identical-philosophy triplet

**EVIDENCE** — Three near-parallel documents at the repo root
**Source:** `CLAUDE.md` (~20 KB), `AGENTS.md` (~20 KB), `WINDSURF.md` (~20 KB)

Each begins with the same framing:

> **AI App Factory — Stark Industries**
> *System prompt for [tool] agentic coding sessions.*

#### CLAUDE.md (Claude Code agent config)

**EVIDENCE** — Stark Industries operating protocol
**Source:** `CLAUDE.md:1-447` (full read by Claude Code at session start; embedded in this very session as the project instructions)

Key directives (full text in CLAUDE.md):
- **Role definition:** "senior software engineer embedded in an agentic coding workflow… You are the hands; Tony is the architect."
- **Plan Mode protocol:** Mandatory `EnterPlanMode` tool call before any file modifications; harness physically removes Write/Edit tools.
- **Disaster recovery:** Write plan to session file BEFORE displaying in CLI; maintain `RECOVERY.md` 3-second-recovery file.
- **Session memory:** `session_YYYY-MM-DD.md` template with PENDING_APPROVAL → APPROVED → IN PROGRESS → COMPLETE state transitions.
- **Tony's Rules:** Scope discipline ("touch only what you're asked to touch"); the "Ironman Rule" ("if you solve one problem but kill a previous feature, you've failed"); push back when warranted; assumption surfacing.
- **TDD flow:** `Build → Unit Test → Integrate → Block Test → System Test → Finalize`.
- **Tech stack preferences declared:** Next.js 15 + Tailwind + Zustand frontend; FastAPI + Uvicorn + Supabase backend (CONTRADICTION — this project's actual backend is Flask, not FastAPI; see 10-RAW-FINDINGS).
- **Failure modes** enumerated (skipping Plan Mode, sycophancy, scope creep, killing working features, etc.).

#### AGENTS.md (likely Cursor/Aider agent config)

**EVIDENCE — File exists, characterized via filename + first lines per `AGENTS.md` (sample at the top of this extraction)**

The first 60 lines (read by Claude Code) match the structure of CLAUDE.md verbatim in many places. A meaningful **delta** noted in the WINDSURF preamble: *"Windsurf does not have a mechanical Plan Mode tool that physically locks your write access. That means the enforcement here is behavioral, not architectural."* This statement explicitly differentiates Windsurf from Claude Code — implying AGENTS.md and WINDSURF.md describe the same Stark protocol calibrated to different tool capabilities.

#### WINDSURF.md (Windsurf/Cascade agent config)

**EVIDENCE** — Same protocol, behavioral enforcement
**Source:** `WINDSURF.md:1-60` (sampled). Subtitle: "*System rules for Windsurf agentic coding sessions.* | *Version: 2.0 | March 2026*". CLAUDE.md is "*Version: 3.0 | March 2026*" — so they have version drift but same source-of-truth ethos.

**INFERENCE** — These three files are intended to be peers across IDE tools and probably need to be kept in sync manually. They are not consumed by the Flask API at runtime — they exist purely for whichever AI assistant is editing the codebase. None are imported, parsed, or referenced by Python code.

#### Notable claims in agent docs that contradict the codebase

**EVIDENCE — CLAUDE.md tech stack section (`CLAUDE.md:441-473`)**
- Claims backend uses **FastAPI + Uvicorn + Supabase**
- Claims Python deps are managed via "requirements.txt + venv + pip (no Poetry/pyproject.toml)" — TRUE
- Claims `/types` folder is the canonical location for "all interfaces and Pydantic models" — but no `/types` folder exists in this repo
- Claims `config_service.py` is the only place env vars should be read — no `config_service.py` exists in this repo (74 `os.environ.get` calls scattered across `pharmacybooks_api/`)

**INFERENCE** — CLAUDE.md describes the broader "Stark Industries" project family, not just this repo. The agent-config files were likely copied from another project and apply only loosely here. A future refactor toward Supabase / FastAPI would align the codebase with these aspirations. See 10-RAW-FINDINGS.

---

### Category 2: User-Facing Templates

#### Password Reset Email (text + HTML)

**EVIDENCE** — Email body templates inline in code
**Source:** `email_utils.py:97-130` `send_password_reset_email`

Text body (line 110-116):
```
Hello from {greeting},

A request was made to reset the password for the account '{username}'. Use the link below to choose a new password.

Reset link: {reset_link}

The link expires at {expires_text}. If you did not request this change, you can ignore this email.
```

HTML body (line 118-128): Inline-styled `<a>` button with brand teal (`#1b72e8`) and same content.

`{greeting}` = `business_name or "PharmacyBooks"`. Subject defaults to env `PASSWORD_RESET_EMAIL_SUBJECT` else `"PharmacyBooks password reset"`.

#### Password Reset HTML Page

**EVIDENCE** — Branded standalone HTML
**Source:** `pharmacybooks_api/templates/reset_password.html` (311 lines)

- Title: "Pharmacy Books | Reset Password"
- Logo URL hardcoded: `https://storage.googleapis.com/msgsndr/COgRLDxQx2jv6R6JbAhw/media/68a478abfcbcd61a6c4be41e.png`
- Color palette defined as CSS custom properties: `--pb-blue: #235179`, `--pb-teal: #2cbfae`, `--pb-coral: #e74a3d`, `--pb-dark: #1d3147`
- Behavior: client-side JS calls `GET /api/password-resets/email-link/verify?token=…` to validate, then displays the token for the user to **paste into the desktop app**. Reset is completed in the desktop client, not in this web page.

**INFERENCE** — The web page is a **token relay** for desktop clients, not a self-contained password reset UI. Server-side `web.py:7` route renders Jinja with `token` from query string, but the page also re-reads the token client-side as a fallback.

#### Stripe-related "branded thank you" defaults

**EVIDENCE** — Hardcoded redirect URLs
**Source:**
- `payments.py:82, 173` — default `success_url='https://pharmacybooks.com/thankyou'`
- `business.py:282` — default `success_url='https://pharmacybooks.com/app/subscription/success'`
- `business.py:288` — default `cancel_url='https://pharmacybooks.com/app/subscription/cancel'`
- `business.py:299` — default `return_url='https://pharmacybooks.com/app/subscription/manage'`
- `integrations.py:2101-2102, 2923-2924` — same brand defaults
- `auth.py:526-527` — `'https://your-app.com/success'` (placeholder — different default than payments.py!)

**INFERENCE** — Brand URLs are defaults only; `success_url` / `cancel_url` are typically overridden by callers. The `auth.py` default `'https://your-app.com/success'` is suspicious — looks like a leftover placeholder. See 10-RAW-FINDINGS.

---

### Category 3: API Error and Informational Messages

**EVIDENCE — Consistent message structure**
- All errors: `{"error": "<short>", "message": "<actionable>"}` plus optional `"details": ...`
- Success messages: `{"success": true, "message": "...", ...}`

**EVIDENCE — HIPAA-aware error sanitization**
**Source:** `auth.py:1370-1376`:
```python
# HIPAA Compliance: Sanitize error messages to prevent PHI exposure
logger.error(f"Login error: {str(e)}", exc_info=True)
return jsonify({
    'error': 'Login failed',
    'message': 'An error occurred during login. Please try again.'
}), 500
```
Pattern repeated across `dashboard.py:127, 167, 273`, `auth.py`, etc.

**INFERENCE — Error responses do not leak DB error text** to the client in the typical path. Stack traces go to server logs only. EXCEPTION at `auth.py:731` `return jsonify({'error': f'Registration failed: {str(e)}'}), 500` — actual Stripe error message is interpolated and returned. This is inconsistent with the HIPAA sanitization pattern elsewhere.

#### Stripe key-validation messages (long-form, action-oriented)

**EVIDENCE** — Multi-line error messages with remediation steps
**Source:** `__init__.py:486-518` — when Stripe key validation fails (publishable key, restricted key, test in prod, live in non-prod):
```
STRIPE CONFIGURATION ERROR: <reason>
ACTION REQUIRED: Set STRIPE_SECRET_KEY to a valid secret key.
- For production: Add STRIPE_SECRET_KEY to Google Secret Manager (value should start with sk_live_)
- For development: Set STRIPE_SECRET_KEY_TEST environment variable (value should start with sk_test_)
- Get your secret key from: https://dashboard.stripe.com/apikeys
See README.md and STRIPE_SECRETS_MANAGEMENT.md for detailed setup instructions.
```
These prevent app startup (raise `ValueError`) and are also logged.

#### Webhook security messages

**EVIDENCE** — Detailed remediation in webhook handler errors
**Source:** `payments.py:386-393, 410-415, 436-446, 449-460`. Each error case includes:
- The cause
- Likely root causes (1, 2, 3 enumeration)
- Action required for prod vs dev
- A doc reference (`docs/stripe_webhook_setup.md`, `SECRET_ROTATION_GUIDE.md`)

**INFERENCE** — These long-form messages are intended for ops/server logs, NOT for client display. They are returned to the webhook caller (Stripe), which only cares about HTTP status code anyway.

---

### Category 4: Log Message Style

**EVIDENCE — Heavy emoji prefixes for categorization**
**Source:** Throughout `payments.py`, `integrations.py`, `auth.py`, `desktop.py`:
- 🎯 webhook entry
- 🔐 signature verification
- 📦 payload parsing
- 🔍 lookup
- ✅ success step
- ⚠️ warning
- ❌ failure
- 🔑 activation key
- 🔄 GHL sync
- 📥/📤 external call
- 🎬 handler entry
- 🏁 handler exit / WEBHOOK EXIT
- 💾 DB update
- 🚪 route entry
- 🆕 create

**INFERENCE — Intent: log-grep ergonomics.** These emoji-prefixed log lines make grep-based investigation easier (`grep '🔑' logs.txt` → all activation key events). Side effect: log volume is high (the webhook handler logs ~30+ lines per request). This may not survive a transition to Cloud Logging structured logs.

---

### CRM-Mastered Field Definition (semi-prompt-like configuration)

**EVIDENCE — Field set drives audit and read-only behavior**
**Source:** `audit_utils.py:15-41`:
```python
CRM_MASTERED_FIELDS = {
    'pharmacy_name', 'address', 'city', 'state', 'zip', 'phone',
    'ncpdp', 'npi',
    'pharmacy_license_number', 'pharmacist_license',
    'contact_person', 'contact_person_last_name',
    'business_email', 'website_url',
    'time_zone', 'preferred_contact_method',
    'country', 'pharmacy_software_system', 'role_in_pharmacy', 'mobile_number',
    'stripe_session_id', 'stripe_customer_id',
    'ghl_contact_id', 'ghl_company_id',
    'date_of_registration',
}
```

This declares which Business fields are "owned by CRM" — read-only for non-super-admins (`pharmacy.py:84-98`) and auto-audit-logged on update (`admin.py:633`).

**EVIDENCE — Sensitive field set drives masking**
**Source:** `audit_utils.py:43-53`:
```python
SENSITIVE_FIELDS = {'password', 'password_hash', 'api_key', 'secret', 'token',
                    'credit_card', 'ssn', 'tax_id'}
```
Used by `mask_sensitive_value` and `mask_sensitive_dict` (substring match on field name).

---

### Persona / Tone

**EVIDENCE — No "persona" in the LLM sense.** API responses use neutral SaaS phrasing. Email templates are friendly but functional.

**EVIDENCE — Internal log messages have a quirky, owner-driven personality** in CLAUDE.md / AGENTS.md / WINDSURF.md ("Stark Industries", "Tony", "Ironman Rule"). This personality belongs to the project owner ("Tony Stark" alias for the user) and shapes how AI agents are expected to collaborate, but it does not appear in user-facing copy.

---

## Open Questions

1. Are the three agent-config docs (CLAUDE.md, AGENTS.md, WINDSURF.md) maintained manually in sync, or is one canonical?
2. Does the FastAPI/Supabase tech-stack claim in CLAUDE.md represent intent for this project's future state? (The user's brief mentioned "we may need to migrate this Flask API to Supabase" — confirming alignment with CLAUDE.md.)
3. Is the password-reset HTML page's "paste reset code into desktop app" workflow intentional (token-relay) or a transitional UX?
4. Why does `auth.py:526-527` use placeholder `'https://your-app.com/success'` while other paths use `'https://pharmacybooks.com/thankyou'`?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (CLAUDE.md tech-stack mismatch, agent-doc drift between tools)
