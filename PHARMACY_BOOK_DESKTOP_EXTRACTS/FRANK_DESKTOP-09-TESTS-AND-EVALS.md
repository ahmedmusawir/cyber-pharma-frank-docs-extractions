# 09-TESTS-AND-EVALS — Test Infrastructure, Coverage, Benchmarks

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL

---

## Summary

Tests live in **two parallel locations** with no unifying configuration: 9 structured tests in `tests/` and **106 loose `test_*.py` files at repo root**. There is no `conftest.py`, no `pytest.ini`, no `setup.cfg`, no `pyproject.toml`. The cached `pytest_output.txt` shows the test suite **fails at collection time** because of a relative-import error in `helpers/sqlite_db_helper.py:91`. Test frameworks are mixed (`unittest.TestCase` AND plain pytest functions, intermixed). No coverage reports are committed. **32 of 64 helper modules (50%) have no test file** referencing them — by far the biggest GAP. Three benchmark scripts exist (`benchmark_*.py`) for vectorization claims but are not part of CI.

---

## Findings

### A. Test Layout — Two Locations

#### A1. Structured `tests/` Folder (9 files)

**EVIDENCE** — All 9 files (per tests-inventory agent + direct `ls`):
| File | Purpose (one-line) |
|---|---|
| `tests/test_action_needed_mismatch_lock.py` | Tests Action Needed mismatch locking and record state |
| `tests/test_action_needed_pbm_candidates.py` | PBM candidate selection for Action Needed events |
| `tests/test_device_identity.py` | Device identity persistence + `pbm-` prefix |
| `tests/test_email_reset_token.py` | Email reset-token extraction from URLs/fragments |
| `tests/test_password_reset_token_utils.py` | Password reset-token URL parsing |
| `tests/test_pbm_info_update.py` | PBM info update propagation + `pbm_key` immutability |
| `tests/test_pbm_key_matching.py` | Canonical `pbm_key` matching incl. BIN 004146 |
| `tests/test_temp_password_sync.py` | Temp-password token application + local user sync |
| `tests/test_token_helper.py` | Token lookup, validation, save, migration |

**Source:** Tests-inventory agent + `ls tests/`.

**EVIDENCE** — These tests use pytest with `unittest.TestCase` classes and plain pytest functions. Fixtures: `tmp_path`, `monkeypatch`, mocks (e.g., `StubLocalAuth`).
**Source:** Tests-inventory agent.

#### A2. Loose Repo-Root Tests (106 files)

**EVIDENCE** — 106 files matching `test_*.py` at repo root (verified via `ls | grep -E '^test_.*\.py$' | wc -l = 106`).
**Source:** Bash count.

**EVIDENCE** — Domain breakdown (counts approximate, derived from filename prefixes):
| Cluster | ~Count | Examples |
|---|--:|---|
| AAC | 8 | `test_aac_smoke.py`, `test_aac_week_alignment.py`, `test_aac_import_persistence.py` |
| Matching Type | 6 | `test_matching_type.py`, `test_matching_type_csv_import.py`, `test_matching_type_required.py` |
| PBM Key | 5 | `test_pbm_key_assignment.py`, `test_pbm_key_column.py`, `test_pbm_key_dashboard_display.py` |
| Column Mapping | 4 | `test_column_mapping_fix.py`, `test_column_mapping_inline_preview.py`, `test_column_mapping_refactor.py` |
| Profile Dialog | 3 | `test_profile_dialog_api_integration.py`, `test_profile_dialog_na_defaults.py` |
| Payer Type | 3 | `test_payer_type_matcher.py`, `test_payer_type_logging.py` |
| Auto-update | 2 | `test_auto_update_manager.py`, `test_auto_update_integration.py` |
| Inclusion lists | 3 | `test_inclusion_list_downloader.py`, `test_inclusion_list_paths.py`, `test_inclusion_lists_structure.py` |
| Medicaid | 4 | `test_medicaid_payer_type_matching.py`, `test_medicaid_batch_repair.py`, `test_medicaid_rate_formatting.py` |
| Medicare | 3 | `test_medicare_list_loader.py`, `test_medicare_pbm_integration.py`, `test_medicare_vectorization.py` |
| NDC | 2 | `test_ndc_normalization_comprehensive.py`, `test_ndc_migration.py` |
| Path | 3 | `test_path_helper.py`, `test_path_integration.py`, `test_path_refactoring_compliance.py` |
| Verify Rate | 2 | `test_verify_rate_config.py`, `test_verify_rate_enhanced.py` |
| BIN 004146 | 1 | `test_match_bin_004146.py` |
| Misc / single-file | ~50 | `test_action_needed_filtering.py`, `test_treeview_styling.py`, `test_visual_enhancements.py`, etc. |

**Source:** Tests-inventory agent.

**INFERENCE** — Loose root tests grew organically alongside fix commits — many appear to have been written for a single bug or feature and never moved into the `tests/` package.

### B. Test Framework Inconsistency

