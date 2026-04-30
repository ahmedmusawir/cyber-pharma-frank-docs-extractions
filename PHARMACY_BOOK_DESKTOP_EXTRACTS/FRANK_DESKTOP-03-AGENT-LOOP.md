# 03-AGENT-LOOP — Request/Response Lifecycle (Reframed)

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — REFRAMED (this is not an LLM agent; documenting the GUI/event lifecycle instead)

---

## Summary

This is **not an agent**. It is a Tkinter desktop application; the relevant "loop" is the **Tk event loop** plus a small set of background **daemon-thread loops** (sync worker, auto-update polling, subscription polling, scraper workflows). User actions arrive as widget events, are handled synchronously in the main thread, and may dispatch I/O work to daemon threads. Cross-thread results return to the main thread via `root.after(0, callback)`. The GUI startup is a sequential phase machine (activation → login → reference data init → main window) with no formal state machine — control flow is encoded directly in `pharmacybooks.py:9906-13200`.

---

## Findings

### The Three Loops

**EVIDENCE** — There are three concurrent loops at runtime:

| Loop | Driver | Cadence | Source |
|---|---|---|---|
| 1. Tk main event loop | `root.mainloop()` | Per-event (user input, redraw) | `pharmacybooks.py:13200` |
| 2. Background sync worker | `BackgroundSyncWorker._worker_thread` | `periodic_interval=300s` (5 min default) | `helpers/background_sync_worker.py:73, 96-99` |
| 3. After-callback timers | `root.after(ms, fn)` | Various (subscription 30 min; opportunities badge 1.5 s) | `pharmacybooks.py:10223, 13182` |

**Source:** Direct read of `helpers/background_sync_worker.py:67-99` + architecture-mapper agent line citations.

**INFERENCE** — Loop #2 and Loop #3 cooperate with Loop #1: workers do not touch widgets directly, they enqueue updates back into the Tk loop via `root.after(0, ...)`.

### Loop 1: Tk Main Event Loop — Startup Phase Machine

**EVIDENCE** — Sequential phase order on first activated launch (verified at `pharmacybooks.py:9906-10500`+):

1. **Pre-window phase** — `StartupPhaseTimer` instrumented (`pharmacybooks.py:121`), `BASE_DIR = get_app_data_dir()` (line 229), `db = DatabaseHelper(BASE_DIR)` (line 9951).
2. **Tk root** created with `ttkbootstrap.Window(themename="flatly")` if available, else fallback `tk.Tk` (line 9937).
3. **Activation gate** — `APIClient.is_activated()` (line 9966). If not activated:
   - `ActivationKeyDialog.show()` (line ~9973)
   - `LocalAccountSetupDialog.show()` (lines 9995-10010)
   - `ColumnMappingDialog.show()` (lines 10025-10047) — required mapping before continuing
4. **Login gate** — `LocalLoginDialog.show()` (line 10061). The login dialog uses `local_auth_helper` (SQLite-backed user table).
5. **Reference data init** — `initialize_reference_data(db, progress_dialog)` (line 10121, function defined at 9598-9837). Loads AAC/WAC/PBM lists with progress UI.
6. **Profile completeness check** — if profile fields are blank, `ProfileDialog` is shown (line 10207).
7. **Background subsystems start:**
   - `initialize_sync_manager()` + `start_background_sync()` (line 10214-10215) — kicks off Loop #2
   - PBM candidate export thread spawn (line 10220, daemon)
   - Subscription periodic check scheduled via `root.after(SUBSCRIPTION_CHECK_INTERVAL, check_subscription_status_periodic)` (line 10223). `SUBSCRIPTION_CHECK_INTERVAL = 30 minutes` per agent.
   - Business-status one-shot startup check (line 10277-10398)
   - Auto-update startup thread (line 10470-10504, daemon)
8. **Sidebar + content frame construction** (lines 10512-13182): logo (220×220 px), sidebar buttons, three book buttons.
9. **Closing handler registered:** `root.protocol("WM_DELETE_WINDOW", on_closing)` (line 13196). `on_closing()` (lines 13185-13194) backs up DB, stops sync worker, destroys root.
10. **`root.mainloop()`** — line 13200. Returns when root is destroyed.

