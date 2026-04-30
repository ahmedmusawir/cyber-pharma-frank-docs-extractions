# 00-REPO-PROFILE — What's in the Box

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

Pharmacy Books Desktop is a Python 3.8+ Tkinter (ttkbootstrap) desktop application for U.S. pharmacy reimbursement tracking, with a Google App Engine remote API backend (NOT in this repo) and a Google Cloud Storage update channel. Version 1.5.18. The repository is a vibe-coded sprawl: a 577 KB single-file `pharmacybooks.py` orchestrator (13,200 lines), ~63 modules under `helpers/`, ~32 standalone scripts under `tools/`, and ~80 loose `*.py` and 127 loose `*.md` artifacts at the repo root. There is no `pyproject.toml`, `setup.py`, `Makefile`, or `package.json`. Inno Setup builds the Windows installer.

---

## Findings

### Identity & Versioning

**EVIDENCE** — Application name and version. `__version__ = "1.5.18"`, `__version_info__ = (1, 5, 18)`.
**Source:** `version.py:5-6`

**EVIDENCE** — Auto-update manifest URL and intervals.
- `UPDATE_MANIFEST_URL = "https://storage.googleapis.com/downloads.pharmacybooks.com/latest-version.json"`
- `UPDATE_CHECK_INTERVAL = 3600 * 6` (6 hours)
- `UPDATE_DOWNLOAD_TIMEOUT = 300` (5 minutes)
**Source:** `version.py:10-12`

**EVIDENCE** — Remote API base URL.
- Default: `https://pharmacy-books-desktop.ue.r.appspot.com/api`
- Override via `[api]` section in `config.ini`, or by `PHARMACYBOOKS_API_BASE_URL` env var (per dev-environment loader in `pharmacybooks.py:248-294`).
**Source:** `config.ini.template:11-13`, `helpers/api_client.py:86-88`

### Language, Runtime & Platform Support

**EVIDENCE** — Python 3.8+ required (per README).
**Source:** `README.md:119`

**EVIDENCE** — Cross-platform user-data directory via `helpers/path_helper.get_app_data_dir()`. README claims:
- Windows: `%LOCALAPPDATA%\PharmacyBooks`
- macOS: `~/Library/Application Support/PharmacyBooks`
- Linux: `~/.config/PharmacyBooks`
**Source:** `README.md:101-105`

**EVIDENCE** — Windows-specific dependency `pywin32`.
**Source:** `requirements.txt:16`

**INFERENCE** — Despite README's cross-platform claims, `pywin32` is a hard requirement on macOS/Linux installs unless `requirements_linux.txt` excludes it (the file exists per README:121 but its contents were not read).

### Entry Points

**EVIDENCE** — Single entry point: `python pharmacybooks.py`.
**Source:** `README.md:43`, `pharmacybooks.py:9906` (`if __name__ == '__main__':`)

**EVIDENCE** — No console-script entry points (no `setup.py`/`pyproject.toml`).
**Source:** GAP — neither file present in repo root.

### Top-Level Dependencies (requirements.txt)

**EVIDENCE** — Full dependency list with version pins:
```
sqlalchemy>=2.0.27,<3.0.0
pandas>=2.2.2,<3.0.0
openpyxl>=3.1.2,<4.0.0
reportlab>=4.0.9,<5.0.0
pypdf==3.17.4
ttkbootstrap>=1.10.1,<2.0.0
tkcalendar>=1.6.1,<2.0.0
Pillow>=10.3.0,<11.0.0
keyring>=25.2.0,<26.0.0
cryptography>=43.0.0,<45.0.0
google-cloud-storage>=2.15.0,<3.0.0
requests>=2.31.0,<3.0.0
beautifulsoup4>=4.12.0,<5.0.0
lxml>=5.0.0,<6.0.0
selenium>=4.0.0,<5.0.0
pywin32>=300
Jinja2>=3.1.2,<4.0.0
```
**Source:** `requirements.txt:1-17`

**INFERENCE** — `sqlalchemy` is listed but `helpers/sqlite_db_helper.py` uses raw `sqlite3` (line 9). SQLAlchemy may be unused or used by another module not yet inspected.

**INFERENCE** — Both `requests` and `selenium` present; remote API calls go through `requests`, AL Medicaid portal scraping uses Selenium.

### Configuration Files

