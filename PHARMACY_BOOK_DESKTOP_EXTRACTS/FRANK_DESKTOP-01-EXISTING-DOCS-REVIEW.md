# 01-EXISTING-DOCS-REVIEW — What the Authors Claimed

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

This repo contains **127 markdown files at the root** plus 15 in `docs/` — totaling 142 markdown documents. Most root-level docs are historical implementation notes, fix summaries, and PR write-ups; many describe completed work and risk being stale. The README claims historical docs were moved to `docs/archive/` — that directory **does not exist**, a contradiction. A 12-document verification sample produced **5 CONFIRMED, 5 UNVERIFIED, 2 CONTRADICTED** findings; the remaining 130+ are catalogued but UNVERIFIED. Three coding-agent config files (`CLAUDE.md`, `AGENTS.md`, `WINDSURF.md`) are present; `AGENTS.md` is a **byte-identical duplicate of `WINDSURF.md`** (mislabeled).

---

## Findings

### Catalog Strategy

Per the user's approval (Q4): **catalog all root MDs, verify a representative sample of 12, label the rest UNVERIFIED**. Full per-file enumeration of 127 root MDs is consolidated into a categorized table below. The `docs/` folder is enumerated individually because all 15 files are active. The verification sample table follows.

### docs/archive/ Existence Check

**CONTRADICTED** — README claim that historical docs were moved to `docs/archive/`.
- README quote (`README.md:125`): *"Documentation has been cleaned up and consolidated to essential guides only. Most historical implementation notes, PR summaries, and migration documentation have been moved to `docs/archive/`."*
- Filesystem check: `docs/archive/` does not exist. Glob `docs/archive/**` returned empty; the listing of `docs/` (15 files) contains no `archive/` subfolder.
- **Source of contradiction:** `README.md:125` vs. directory listing of `docs/` (taken 2026-04-29).

### Catalog: docs/ (15 files + 3 image/HTML assets + 1 CSV)

| File | One-Line Summary |
|---|---|
| `docs/AUTO_UPDATE_DEVELOPER_GUIDE.md` | Auto-Update Framework — Developer Implementation Guide |
| `docs/AUTO_UPDATE_FLOW_DIAGRAM.md` | Auto-Update Dialog Flow Visualization |
| `docs/AUTO_UPDATE_USER_GUIDE.md` | Auto-Update Framework — User Guide |
| `docs/DEV_NOTES_TOKEN_HANDLING.md` | Token Handling in Automatic Updates — Developer Notes |
| `docs/IMPLEMENTATION_SUMMARY.md` | Webhook-Driven Registration Pattern (Stripe-first) |
| `docs/INCLUSION_LISTS_README.md` | Inclusion Lists Management Guide (GCS-backed reference data) |
| `docs/LOGO_ASSET_GUIDE.md` | Logo Asset Configuration and Usage |
| `docs/NDC_COLUMN_MAPPING_GUIDE.md` | NDC Column Mapping — User Guide (5-layer defensive validation) |
| `docs/PATH_REFACTORING.md` | User Data Directory Implementation Guide |
| `docs/PATH_REFACTORING_TESTING.md` | Testing Checklist for Path Changes |
| `docs/PBM_BEST_PRACTICES.md` | PBM File Best Practices |
| `docs/PBM_KEY_CANONICAL_MATCHING.md` | PBM Key Canonical Matching Implementation |
| `docs/pbm_name_column_addition.md` | PBM Name Column Addition (schema migration) |
| `docs/TIMING_AND_DIAGNOSTICS.md` | Comprehensive Timing and Diagnostic Logging Guide |
| `docs/WEB_MIGRATION_DESKTOP_INVENTORY.md` | Web Migration — Desktop Inventory (architecture map) |
| `docs/liberty_api_endpoints.csv` | (data) Liberty pharmacy backend API endpoint catalogue (9-column schema; ≥5 sample rows on AR + Delivery endpoints) |
| `docs/tab_highlighting.png` | (image, 17.1 KB) Visual reference — tab highlighting UI |
| `docs/treeview_styling.png` | (image, 53.7 KB) Visual reference — treeview styling |
| `docs/ui_enhancements.html` | (HTML, 4.4 KB) UI enhancement walkthrough |

