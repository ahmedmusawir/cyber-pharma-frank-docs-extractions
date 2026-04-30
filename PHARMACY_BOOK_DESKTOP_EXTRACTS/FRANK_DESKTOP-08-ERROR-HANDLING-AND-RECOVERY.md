# 08-ERROR-HANDLING-AND-RECOVERY — Failures, Retries, Recovery, Interrupts

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — PRIMARY focus

---

## Summary

The application has **two well-developed recovery subsystems** (auto-update with hash-verified rollback markers, and a sync worker with exponential-backoff retries) and **broad ad-hoc try/except handling** elsewhere. Network errors against the App Engine API surface as `APIError` and are caught at call sites; the app degrades to local SQLite-only mode rather than failing hard. The auto-update path tracks pending recalculate policies in two JSON marker files (`pending_recalculate.json`, `applied_recalculate_hash.json`) so that a recalc triggered by a server-pushed policy survives across restarts. There is no formal circuit-breaker, no dead-letter queue, and no centralized error-reporting/telemetry pipeline. Most "recovery" is implicit: the user re-launches and the next periodic poll/sync succeeds.

---

## Findings

### A. Error-Type Surface

**EVIDENCE** — `APIError(Exception)` is the central API exception (`helpers/api_client.py:33-39`):
```python
class APIError(Exception):
    def __init__(self, message: str, status_code: Optional[int] = None,
                 response_data: Optional[Dict] = None):
        self.message = message
        self.status_code = status_code
        self.response_data = response_data or {}
        super().__init__(self.message)
```
**Source:** Direct read.

**EVIDENCE** — Caught network-level exceptions at API call sites: `requests.ConnectionError`, `requests.Timeout`, `requests.RequestException` (architecture-mapper agent reading `helpers/api_client.py:627-633`).
**Source:** Architecture-mapper agent.