**EVIDENCE** — `config.ini.template` ships with sections: `[mysql]` (legacy/unused per its own comment), `[api]` (base_url, timeout=30), `[selenium]` (use_js_for_fields), `[portal]` (CLI integration for AL Medicaid scraper).
**Source:** `config.ini.template:1-41`

**EVIDENCE** — Dev environment overrides loaded from disk:
- `PHARMACYBOOKS_UPLOAD_BUCKET`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `PHARMACYBOOKS_API_BASE_URL`
**Source:** `pharmacybooks.py:248-294` (per architecture-mapper agent report)

**GAP** — `config.ini` (the actual filled-in file) is not committed; only `.template` exists. Activation/operation requires it per `README.md:23-26`.

### Build & Packaging

**EVIDENCE** — Windows installer scripts present.
**Source:** `installer/setup_script.iss`, `installer/setup_script(one dir).iss`

**CLAIM** — `BUILD.md` references PyInstaller-based packaging.
**Source:** `BUILD.md` (read inventoried by sub-agent — full code-level verification not performed).

**EVIDENCE** — `verify_version_consistency.py` at repo root claims to validate that `version.py`, `installer/setup_script.iss` AppVersion, and the About dialog string all match.
**Source:** `verify_version_consistency.py` (per file listing). Sub-agent did not verify the `.iss` file's actual `AppVersion` matches `version.py` — flagged as GAP (see 10-RAW-FINDINGS).

### File Tree (Top 2 Levels — Counts & Notable Items)

**EVIDENCE** — Repo-root file count and breakdown (verified via `ls`):
- ~321 entries at root
- 127 `*.md` files at root
- 106 `test_*.py` files at root
- 1 `pharmacybooks.py` (577,060 bytes; 13,200 lines)
- ~80 other loose Python scripts at root (`demo_*`, `check_*`, `debug_*`, `verify_*`, `visualize_*`, `benchmark_*`, `fix_*`, `inspect_*`, `find_*`, `grok_test_*`)
- 6 `.txt` files (most are visual comparison artifacts)
- 4 CSVs / JSONs at root (`preview_keys.csv`, `preview_keys.fixed.csv`, `tmp.csv`, `verify_rate_selectors.json`)
- 1 `pytest_output.txt`
**Source:** Bash `ls C:/_MY_PROJECTS/CYBER_PHARMA/pharmacybooks-desktop-main` (trunc/full listings during scan)

**EVIDENCE** — Subdirectories at root:
| Folder | Purpose (one-line) |
|---|---|
| `helpers/` | ~63 module files — primary code organization |
| `tools/` | ~32 diagnostic + migration scripts (+ `IMPORT_DEBUG.md`) |
| `tests/` | 9 structured pytest/unittest files |
| `docs/` | 15 active doc files + 3 image/HTML assets + 1 CSV (`liberty_api_endpoints.csv`) |
| `examples/` | 17 runnable scripts + 4 sample CSVs + 1 JSON + 2 README MDs |
| `icons/` | Logo PNGs/ICOs + `archive/` subfolder |
| `installer/` | 2 Inno Setup `.iss` scripts |
| `scripts/` | 2 maintenance scripts (`backfill_payer_type_matching.py`, `update_icon_rounded_corners.py`) |
| `download-site/` | Static marketing site (`index.html`, `main.js`, `styles.css`) |
| `site - registration/` | 2 static HTML pages (NOTE literal space in folder name) |
| `dev_app_data/` | Dev sandbox runtime data (`column_mappings.json`, `ReimbursementReports/`) |
| `ReimbursementReports/` | Runtime artifact dir (`Signatures/`) |
| `brain-drain/` | The skill running this extraction (do not touch) |
| `_EXTRACTIONS/` | This extraction's output |

**EVIDENCE** — `helpers/__init__.py` is a placeholder with no exports.
**Source:** `helpers/__init__.py:1-2` ("This file marks the helpers directory as a Python package. It can be empty.")

### Helpers Module Roster (63 files)

Grouped by domain. One-liner per module. (Detailed in `02-ARCHITECTURE-MAP.md`.)

**Auth / Login (8):**
`local_login_dialog.py`, `local_auth_helper.py`, `api_login_dialog.py`, `activation_key_dialog.py`, `local_account_setup_dialog.py`, `support_access.py`, `temp_password_crypto.py`, `temp_password_sync.py`

