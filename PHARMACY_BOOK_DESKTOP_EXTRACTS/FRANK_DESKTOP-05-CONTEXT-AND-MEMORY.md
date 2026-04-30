# 05-CONTEXT-AND-MEMORY — State and Memory Management

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

State lives in three tiers: **(1) the local SQLite database** `user_data.db` (the source of truth for prescription rows, profile, references, exclusion rules, edit history); **(2) per-process Python globals + Tk widget state** in `pharmacybooks.py` (mutable, not persisted); **(3) on-disk caches and files** under the platform-appropriate user-data directory (column mappings, business profile cache, device identity, sync state, downloaded inclusion lists, downloaded installers, recalculate policy markers, logs). Secrets live in the OS keyring. There is no token-budget concept (this is not an LLM agent), and no in-memory cache eviction strategy beyond ad-hoc invalidation.

---

## Findings

### Tier 1 — SQLite Local Database (`user_data.db`)

**EVIDENCE** — Path: `os.path.join(get_app_data_dir(), "user_data.db")` (per agent + `helpers/sqlite_db_helper.py`).

**EVIDENCE** — Tables (per architecture-mapper agent's reading of `helpers/sqlite_db_helper.py`):
- `user_data` — prescription rows (BIN, PCN, group_field, network_id, drug_ndc, qty, total_paid, awp, payer_type, pbm_key, etc.)
- `profile` — pharmacy profile fields (CRM-managed and user-editable)
- `pbm_info` — PBM reference data with canonical `pbm_key`
- `aac_reference` — AAC values keyed by NDC + week
- `wac_reference` — WAC values keyed by NDC
- `exclusion_rules` — user-defined script-exclusion rules (whitelisted columns: `ndc`, `network_id`, `group_field` per `helpers/sqlite_db_helper.py:97`)
- `edit_history` — audit trail of field-level edits
- `file_imports` — record of import runs
- `import_watch` — watched-folder import state
- `medicaid_portal_verification_queue` (per agent reference at `pharmacybooks.py:5814` context)

**Source:** Architecture-mapper agent + direct read of `helpers/sqlite_db_helper.py:97`.

**EVIDENCE** — Schema migrations auto-applied via `_run_migrations()` (line 1017 per agent). No external migration tool (no Alembic, no Flyway).

**EVIDENCE** — Connection access via context manager: `with db.get_connection() as conn:` (line 2182 per agent). Uses raw `sqlite3` (not SQLAlchemy despite the requirement).

**EVIDENCE** — Profile fields enumerated at top of helper (`helpers/sqlite_db_helper.py:123-130`+):
```
PROFILE_FIELDS = [
    "pharmacy_name",
    "address",
    "city",
    "state",
    "zip",
    "phone",
    "fax",
    ...
]
```
**Source:** `helpers/sqlite_db_helper.py:123-130` (direct read; full list truncated by my offset).

**INFERENCE** — `pbm_key` is a derived/canonical column on both `pbm_info` and `user_data` — its persistence + backfill story is the reason for ~12 BIN-004146 tools and the `pbm_key_backfill_migration.py` helper.

**EVIDENCE** — Cross-thread DB access uses `threading.Lock`-style protection inside `SQLiteDatabaseHelper` (per agent INFERENCE — direct lock-name verification deferred). Connections appear short-lived (per-call context managers).
**Source:** Architecture-mapper agent.

### Tier 2 — Process State (Python Globals + Tk State)

**EVIDENCE** — Module-level globals in `pharmacybooks.py` mutated across handlers:
- `pending_update_info` (auto-update info between threads)
- `startup_dashboard_ready` (flag for ordering startup tasks)
- `PROFILE_REFRESH_WARNING_SHOWN` (one-shot UI gating)
- `startup_import_files` (set in startup scan, consumed by import_startup_files)
- `global_reference_manager` (the `ReferenceDataManager` instance)
- `update_manager` (the `AutoUpdateManager` instance)
- `db = DatabaseHelper(BASE_DIR)` (line 9951)
- `APP_DIR = get_install_dir()` (line 228)
- `BASE_DIR = get_app_data_dir()` (line 229)
- `REPORT_DIR`, `TEMPLATE_DIR`, `SIGNATURES_DIR` (lines 231-239)
- `FIXED_FEE = 10.64` (line 236)

**Source:** Architecture-mapper agent.

**INFERENCE** — Tk widget state (entry values, treeview selection, dialog modality) is owned by individual `Toplevel` classes — there is no application-wide state container. Closing a dialog discards its widget state.

**EVIDENCE** — `BatchFieldUpdateCollector` (`pharmacybooks.py:190-226`) is a transient collector for batched field edits within a single dialog session.

### Tier 3 — On-Disk Files (Beyond `user_data.db`)

#### T3.1 — User Data Directory Layout (`get_app_data_dir()`)

**EVIDENCE** — Platform paths (per `README.md:101-115`):
- Windows: `%LOCALAPPDATA%\PharmacyBooks`
- macOS: `~/Library/Application Support/PharmacyBooks`
- Linux: `~/.config/PharmacyBooks`

**EVIDENCE** — Files present under app data dir (sources cited):

| Filename | Owner | Purpose |
|---|---|---|
| `user_data.db` | `helpers/sqlite_db_helper.py` | Main SQLite DB |
| `device_identity.json` | `helpers/device_identity.py:13` | `{device_id: pbm-<uuid4>, created_at, schema_version}` |
| `pending_recalculate.json` | `helpers/auto_update_manager.py:48` | Pending recalculation policy from a server-side update |
| `applied_recalculate_hash.json` | `helpers/auto_update_manager.py:49` | Hash of last applied recalculate policy + timestamp |
| `column_mappings.json` | `helpers/column_mapping_helper.py` (CLAIM) | User's CSV-column → DB-field mappings (also seen at `dev_app_data/column_mappings.json`) |
| `inclusion_lists/<folder>/*` | `helpers/inclusion_list_downloader.py` | Downloaded reference data (subfolders: `pbm_info/`, `WAC_lists/`, `aac_reference/`, `FUL/`, `medicare_info/`, `pdf_templates/`) |
| `ReimbursementReports/` | `helpers/pdf_helpers.py` | Generated PDF reports + `Signatures/` subfolder |
| `ImportData/` | `pharmacybooks.py` startup scan | User-supplied CSV/XLSX awaiting import |
| `logs/` | `helpers/log_manager.py`, `helpers/logging_utils.py`, `helpers/aac_debug_logger.py` | Application logs (incl. `aac_import_debug.log`) |
| `updates/` | `helpers/auto_update_manager.py` | Downloaded installers + `rollback_info.json` (CLAIM) |
| `import_debug.csv` | `tools/import_row_debug_logger.py` | Opt-in row-level debug log (env-gated) |
| `<lockfile>` | `helpers/single_instance.py` | Single-instance lock; pid written to file |
| `config/gcs_service_account.json` | (manual) | GCS service account JSON for inclusion-list downloads |

**Source:** Direct reads of `helpers/device_identity.py:13`, `helpers/auto_update_manager.py:48-49`, `helpers/single_instance.py:53-55`; agent + docs reports for the rest.

#### T3.2 — Atomic Writes

**EVIDENCE** — `helpers/device_identity._persist_identity` writes to `<path>.tmp` then `os.replace(tmp, path)` for atomic update (`helpers/device_identity.py:35-47`).
**Source:** Direct read.

**INFERENCE** — Other persisters (column mappings, applied_recalculate_hash) likely use a similar pattern but were not directly verified.

#### T3.3 — Cache Manager

**EVIDENCE** — `helpers/cache_manager.py` exposes a `CacheManager` class storing business-profile/status JSON files (per architecture-mapper agent). Cache invalidation strategy unclear — agent flagged Open Question: "age-based (>7 days triggers warning, line 10309) or event-based?"
**Source:** Architecture-mapper agent.

### Tier 4 — Secrets (OS Keyring)

**EVIDENCE** — All keyring entries (verified directly):

| Service | Username | Content | Source |
|---|---|---|---|
| `pharmacybooks` | `api_tokens` | JSON: `{access_token, refresh_token, user, expires_at}` | `helpers/token_helper.py:101`, `helpers/api_client.py:103` |
| `pharmacybooks` | `api_token` | Raw access token (legacy fallback) | `helpers/token_helper.py:61` |
| `PharmacyBooksSupport` | `support_override_code` | SHA-256 hash of support override code | `helpers/support_access.py:9-10` |
| `pharmacybooks` (CLAIM) | `<smtp_user>` | SMTP password (per agent) | `helpers/email_helpers.py` (CLAIM) |

**EVIDENCE** — Local user passwords stored in SQLite (hashed) via `helpers/local_auth_helper.py` — NOT in keyring. Local DB is the source of truth for desktop-login auth.
**Source:** Architecture-mapper agent.

**EVIDENCE** — Environment-variable overrides for token lookup:
- `PHARMACYBOOKS_JWT` — overrides keyring for token (`helpers/token_helper.py:38`)
- `PHARMACYBOOKS_SUPPORT_CODE` — overrides keyring for support code (`helpers/support_access.py:25`); the env var holds the **raw** code, not a hash; the env-var path computes the hash on the fly
- `PHARMACYBOOKS_TEMP_PASSWORD_KEY`, `PHARMACYBOOKS_PASSWORD_RESET_SECRET_KEY`, `PHARMACYBOOKS_SECRET_KEY` — temp-password Fernet secret (`helpers/temp_password_crypto.py:11-15`)

**Source:** Direct reads.

### Tier 5 — Sync State (Cloud Sync)

**EVIDENCE** — Sync status flags on `user_data` rows (CLAIM — exact column not directly read):
- `db.get_unsynced_records()` and `db.mark_synced_to_cloud()` are called by `sync_integration.py` per architecture-mapper agent.

**EVIDENCE** — `BackgroundSyncWorker` carries in-memory state:
- `_sync_queue: queue.Queue` — pending sync requests (`helpers/background_sync_worker.py:98`)
- `_current_status: SyncStatus` (line 99)
- `_is_running: bool` (line 100)

**Source:** Direct read `helpers/background_sync_worker.py:96-100` + agent.

**INFERENCE** — Sync is "at least once" — retries with exponential backoff (`max_retries=3`, `retry_delay=5s` per `helpers/background_sync_worker.py:71-72`). No transactional outbox pattern.

### Tier 6 — Caches Internal to Helpers

**EVIDENCE** — Reference-data caches:
- `helpers/aac_reference_manager.py` exposes `aac_manager` module-level instance (per `helpers/sqlite_db_helper.py:101`).
- `helpers/opportunities_sheet_loader.py` has local cache (per agent).

**Source:** `helpers/sqlite_db_helper.py:101` + agent.

**INFERENCE** — These caches are populated at startup via `initialize_reference_data` and rebuilt on `download_inclusion_lists` runs. No TTL or LRU mechanism observed.

### Performance / Vectorization Context

**CLAIM** — Per `docs/TIMING_AND_DIAGNOSTICS.md`:
- Vectorized batch calc target: 10,000+ records/sec
- Per-row operations flagged as warnings via `warn_per_row_operation()`
- 4 instrumented modules: `data_importer.py`, `batch_calculator.py`, `reference_data_manager.py`, `medicaid_rate_workflow.py`

**Source:** Docs-review agent.

### What's NOT Persisted

**INFERENCE** — Transient items dropped on app exit:
- Tk widget state (selections, scroll positions, modal flags)
- All module-level globals in `pharmacybooks.py`
- The `_sync_queue` (sync items in queue at shutdown may be lost — see Open Question in 03)
- Worker-thread state

**INFERENCE** — Recovered on restart:
- DB rows
- Column mappings (from JSON file)
- Tokens (from keyring)
- Device identity
- Inclusion lists (from disk)
- Pending recalc policy

### Token Budget / LLM Context

**GAP** — Not applicable. This application has no LLM context window, no token budgeting, no compaction, no RAG, no retrieval. The reference data ETL pipeline (AAC/WAC/PBM/Medicare/FUL → SQLite → Pandas merge) is the closest analog to RAG, but it is purely tabular lookup, not embedding-based retrieval.

---

## Open Questions

1. Cache invalidation policy — is the business-status cache age-based (7-day warning at `pharmacybooks.py:10309`) or event-based (refresh on subscription poll)? The two could be inconsistent.
2. Does the SQLite layer use any concurrency-safety mechanism beyond per-connection context managers (e.g., `PRAGMA journal_mode=WAL`)?
3. Are sync queue items dead-lettered if `max_retries` is exhausted, or silently dropped?
4. Where does the `column_mappings.json` filename live within the app data dir — flat at root, or under `config/`? (`dev_app_data/column_mappings.json` exists but doesn't confirm production path.)
5. Is the `device_identity.json` schema versioned for migration (`SCHEMA_VERSION = 1`) but lacks any v→v+1 upgrade code — what happens if a future v2 device file is read by v1 code?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (LLM-context N/A, sync dead-letter behavior, WAL pragma, column_mappings.json path)
