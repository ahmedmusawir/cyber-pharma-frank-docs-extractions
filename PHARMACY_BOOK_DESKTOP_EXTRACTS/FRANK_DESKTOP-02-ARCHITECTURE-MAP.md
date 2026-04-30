# 02-ARCHITECTURE-MAP — System Structure

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — PRIMARY focus

---

## Summary

The application is a **monolithic procedural Python desktop client** built around a single 13,200-line orchestrator (`pharmacybooks.py`) and a flat `helpers/` package of ~63 modules. It uses Tkinter (via `ttkbootstrap`) for the GUI; SQLite (`user_data.db`) for local storage; an HTTPS REST API on Google App Engine for remote business data; Google Cloud Storage for reference data downloads and auto-update artifacts; and Selenium for one external integration (Alabama Medicaid Portal). There is no service layer, no facade, no dependency injection — the orchestrator imports helpers directly and mutates module-level globals to coordinate state. The remote API backend is **not in this repo** (the User confirmed extraction scope).

---

## Findings

### Top-Level Diagram (Logical)

```
                  ┌──────────────────────────────────────────────────────┐
                  │                pharmacybooks.py (13,200 LOC)         │
                  │  • Tk root window + ttkbootstrap theme               │
                  │  • Sidebar (3 books) + content frame                 │
                  │  • All Toplevel dialogs as inner classes             │
                  │  • Module-level globals for cross-handler state      │
                  │  • root.after() timers + threading.Thread() workers  │
                  └─────┬─────────┬─────────┬────────┬──────────┬────────┘
                        │         │         │        │          │
                  ┌─────▼─┐ ┌─────▼─┐ ┌─────▼─┐ ┌────▼────┐ ┌───▼──────┐
                  │helpers│ │helpers│ │helpers│ │helpers/ │ │tools/    │
                  │  GUI  │ │ data  │ │ auth  │ │sync+upd │ │ scripts  │
                  └───┬───┘ └───┬───┘ └───┬───┘ └────┬────┘ └──────────┘
                      │         │         │          │
                      │   ┌─────▼─────────▼──────────▼────┐
                      │   │   helpers/api_client.py       │
                      │   │   (single HTTP gateway)       │──────► GCP App Engine API
                      │   └────────────────┬──────────────┘        (NOT in repo)
                      │                    │
                      │            ┌───────▼────────┐
                      └───────────►│ helpers/        │
                                   │ sqlite_db_helper│──► user_data.db (local)
                                   └────────────────┘
                          (helpers also call: keyring,
                           selenium, google-cloud-storage,
                           reportlab, pandas)
```

### Layer 1 — `pharmacybooks.py` Orchestrator

**EVIDENCE** — Single 577 KB / 13,200-line module with NO main `App` class. Architecture is procedural with module-level globals mutated by handler functions. Entry point is `if __name__ == '__main__':` at line **9906**.
**Source:** `pharmacybooks.py:9906` (architecture-mapper agent verified).

**EVIDENCE** — Major top-level classes (extracted via `^class ` grep):
| Line | Class | Role |
|---|---|---|
| 121 | `StartupPhaseTimer` | Context manager for startup-phase timing |
| 190 | `BatchFieldUpdateCollector` | Collects batched field-edit updates |
| 557 | `ProfileDialog(tk.Toplevel)` | Pharmacy profile editor + signature capture (lines 557-1415) |
| 1416 | `SignatureCaptureDialog` | Drawing-based signature capture |
| 1465 | `EditFieldDialog` | Inline field editor |
| 1594 | `ExclusionDialog` | Rule-based script exclusion |
| 1798 | `ScriptLookupDialog` | RX/NDC lookup |
| 1884 | `CheckPaymentDialog` | Payment reconciliation |
| 2175 | `EditPaymentDialog` | Edit payment records |
| 2346 | **`ReimbursementComparer`** | **6,156-line core data analysis engine** (lines 2346-8501) |
| 8724 | `OpportunitiesDashboard` | Tabbed Notebook with 6 tabs (4 are "Coming soon" stubs) |

**Source:** Architecture-mapper agent's grep results on `pharmacybooks.py`.

