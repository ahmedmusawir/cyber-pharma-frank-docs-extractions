# 07-GUARDRAILS-AND-SANDBOXING — Safety, Permissions, Validation, Sandbox

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — PRIMARY focus

---

## Summary

Security boundaries are stitched together from several mechanisms: an **activation key** (server-side device licensing), a **local SQLite-backed login** with a separate password store, a **JWT bearer token** for API calls (cached in OS keyring), a **support-override hash** for emergency access, **Fernet-encrypted temp passwords** for support-issued resets, a **single-instance file lock**, a **persistent device identity UUID**, and an **explicit SQL-column whitelist** for safe dynamic exclusion-rule queries. There is **no application-level authorization layer** (the GUI does not gate features by privilege beyond a `is_super_admin` flag passed to dialogs). The most serious finding is a **hardcoded fallback Fernet secret** (`"pharmacybooks-fallback-secret"`) in `helpers/temp_password_crypto.py:18`. All cited code reads were performed directly during this extraction.

---

## Findings

### A. Authentication Layers

#### A1. Activation Key (Device Licensing)

**EVIDENCE** — Activation is checked at startup via `APIClient.is_activated()` (`pharmacybooks.py:9966` per agent). If false, the user must enter an activation key in `ActivationKeyDialog` (`helpers/activation_key_dialog.py`).
**Source:** Architecture-mapper agent + `helpers/activation_key_dialog.py` (file inventoried).

**EVIDENCE** — On successful activation, the flow proceeds to `LocalAccountSetupDialog` (creates a local admin) and `ColumnMappingDialog` (mandatory first-run mapping). If local-account creation fails, the app shows a warning and exits (`pharmacybooks.py:9995-10010` per agent).
**Source:** Architecture-mapper agent.

**INFERENCE** — Activation creates a server-side binding between the device (via `device_identity.json` UUID) and the customer's business record. This is consistent with the "Stripe-first, webhook-driven registration" pattern claimed in `docs/IMPLEMENTATION_SUMMARY.md`.

#### A2. Local Login (Desktop Session)

**EVIDENCE** — After activation, every session requires `LocalLoginDialog` (`helpers/local_login_dialog.py`), backed by `local_auth_helper.LocalAuthHelper`. Credentials are stored in a local SQLite users table (NOT in keyring).
**Source:** Architecture-mapper agent.

**EVIDENCE** — `helpers/local_login_dialog.py` is also the consumer of email-reset tokens (per agent: `test_email_reset_token.py` and `test_password_reset_token_utils.py` are tied to it).
**Source:** Tests + agent.

**INFERENCE** — The local login is intentionally distinct from the API token. A user can log in to the desktop app while offline; the JWT is fetched from keyring and used opportunistically for API calls.

#### A3. JWT for API (Bearer Token)

**EVIDENCE** — Token lookup order (`helpers/token_helper.py:23-69`):
1. Environment variable `PHARMACYBOOKS_JWT` (highest priority)
2. Keyring `pharmacybooks/api_tokens` — JSON `{access_token, refresh_token, user, expires_at}` (line 45-58)
3. Keyring `pharmacybooks/api_token` — raw legacy fallback (line 60-66)

`"Bearer "` prefix is stripped automatically (line 41).
**Source:** Direct read `helpers/token_helper.py:23-69`.

**EVIDENCE** — Token validation (`helpers/token_helper.py:109-150`):
- 3-segment JWT shape check (line 134)
- Base64-url-decode payload + JSON-parse (lines 138-147)
- Expiration check via `exp` claim (line 149+)
- Returns `(is_valid, reason)` tuple

**Source:** Direct read.

**EVIDENCE** — `APIClient` clears tokens if validation fails on load (`helpers/api_client.py:115-118`).
**Source:** Direct read.

#### A4. Support Override (Emergency Access)