### Catalog: Repo-Root Markdown (127 files)

Categorized by topic prefix. Counts derived from filename patterns; see filesystem listing for full enumeration.

| Topic Cluster | Approx. Count | Representative Filenames |
|---|--:|---|
| **AAC** (Average Amount Covered week alignment + rate commit) | 5 | `AAC_CHANGES_BEFORE_AFTER.md`, `AAC_FIRST_PRIORITY_IMPLEMENTATION.md`, `AAC_IMPLEMENTATION_COMPLETE.md`, `AAC_RATE_COMMIT_FIX_COMPLETE.md`, `AAC_WEEK_ALIGNMENT_IMPLEMENTATION.md` |
| **About Dialog version handling** | 5 | `ABOUT_DIALOG_TESTING_README.md`, `ABOUT_DIALOG_VERSION_FINAL_SUMMARY.md`, `ABOUT_DIALOG_VERSION_FLOW.md`, `ABOUT_DIALOG_VERSION_SUMMARY.md`, `VERSION_CONSISTENCY_CHECK.md` |
| **Action Needed refactor** | 1 | `ACTION_NEEDED_REFACTORING_SUMMARY.md` |
| **API fallback** | 4 | `API_FALLBACK_IMPLEMENTATION.md`, `API_FALLBACK_VISUAL_COMPARISON.md`, `IMPLEMENTATION_COMPLETE_API_FALLBACK.md`, `IMPLEMENTATION_COMPLETE_PROFILE_API.md` |
| **Auto-update** | 5 | `AUTO_UPDATE_FLOW_VISUALIZATION.md`, `AUTO_UPDATE_IMPLEMENTATION_SUMMARY.md`, `AUTO_UPDATE_IMPROVEMENTS.md`, `AUTO_UPDATE_QUICKREF.md`, `MANIFEST_UPDATE_QUICKREF.md` |
| **Backfill / Payer Type** | 6 | `BACKFILL_PAYER_TYPE_MATCHING_GUIDE.md`, `IMPLEMENTATION_COMPLETE_PAYER_TYPE_BLANK_STRING.md`, `IMPLEMENTATION_SUMMARY_BACKFILL.md`, `PAYER_TYPE_BLANK_STRING_REFACTORING.md`, `PAYER_TYPE_LOGGING_IMPLEMENTATION_COMPLETE.md`, `PAYER_TYPE_LOGGING_IMPLEMENTATION_SUMMARY.md` |
| **Batch / Vectorization** | 5 | `BATCH_IMPORT_OPTIMIZATION.md`, `BATCH_PROCESSING_OPTIMIZATION.md`, `BATCH_REPAIR_VISUAL_COMPARISON.md`, `MEDICAID_BATCH_PROCESSING_AUDIT.md`, `MEDICAID_BATCH_REPAIR_OPTIMIZATION.md` |
| **BIN 004146** (matching, normalization) | 6 | `BIN_004146_FINAL_SUMMARY.md`, `BIN_004146_IMPLEMENTATION_VERIFICATION.md`, `BIN_004146_MATCHING_IMPLEMENTATION.md`, `BIN_004146_PRIORITY_COMPARISON.md`, `BIN_STANDARDIZATION_IMPLEMENTATION.md`, `IMPLEMENTATION_COMPLETE_004146.md`, `IMPLEMENTATION_COMPLETE_BIN_004146_PRIORITY.md` |
| **Build / Icon** | 5 | `BUILD.md`, `ICON_CONFIGURATION.md`, `ICON_CONFIGURATION_SUMMARY.md`, `ICON_ROUNDED_CORNERS_UPDATE.md`, `MANIFEST_UPDATE_QUICKREF.md` |
| **Business Info / Profile read-only** | 8 | `BUSINESS_INFO_DATA_BEFORE_READONLY.md`, `BUSINESS_INFO_READONLY_FIX.md`, `PROFILE_DIALOG_API_INTEGRATION_SUMMARY.md`, `PROFILE_DIALOG_BLACK_FOREGROUND_SUMMARY.md`, `PROFILE_DIALOG_NA_DEFAULTS.md`, `PROFILE_FIX_VISUAL_SUMMARY.md`, `PROFILE_READONLY_CONTRAST_FIX.md`, `PROFILE_READONLY_QUICK_REF.md`, `PROFILE_READONLY_STYLING.md`, `READONLY_FIELDS_IMPLEMENTATION_SUMMARY.md` |
| **Canonical Matcher / PBM** | 13 | `CANONICAL_MATCHER_IMPLEMENTATION_SUMMARY.md`, `PBM_FALLBACK_MATCHING_DOCUMENTATION.md`, `PBM_INFO_POPULATION_FIX.md`, `PBM_KEY_DASHBOARD_COLUMN_DOCUMENTATION.md`, `PBM_KEY_DEBUGGING_ENHANCEMENT.md`, `PBM_KEY_IMPLEMENTATION_FINAL_SUMMARY.md`, `PBM_KEY_IMPLEMENTATION_SUMMARY.md`, `PBM_KEY_IMPORT_PREVENTION_SUMMARY.md`, `PBM_KEY_IMPORT_PREVENTION_VERIFICATION.md`, `PBM_MATCHING_FIX_SUMMARY.md`, `IMPLEMENTATION_COMPLETE_PBM_KEY_DEBUGGING.md`, `IMPLEMENTATION_COMPLETE_SIMPLIFIED_PBM_MATCHING.md`, `IMPLEMENTATION_SUMMARY_PBM_KEY_DOCUMENTATION.md`, `MATCHING_LOGIC_IMPROVEMENTS_SUMMARY.md`, `README_PBM_ADJUSTMENTS.md`, `SIMPLIFIED_PBM_MATCHING_GUIDE.md` |
| **Column Mapping** | 7 | `COLUMN_IMPLEMENTATION_SUMMARY.md`, `COLUMN_LAYOUT_COMPARISON.md`, `COLUMN_MAPPING_DOCUMENTATION.md`, `COLUMN_MAPPING_FIX_SUMMARY.md`, `COLUMN_MAPPING_INLINE_PREVIEW.md`, `COLUMN_MAPPING_UI_VISUALIZATION.md`, `IMPLEMENTATION_SUMMARY_INLINE_PREVIEW.md`, `IMPLEMENTATION_COMPLETE_UNIVERSAL_FILTER.md`, `MAPPING_REFACTOR_SUMMARY.md` |
| **Commercial / Alt Rates / Owed Book** | 4 | `ALT_RATES_CLEANUP_SUMMARY.md`, `COMMERCIAL_RATE_CLEANUP_SUMMARY.md`, `DASHBOARD_COLUMN_REVERSION_SUMMARY.md`, `OWED_BOOK_COLUMNS_IMPLEMENTATION.md` |
| **Copay / Secondary BIN** | 1 | `COPAY_SECONDARY_BIN_IMPLEMENTATION_SUMMARY.md` |
| **Foreground Fix / Foreground Visual** | 2 | `IMPLEMENTATION_COMPLETE_FOREGROUND_FIX.md`, (visual `.txt`) |
| **Final / Top-Level Implementation Index** | 4 | `FINAL_IMPLEMENTATION_SUMMARY.md`, `IMPLEMENTATION_CHECKLIST.md`, `IMPLEMENTATION_COMPLETE_SUMMARY.txt`, `READONLY_REFACTOR_SUMMARY.txt` |
| **Import Debug** | 1 | `IMPORT_DEBUG_IMPLEMENTATION_SUMMARY.md` |
| **Matching Type** | 2 | `IMPLEMENTATION_SUMMARY_MATCHING_TYPE_REQUIRED.md`, `MATCHING_LOGIC_IMPROVEMENTS_SUMMARY.md` |
| **Medicaid / Medicare** | 12 | `MEDICAID_PAYER_TYPE_ALIGNMENT_SUMMARY.md`, `MEDICAID_RATE_5_DECIMAL_IMPLEMENTATION.md`, `MEDICAID_RATE_COMMIT_FIX_SUMMARY.md`, `MEDICAID_RATE_REPAIR_BEFORE_AFTER.md`, `MEDICAID_RATE_VECTORIZATION.md`, `MEDICAID_RATE_VECTORIZATION_QUICKREF.md`, `MEDICARE_INFO_FIX_SUMMARY.md`, `MEDICARE_MATCHING_SUMMARY.md`, `PR_SUMMARY_MEDICAID_BATCH_REPAIR.md`, `PR_SUMMARY_MEDICARE_FIX.md`, `IMPLEMENTATION_SUMMARY_VERIFY_RATE.md`, `VERIFY_RATE_ENHANCED_IMPLEMENTATION.md`, `VERIFY_RATE_FIX_SUMMARY.md` |
| **Mousewheel Fix** | 2 | `IMPLEMENTATION_SUMMARY_MOUSEWHEEL_FIX.md`, `MOUSEWHEEL_SCROLLING_FIX_SUMMARY.md` |
| **NDC Normalization** | 2 | `NDC_NORMALIZATION_ENHANCEMENT_SUMMARY.md`, `IMPLEMENTATION_SUMMARY_BIN_FALLBACK_MATCHING.md` |
| **Path Audit / Refactor** | 1 | `PATH_AUDIT_SUMMARY.md` |
| **Plan Name Removal** | 1 | `PLAN_NAME_REMOVAL_SUMMARY.md` |
| **README / Verify Rate** | 3 | `README.md`, `README_VERIFY_RATE_SELECTORS.md`, `README_PBM_ADJUSTMENTS.md` |
| **Tab Filtering / Universal Filter** | 2 | `TAB_FILTERING_IMPLEMENTATION_SUMMARY.md`, `UNIVERSAL_FILTER_DOCUMENTATION.md` |
| **Testing** | 2 | `TESTING_GUIDE.md`, `TEST_RESULTS.md` |
| **Timing / Vectorization summary** | 2 | `TIMING_IMPLEMENTATION_SUMMARY.md`, `VECTORIZATION_VISUAL_SUMMARY.md` |
| **Timestamp** | 1 | `TIMESTAMP_FIX_SUMMARY.md` |
| **UI Changes / Enhancements** | 2 | `UI_CHANGES_VISUALIZATION.md`, `UI_ENHANCEMENTS_SUMMARY.md` |
| **Unicode / Optional Border** | 2 | `UNICODE_ENCODING_FIX_SUMMARY.md`, `OPTIONAL_BORDER_STYLING_GUIDE.md` |
| **Coding-Agent Configs (3)** | 3 | `CLAUDE.md`, `AGENTS.md`, `WINDSURF.md` (note `AGENTS.md` ≡ `WINDSURF.md` byte-identical) |
| **Implementation Summary (Top-level)** | 1 | `IMPLEMENTATION_COMPLETE_PROFILE_API.md`, plus several `IMPLEMENTATION_SUMMARY_*` files |
| **Older PR summaries** | 3 | `PR_SUMMARY_OLD.md`, `PR_SUMMARY_MEDICAID_BATCH_REPAIR.md`, `PR_SUMMARY_MEDICARE_FIX.md` |