**API / Identity / Tokens (4):**
`api_client.py` (central HTTP gateway), `api_db_helper.py` (legacy/unused), `token_helper.py`, `device_identity.py`

**SQLite / Cache (4):**
`sqlite_db_helper.py` (4500+ lines per agent report), `sqlite_type_converter.py`, `cache_manager.py`, `log_manager.py`

**Data Import / Mapping (5):**
`data_importer.py`, `column_mapping_dialog.py`, `column_mapping_helper.py`, `column_name_sanitizer.py`, `csv_utils.py`

**Reference Data (AAC / WAC / FUL / PBM / Medicare / Opportunities) (10):**
`reference_data_manager.py`, `aac_reference_manager.py`, `aac_enrichment.py`, `aac_debug_logger.py`, `ful_reference_manager.py`, `pbm_file_loader.py`, `pbm_utils.py`, `pbm_key_backfill_migration.py`, `medicare_list_loader.py`, `opportunities_sheet_loader.py`, `inclusion_list_downloader.py`

**Medicaid Workflows (Selenium scrapers + verification) (6):**
`medicaid_rate_workflow.py`, `al_medicaid_scraper.py`, `al_medicaid_batch.py`, `al_medicaid_http_prototype.py`, `al_medicaid_hybrid_batch.py`, `medicaid_portal_scraper.py`, `verify_rate_dialog.py`, `verify_rate_config.py`

**Payer-Type Matching (3):**
`payer_type_matcher.py`, `matching_type_constants.py`, `payer_type_logger.py`

**Background Sync / Auto-Update (4):**
`sync_integration.py`, `background_sync_worker.py`, `auto_update_manager.py`, `auto_update_dialogs.py`

**UI / Dialogs / Reports (6):**
`ui_helpers.py`, `treeview_border_styling.py`, `pdf_helpers.py`, `email_helpers.py`, `subscription_manager_dialog.py`, `data_init_progress_dialog.py`, `business_registration_dialog.py`

**Utilities (8):**
`path_helper.py`, `config_helper.py`, `date_utils.py`, `logging_utils.py`, `timing_logger.py`, `batch_calculator.py`, `single_instance.py`, `pbm_debug_logger.py`

### Loose-Scripts Inventory (Repo Root)

Categorized counts (verified by Glob):

| Pattern | Count | Sample files |
|---|--:|---|
| `test_*.py` | 106 | `test_pbm_key_assignment.py`, `test_aac_smoke.py`, `test_match_bin_004146.py` |
| `demo_*.py` | 25 | `demo_about_version.py`, `demo_pbm_workflow.py`, `demo_verify_rate_enhanced.py` |
| `check_*.py` | 8 | `check_db.py`, `check_pbm_data.py`, `check_keyring_tokens.py` |
| `debug_*.py` | 3 | `debug_profile.py`, `debug_bin_004146.py`, `debug_test.py` |
| `verify_*.py` | 4 | `verify_version_consistency.py`, `verify_icon_config.py`, `verify_implementation.py`, `verify_ui_changes.py` |
| `visualize_*.py` | 5 | `visualize_business_info_changes.py`, `visualize_column_mapping_redesign.py` |
| `benchmark_*.py` | 3 | `benchmark_medicaid_batch_repair.py`, `benchmark_medicaid_vectorization.py`, `benchmark_medicare_optimization.py` |
| `fix_*.py` | 1 | `fix_corrupted_file.py` |
| `inspect_*.py` | 1 | `inspect_jwt_payload.py` |
| `find_*.py` | 0 | (none at root; 2 in `tools/`) |
| `grok_test_*.py` | 2 | `grok_test_004146.py`, `grok_test_004146b.py` |
| `*_SUMMARY.md` / `*_IMPLEMENTATION*.md` | ~80 | (catalogued in `01-EXISTING-DOCS-REVIEW.md`) |

**INFERENCE** — These ~80 loose Python scripts at the repo root are organic accretion: many are one-shot demos, ad-hoc diagnostic queries, or "before/after" visualizers used during development. They have no entry-point registration, no shared `conftest.py`, and no dependency manifest separating runnable scripts from importable modules. The sprawl is itself a finding (see `10-RAW-FINDINGS-AND-QUESTIONS.md`).

### Tools Folder Roster (32 files)