**EVIDENCE** — `MissingMappedColumnsError` raised by `data_importer.py` when required column mappings are absent (per architecture-mapper agent's note on the import flow).
**Source:** Architecture-mapper agent.

**EVIDENCE** — `OSError` swallowed in identity persistence (`helpers/device_identity.py:41-47`):
```python
except OSError:
    # Persistence failures should never block the app.
    try:
        if os.path.exists(tmp_path):
            os.remove(tmp_path)
    except OSError:
        pass
```
**Source:** Direct read.

**EVIDENCE** — `cryptography.fernet.InvalidToken` caught in `decrypt_secret` and converted to `None` return (`helpers/temp_password_crypto.py:57-58`):
```python
except InvalidToken:
    return None
```
**Source:** Direct read.

**INFERENCE** — Returning `None` for decryption failure is a deliberate choice to avoid leaking decryption-failure details. Caller must handle `None` to surface a user-friendly error.

### B. Auto-Update Recovery System

#### B1. Update Manifest Fetch Failures

**EVIDENCE** — From `helpers/auto_update_manager.py` docstring (lines 1-15):
- Background, non-blocking checks
- HTTPS-only connections
- SHA-256 hash verification
- User confirmation required before applying
- Rollback mechanism (keep previous version until update succeeds)
- Comprehensive logging

**Source:** Direct read `helpers/auto_update_manager.py:1-16`.

**EVIDENCE** — On manifest read failure: warning logged, no update returned (`helpers/auto_update_manager.py:73-78`):
```python
def load_pending_recalculate_policy() -> Optional[Dict[str, Any]]:
    path = _get_pending_recalculate_path()
    if not os.path.exists(path):
        return None
    try:
        with open(path, 'r', encoding='utf-8') as f:
            return json.load(f)
    except Exception as e:
        logger.warning(f"[AUTO-UPDATE] Error reading pending recalculation policy: {e}")
        return None
```
**Source:** Direct read.

**INFERENCE** — Pattern: catch broad `Exception`, log warning, return `None`. Repeated for `clear_pending_recalculate_policy` (lines 81-87), `load_applied_recalculate_hash` (lines 90-100), `save_applied_recalculate_hash` (lines 103-115).

#### B2. Recalculate Policy Persistence (Recovery Across Restarts)

**EVIDENCE** — Two JSON marker files in app data dir (`helpers/auto_update_manager.py:48-49`):
- `pending_recalculate.json` — server-pushed policy waiting to be applied
- `applied_recalculate_hash.json` — hash of the last applied policy + ISO timestamp

**Source:** Direct read.

**EVIDENCE** — `should_apply_recalculate_policy(policy, current_version)` (line 118+) compares `recalculate_hash` against the saved applied hash to avoid re-running the same policy.
**Source:** Direct read (partial).

**INFERENCE** — This is the only durable "outbox-like" pattern in the codebase. It survives crashes and ensures server-mandated recalculations execute exactly once across restarts.

#### B3. Download Failure Modes (per `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md`)

**CLAIM** — Per `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md` (extracted by docs-review agent):
| Failure | Recovery |
|---|---|
| Network error | Log + retry on next 6-hour check (non-blocking) |
| 404 manifest | Log warning, no-op |
| Malformed JSON | Log error, no-op |
| Missing required fields | Log error, no-op |
| Download interrupted | User notified + can retry |
| Hash mismatch | Delete file, notify user, allow retry |
| Installer launch failed | Show error + provide manual installer path |

**Source:** Docs-review agent / `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md` (CLAIM).

#### B4. Rollback Mechanism

**CLAIM** — `helpers/auto_update_manager.py:8` docstring claims "Rollback mechanism (keeping previous version until update succeeds)". `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md` mentions a `rollback_info.json` written before installer launch, plus a `.last_update_check` timestamp file.
**Source:** Direct read of docstring + docs-review agent.

**GAP** — Code-level verification of the actual rollback logic (which files are preserved, where, for how long) was not performed in this extraction.

### C. Background Sync Worker Recovery

**EVIDENCE** — Defaults from `helpers/background_sync_worker.py:67-93`:
- `batch_size=500`
- `max_retries=3`
- `retry_delay=5` (seconds, base for backoff)
- `periodic_interval=300` (seconds)

**Source:** Direct read.

**EVIDENCE** — `SyncStatus` enum values include `ERROR` and `PAUSED` (`helpers/background_sync_worker.py:35-41`). Errors are surfaced via `status_callback(SyncMessage(SyncStatus.ERROR, message, details))` rather than raising.

**Source:** Direct read.

**EVIDENCE** — Stop signal: `_stop_event = threading.Event()` (line 97). Worker thread checks the event in its loop (CLAIM, body of worker loop not directly read).

**Source:** Direct read of constructor, full worker loop deferred.

**INFERENCE** — Sync errors are non-fatal — the next periodic tick re-tries unsynced records. No dead-letter queue: if `max_retries` is exhausted, the records remain "unsynced" and will be picked up again on the next cycle (assuming the underlying error has resolved).

**INFERENCE** — There is **no idempotency token** sent with sync uploads (CLAIM — based on no evidence of one in the constructor signature). If a request is sent and the response is lost, a retry could double-write on the server. Server-side de-duplication is implied but not verifiable from this repo.

### D. Profile / API Refresh Fallback

**EVIDENCE** — `helpers/sqlite_db_helper.get_profile()` (line 3781 per agent) supports `use_cache=True/False`. With `use_cache=False`, calls `_refresh_profile_from_api()` (line 4007), which calls `api_client.get_business_info()` (line 4015). If that fails, falls back to local cache.
**Source:** Architecture-mapper agent.

**EVIDENCE** — Per `API_FALLBACK_IMPLEMENTATION.md` and `IMPLEMENTATION_COMPLETE_API_FALLBACK.md` (CLAIM): on API errors, the app degrades to read-only local SQLite mode rather than failing hard. README also describes this pattern.
**Source:** Docs-review agent.

**INFERENCE** — Degradation is silent — the user may not notice that the app is showing stale data. No banner or status-bar indicator was identified in this pass.

### E. Reference Data Recovery

**EVIDENCE** — Inclusion-list downloads (per `docs/INCLUSION_LISTS_README.md`):
- Retry with exponential backoff (max 3 retries: 1s, 2s, 4s)
- Session-level cooldown for empty folders to prevent log spam
- Three strategies: public HTTP listing → authenticated GCS → manifest-based fallback

**Source:** Docs-review agent / `docs/INCLUSION_LISTS_README.md` (CLAIM).

**INFERENCE** — Three-strategy fallback ladder is a deliberate degradation chain — each tier requires less privilege than the last.

### F. Database Migration & Schema Recovery

**EVIDENCE** — `_run_migrations()` (`helpers/sqlite_db_helper.py:1017` per agent) is auto-applied on startup.
**Source:** Architecture-mapper agent.

**INFERENCE** — There is no rollback for failed migrations. A migration that crashes mid-way could leave the DB in a partially-upgraded state. **GAP:** transactional migration verification deferred.

**EVIDENCE** — `tools/` contains explicit backfill/repair scripts for situations where the DB has gotten into a bad state:
- `tools/backfill_pbm_key_and_check_004146.py`
- `tools/normalize_and_backfill_004146.py`
- `tools/fix_normalized_bin_and_recompute_pbm_keys.py`
- `tools/migrate_normalize_bins.py`
- `tools/migrate_keyring_tokens.py`

**Source:** File listing.

**INFERENCE** — The volume of these scripts suggests recurring, recoverable corruption modes. The fact that they exist as standalone scripts (not integrated repair flows in the GUI) implies they require operator intervention.

### G. Single-Instance Failure Mode

**EVIDENCE** — `SingleInstanceGuard.__enter__` raises `RuntimeError("Another instance is already running")` if the lock cannot be acquired (`helpers/single_instance.py:97`).
**Source:** Direct read.

**INFERENCE** — Caller must catch this and show a friendly dialog rather than crashing. Whether `pharmacybooks.py` does this gracefully was not directly read in this pass.

**EVIDENCE** — Lock release is registered with `atexit` (`helpers/single_instance.py:58`). In a kill-9 scenario the lockfile is left on disk; the next startup will attempt to acquire it, possibly fail because the OS still holds the lock briefly, and may be unable to start until the OS releases the lock.
**Source:** Direct read.

### H. Selenium / Scraper Failure Modes

**INFERENCE** — `helpers/al_medicaid_scraper.py` runs Selenium in a daemon thread. Scraper exceptions bubble up to the calling function (`pharmacybooks.py:5814` per agent). Specific catch+retry behavior not directly verified.

**EVIDENCE** — `verify_rate_config.py` exposes a `turbo_fail` setting (per agent) — a fast-fail mode for impatient users.
**Source:** Architecture-mapper agent.

**INFERENCE** — Three scraper variants exist (Selenium, HTTP-only, hybrid) so that if one fails, an alternative path is configured. The hybrid scraper auto-falls-back from HTTP to Selenium per `al_medicaid_hybrid_batch.py`.

### I. Timing-Logger Performance Warnings

**CLAIM** — Per `docs/TIMING_AND_DIAGNOSTICS.md`:
- Per-row operations flagged via `warn_per_row_operation()`
- High SQL query counts (>100) flagged
- Targets: <100 records/sec = bad, >10,000/sec = excellent

**Source:** Docs-review agent.

**INFERENCE** — These are observability hooks, not corrective actions. The app does not auto-tune; an operator reads logs and decides.

### J. Test Collection Failure (Operational Risk)

**EVIDENCE** — `pytest_output.txt` (per tests-inventory agent) shows the test suite **fails at collection time** due to a relative-import error in `helpers/sqlite_db_helper.py:91`:
```
ImportError: attempted relative import with no known parent package
... from .api_client import get_api_client, APIError, APIClient
```
244 tests collected, but 1 collection error halts the entire run.

**Source:** Tests-inventory agent reading of `pytest_output.txt`.

**INFERENCE** — There is no `conftest.py`, no `pytest.ini`, no `setup.cfg`, no `pyproject.toml`. With no package configuration, the test runner cannot resolve the `helpers` package consistently. This is a recovery GAP at the testing infrastructure level — see `09-TESTS-AND-EVALS.md`.

### K. Logging & Diagnostics

**EVIDENCE** — Logging modules:
- `helpers/log_manager.py` — log file rotation/management
- `helpers/logging_utils.py` — file-based logging setup
- `helpers/aac_debug_logger.py` — opt-in AAC lookup tracing
- `helpers/payer_type_logger.py` (`PayerTypeMatchLogger`) — payer-type decision tracing
- `helpers/timing_logger.py` (`TimingLogger`) — hierarchical performance timing
- `helpers/pbm_debug_logger.py` — DataFrame diagnostic printing
- `tools/import_row_debug_logger.py` — row-level debug log (env-gated `PBM_IMPORT_DEBUG=true`, output `import_debug.csv`)

**Source:** Architecture-mapper agent + `tools/IMPORT_DEBUG.md`.

**INFERENCE** — Logging is rich but uncoordinated — each subsystem owns its own logger. There is no centralized error reporter (e.g., Sentry-style telemetry). Errors stay local to the user's machine unless support is contacted.

### L. Crash-Recovery Patterns Observed

**EVIDENCE** — Atomic-write pattern in `helpers/device_identity.py:35-47` (write tmp + `os.replace`). Likely repeated elsewhere (column mapping helper, applied_recalculate_hash) but not directly verified.

**EVIDENCE** — `import_row_debug_logger` flushes every 100 rows for crash safety (per `tools/IMPORT_DEBUG.md`).

**Source:** Direct read + docs-review agent.

### M. Interrupt Handling

**INFERENCE** — Tk dialogs receive `WM_DELETE_WINDOW` events. The app-level handler is at `pharmacybooks.py:13196` (`root.protocol("WM_DELETE_WINDOW", on_closing)`). Per-dialog handlers were not enumerated.

**INFERENCE** — Long-running operations (import, export, scraper batch) appear to lack a "Cancel" button verification — the architecture-mapper agent noted "User can cancel" for the update-download dialog (`UpdateDownloadProgressDialog`), but other progress dialogs were not verified.

---

## Open Questions

1. Does the sync worker drain its queue at shutdown (`on_closing`), or are queued sync-jobs lost?
2. What happens if an auto-update download is interrupted mid-stream — is the partial file resumable, or does it always restart?
3. Is there any client-side dead-letter / quarantine file for permanently-failed sync records?
4. Does the migration runner (`_run_migrations`) execute each step in its own SQLite transaction, or are migrations free-form?
5. The pytest `ImportError` — does the test suite ever pass? Is anyone running it, or is it considered abandoned?
6. The `silent` API-fallback pattern (degrades to local SQLite) — is there any UI surfacing of "Working offline" state, or does the user just see stale data?
7. What's the worst case if `pending_recalculate.json` is corrupted? The current pattern returns `None` and proceeds — does that mean a server-pushed recalc just gets dropped?
8. Are SMTP/email send failures reported to the user, or silently logged?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (rollback file logic, sync queue drain, transactional migrations, partial-download resumability)
- [x] Test-suite collection failure documented as a recovery GAP at the infrastructure level