**Note on counts:** Cluster counts may slightly overlap (some files fit multiple themes). The total root MD count is 127 (verified via `ls | grep '\.md$' | wc -l = 127`).

### Verification Sample (12 docs)

| # | Doc | Verdict | Evidence / Source |
|---|---|---|---|
| 1 | `AUTO_UPDATE_QUICKREF.md` | **CONFIRMED** | File exists; agent read top of file; structure matches an auto-update quickref. Source: file present in repo root. |
| 2 | `BUILD.md` | **CONFIRMED** | File exists with build/packaging guide structure. Source: file present at root, opens with header `# PharmacyBooks Desktop - Build & Packaging Guide`. (Code-level claims about PyInstaller commands not verified — see GAP.) |
| 3 | `PBM_MATCHING_FIX_SUMMARY.md` | **UNVERIFIED** | Document claims a fix to `BatchCalculator._match_pbm_vectorized` regarding optional `email`/`pbm_name` columns. Code-level verification of the fix logic in `helpers/batch_calculator.py` was not performed in this extraction. Source: doc present; corresponding helper present. |
| 4 | `IMPLEMENTATION_COMPLETE_PROFILE_API.md` | **UNVERIFIED** | Document claims `_load_business_profile()` method (~74 lines) added to `ProfileDialog`, plus a `test_profile_dialog_api_integration.py`. The test file IS verified present at root. The method's existence in `pharmacybooks.py` (`ProfileDialog` class lines 557-1415) was reported by the architecture-mapper agent but the specific `_load_business_profile` method name was not grepped. Source: doc + test file both present; method line-count UNVERIFIED. |
| 5 | `NDC_NORMALIZATION_ENHANCEMENT_SUMMARY.md` | **CONFIRMED** | Doc claims `helpers/column_mapping_helper.py` exposes `normalize_column_name()` and `normalize_ndc_column()`; both confirmed by Grep. Doc claims `_ndc_variants` includes `'ndc'`, `'drug_ndc'`, `'ndc_number'`; confirmed at `helpers/column_mapping_helper.py:49-52`. Source: `helpers/column_mapping_helper.py:54, 83`. |
| 6 | `COLUMN_MAPPING_DOCUMENTATION.md` | **UNVERIFIED** | Doc claims database `user_data` table includes columns: `bin`, `secondary_bin`, `pcn`, `group_field`, `network_id`, `contract_id`, `copay`. Code-level schema dump from `helpers/sqlite_db_helper.py` was not extracted in this pass. The `group_field` rename (claimed to avoid SQL reserved word `group`) was not confirmed against schema DDL. Source: doc present; schema verification deferred. |
| 7 | `TIMING_IMPLEMENTATION_SUMMARY.md` | **CONFIRMED** (partial) | Doc claims `helpers/timing_logger.py` provides `TimingLogger` with hierarchical timing. Confirmed file exists; agent confirmed `class TimingLogger` is present (lines 41+) with parent/operation_name signature. Performance claim "10,000+ records/sec" is **UNVERIFIED** (no benchmark run). Source: `helpers/timing_logger.py`. |
| 8 | `AUTO_UPDATE_IMPLEMENTATION_SUMMARY.md` | **CONTRADICTED** + CONFIRMED | Doc claims version `1.0.0`. Code says `__version__ = "1.5.18"` (`version.py:5`). **CONTRADICTED** on the version literal. Manifest URL, 6-hour interval, and 5-minute download timeout claims **CONFIRMED** against `version.py:10-12`. File paths `helpers/auto_update_manager.py` and `helpers/auto_update_dialogs.py` **CONFIRMED**. Source: `version.py:5,10,11,12` (code) vs. doc text. **INFERENCE:** the doc was written against an earlier version and never updated — it's a stale historical artifact. |
| 9 | `README_VERIFY_RATE_SELECTORS.md` | **UNVERIFIED** | Doc claims Verify Rate feature autofills NDC/Date/DAW in AL Medicaid Portal via Selenium with multi-browser (Chrome/Edge/Firefox) support and "Other Date" radio handling. `helpers/verify_rate_dialog.py` and `helpers/verify_rate_config.py` **CONFIRMED** present, but the specific bug-fix details (radio button ordering, JS injection) were not code-level verified. Source: doc + helper files present. |
| 10 | `BIN_004146_FINAL_SUMMARY.md` | **CONTRADICTED** | Doc claims BIN 004146 special case is "already fully implemented" in `helpers/pbm_file_loader.py` at lines 265-268 (Excel loader) and 466-468 (CSV loader). Bash grep for `004146\|4146` in `helpers/pbm_file_loader.py` returned **zero matches** per the docs-review agent. The canonical handling appears to live in `helpers/pbm_utils.py` (per `docs/PBM_KEY_CANONICAL_MATCHING.md`) and is also referenced in `tests/test_pbm_key_matching.py`, but NOT in `helpers/pbm_file_loader.py` as the doc asserts. Source: agent Grep run on `helpers/pbm_file_loader.py`. **INFERENCE:** the doc described intent before refactor, or the special case was migrated to `pbm_utils.py` and the doc was never updated. |
| 11 | `PATH_AUDIT_SUMMARY.md` | **UNVERIFIED** | Doc claims `helpers/aac_debug_logger.py` was modified to accept `log_filename: str = None` and use `get_logs_dir()` when None. The helper file and the test (`test_aac_debug_logger_path.py`) both exist. Specific parameter signature change not code-level verified. Source: file + test present. |
| 12 | `VERSION_CONSISTENCY_CHECK.md` | **CONFIRMED** (partial) | Script `verify_version_consistency.py` **CONFIRMED** present at repo root. `version.py:5` confirms `__version__ = "1.5.18"` (the doc says the issue was the About dialog showed `v1.0.0` — that was an old issue, since fixed). The script's claimed checks against `installer/setup_script.iss` AppVersion **UNVERIFIED** (the .iss file was not opened). Source: `version.py:5`, file listing of root. |