**INFERENCE** — `ReimbursementComparer` is the de-facto core domain class — by line count it dwarfs every other class in the codebase. Splitting it is a candidate refactor target, but extraction is read-only.

**EVIDENCE** — Startup sequence (verified at `pharmacybooks.py:9906-13200`):
1. Create Tk root (`ttkbootstrap.Window` if available, else `tk.Tk`) — line 9937 sets theme `"flatly"`
2. Initialize `db = DatabaseHelper(BASE_DIR)` — line 9951 (note: `DatabaseHelper` is an alias, actually `SQLiteDatabaseHelper`)
3. `APIClient.is_activated()` check — line 9966
4. If not activated → `ActivationKeyDialog` → `LocalAccountSetupDialog` → `ColumnMappingDialog` (lines 9973-10047)
5. `LocalLoginDialog` — line 10061
6. `initialize_reference_data()` with progress dialog — line 10121
7. ProfileDialog if profile incomplete — line 10207
8. `initialize_sync_manager()` + `start_background_sync()` — line 10214
9. Subscription check periodic (every 30 min via `root.after`) — line 10223
10. PBM candidate export thread (daemon) — line 10220
11. Update check thread (daemon) — line 10470
12. Sidebar + content frame construction — lines 10512-13182
13. `root.protocol("WM_DELETE_WINDOW", on_closing)` — line 13196
14. `root.mainloop()` — line 13200

**Source:** Architecture-mapper agent.

**EVIDENCE** — Sidebar buttons and their handlers (lines 10512-13182):
| Button | Handler | Line |
|---|---|---|
| Settings ▼ | `show_settings_menu()` | 12464 |
| Imports ▼ | `show_imports_menu()` | 12960 |
| Manage Saved PDFs | `show_manage_saved_pdfs_dialog()` | 11713 |
| Owed Book | `show_dashboard("Owed Book")` | 13181 |
| KPI book | `show_dashboard("KPI book")` | (per agent) |
| Opportunities Book | `show_dashboard("Opportunities Book")` | 13120 |

**Source:** Architecture-mapper agent.

**EVIDENCE** — `show_dashboard(book_name)` is the dashboard router (lines 9339-9370). It instantiates `OpportunitiesDashboard` for the Opportunities Book and other book-specific views in the content frame.
**Source:** Architecture-mapper agent.

**INFERENCE** — There is no UI framework abstraction. Each Toplevel class wires its own widgets, validation, and event bindings. Adding a new dialog requires editing `pharmacybooks.py` directly.

### Layer 2 — `helpers/` Modules

#### Cluster A — Authentication & Activation (8 modules)

**EVIDENCE** — Modules:
- `local_login_dialog.py` — local SQLite-backed login UI
- `local_auth_helper.py` — local password hash + user mgmt (class `LocalAuthHelper`)
- `api_login_dialog.py` — API-based login (legacy/alternative path)
- `activation_key_dialog.py` — first-run activation
- `local_account_setup_dialog.py` — first-run admin account creation
- `support_access.py` — support-override authentication via SHA-256 hash + keyring (verified directly: `helpers/support_access.py:13-50`)
- `temp_password_crypto.py` — Fernet decryption of support-issued temp passwords; **uses hardcoded fallback secret `"pharmacybooks-fallback-secret"`** (`helpers/temp_password_crypto.py:18`) — flagged in 07
- `temp_password_sync.py` — applies decrypted temp passwords to local users (`TempPasswordApplyResult`)

**Source:** Architecture-mapper agent + direct reads of `support_access.py`, `temp_password_crypto.py`.

#### Cluster B — API & Identity (4 modules)

**EVIDENCE** — `helpers/api_client.py` is the single HTTP gateway.
- Class `APIClient(base_url, timeout=30)` — `helpers/api_client.py:42-50`
- Default base URL fallback: `"https://pharmacy-books-desktop.ue.r.appspot.com/api"` (line 88)
- `requests.Session()` for connection reuse (line 60)
- Sets `User-Agent: 'PharmacyBooks-Desktop/1.0'` (line 73). **CONTRADICTION:** the literal user-agent says 1.0 but `__version__` is "1.5.18" — see 10-RAW-FINDINGS.
- Custom `APIError(Exception)` carrying `status_code` and `response_data` (lines 33-39)
- Loads stored token via `token_helper.get_stored_token()` and validates via `validate_token_shape_and_exp` (lines 90-118). If validation fails, calls `_clear_tokens()`.
- Token storage: keyring service `"pharmacybooks"`, username `"api_tokens"` (JSON) — line 103.

