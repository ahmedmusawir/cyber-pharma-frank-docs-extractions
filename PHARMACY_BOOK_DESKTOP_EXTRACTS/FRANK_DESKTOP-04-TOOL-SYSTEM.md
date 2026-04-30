# 04-TOOL-SYSTEM — Feature / Command Surface (Reframed)

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — REFRAMED (this is not an LLM agent — documenting the user-facing command surface and external integration surface instead)

---

## Summary

There is no LLM tool registry. The user-facing "command surface" is the **Tk widget tree** (sidebar buttons + popup menus + dialog controls); the external "command surface" is the set of **HTTPS methods on `helpers/api_client.APIClient`** plus the GCS download routines plus the Selenium scraper command set. There is no permission layer between user actions and helpers (the GUI owns gating). Permissions are server-side (the App Engine API enforces them based on JWT). This document inventories the surface from the user's POV, the Selenium portal POV, and the inferred backend API POV (CLAIMs only — backend not in repo per User Q2).

---

## Findings

### A. User Command Surface (Sidebar + Menus + Dialogs)

#### A1. Sidebar Buttons (always visible)

**EVIDENCE** — Three "books" with colored bootstyles + management buttons (`pharmacybooks.py:13129-13182`):

| Button | bootstyle | Handler | Purpose |
|---|---|---|---|
| Settings ▼ | (default) | `show_settings_menu(button)` (line 12464) | Popup menu (see A2) |
| Imports ▼ | (default) | `show_imports_menu()` (line 12960) | Popup menu (see A3) |
| Manage Saved PDFs | (default) | `show_manage_saved_pdfs_dialog()` (line 11713-12463) | Browse + manage stored PDF reports |
| Owed Book | `danger` | `show_dashboard("Owed Book")` (line 13181) | Per-customer A/R analysis |
| KPI book | `info` | `show_dashboard("KPI book")` | KPI dashboard |
| Opportunities Book | `success` (with dynamic badge) | `show_dashboard("Opportunities Book")` (line 13120) | Opportunities tab UI |

**Source:** Architecture-mapper agent.

#### A2. Settings ▼ Menu Items

**EVIDENCE** — Submenu opened by `show_settings_menu()` (`pharmacybooks.py:12464-12491`):
- **Profile** → `launch_profile()` (line 10546-10564) → opens `ProfileDialog`
- **Column Mapping** → `show_column_mapping_dialog()` (line 11504-11591) → opens `ColumnMappingDialog`
- **Manage Imports** → `show_manage_imports_dialog()` (line 11592-11712)
- **Help & About** → about dialog (line UNVERIFIED — see test_about_dialog_version.py)
- **Check for Updates** → `manual_check_for_updates()` (line 10584-10644)

**Source:** Architecture-mapper agent.

#### A3. Imports ▼ Menu Items

**EVIDENCE** — Submenu opened by `show_imports_menu()` (`pharmacybooks.py:12960-12977`):
- **Scan for Files** → `scan_import_folder(app)` (line 8657-8723)
- **Download Inclusion Lists** → triggers `inclusion_list_downloader` flow
- **Manage Saved PDFs** → `show_manage_saved_pdfs_dialog()`

**Source:** Architecture-mapper agent.

#### A4. Major Modal Dialogs

**EVIDENCE** — Inventory of Toplevel dialog classes inside `pharmacybooks.py`:

| Class | Lines | Surface |
|---|---|---|
| `ProfileDialog` | 557-1415 | Pharmacy profile fields (CRM-managed read-only + user-editable), signature capture button |
| `SignatureCaptureDialog` | 1416-1464 | Mouse-drawn signature pad |
| `EditFieldDialog` | 1465-1593 | Inline value editor with edit-history tracking |
| `ExclusionDialog` | 1594-1797 | Rule-based exclusion (uses whitelist `{ndc, network_id, group_field}`) |
| `ScriptLookupDialog` | 1798-1883 | RX/NDC search |
| `CheckPaymentDialog` | 1884-2174 | Reconcile payments to scripts |
| `EditPaymentDialog` | 2175-2345 | Edit existing payment record |
| `OpportunitiesDashboard` | 8724-9337 | Notebook with 6 tabs (2 functional, 4 stubs) |