**Source:** Architecture-mapper agent's pharmacybooks.py walk (lines cited above are from the agent's structured report).

**INFERENCE** — There is no explicit state-machine class. Phase order is encoded as a flat sequence of statements + early-exit conditionals. A failed activation kills the process; a failed login retries within the dialog.

### Loop 1: Per-Event Handlers (Within mainloop)

**EVIDENCE** — Common event-handler patterns:

| Trigger | Handler | Behavior |
|---|---|---|
| Sidebar button click → "Owed Book" / "KPI book" / "Opportunities Book" | `show_dashboard(book_name)` (`pharmacybooks.py:9339-9370`) | Clears content frame, instantiates dashboard view |
| "Settings ▼" click | `show_settings_menu()` (`pharmacybooks.py:12464-12491`) | Posts a popup `tk.Menu` |
| "Imports ▼" click | `show_imports_menu()` (`pharmacybooks.py:12960-12977`) | Posts a popup `tk.Menu` |
| "Manual Check for Updates" | `manual_check_for_updates()` (`pharmacybooks.py:10584-10644`) | Spawns daemon thread for HTTPS manifest fetch; updates UI via `root.after` |
| Profile edit | `ProfileDialog.__init__` → `_load_business_profile()` (per `IMPLEMENTATION_COMPLETE_PROFILE_API.md` CLAIM, line UNVERIFIED) | API fetch on dialog open |
| Treeview row double-click (in Owed Book) | `EditFieldDialog` opened (`pharmacybooks.py:1465-1593`) | Edit-tracked field update |

**Source:** Architecture-mapper agent.

### Loop 2: Background Sync Worker

**EVIDENCE** — `BackgroundSyncWorker` is a thread that processes a queue and runs periodic syncs.
- Constructor: `helpers/background_sync_worker.py:67-93`
- Defaults: `batch_size=500`, `max_retries=3`, `retry_delay=5 seconds`, `periodic_interval=300 seconds`
- Internal state: `_worker_thread`, `_stop_event = threading.Event()`, `_sync_queue = queue.Queue()`, `_current_status = SyncStatus.IDLE`, `_is_running` (lines 96-100)
- Status enum: `IDLE`, `SYNCING`, `COMPLETE`, `ERROR`, `PAUSED` (lines 35-41)
- `SyncMessage` carries `status, message, details, timestamp` to a `status_callback` (lines 44-53)

**Source:** Direct read `helpers/background_sync_worker.py:1-100`.

**EVIDENCE** — Per `examples/sync_worker_demo.py` and the docs:
- Auto-syncs after imports
- Periodic background sync while app is running
- Retries on failure with exponential backoff (per docstring at `helpers/background_sync_worker.py:11`)

**Source:** `helpers/background_sync_worker.py:1-14` (docstring).

### Loop 3: After-Callback Timers

**EVIDENCE** — Periodic UI work scheduled via `root.after()`:
- **Subscription poll:** `root.after(SUBSCRIPTION_CHECK_INTERVAL, check_subscription_status_periodic)` (`pharmacybooks.py:10223`); 30-minute interval
- **Opportunities badge refresh:** `root.after(1500, update_opportunities_badge)` scheduled at startup (`pharmacybooks.py:13182`); 1.5-second cadence per agent

**Source:** Architecture-mapper agent.

**INFERENCE** — `root.after(ms, fn)` schedules a one-shot; periodic behavior is achieved by handlers re-scheduling themselves at the end of each invocation.

### Cross-Thread Dispatch Pattern

**EVIDENCE** — Workers update the UI by deferring back to the main thread:
- Update download progress callback (`pharmacybooks.py:10736`): `root.after(0, lambda: progress_dialog.update_progress(...))`

**Source:** Architecture-mapper agent.

**INFERENCE** — There is no central message bus. Each thread owns its own `status_callback` (e.g., `BackgroundSyncWorker.status_callback`) and threads/dialogs each set up their own `root.after` plumbing.

### Selenium Workflow Loop (Subordinate to Loop 1)