**Source:** Direct read of `helpers/api_client.py:1-120`.

**EVIDENCE** — `helpers/token_helper.py` provides canonical token lookup:
- Order: `PHARMACYBOOKS_JWT` env var → keyring `pharmacybooks/api_tokens` (JSON) → keyring `pharmacybooks/api_token` (raw legacy)
- Strips `"Bearer "` prefix
- `validate_token_shape_and_exp(token)` checks 3 segments, decodes payload, verifies `exp` claim
- `save_tokens(access_token, refresh_token, user, expires_at)` writes JSON to keyring

**Source:** Direct read of `helpers/token_helper.py:23-150`.

**EVIDENCE** — `helpers/device_identity.py` generates and persists `device_id` of the form `pbm-<uuid4>` to `device_identity.json` in the app data dir.
- `IDENTITY_FILENAME = "device_identity.json"`, `SCHEMA_VERSION = 1`
- Atomic write via `tmp_path` + `os.replace` (lines 36-47)
- Used by `helpers/api_client.py:26` (`get_or_create_device_identity`).

**Source:** Direct read of `helpers/device_identity.py:1-77`.

**INFERENCE** — `helpers/api_db_helper.py` is the legacy MySQL/API-based DB helper, superseded by `sqlite_db_helper.py` (per architecture-mapper agent — agent observed it has no test coverage and is no longer the active path). **GAP:** code-level confirmation that no caller uses `api_db_helper` was not performed.

#### Cluster C — Local Database (4 modules)

**EVIDENCE** — `helpers/sqlite_db_helper.py` (~4,500 lines per agent report) is the local DB gateway.
- DB path: `os.path.join(base_dir, "user_data.db")`
- Tables (per agent): `user_data`, `profile`, `pbm_info`, `aac_reference`, `wac_reference`, `exclusion_rules`, `edit_history`, `file_imports`, `import_watch`
- Connection helper: `get_connection()` yields `sqlite3.Connection` as context manager (line 2182)
- Migrations auto-applied via `_run_migrations()` (line 1017)
- Imports `api_client` directly (lines 116-118): `from .api_client import get_api_client, APIError, APIClient`
- **Whitelist for safe dynamic SQL:** `VALID_EXCLUSION_RULE_COLUMNS = {'ndc', 'network_id', 'group_field'}` (`helpers/sqlite_db_helper.py:97`)

**Source:** Direct read of `helpers/sqlite_db_helper.py:1-130` + architecture-mapper agent.

**EVIDENCE** — Other DB-adjacent modules:
- `sqlite_type_converter.py` — Python ↔ SQLite type conversion
- `cache_manager.py` — JSON-file cache of business profile/status
- `log_manager.py` — log file rotation/management

**Source:** Architecture-mapper agent.

#### Cluster D — Data Import & Mapping (5 modules)

**EVIDENCE** — Pipeline:
- `data_importer.py` — class `DataImporter`. Reads CSV/XLSX, applies column mapping, normalizes NDC, runs payer-type matching, enriches with PBM/AAC.
- `column_mapping_dialog.py` — UI for column → field mapping
- `column_mapping_helper.py` — load/save mappings to JSON; `normalize_column_name()` (line 54), `normalize_ndc_column()` (line 83), `_ndc_variants` (lines 49-52)
- `column_name_sanitizer.py` — header normalization (whitespace + lowercase)
- `csv_utils.py` — delimiter detection (`,` `;` `\t`)

**Source:** Architecture-mapper + docs-review agents.

#### Cluster E — Reference Data & Inclusion Lists (10 modules)