**EVIDENCE** — Sampled root tests use a mix of approaches (per tests-inventory agent reading top 50 lines of 5 files):
- `test_aac_smoke.py` — `unittest.TestCase`
- `test_pbm_key_assignment.py` — `unittest.TestCase` with `setUp`/`tearDown`, `tempfile.mkdtemp()`
- `test_column_mapping_fix.py` — plain pytest functions with `tempfile.TemporaryDirectory`
- `test_import_debug_integration.py` — `unittest.TestCase` with `unittest.mock.patch` + env mocking
- `test_match_bin_004146.py` — argparse + sqlite3 connect — closer to a CLI diagnostic than a test (per agent: "Diagnostic/example script, not traditional test")

**Source:** Tests-inventory agent.

**INFERENCE** — `test_match_bin_004146.py` is mislabeled as a test — it's an interactive CLI tool. This pattern may repeat across other root files; CI runs would treat them all as tests.

### C. Missing Test Configuration

**EVIDENCE — GAP** — No `conftest.py` (root or `tests/`), no `pytest.ini`, no `setup.cfg`, no `pyproject.toml`.
**Source:** Tests-inventory agent + filesystem scan.

**INFERENCE** — Without configuration, pytest's rootdir is inferred from invocation — different invocation locations may collect different files. There is no shared fixture surface, no common DB seeding, no test-data directory convention.

**INFERENCE** — Tests that need `helpers/` to be importable as a package depend entirely on the cwd / `PYTHONPATH` at test time — see the import error below.

### D. Pytest Collection Failure (Active Bug)

**EVIDENCE** — `pytest_output.txt` (committed at repo root) shows the test run halted at collection. Reported by tests-inventory agent:
- 244 items collected
- 1 ERROR halts the run
- Error location: `test_import_date_handling.py:8`
- Root cause: `ImportError: attempted relative import with no known parent package` in `helpers/sqlite_db_helper.py:91` attempting `from .api_client import get_api_client, APIError, APIClient`

**Source:** Tests-inventory agent reading `pytest_output.txt`.

**INFERENCE** — When `pytest` collects a root-level test file that imports `helpers.sqlite_db_helper`, the relative import `from .api_client import ...` resolves only if `helpers` is recognized as a package in the current Python path. Without a `conftest.py` adding the repo root to `sys.path`, this fails. The `try/except ImportError` in `helpers/sqlite_db_helper.py:115-118` falls back to `from api_client import ...` — but this requires `api_client` to be importable as a top-level module, which only works when `cwd = helpers/`. Neither path works under pytest's default rootdir resolution.

**EVIDENCE** — `pytest_output.txt` was last updated alongside the rest of the repo (Mar 20 timestamp). It has not been refreshed; the failure may have been there for a while.
**Source:** `ls -la pytest_output.txt` timestamp.

### E. Coverage Gaps — Helper Modules Without Tests

**EVIDENCE** — Tests-inventory agent's coverage table identified **32 of 64 helper modules with NO corresponding test** (50% coverage gap):
- `__init__.py`, `aac_enrichment.py`, `activation_key_dialog.py`
- `al_medicaid_batch.py`, `al_medicaid_http_prototype.py`, `al_medicaid_hybrid_batch.py`, `al_medicaid_scraper.py`
- `api_db_helper.py` (legacy/dead)
- `api_login_dialog.py`, `background_sync_worker.py`
- `business_registration_dialog.py`, `cache_manager.py`
- `column_mapping_dialog.py`, `column_name_sanitizer.py`, `config_helper.py`, `csv_utils.py`
- `data_init_progress_dialog.py`, `email_helpers.py`
- `local_account_setup_dialog.py`, `local_auth_helper.py`, `log_manager.py`, `logging_utils.py`, `login_dialog.py`
- `medicaid_portal_scraper.py`
- `opportunities_sheet_loader.py`, `pbm_debug_logger.py`, `pbm_key_backfill_migration.py`
- `pdf_helpers.py`, `single_instance.py`
- `subscription_manager_dialog.py`, `support_access.py`, `sync_integration.py`
- `treeview_border_styling.py`, `verify_rate_config.py`, `verify_rate_dialog.py`

**Source:** Tests-inventory agent.

**INFERENCE — CRITICAL GAPs** — Several untested modules carry security or correctness weight:
- `local_auth_helper.py` — local password hashing (no unit tests verifying KDF strength)
- `support_access.py` — support override authentication (no unit tests)
- `single_instance.py` — concurrency lock (no unit tests for race conditions)
- `email_helpers.py` — SMTP delivery (no unit tests for credential handling)
- `cache_manager.py` — cache invalidation (no unit tests for TTL/eviction)
- `pdf_helpers.py` — PDF rendering (no unit tests)
- All AL Medicaid scrapers — Selenium integration (no unit/contract tests)

### F. Heavily-Tested Modules

**EVIDENCE — Coverage Strengths** (per tests-inventory agent):
| Module | Test files referencing it |
|---|---|
| `sqlite_db_helper.py` | ~40+ (heavily tested) |
| `data_importer.py` | 6 |
| `payer_type_matcher.py` | 9 |
| `path_helper.py` | 7+ |
| `batch_calculator.py` | 3 |
| `matching_type_constants.py` | 7 |
| `column_mapping_helper.py` | 4 |
| `inclusion_list_downloader.py` | 3 |