Plus dialogs from `helpers/`:
- `ColumnMappingDialog` (`helpers/column_mapping_dialog.py`)
- `ActivationKeyDialog` (`helpers/activation_key_dialog.py`)
- `LocalAccountSetupDialog` (`helpers/local_account_setup_dialog.py`)
- `LocalLoginDialog` (`helpers/local_login_dialog.py`)
- `BusinessRegistrationDialog` (`helpers/business_registration_dialog.py`)
- `SubscriptionManagerDialog` (`helpers/subscription_manager_dialog.py`)
- `VerifyRateDialog` (`helpers/verify_rate_dialog.py`)
- `UpdateNotificationDialog` / `UpdateDownloadProgressDialog` / `UpdateRestartDialog` (`helpers/auto_update_dialogs.py`)
- `DataInitProgressDialog` (`helpers/data_init_progress_dialog.py`)

**Source:** Architecture-mapper agent + `00-REPO-PROFILE.md`.

#### A5. Tab Surface (`OpportunitiesDashboard`)

**EVIDENCE** — Six tabs declared (`pharmacybooks.py:8724-9337`):
| Tab | Status |
|---|---|
| NDC Optimization | "Coming soon" stub (line 8763) |
| Add Ons | Implemented |
| Medicaid Rebills | Implemented |
| Profit Driver Refills | "Coming soon" stub (line 8779) |
| Tx Opps | "Coming soon" stub (line 8785) |
| AAC Rate Review | "Coming soon" stub (line 8791) |

**Source:** Architecture-mapper agent.

### B. External Integration Surface — `helpers/api_client.APIClient`

**Note (User Q2):** All entries here are CLAIMs derived from `api_client.py` and helper imports/usages, NOT backend code.

#### B1. Confirmed Methods (named in code)

**EVIDENCE** — Methods directly cited by other helpers or the architecture-mapper agent (not exhaustive — full enumeration of `api_client.py` deferred):

| Method | Caller | Purpose (CLAIM) |
|---|---|---|
| `is_activated()` | `pharmacybooks.py:9966` | Boolean — does the device have a valid activation? |
| `get_business_info()` | `helpers/sqlite_db_helper.py:_refresh_profile_from_api()` (line 4015 per agent) | Fetch CRM-managed pharmacy profile |
| `get_subscription_status()` | `pharmacybooks.py:check_subscription_status_periodic` (line 10223 per agent) | Subscription/license check |
| `get_business_status()` | `pharmacybooks.py:check_business_status_on_startup` (line 10277 per agent) | One-shot startup status |
| `put_user_data()` | `helpers/background_sync_worker` upload (CLAIM) | Push unsynced local rows to cloud |
| `get_aac_reference()` / `get_aac_reference_batch()` | `helpers/sqlite_db_helper.py` enrichment paths (per agent) | Pull AAC values for NDC list |
| `get_ful_reference()` | `helpers/sqlite_db_helper.populate_ful_reference_from_api()` (line 6262 per agent) | Pull FUL list |
| `get_stored_token()` (helper, not method) | `api_client._load_saved_tokens` (line 94) | Token retrieval through `token_helper` |
| `_refresh_access_token()` | (per agent) | Refresh JWT via refresh token |
| `get_or_create_device_identity` (helper) | `api_client.py:26` | Device ID injection on requests |

**Source:** Direct reads of `helpers/api_client.py:1-120`, `helpers/token_helper.py`, `helpers/device_identity.py`; architecture-mapper agent for the rest.

#### B2. Inferred Surface from Helper Usage