**EVIDENCE** — Modules:
- `reference_data_manager.py` — orchestrator (AAC/WAC/PBM/Medicare/FUL)
- `aac_reference_manager.py` — local AAC files (Average Amount Covered)
- `aac_enrichment.py` — joins AAC values onto user data
- `aac_debug_logger.py` — opt-in AAC lookup tracing
- `ful_reference_manager.py` — FUL (Federal Upper Limit)
- `pbm_file_loader.py` — class `PBMFileLoader`; CSV/XLSX PBM info loading
- `pbm_utils.py` — canonical `pbm_key` builder (BIN 004146 special case lives here per `docs/PBM_KEY_CANONICAL_MATCHING.md`, **not** in `pbm_file_loader.py` despite `BIN_004146_FINAL_SUMMARY.md` claim)
- `pbm_key_backfill_migration.py` — backfill historical `pbm_key` values
- `medicare_list_loader.py` — class `MedicareListLoader`
- `opportunities_sheet_loader.py` — load opportunities data with local cache
- `inclusion_list_downloader.py` — GCS downloader for `inclusion_lists/<folder>/<file>` (folders: `pbm_info/`, `WAC_lists/`, `aac_reference/`, `FUL/`, `medicare_info/`, `pdf_templates/`)

**Source:** Architecture-mapper + docs-review agents (`docs/INCLUSION_LISTS_README.md`, `docs/PBM_KEY_CANONICAL_MATCHING.md`).

#### Cluster F — Medicaid Workflows (7 modules)

**EVIDENCE** — Selenium-driven Alabama Medicaid Portal automation:
- `medicaid_rate_workflow.py` — class `MedicaidRateWorkflow` (orchestrator)
- `al_medicaid_scraper.py` — class `AlabamaMedicaidScraper`. Uses Selenium WebDriver (Chrome/Edge/Firefox). No bundled driver — relies on Selenium Manager.
- `al_medicaid_batch.py` — batch wrapper
- `al_medicaid_http_prototype.py` — HTTP-only prototype (no Selenium)
- `al_medicaid_hybrid_batch.py` — hybrid (HTTP first, Selenium fallback)
- `medicaid_portal_scraper.py` — Selenium element-finding utilities
- `verify_rate_dialog.py` — manual verification UI
- `verify_rate_config.py` — Selenium configuration loader

**Source:** Architecture-mapper agent + `docs/README_VERIFY_RATE_SELECTORS.md`.

#### Cluster G — Payer-Type Matching (3 modules)

**EVIDENCE** — Modules:
- `payer_type_matcher.py` — name-based match to payer category (Medicaid, Medicare, Commercial, Cash, Action Needed)
- `matching_type_constants.py` — enum/int mapping (`MATCHING_TYPE_BIN`, `MATCHING_TYPE_BIN_PCN`, `MATCHING_TYPE_BIN_PCN_GROUP`, `MATCHING_TYPE_BIN_PCN_GROUP_NETWORK_ID`, etc.) — verified at `helpers/sqlite_db_helper.py:34-47` (imported as a dependency)
- `payer_type_logger.py` — class `PayerTypeMatchLogger` for decision tracing

**Source:** Direct read of `helpers/sqlite_db_helper.py:34-47` + architecture-mapper agent.

#### Cluster H — Sync, Auto-Update (4 modules)

**EVIDENCE** — Background sync:
- `sync_integration.py` — class `SyncManager` (top-level orchestrator)
- `background_sync_worker.py` — class `BackgroundSyncWorker(db, api_client, batch_size=500, max_retries=3, retry_delay=5, periodic_interval=300, status_callback)` (`helpers/background_sync_worker.py:67-99`). Defines `SyncStatus` enum (IDLE, SYNCING, COMPLETE, ERROR, PAUSED) and `SyncMessage` for UI updates.
- `auto_update_manager.py` — manifest fetch, hash verification, recalc-policy file management (`pending_recalculate.json`, `applied_recalculate_hash.json`)
- `auto_update_dialogs.py` — `UpdateNotificationDialog`, `UpdateDownloadProgressDialog`, `UpdateRestartDialog`

**Source:** Direct read of `helpers/background_sync_worker.py:1-100`, `helpers/auto_update_manager.py:1-120`.

#### Cluster I — UI / Reports / Email (6 modules)