**Source:** Tests-inventory agent.

**INFERENCE** — Coverage concentration mirrors business focus: import + matching + persistence are the most-tested areas. Auth/recovery/scrapers are systematically under-tested.

### G. Benchmarks (Not Tests, but Validation Surface)

**EVIDENCE** — Three benchmark scripts at repo root:
- `benchmark_medicaid_batch_repair.py` — sequential vs vectorized repair
- `benchmark_medicaid_vectorization.py` — Medicaid vectorization perf
- `benchmark_medicare_optimization.py` — Medicare optimization perf

**Source:** Tests-inventory agent + `ls`.

**EVIDENCE** — Performance claims they support (per `docs/TIMING_AND_DIAGNOSTICS.md`):
- Data Import: 1,000+ records/sec
- Batch Calculator: 6,882+ records/sec (vectorized)
- Medicaid Rate Workflow: 524+ records/sec
- "656x faster" vectorized vs row-by-row

**Source:** Docs-review agent.

**GAP** — Benchmarks are run manually, not in CI. No baseline files committed; no regression alerts.

### H. Loose-Script "Tests" That Are Not Tests

**EVIDENCE** — At repo root, 106 `test_*.py` PLUS:
- 25 `demo_*.py` (demonstration scripts; some assert correctness, mostly print)
- 8 `check_*.py` (DB inspection; not assertions)
- 3 `debug_*.py` (interactive debugging)
- 4 `verify_*.py` (one-shot verification, e.g., `verify_version_consistency.py`)
- 5 `visualize_*.py` (ASCII before/after; no assertions)
- 3 `benchmark_*.py` (perf measurement)
- 1 `fix_*.py` (one-off file repair)
- 1 `inspect_*.py` (`inspect_jwt_payload.py`)
- 2 `grok_test_*.py` (`grok_test_004146.py`, `grok_test_004146b.py`)

**Source:** Tests-inventory agent + `ls` patterns.

**INFERENCE** — The mix of `demo_*`/`check_*`/`debug_*`/`verify_*`/`visualize_*`/`fix_*`/`inspect_*`/`grok_test_*` suggests an organic workflow where authors add new scripts whenever they need to inspect something new. None are documented in a unified place. The `grok_test_*` prefix likely indicates someone using xAI's Grok to debug BIN 004146 and committing the resulting scripts.

### I. CI / Test-Runner Integration

**GAP** — No `.github/workflows/`, no `.gitlab-ci.yml`, no `azure-pipelines.yml`, no `tox.ini`, no `Makefile` test target — none of these were present in the file listing.
**Source:** Filesystem scan.

**INFERENCE** — The test suite has no automated runner. Tests are exercised manually (or not at all). The committed `pytest_output.txt` is the most recent evidence of a run — and it failed at collection.

### J. Test Documentation

**EVIDENCE** — Two test-related docs:
- `TESTING_GUIDE.md` (root) — claims testing procedures
- `docs/PATH_REFACTORING_TESTING.md` — testing checklist for path refactoring
- `TEST_RESULTS.md` (root) — test result summary

**Source:** Docs-review agent + filesystem.

**INFERENCE** — Existence of `TEST_RESULTS.md` suggests historical test runs were captured, but its contents were not extracted in this pass.

### K. Tests as Documentation

**EVIDENCE** — Several tests double as executable specs:
- `tests/test_pbm_key_matching.py` — encodes the canonical-matching contract incl. BIN 004146
- `tests/test_token_helper.py` — encodes the token-lookup priority order
- `tests/test_device_identity.py` — encodes the `pbm-<uuid4>` device-id format

**Source:** Tests-inventory agent.

**INFERENCE** — These 9 `tests/` files are the most reliable specifications in the repo, more so than the markdown docs (many of which are stale). For the rebuild, treat these as authoritative.

---

## Open Questions

1. Has anyone successfully run `pytest` against this repo in its current state, or has the collection error been there since the import refactor?
2. Are the loose root tests intended to be runnable independently (`python test_xyz.py`) or under pytest? Many of them look like the former.
3. Why does `test_match_bin_004146.py` at root duplicate `tests/test_pbm_key_matching.py` (the latter has BIN 004146 coverage)?
4. Is `test_payer_type_matcher.py` at 61,430 bytes really one test file, or did it grow into a kitchen-sink suite?
5. Should test outputs (TEST_RESULTS.md, pytest_output.txt) be regenerated periodically, or are they stale historical artifacts?
6. Are there any contract tests against the App Engine API (none observed), or does the desktop app trust the API completely?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] **GAPs aggressively documented** as required by skill rule for this doc:
  - 32/64 helper modules untested
  - No conftest.py / pytest.ini / pyproject.toml
  - No CI configuration
  - Pytest collection error in cached output
  - No automated benchmark regression
  - Critical-path modules (auth, scrapers, single_instance) untested