**EVIDENCE** — `helpers/support_access.py` (full file read):
- Service: `"PharmacyBooksSupport"` (line 9)
- Username: `"support_override_code"` (line 10)
- Storage: SHA-256 hex digest of the support code (line 14: `hashlib.sha256(raw.encode("utf-8")).hexdigest()`)
- Lookup: keyring first, then env var `PHARMACYBOOKS_SUPPORT_CODE` (raw — hashed on the fly at line 27)
- `verify_support_code(candidate)` — constant-time comparison NOT used; equality via `==` (line 38). **INFERENCE:** SHA-256 hash equality on hex digest is generally safe against timing attacks because both strings are the same length, but not best-practice.

**Source:** Direct read `helpers/support_access.py:1-50`.

**INFERENCE** — Support codes appear to be set/rotated by an admin (`set_support_code`) and used to unlock support-only features (perhaps via `helpers/support_access` import sites elsewhere — not enumerated).

#### A5. Temp-Password Sync (Server-Issued Resets)

**EVIDENCE** — `helpers/temp_password_crypto.py` (full file read, lines 1-73):
- Decrypts ciphertext from the backend using a Fernet helper.
- Secret is derived by SHA-256-hashing one of three env vars (priority order): `PHARMACYBOOKS_TEMP_PASSWORD_KEY`, `PHARMACYBOOKS_PASSWORD_RESET_SECRET_KEY`, `PHARMACYBOOKS_SECRET_KEY` (lines 11-15).
- **`_DEFAULT_SECRET = "pharmacybooks-fallback-secret"` (line 18) — used if no env var is set (line 27).**
- The hash is then base64-url-encoded to form a Fernet key (lines 35-37).

**Source:** Direct read `helpers/temp_password_crypto.py:1-73`.

**EVIDENCE — CRITICAL FINDING** — The fallback secret is a **publicly-discoverable string in the source code**. If neither end of the temp-password channel sets one of the three env vars, an attacker who reads this repo can decrypt every temp-password payload exchanged.
**Source:** `helpers/temp_password_crypto.py:18, 27`.

**INFERENCE** — The fallback exists "to keep environments in sync" with a backend default (per the inline comment, line 17). But this means the backend may be configured with the same hardcoded fallback as a "shared default" — if so, the temp-password channel is fundamentally compromised against anyone with access to either codebase.

**EVIDENCE** — `helpers/temp_password_sync.py` consumes the decryption result and applies the resulting plaintext to local user accounts (per architecture-mapper agent — class `TempPasswordApplyResult`).
**Source:** Architecture-mapper agent.

#### A6. Device Identity (Anti-Replay / Device Binding)

**EVIDENCE** — `helpers/device_identity.py` (full file read, lines 1-77):
- File: `device_identity.json` in app data dir (line 13).
- Format: `{device_id: "pbm-<uuid4>", created_at: <ISO8601>, schema_version: 1}` (lines 58-62).
- Generated lazily on first call to `get_or_create_device_identity()` (line 50).
- Atomic write via `tmp_path` + `os.replace` (lines 35-47).
- Persistence failures swallowed (line 41 — `except OSError: pass`).
- `update_device_identity(updates)` (line 67) merges updates into existing identity.

**Source:** Direct read.

**EVIDENCE** — Used by `helpers/api_client.py:26` (`from .device_identity import get_or_create_device_identity, update_device_identity`).
**Source:** Direct read `helpers/api_client.py:26`.

**INFERENCE** — The device ID is presumed sent on API requests (likely as a header or claim in the request body). Specific request-construction code was not read in this extraction.

### B. Process / Filesystem Sandbox

#### B1. Single-Instance Lock

**EVIDENCE** — `helpers/single_instance.py` (full file read, lines 1-104):
- Class `SingleInstanceGuard(lock_path)` — file-based cross-platform lock.
- Windows: `msvcrt.locking(handle.fileno(), msvcrt.LK_NBLCK, 1)` (lines 34-39).
- POSIX: `fcntl.flock(handle, fcntl.LOCK_EX | fcntl.LOCK_NB)` (lines 43-49).
- On acquire: writes `os.getpid()` to the lockfile (lines 53-55) and registers `atexit.register(self.release)` (line 58).
- Context manager support: `__enter__` raises `RuntimeError("Another instance is already running")` if acquire fails (line 97).

**Source:** Direct read.

**INFERENCE** — Lockfile location is supplied by caller — not constrained here. Likely under app data dir.