**EVIDENCE** — Modules:
- `ui_helpers.py` — Treeview factory, context menus, window icon
- `treeview_border_styling.py` — optional column borders
- `pdf_helpers.py` — class `PDFHelper`, `fill_pdf_form()` (PyPDF + ReportLab)
- `email_helpers.py` — class `EmailHelper`; SMTP for Gmail / Outlook / Yahoo / custom
- `subscription_manager_dialog.py` — subscription/license UI
- `data_init_progress_dialog.py` — class `DataInitProgressDialog`
- `business_registration_dialog.py` — registration flow UI

**Source:** Architecture-mapper agent.

#### Cluster J — Utilities (8 modules)

**EVIDENCE** — Modules:
- `path_helper.py` — `get_install_dir()`, `get_app_data_dir()`, `get_logs_dir()`, `get_inclusion_lists_dir()`, `get_reports_dir()`, `get_import_data_dir()`, `get_icons_dir()` (per docs)
- `config_helper.py` — INI loader (`get_api_config(base_dir)`)
- `date_utils.py` — `get_aac_wednesday_start`, `get_most_recent_wednesday`
- `logging_utils.py` — file-based logging setup
- `timing_logger.py` — class `TimingLogger` (hierarchical timing context manager)
- `batch_calculator.py` — class `BatchCalculator` (vectorized Medicaid calc)
- `single_instance.py` — class `SingleInstanceGuard` (file-lock; verified directly, lines 1-104)
- `pbm_debug_logger.py` — DataFrame diagnostic printer

**Source:** Direct read of `helpers/single_instance.py` + architecture-mapper agent.

### Layer 3 — `tools/` and `scripts/`

**EVIDENCE** — `tools/` houses 32 standalone diagnostic and migration scripts. They are NOT imported by `pharmacybooks.py` and run independently. Heavy concentration on BIN 004146 diagnosis (12 scripts) and PBM key debugging.
**Source:** `00-REPO-PROFILE.md` cluster table.

**EVIDENCE** — `tools/import_row_debug_logger.py` IS imported by `helpers/sqlite_db_helper.py:110` (`from tools.import_row_debug_logger import log_import_row`). This is the only `tools/` module imported by helpers — an unusual coupling because `tools/` is otherwise standalone.
**Source:** `helpers/sqlite_db_helper.py:108-112` (direct read).

**EVIDENCE** — `scripts/` has 2 maintenance scripts: `backfill_payer_type_matching.py`, `update_icon_rounded_corners.py`.
**Source:** Direct `ls`.

### Cross-Cutting Boundaries

#### B1 — GUI ↔ Helpers

**EVIDENCE** — `pharmacybooks.py` imports ~20 helpers directly at file top (lines 65-105 per agent). Pattern: `from helpers.<module> import <Class>`. There is **no facade** (`helpers/__init__.py` is empty).
**Source:** `helpers/__init__.py:1-2` (direct read).

**INFERENCE** — Adding a helper requires no registration; existing helpers are added by adding an import statement to `pharmacybooks.py`. This makes the orchestrator file the single point of coupling.

**INFERENCE** — Helpers do **not** import `pharmacybooks` (no circular imports observed by the architecture-mapper agent). Communication goes one way.

#### B2 — Helpers ↔ Remote API (CLAIM-only per scope)

**EVIDENCE** — `helpers/api_client.py` is the single HTTP gateway. Imported by:
- `helpers/sqlite_db_helper.py:116-118` (`get_api_client`, `APIError`, `APIClient`)
- `helpers/data_importer.py` (per agent — exact line not extracted)
- `helpers/sync_integration.py` and `helpers/background_sync_worker.py:25` (`from .api_client import APIClient, APIError`)
- `helpers/auto_update_manager.py` (uses `requests` directly, not `api_client`, for the public manifest/installer URLs — verified at `helpers/auto_update_manager.py:30`)
- `pharmacybooks.py` (per agent)

**Source:** Direct reads of `helpers/sqlite_db_helper.py:116`, `helpers/background_sync_worker.py:25`, `helpers/auto_update_manager.py:30`.