Heavy concentration on BIN 004146 diagnostics (≥12 files reference 004146 or 4146):

**BIN 004146 / 4146 (12):** `check_004146.py`, `check_4146.py`, `query_004146.py`, `inspect_bin_004146.py`, `diagnose_004146.py`, `diagnose_004146_v2.py`, `find_4146_variants.py`, `find_4146_variants_arg.py`, `importdata_for_004146.py`, `normalize_and_backfill_004146.py`, `backfill_pbm_key_and_check_004146.py`, `detect_invisible_bin.py` (null-byte detection in BIN field)

**Migrations (3):** `migrate_keyring_tokens.py`, `migrate_normalize_bins.py`, `fix_normalized_bin_and_recompute_pbm_keys.py`

**Backfills (2):** `backfill_medicare_hierarchy.py`, `backfill_pbm_key_and_check_004146.py`

**Inspection / Diagnostics (8):** `check_pbm_info_table.py`, `check_sample_rows.py`, `check_import_debug.py`, `list_pbm_info_610097.py`, `list_pbm_info_keys_610097.py`, `show_user_data_610097.py`, `inspect_xlsx_cells.py`, `preview_match_keys.py`

**Normalizers / Helpers (2):** `bin_normalizer.py`, `import_row_debug_logger.py`

**Apply / Update (1):** `apply_updates_from_csv.py`

**Test scaffolding (3):** `create_test_write.py`, `write_test_debug.py`, `helpers_payer_type_matcher_Version11.py` (vendored backup of payer_type_matcher)

**External tool (1):** `al_medicaid_portal.py` (standalone Selenium scraper script)

**Doc (1):** `IMPORT_DEBUG.md`

### Examples Folder

**EVIDENCE** — 17 runnable `.py` scripts demonstrating: AAC reference download/sync workflows, AL Medicaid scraper variants (HTTP, Selenium, hybrid), logging patterns, sync worker demo, WAC import.
**EVIDENCE** — 4 sample CSVs: `pbm_info_example.csv`, `sample_medicare_list.csv`, `sample_template_extended.csv`, `sample_template_minimal.csv`, `sample_template_synonyms.csv`.
**EVIDENCE** — 1 `latest-version.json` (auto-update manifest template).
**EVIDENCE** — 2 README MDs: `examples/README.md`, `examples/IMPORT_TEMPLATES_README.md`.
**Source:** `examples/` listing.

### Build / Run Commands

**EVIDENCE** — Run: `python pharmacybooks.py` after `cp config.ini.template config.ini` and editing.
**Source:** `README.md:22-26, 43`

**CLAIM** — Build via PyInstaller per `BUILD.md` (not verified at code level in this extraction; agent only inventoried).
**Source:** `BUILD.md` (one-liner: "PharmacyBooks Desktop - Build & Packaging Guide").

### Tech-Stack Snapshot

| Layer | Technology |
|---|---|
| GUI | Tkinter via `ttkbootstrap` (theme: "flatly") |
| HTTP client | `requests` |
| Local DB | `sqlite3` (raw, NOT SQLAlchemy despite the dep) |
| Spreadsheet I/O | `pandas`, `openpyxl` |
| PDF | `reportlab`, `pypdf` |
| Browser automation | `selenium` (Chrome/Edge/Firefox via Selenium Manager) |
| Crypto | `cryptography` (Fernet for support temp passwords); `hashlib.sha256` (support codes, update verification) |
| Secret storage | `keyring` (Windows DPAPI / macOS Keychain / Linux Secret Service) |
| Cloud / object store | `google-cloud-storage` |
| Templating | `Jinja2` |
| Imaging | `Pillow` |
| Calendar widget | `tkcalendar` |
| Windows integration | `pywin32` |

---

## Open Questions

1. Is `sqlalchemy` actually used anywhere, or is it an unused dependency? (No imports found in the files inspected.)
2. What is in `requirements_linux.txt`? README references it (`README.md:121`) but the file was not read in this extraction.
3. Is the `download-site/` folder shipped with the app or merely co-located in the repo? (It looks like a separate marketing site.)
4. The `site - registration/` folder name contains a literal space — is this intentional, or accidental?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist (paths drawn from direct `ls`, `Read`, or sub-agent reports with citations)
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (`config.ini`, `requirements_linux.txt` contents, sqlalchemy usage, BUILD.md depth)