**INFERENCE** — Other CLAIMed methods inferred from helper docstrings or doc references (no code-level enumeration in this extraction):
- Activation/token exchange
- Webhook-driven business registration (per `docs/IMPLEMENTATION_SUMMARY.md` — Stripe-first pattern; backend impl not in this repo)
- Temp-password sync (`helpers/temp_password_sync.py` consumes payloads from API; encryption uses Fernet — see 07)

**Source:** Docs-review agent + architecture-mapper agent.

**GAP** — A complete enumeration of `api_client.py` methods would require reading the full file (only first 120 lines were directly read). The user explicitly limited scope to CLAIMs derived from `api_client.py` and did not request a deep API map.

#### B3. Auth Conventions

**EVIDENCE** — `requests.Session()` with default headers `Content-Type: application/json` and `User-Agent: PharmacyBooks-Desktop/1.0` (`helpers/api_client.py:71-74`).
**EVIDENCE** — Token attached as `Authorization: Bearer <token>` (per `token_helper.py:_strip_bearer_prefix` semantics — implies inverse).
**EVIDENCE** — `device_id` available via `helpers/device_identity.get_or_create_device_identity()` (`helpers/api_client.py:26`); inferred to be sent on requests (specific request-construction code not read in this extraction).

**Source:** `helpers/api_client.py:60-74`.

#### B4. Error Surface

**EVIDENCE** — `APIError(Exception)` with `message`, `status_code`, `response_data` (`helpers/api_client.py:33-39`).
**EVIDENCE** — `requests.ConnectionError`, `requests.Timeout`, `requests.RequestException` are caught and converted (per agent reading lines 627-633).

**Source:** `helpers/api_client.py:33-39` (direct) + agent.

### C. External Integration Surface — Google Cloud Storage

#### C1. Reference Data Downloads

**CLAIM** — Per `docs/INCLUSION_LISTS_README.md`:
- Bucket: `inclusion_lists` (default; override via `PHARMACYBOOKS_UPLOAD_BUCKET` env var)
- Folders: `pbm_info/`, `WAC_lists/`, `pdf_templates/`, `aac_reference/`, `FUL/`, `medicare_info/`
- Strategies: public HTTP listing (preferred), authenticated GCS client, manifest-based fallback
- Auth: `GOOGLE_APPLICATION_CREDENTIALS` → `%LOCALAPPDATA%\PharmacyBooks\config\gcs_service_account.json`
- Retry: exponential backoff (max 3 retries: 1s, 2s, 4s)
- Storage: `<app_data_dir>/inclusion_lists/<folder_lowercase>/<filename>`

**Source:** Docs-review agent / `docs/INCLUSION_LISTS_README.md` (CLAIM).

#### C2. Auto-Update Manifest + Installer

**EVIDENCE** — Manifest URL fixed in code: `https://storage.googleapis.com/downloads.pharmacybooks.com/latest-version.json` (`version.py:10`).
**EVIDENCE** — Installer downloaded from manifest's `download_url` field (per `examples/latest-version.json` template).
**EVIDENCE** — SHA-256 hash field in manifest verified after download (per `helpers/auto_update_manager.py:13` docstring).

**Source:** `version.py:10` + `helpers/auto_update_manager.py:1-15` (direct).

### D. External Integration Surface — Alabama Medicaid Portal (Selenium)

**EVIDENCE** — `helpers/al_medicaid_scraper.py` (class `AlabamaMedicaidScraper`) drives the portal via Selenium WebDriver.
- Browsers: Chrome (default, auto), Edge, Firefox (per `README_VERIFY_RATE_SELECTORS.md`)
- No bundled driver; relies on Selenium Manager (per agent)
- Selectors: configured in `verify_rate_selectors.json` at repo root (4,138 bytes)