**CLAIM** — Per the user's Q2 directive, the remote API surface is documented as CLAIMs derived from `api_client.py` only — see `04-TOOL-SYSTEM.md`. Methods inferred from agent reports include `get_business_info()`, `get_subscription_status()`, `get_business_status()`, `put_user_data()`, `get_aac_reference()`, `get_aac_reference_batch()`, `get_ful_reference()`, etc. The backend repo is not present in this extraction.
**Source:** Architecture-mapper agent (CLAIM tag — exact method enumeration deferred to 04).

#### B3 — Helpers ↔ SQLite

**EVIDENCE** — `helpers/sqlite_db_helper.py` is the single SQLite gateway. Used by:
- `pharmacybooks.py` (heavily — most dialog Save handlers call `db.update_field_with_edit_tracking`, `db.insert_user_data`, etc.)
- `helpers/data_importer.py` (`db.insert_user_data` after import)
- `helpers/reference_data_manager.py` (`db.upsert_aac_reference_data`, `db.get_pbm_info`)
- `helpers/sync_integration.py` (`db.get_unsynced_records`, `db.mark_synced_to_cloud`)
- `helpers/background_sync_worker.py:24` (`from .sqlite_db_helper import SQLiteDatabaseHelper`)

**Source:** Direct read of `helpers/background_sync_worker.py:24`, agent.

**EVIDENCE** — Raw SQL is written using a mix of f-strings (for schema DDL) and parameterized statements (for queries). The whitelist `VALID_EXCLUSION_RULE_COLUMNS = {'ndc', 'network_id', 'group_field'}` (line 97) is the explicit guard against SQL injection in dynamic exclusion rule clauses.
**Source:** `helpers/sqlite_db_helper.py:97`.

#### B4 — Threading & Async

**EVIDENCE** — No `asyncio` anywhere. All concurrency uses:
- `threading.Thread(..., daemon=True)` for I/O work
- `root.after(ms, callback)` for UI dispatch back from worker threads
- `queue.Queue` in `BackgroundSyncWorker._sync_queue` (line 98)
- `threading.Event` as stop signal (`_stop_event` at line 97)

**Background threads spawned at startup** (per agent):
| Line | Thread purpose |
|---|---|
| 9903 | PBM candidate export (daemon) |
| 10215 | Sync worker initialization |
| 10220 | PBM candidate export trigger |
| 10470 | Update check (daemon) |
| 10764 | Update download (daemon) |
| 11010 | Recalculate task (daemon) |
| 5814 | Medicaid scraper workflow task (daemon) |

**Source:** Architecture-mapper agent.

**INFERENCE** — UI dispatch from threads uses the `root.after(0, lambda: ...)` pattern. There is no central event bus; each thread has bespoke callbacks.

#### B5 — Boundaries to External Systems

**EVIDENCE** — Egress points:
| Helper | External System | Protocol |
|---|---|---|
| `api_client.py` | App Engine REST API | HTTPS via `requests` |
| `auto_update_manager.py` | GCS public bucket `downloads.pharmacybooks.com` | HTTPS via `requests` |
| `inclusion_list_downloader.py` | GCS bucket `inclusion_lists` (configurable via `PHARMACYBOOKS_UPLOAD_BUCKET`) | google-cloud-storage SDK + public HTTP listing |
| `al_medicaid_scraper.py` | AL Medicaid Portal | Selenium WebDriver |
| `al_medicaid_http_prototype.py` | AL Medicaid Portal | HTTPS via `requests` (no JS) |
| `email_helpers.py` | SMTP servers (Gmail/Outlook/Yahoo/custom) | SMTP |
| `keyring` (multiple consumers) | OS credential store (DPAPI/Keychain/Secret Service) | platform API |

**Source:** Direct reads + architecture-mapper agent + `docs/INCLUSION_LISTS_README.md`.

### Notable Schema / Column Conventions

**EVIDENCE** — Canonical NDC column is `drug_ndc`, not `ndc`. Five-layer defensive validation across `helpers/data_importer.py`, `prepare_user_data_dict()`, `calculate_user_data_columns()`, `batch_calculator.calculate_batch()`, and `insert_user_data()`. Per `docs/NDC_COLUMN_MAPPING_GUIDE.md` (CLAIM, partial code confirmation in `helpers/column_mapping_helper.py:49-83`).
**Source:** `helpers/column_mapping_helper.py:49-83`.