**EVIDENCE** — The AL Medicaid Portal scraper runs as a daemon thread (`pharmacybooks.py:5814` per agent). Per browser session:
1. Driver init (Chrome/Edge/Firefox via Selenium Manager)
2. Navigate to AL Medicaid portal
3. For each NDC in queue:
   - Fill NDC field
   - Select "Other Date" radio (must be selected BEFORE date field per `README_VERIFY_RATE_SELECTORS.md`)
   - Fill Date of Service
   - Fill DAW
   - Click search; parse rate
4. Driver shutdown

**Source:** `README_VERIFY_RATE_SELECTORS.md` (CLAIM) + agent.

**INFERENCE** — Each rate-verification run owns a distinct WebDriver instance — no driver reuse across batches in default config (the scraper docstring claims session reuse via `al_medicaid_batch.py` but this was not code-level verified).

### Auto-Update Lifecycle Loop

**EVIDENCE** — `auto_update_manager` checks for updates at startup (in a daemon thread) and has a configurable periodic interval `UPDATE_CHECK_INTERVAL = 21,600 seconds` (6 hours) per `version.py:11`. Lifecycle:
1. **Check** — HTTPS GET `latest-version.json` from GCS public URL (`version.py:10`)
2. **Compare** — `_parse_version_tuple()` (`helpers/auto_update_manager.py:52-56`) compares against `__version__`
3. **Download** — daemon thread, progress callback (`pharmacybooks.py:10712-10831`)
4. **Verify** — SHA-256 hash check (per docstring at `helpers/auto_update_manager.py:13`)
5. **Notify** — `UpdateNotificationDialog` (Download / Remind Me Later / Skip This Version)
6. **Install** — launch installer with `/SILENT` flag (per `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md`); app exits; user relaunches

**Source:** Direct read of `helpers/auto_update_manager.py:1-60` + `version.py:10-12` + agent + docs.

### Data Import Lifecycle (One-Shot)

**EVIDENCE** — On user-triggered import or startup scan:
1. `scan_import_folder_startup(db)` (`pharmacybooks.py:8574-8603`) — startup-time scan
2. `import_startup_files(app)` (line 8605-8654) — apply saved column mappings, run `DataImporter`
3. `DataImporter.run()` (per `docs/TIMING_AND_DIAGNOSTICS.md` CLAIM): 7 instrumented stages — Read, Column Mapping, Cleaning, Prepare, Batch Calc, Database Insert
4. `BatchCalculator.calculate_batch()` runs vectorized AAC/PBM/Medicare/Cash/Medicaid/Financial sub-stages (per `docs/TIMING_AND_DIAGNOSTICS.md`)
5. `db.insert_user_data(records)` finalizes
6. `BackgroundSyncWorker` is notified to push unsynced records (Loop 2)

**Source:** Architecture-mapper + docs-review agents.

**INFERENCE** — Imports are not chunked-streaming; they buffer the entire spreadsheet into a pandas DataFrame, run vectorized transforms, then bulk-insert.

### Shutdown

**EVIDENCE** — `on_closing()` at `pharmacybooks.py:13185-13194`:
1. Backup `user_data.db` (per agent's reading)
2. Stop sync worker (likely sets `_stop_event` per `helpers/background_sync_worker.py:97`)
3. `root.destroy()`

**Source:** Architecture-mapper agent.

**EVIDENCE** — `SingleInstanceGuard.release()` is registered with `atexit` (`helpers/single_instance.py:58`) and unlinks the lockfile.

**Source:** Direct read `helpers/single_instance.py:58, 64-93`.

---

## Open Questions

1. Does the sync worker drain its queue on shutdown, or is in-flight work lost?
2. Is the Selenium driver reused across rate-verification batches, or instantiated per call?
3. Are auto-update downloads restartable across app launches if the app exits mid-download?
4. The 30-minute subscription poll is hardcoded — what happens if the user's network is offline at the polling tick? (Likely silent retry, but unverified.)
5. The opportunities-badge `root.after(1500, ...)` cadence (every 1.5 s) seems aggressive for a UI badge — is this a bug or intentional tight refresh?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (Selenium driver reuse, mid-download restartability)
- [x] Reframed correctly: documented Tk event loop + worker loops, not an LLM agent loop