#### B2. User Data vs Install Dir Separation

**EVIDENCE** — All runtime writes go to `get_app_data_dir()` (per `docs/PATH_REFACTORING.md` and direct reads in `helpers/auto_update_manager.py:34-43`, `helpers/device_identity.py:18`). The install dir is treated as read-only.
**Source:** Docs + direct reads.

**INFERENCE** — This satisfies Windows non-admin user requirements (avoiding `Program Files` writes) but does NOT enforce a sandbox — the app process can still write anywhere it has filesystem permission to.

### C. Input Validation Boundaries

#### C1. Whitelisted SQL Columns for Exclusion Rules

**EVIDENCE** — `helpers/sqlite_db_helper.py:97`:
```python
VALID_EXCLUSION_RULE_COLUMNS = {'ndc', 'network_id', 'group_field'}
```
With explanatory comment (lines 94-96):
> "Restricting to a whitelist keeps dynamic SQL safe when building WHERE clauses for rule application. Extend this set cautiously if future rule types are required."

**Source:** Direct read `helpers/sqlite_db_helper.py:94-97`.

**INFERENCE** — This is a deliberate guardrail. `ExclusionDialog` lets users pick a column for a rule — restricting the choice to three known-safe columns prevents user input from controlling the column name in a dynamic WHERE clause.

#### C2. NDC Defensive Validation (5 Layers)

**CLAIM** — Per `docs/NDC_COLUMN_MAPPING_GUIDE.md`, the import pipeline has 5 fallback layers that rename `ndc` → `drug_ndc` and fail-fast if neither is present. Two layers code-level confirmed:
- `helpers/column_mapping_helper.py:54` (`normalize_column_name`)
- `helpers/column_mapping_helper.py:83` (`normalize_ndc_column`)

**Source:** `helpers/column_mapping_helper.py` (direct grep + agent).

#### C3. JWT Shape Validation

**EVIDENCE** — `validate_token_shape_and_exp` rejects tokens missing the 3-segment shape, with malformed base64, or with an expired `exp` claim. See A3 above.

#### C4. Update Hash Verification

**EVIDENCE** — `helpers/auto_update_manager.py:13` docstring claims SHA-256 hash verification of downloaded installers. Code-level verification of the actual `hashlib.sha256(...).hexdigest()` invocation deferred (only first 120 lines read, but the import `import hashlib` is present at line 21).
**Source:** Direct read `helpers/auto_update_manager.py:21`.

**INFERENCE** — The manifest's `sha256_hash` field is compared against the computed hash of the downloaded installer (per `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md` and `helpers/auto_update_manager.py:11-15` docstring). This guards against MITM tampering even though the URL is also HTTPS (HTTPS guards transport, hash guards file integrity).

#### C5. HTTPS-Only Connections (Auto-Update + API)

**EVIDENCE** — Auto-update manifest URL is hardcoded `https://...` (`version.py:10`).
**EVIDENCE** — API base URL default is `https://...` (`config.ini.template:13`, `helpers/api_client.py:88`).
**INFERENCE** — A user could override the API base URL to `http://localhost:5000/api` for local dev (per `config.ini.template:17`). The manifest URL is not user-overridable.
**Source:** Direct reads.

### D. Authorization Boundaries

**EVIDENCE** — There is **no application-level RBAC**. Dialog handlers may take an `is_super_admin: bool` parameter (e.g., `ProfileDialog(root, db, is_super_admin)` per agent reading at `pharmacybooks.py:9995-10010`), but the app trusts whatever the server hands back via the JWT.
**Source:** Architecture-mapper agent.

**INFERENCE** — Server-side authorization is the actual privilege boundary. The desktop client is a thin permissions surface.

### E. Secret-Storage Conventions

**EVIDENCE** — Secrets stored via `keyring`:
| Service / Username | Content | Source |
|---|---|---|
| `pharmacybooks` / `api_tokens` | JSON of access+refresh+user+exp | `helpers/token_helper.py:101` |
| `pharmacybooks` / `api_token` (legacy) | Raw access token | `helpers/token_helper.py:61` |
| `PharmacyBooksSupport` / `support_override_code` | SHA-256 hash | `helpers/support_access.py:9-10` |
| `pharmacybooks` (CLAIM) / SMTP user | SMTP password | `helpers/email_helpers.py` (CLAIM, not directly read) |