**EVIDENCE** — `group` field stored as `group_field` to avoid SQL reserved-word conflict. Whitelisted for exclusion rules at `helpers/sqlite_db_helper.py:97`.
**Source:** `helpers/sqlite_db_helper.py:97`.

**EVIDENCE** — `pbm_key` is a derived canonical match key on both `user_data` and `pbm_info`. Format: hyphen-separated, lowercase, leading zeros stripped. Special case: BIN 004146 always normalized to `'4146'` and matched on BIN-only (per `docs/PBM_KEY_CANONICAL_MATCHING.md`). Implementation lives in `helpers/pbm_utils.py`.
**Source:** `docs/PBM_KEY_CANONICAL_MATCHING.md` (CLAIM).

### State Globals in `pharmacybooks.py`

**EVIDENCE** — Module-level globals identified by agent (line numbers approximate):
- `pending_update_info`
- `startup_dashboard_ready`
- `PROFILE_REFRESH_WARNING_SHOWN`
- `startup_import_files`
- `global_reference_manager`
- `update_manager`
- `db = DatabaseHelper(BASE_DIR)` (line 9951)
- `APP_DIR = get_install_dir()` (line 228)
- `BASE_DIR = get_app_data_dir()` (line 229)
- `REPORT_DIR`, `TEMPLATE_DIR`, `SIGNATURES_DIR` (lines 231-239)
- `FIXED_FEE = 10.64` (line 236) — **EVIDENCE of a magic number constant**

**Source:** Architecture-mapper agent.

**INFERENCE** — Mutable globals (`pending_update_info`, `startup_import_files`, etc.) are mutated from multiple handlers — testing in isolation is hard.

### Notable Architectural Smells (Read-Only Observations)

These are observations, not recommendations. Each is tagged.

**INFERENCE / QUESTION** — `OpportunitiesDashboard` has 6 tabs but only 2 are implemented (`Add Ons`, `Medicaid Rebills`); the other 4 (`NDC Optimization`, `Profit Driver Refills`, `Tx Opps`, `AAC Rate Review`) are "Coming soon" stubs (per agent). May be incomplete features.

**INFERENCE / QUESTION** — `ReimbursementComparer` is a 6,156-line class. Its responsibilities cross calculation, export, payment reconciliation, and exclusion logic. A re-architecture for the web rewrite will likely need to split it.

**INFERENCE / QUESTION** — `helpers/api_db_helper.py` may be dead code (legacy MySQL/API helper superseded by `sqlite_db_helper.py`). Code-level dead-code confirmation deferred.

**INFERENCE / QUESTION** — `helpers/temp_password_crypto.py` ships a hardcoded fallback secret. See 07-GUARDRAILS for security implications.

**INFERENCE / QUESTION** — User-Agent string `'PharmacyBooks-Desktop/1.0'` (`api_client.py:73`) does not match runtime version `1.5.18` (`version.py:5`). Hardcoded.

---

## Open Questions

1. Is `helpers/api_db_helper.py` dead code, or still imported somewhere?
2. Are the 4 unimplemented `OpportunitiesDashboard` tabs planned features or vestigial?
3. Is `ReimbursementComparer` re-targetable as a server-side service for the web rewrite, or does its size hide tight coupling to the Tk dialog tree?
4. Why does `pharmacybooks.py` import `tools.import_row_debug_logger` (`helpers/sqlite_db_helper.py:110`) when the rest of `tools/` is standalone?
5. The `User-Agent` hardcoded as `1.0` is a CLAIM the server might use for analytics — is anything keying off it?
6. Why was `sqlalchemy` declared as a dependency if `sqlite_db_helper.py` uses raw `sqlite3`?
7. The `download-site/` and `site - registration/` HTML/JS folders look like marketing/registration pages co-located with the desktop client — should they be in a sibling repo?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist (paths from direct Read or sub-agent reports with line citations)
- [x] No invented function names or file paths (line numbers attributed; uncertain ones tagged INFERENCE)
- [x] No synthesis or recommendations included (smells flagged as observations only)
- [x] GAPs explicitly documented (api_db_helper dead-code status, OpportunitiesDashboard tab status, full method enumeration of api_client deferred to 04)