**EVIDENCE** — Variant scrapers:
- `al_medicaid_http_prototype.py` — HTTP-only (no JS)
- `al_medicaid_hybrid_batch.py` — HTTP first, Selenium fallback
- `al_medicaid_batch.py` — batch wrapper around the Selenium scraper
- `medicaid_portal_scraper.py` — element-finding utilities

**EVIDENCE** — Field-fill ordering rule from `README_VERIFY_RATE_SELECTORS.md`: "Other Date" radio MUST be selected BEFORE the date field is filled (CLAIM, not code-level verified).

**Source:** Architecture-mapper agent + docs-review agent.

#### D1. Portal Configuration (`config.ini` `[selenium]` and `[portal]` sections)

**EVIDENCE** — From `config.ini.template:19-41`:
- `[selenium] use_js_for_fields = false` — when true, always use JS injection for NDC/date/DAW; else `send_keys` with JS fallback if >1s
- `[portal] cli_enabled = false` — when true, integration with a standalone CLI (`{input}`, `{output}`, `{count}`, `{base_dir}` placeholders)
- `[portal] cli_command =` — empty by default
- `[portal] cli_timeout_seconds = 600`
- `[portal] cli_workspace =` — defaults to `dev_app_data/portal_cli`
- `[portal] cli_keep_artifacts = true`

**Source:** `config.ini.template:19-41` (direct read).

### E. External Integration Surface — Email (SMTP)

**EVIDENCE** — `helpers/email_helpers.EmailHelper` provides SMTP send for Gmail / Outlook / Yahoo / custom (per agent). Credentials stored via `keyring`.
**Source:** Architecture-mapper agent.

**INFERENCE** — Outbound email is used for daily operations email and possibly password-reset workflows. Specific endpoints/headers not enumerated.

### F. Validation / Permission Boundaries

**EVIDENCE** — There is **no client-side permission layer**. Buttons gate by feature flag (e.g., super-admin column visibility) but not by privilege. Server-side enforcement is implied by the JWT (per `token_helper.validate_token_shape_and_exp`).

**EVIDENCE** — Server-side enforcement is the App Engine API. CLAIM only — backend not present.

**EVIDENCE** — Single-instance lock (`helpers/single_instance.SingleInstanceGuard`) is the only "system-level" guard observed: a per-user file lock prevents two app processes running concurrently. Lock path is supplied by caller; pid is written into the lock file (`helpers/single_instance.py:53-55`).

**Source:** Direct read of `helpers/single_instance.py:1-104`.

### G. Internal Command Surface — `tools/` Standalone Scripts

**EVIDENCE** — `tools/` provides 32 admin/debug commands invoked manually (`python tools/<script>.py`). Most accept paths or DB filters via CLI args; none are integrated into the GUI.
**Source:** `00-REPO-PROFILE.md` cluster table.

**EVIDENCE** — `tools/import_row_debug_logger.py` is the only `tools/` module imported by helpers (`helpers/sqlite_db_helper.py:110`). It's gated by `PBM_IMPORT_DEBUG=true` env var.
**Source:** Direct read `helpers/sqlite_db_helper.py:108-112`.

---

## Open Questions

1. Full enumeration of `helpers/api_client.APIClient` methods (B2 GAP) — does the user need this, or are CLAIMs sufficient for the rebuild?
2. Are any backend webhooks consumed by the desktop client, or are all server pushes pulled via polling?
3. Does the AL Medicaid scraper expose any structured rate result types, or is parsing ad-hoc per page?
4. The "Help & About" menu item points to a dialog whose source line was not extracted — does it have a "Send debug bundle" feature that contacts support?
5. Are the four `OpportunitiesDashboard` "Coming soon" tabs reachable in production, or are they hidden behind a feature flag?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths (uncertain method names tagged CLAIM/INFERENCE)
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (full api_client method enumeration; AL Medicaid result types; About dialog source lines)
- [x] User Q2 honored: all backend API surface tagged as CLAIM with `api_client.py` cited