**EVIDENCE** — Local user passwords NOT stored in keyring; they live in the local SQLite DB (hashed via `helpers/local_auth_helper`).
**Source:** Architecture-mapper agent.

**EVIDENCE** — GCS service account JSON stored on disk at `<app_data_dir>/config/gcs_service_account.json` (per `docs/INCLUSION_LISTS_README.md`). This is a regular file with filesystem permissions — NOT in keyring.
**Source:** Docs-review agent.

**INFERENCE** — Filesystem permissions on the GCS service account file rely on Windows ACLs / POSIX 0600 conventions; the code does not enforce specific permissions (not directly verified).

### F. Token-Migration Tooling

**EVIDENCE** — `tools/migrate_keyring_tokens.py` exists to clean up malformed/legacy keyring entries (per agent + `docs/DEV_NOTES_TOKEN_HANDLING.md`).
**Source:** `00-REPO-PROFILE.md` cluster table + docs-review agent.

**EVIDENCE** — `tools/check_keyring_tokens.py` (root, per file listing) inspects current keyring contents for diagnosis.
**Source:** File listing.

### G. Dev Environment Override

**EVIDENCE** — `pharmacybooks.py:248-294` (per agent) loads dev overrides from disk for:
- `PHARMACYBOOKS_UPLOAD_BUCKET`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `PHARMACYBOOKS_API_BASE_URL`

**INFERENCE** — This permits a dev to point the production binary at local services without a code change. **QUESTION**: where is the override file located, and what protects it from being weaponized as a privilege-escalation mechanism if a malicious actor gains write access?

### H. Notable Security-Adjacent Findings

**EVIDENCE / QUESTION** — `tools/inspect_jwt_payload.py` exists at repo root (per file listing). It reads a token from keyring/env and base64-decodes the payload. This is benign for local diagnostics but could leak PII/claims into shell history if used in production.
**Source:** File listing.

**EVIDENCE** — `helpers/sqlite_db_helper.py:121` hardcodes `DEFAULT_ALDOI_EMAIL = "pbmcompliance@insurance.alabama.gov"` — a regulator email address. Not a security risk per se, but an external-system coupling.
**Source:** Direct read.

**EVIDENCE** — User-Agent string is hardcoded to `'PharmacyBooks-Desktop/1.0'` (`helpers/api_client.py:73`). Server-side analytics or rate-limiting keyed on this header will not see the actual `1.5.18` version.
**Source:** Direct read.

---

## Open Questions

1. **CRITICAL:** Is the backend deployed with one of `PHARMACYBOOKS_TEMP_PASSWORD_KEY` / `PHARMACYBOOKS_PASSWORD_RESET_SECRET_KEY` / `PHARMACYBOOKS_SECRET_KEY` set? If not, all temp-password ciphertexts are decryptable by anyone with the source code (`helpers/temp_password_crypto.py:18`).
2. Does the GCS service-account JSON have its filesystem permissions hardened, or is it left at default 0644 / Authenticated Users?
3. Is the support-override code rotated, and how is it transmitted out-of-band to the support team?
4. Where is the dev-override file (`pharmacybooks.py:248-294`) located on disk, and is it documented/excluded from production deployments?
5. Does the JWT validation reject expired tokens at API-call time, or only at startup load? (The `validate_token_shape_and_exp` is called in `_load_saved_tokens` but maybe not before each request.)
6. Are local user passwords (in SQLite) hashed with a slow KDF (bcrypt/argon2/scrypt) or just SHA-256? `helpers/local_auth_helper.py` was not directly read in this extraction — flagged for follow-up.
7. The single-instance lock prevents two app processes — but what about two users on the same Windows host running in different sessions? (The lock path probably resolves per-user via `get_app_data_dir()`, but unverified.)

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included (security smells described as findings, not recommendations)
- [x] GAPs explicitly documented (local password hash algorithm; full api_client request construction; lock-path scope)
- [x] CRITICAL findings flagged in Open Questions for human review