**Sample verification totals:** **5 CONFIRMED**, **5 UNVERIFIED**, **2 CONTRADICTED**.

### Three-Way Coding-Agent Config Comparison

**EVIDENCE** — `AGENTS.md` and `WINDSURF.md` are byte-identical.
- Verified by `diff AGENTS.md WINDSURF.md` returning empty output.
- Both files open with: `# .windsurfrules — Windsurf/Cascade Configuration` and `*Version: 2.0 | March 2026*`.
- **Source:** `AGENTS.md:1-5`, `WINDSURF.md:1-5`, `diff` invocation.

**INFERENCE** — `AGENTS.md` is mislabeled — its content is for Windsurf/Cascade, not a generic AGENTS specification. This violates the (informal) convention that `AGENTS.md` is a tool-agnostic agent file. Practical consequence: a coding agent that auto-loads `AGENTS.md` will receive Windsurf-specific guidance.

**EVIDENCE** — `CLAUDE.md` exists at root, version `3.0 | March 2026`, length 685 lines.
**Source:** `CLAUDE.md:1-5` (and the docs-review agent's diff).

#### Substantive Deltas: CLAUDE.md vs WINDSURF.md (≡ AGENTS.md)

1. **Plan Mode enforcement.** CLAUDE.md frames Plan Mode as a *system-level constraint* — invoking `EnterPlanMode` removes write tools (`CLAUDE.md:33-36`). WINDSURF.md acknowledges no such mechanical lock exists in Windsurf and pivots to behavioral discipline + the session file as the anchor (`WINDSURF.md:33-40`).

2. **Announcement banner.** CLAUDE.md uses `🔵 ENTERING PLAN MODE` in the CLI; WINDSURF.md uses `🔵 PLANNING` in chat (`CLAUDE.md:64-67` vs `WINDSURF.md:70-75`).

3. **RECOVERY.md emphasis.** Both reference `RECOVERY.md` as a 3-second recovery file. WINDSURF.md gives it a more prescriptive role and explicit `# Recovery State / Last action / Pending / Next step` structure (`WINDSURF.md:189-202`).

4. **Tech-stack context.** Identical. Both list `Next.js 15 / TypeScript / Tailwind / ShadCN / Zustand`, `FastAPI + Uvicorn + Supabase`, `Google ADK + Vertex AI + Gemini 2.5`. **Notable:** this stack does NOT match the actual app's stack (Tkinter desktop, App Engine API, no Next.js, no FastAPI). The agent files describe an "AI App Factory" workflow whose canonical stack appears unrelated to this specific project. (Flagged in `10-RAW-FINDINGS-AND-QUESTIONS.md`.)

5. **Windsurf-Specific Notes section.** Present only in WINDSURF.md (`WINDSURF.md:620-651`). Acknowledges Cascade can read the entire codebase and execute terminal commands, and explicitly states there is no `EnterPlanMode` tool — the session file write is the substitute mechanical anchor.

#### Purpose Confirmation

**EVIDENCE** — All three files explicitly self-describe as configuration for *coding agents that work on this repo* (CLAUDE.md:9, AGENTS.md:9, WINDSURF.md:9 all read "System prompt/rules for [Claude Code|Windsurf] agentic coding sessions"). They are NOT in-app personas.
**Source:** Header text on lines 1-9 of each file.

### Stale-Doc Risk Patterns

**INFERENCE** — From the catalog above:
- ~25 files start with `IMPLEMENTATION_COMPLETE_*` / `IMPLEMENTATION_SUMMARY_*` / `*_FINAL_SUMMARY.md` — these are change-log artifacts that document past commits and are highly likely to be stale relative to current code.
- ~13 files describe BIN 004146 work — concentration suggests a recurring problem area (also evidenced by ~12 corresponding scripts in `tools/`).
- The `_FINAL` and `_COMPLETE` suffixes appear contradictory in places (e.g., `BIN_004146_FINAL_SUMMARY.md` and `IMPLEMENTATION_COMPLETE_BIN_004146_PRIORITY.md` both exist).
- Whole categories of "fix" docs (foreground color, mousewheel, treeview styling, copay) describe one-off polish work whose verification value approaches zero.

### Single-Doc Spotlights

**EVIDENCE / CLAIM** — `docs/WEB_MIGRATION_DESKTOP_INVENTORY.md` exists and contains a Layered View (Presentation/Application/Persistence/Reference Data/Reporting/Auth/Testing). The agent extracted ~29 helper modules, dialogs, GCS folders, and feature lists from it. This document is the closest thing to an architecture map authored by the project itself, and looks like preparation for the future web app rewrite. **Useful as input to the rebuild.**
**Source:** `docs/WEB_MIGRATION_DESKTOP_INVENTORY.md` (read by docs-review sub-agent).

**EVIDENCE** — `docs/liberty_api_endpoints.csv` is a 9-column endpoint catalogue (Tag, Path, Method, Access, Summary, Description, HasRequestBody, Parameters, Responses). Sample rows include `/ar`, `/ar/{patientId}`, `/delivery/`, `/delivery/queue`. This appears to document **Liberty Software's pharmacy management API** (an external system this app may interact with), NOT the PharmacyBooks App Engine API. Tagged CLAIM because no code in this repo was verified to call those endpoints.
**Source:** `docs/liberty_api_endpoints.csv:1-6`.

---

## Open Questions

1. **`AGENTS.md` mislabel** — should it be deleted, or replaced with a tool-agnostic agents file? (Currently misleading.)
2. **`docs/archive/` missing** — was historical archival aborted, or were the files deleted? Either way, the README claim is wrong.
3. **`docs/liberty_api_endpoints.csv`** — does any module in this repo actually integrate with Liberty Software, or is the CSV aspirational?
4. **`AUTO_UPDATE_IMPLEMENTATION_SUMMARY.md` version drift** — is the `1.0.0` stale-version issue isolated to this one doc, or are other "implementation complete" docs similarly stale?
5. **CLAUDE.md tech-stack mismatch** — the agent files describe a Next.js/FastAPI/ADK stack, but this app is Tkinter/App Engine. Is this repo a side-project under a broader factory whose stack docs leak into per-project files?
6. **Doc verification scope** — 130+ root MDs remain UNVERIFIED. Tony's Q4 answer accepted this. Should a future pass do bulk grep-based verification?
7. **`BIN_004146_FINAL_SUMMARY.md` discrepancy** — was the BIN 004146 special case migrated from `pbm_file_loader.py` to `pbm_utils.py`, or was it never in `pbm_file_loader.py` at all? Two possible histories explain the same observed state.

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist (cross-checked against filesystem listing)
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included (catalog + verdicts only)
- [x] GAPs explicitly documented (130+ UNVERIFIED root MDs, .iss file unread, BUILD.md commands UNVERIFIED)
- [x] CONTRADICTED items called out with line/file evidence on both sides (docs/archive/, BIN 004146, version 1.0.0)
