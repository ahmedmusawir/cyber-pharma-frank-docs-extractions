# 10-RAW-FINDINGS-AND-QUESTIONS — Surprises, Contradictions, Open Questions

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — PRIMARY focus

---

## Summary

This is the catch-all for findings that don't fit cleanly elsewhere: oddities, contradictions, sprawl indicators, and surprising things found during shallow scans. Per User's additional rule: **shallow flags only, every item carries an evidence tag**. The dominant narrative is **organic accretion** — a 13K-line monolith, 127 root markdown files, 80+ loose root scripts, two test-file locations, three duplicate-purpose agent-config files, and a 12-file BIN-004146 debugging fleet. Several documented features contradict the current code state. The most critical finding is the hardcoded fallback Fernet secret (cross-referenced in 07).

---

## Findings

### A. The Sprawl Itself

**EVIDENCE** — Repo-root inventory (verified counts 2026-04-29):
- 321 entries at root
- 127 `.md` files at root (`ls | grep '\.md$' | wc -l = 127`)
- 106 `test_*.py` at root (`ls | grep '^test_.*\.py$' | wc -l = 106`)
- ~80 other loose `.py` scripts at root (`demo_*`, `check_*`, `debug_*`, `verify_*`, `visualize_*`, `benchmark_*`, `fix_*`, `inspect_*`, `find_*`, `grok_test_*`)
- 6 `.txt` files at root (mostly visual-comparison artifacts)
- 1 `pharmacybooks.py` of 577,060 bytes (13,200 lines)

**Source:** Direct Bash counts.

**INFERENCE** — A vibe-coded codebase. The 127 root MDs are mostly `*_SUMMARY.md` / `*_IMPLEMENTATION_*.md` historical artifacts; the 80+ loose scripts are "I needed to check X" one-shots that were committed. Together they form a thick layer of context whose half-life is short. The `docs/` folder represents the deliberate documentation; the root MDs represent the accumulated change log.

**EVIDENCE** — README claims historical docs were moved to `docs/archive/` (`README.md:125`), but `docs/archive/` does not exist on disk. **CONTRADICTED.**
**Source:** Filesystem.

**INFERENCE** — Either the cleanup was abandoned, or the archive was deleted. Either way, the README is wrong.

### B. The Monolith

**EVIDENCE** — `pharmacybooks.py`: 577 KB, 13,200 lines, no top-level `App` class. Architecture is procedural with module-level globals.
**Source:** `00-REPO-PROFILE.md` + architecture-mapper agent.

**EVIDENCE** — Inside the monolith: `ReimbursementComparer` is 6,156 lines (lines 2346-8501).
**Source:** Architecture-mapper agent.

**INFERENCE** — `ReimbursementComparer` is the de-facto domain model. Splitting it is non-trivial because Tk dialog handlers call it directly throughout the file.

**QUESTION** — How many of those 13,200 lines are actively reachable? Static analysis or coverage instrumentation would help, but neither has been run.

### C. Three Coding-Agent Config Files

**EVIDENCE** — `CLAUDE.md`, `AGENTS.md`, `WINDSURF.md` all live at repo root. `AGENTS.md` and `WINDSURF.md` are **byte-identical** (`diff` returns empty). Both are headed `# .windsurfrules — Windsurf/Cascade Configuration`.
**Source:** Bash `diff` invocation.

**INFERENCE** — `AGENTS.md` is mislabeled. Its filename suggests a tool-agnostic agents file, but its content is Windsurf-specific.

**EVIDENCE — CONTRADICTED** — All three files describe a tech stack of `Next.js 15 / FastAPI / Vertex AI / ADK / Gemini 2.5` (`CLAUDE.md:611-613`, `WINDSURF.md:582-586`). The actual repo has none of those — it is Tkinter desktop with `requests` against an App Engine API.
**Source:** `requirements.txt` + repo-root scan.

**INFERENCE** — These files appear to belong to a multi-project working environment whose default tech-stack table got copied verbatim into a project where it does not belong.

### D. BIN 004146 Obsession

**EVIDENCE — INFERENCE** — The codebase contains an unusually heavy concentration of BIN 004146 artifacts:
- ~12 scripts in `tools/` referencing 004146 or 4146 (`check_004146.py`, `diagnose_004146.py`, `find_4146_variants.py`, `query_004146.py`, `inspect_bin_004146.py`, `importdata_for_004146.py`, `normalize_and_backfill_004146.py`, `backfill_pbm_key_and_check_004146.py`, etc.)
- 7+ markdown summaries at root (`BIN_004146_FINAL_SUMMARY.md`, `BIN_004146_IMPLEMENTATION_VERIFICATION.md`, `BIN_004146_MATCHING_IMPLEMENTATION.md`, `BIN_004146_PRIORITY_COMPARISON.md`, `IMPLEMENTATION_COMPLETE_004146.md`, `IMPLEMENTATION_COMPLETE_BIN_004146_PRIORITY.md`, etc.)
- Special-case logic in `helpers/pbm_utils.py` (per `docs/PBM_KEY_CANONICAL_MATCHING.md`)
- 3+ test files (`test_match_bin_004146.py`, `test_pbm_file_loader_bin_004146.py`, `tests/test_pbm_key_matching.py`)
- 2 "grok_test" files (`grok_test_004146.py`, `grok_test_004146b.py`) — possibly Grok-AI-assisted debugging

**Source:** File listings.

**INFERENCE** — BIN 004146 represents a real, recurring matching problem (likely a PBM that doesn't follow standard BIN+PCN conventions). The volume of scripts suggests it's been hit-and-patched repeatedly rather than solved structurally.

**EVIDENCE — CONTRADICTED** — `BIN_004146_FINAL_SUMMARY.md` claims the special case is "already fully implemented" in `helpers/pbm_file_loader.py` lines 265-268 (Excel) and 466-468 (CSV). Grep for `004146\|4146` in `helpers/pbm_file_loader.py` returned **zero matches**.
**Source:** Docs-review agent.

**INFERENCE** — Either the special case was migrated to `helpers/pbm_utils.py` (likely, per the docs), or the doc described intent before any code was written. Either way, the doc is wrong about its current location.

### E. Version Drift

**EVIDENCE — CONTRADICTED** — `AUTO_UPDATE_IMPLEMENTATION_SUMMARY.md` claims version `1.0.0`. Code says `__version__ = "1.5.18"` (`version.py:5`).
**Source:** Direct read.

**EVIDENCE — STALE** — `helpers/api_client.py:73` hardcodes `User-Agent: 'PharmacyBooks-Desktop/1.0'`. This is sent on every API request and does not match `__version__`.
**Source:** Direct read.

**INFERENCE** — Multiple version strings drift independently. Server-side analytics keyed on User-Agent see only "1.0".

### F. Hardcoded Fallback Secret (CRITICAL)

**EVIDENCE — CRITICAL** — `helpers/temp_password_crypto.py:18`:
```python
_DEFAULT_SECRET = "pharmacybooks-fallback-secret"
```
Used at line 27 if no env var is set. Cross-referenced in `07-GUARDRAILS-AND-SANDBOXING.md`.
**Source:** Direct read.

**INFERENCE** — Anyone with this repo can compute the Fernet key (`base64url(sha256("pharmacybooks-fallback-secret"))`) and decrypt any temp-password ciphertext that was encrypted under the fallback. Whether this matters depends on whether the backend ever uses the fallback.

### G. Pytest Suite Broken

**EVIDENCE** — `pytest_output.txt` shows the test suite fails at collection with `ImportError: attempted relative import with no known parent package` in `helpers/sqlite_db_helper.py:91`.
**Source:** Tests-inventory agent.

**INFERENCE** — There is no `conftest.py` to add the repo root to `sys.path`. Without it, the `helpers` package is not importable for tests collected from the root. This may have been broken since the API-client refactor that introduced relative imports.

**QUESTION** — Has anyone successfully run the test suite in its current state?

### H. Two Test Locations

**EVIDENCE** — 9 tests in `tests/`, 106 `test_*.py` at root, with no `conftest.py` linking them.
**Source:** Tests-inventory agent.

**INFERENCE** — `tests/` looks like a deliberate, clean home that was started but never fully populated. The root accretion continued in parallel.

### I. Loose Scripts Junkyard

**EVIDENCE** — Per category counts at root:
| Pattern | Count |
|---|--:|
| `demo_*.py` | 25 |
| `check_*.py` | 8 |
| `debug_*.py` | 3 |
| `verify_*.py` | 4 |
| `visualize_*.py` | 5 |
| `benchmark_*.py` | 3 |
| `fix_*.py` | 1 |
| `inspect_*.py` | 1 |
| `grok_test_*.py` | 2 |

**Source:** Glob.

**INFERENCE** — Many of these are one-shot scripts authored to debug a specific issue, then committed. None have a unified registration; new contributors won't know which still work. Examples:
- `fix_corrupted_file.py` — UTF-16-to-UTF-8 file repair (likely a one-off fix)
- `grok_test_004146.py` (415 bytes) — looks like a Grok-AI scratchpad
- `inspect_jwt_payload.py` — diagnostic, also dangerous if used in production

### J. Vibe-Coded Globals

**EVIDENCE** — `pharmacybooks.py` mutates module-level globals across handlers:
- `pending_update_info`, `startup_dashboard_ready`, `PROFILE_REFRESH_WARNING_SHOWN`, `startup_import_files`, `global_reference_manager`, `update_manager`
**Source:** Architecture-mapper agent.

**INFERENCE** — These are testability and reasoning hazards. Order-of-initialization bugs become possible.

**EVIDENCE** — `FIXED_FEE = 10.64` (`pharmacybooks.py:236` per agent) is a magic-number constant.
**Source:** Architecture-mapper agent.

### K. "Coming Soon" Tabs in Production

**EVIDENCE** — `OpportunitiesDashboard` declares 6 tabs but only 2 are functional. Lines 8763, 8779, 8785, 8791 (per agent) are "Coming soon" placeholders for `NDC Optimization`, `Profit Driver Refills`, `Tx Opps`, `AAC Rate Review`.
**Source:** Architecture-mapper agent.

**QUESTION** — Are these visible in shipped builds, or hidden behind a feature flag?

### L. Doc/Code Mismatches in the Verification Sample

From `01-EXISTING-DOCS-REVIEW.md`'s 12-doc sample:
- 5 CONFIRMED, 5 UNVERIFIED, **2 CONTRADICTED** (BIN 004146 location, version 1.0.0 in auto-update doc)
- Plus the README.md `docs/archive/` claim **CONTRADICTED**

**INFERENCE** — At a 17% contradiction rate on a small sample, the 130 unverified root MDs likely contain dozens of similar drift. Treat the historical docs as untrustworthy unless cross-checked.

### M. SQLAlchemy Declared but Unused

**EVIDENCE** — `requirements.txt:1` declares `sqlalchemy>=2.0.27,<3.0.0`. `helpers/sqlite_db_helper.py` uses raw `sqlite3` (line 9 imports). No SQLAlchemy import was observed in the files inspected.
**Source:** Direct reads.

**QUESTION** — Is SQLAlchemy used anywhere, or is it a leftover from an earlier architecture?

### N. The "Liberty API" CSV

**EVIDENCE** — `docs/liberty_api_endpoints.csv` documents 9-column API endpoint metadata (Tag, Path, Method, Access, Summary, Description, HasRequestBody, Parameters, Responses). Sample rows describe `/ar`, `/ar/{patientId}`, `/delivery/`, `/delivery/queue` — these look like Liberty Software's pharmacy management API.
**Source:** Docs-review agent.

**QUESTION** — Does any code in this repo integrate with Liberty Software, or is this CSV aspirational? No code-level reference to "liberty" was identified in this pass (not exhaustively grepped).

### O. Folder Name with Literal Space

**EVIDENCE** — Folder `site - registration/` contains a literal space in its name (with surrounding spaces).
**Source:** Direct `ls`.

**INFERENCE** — This will break command-line tools that don't quote paths. Probably accidental.

### P. Co-Located Marketing Site

**EVIDENCE** — `download-site/` (3 files: `index.html`, `main.js`, `styles.css`) and `site - registration/` (2 HTML files) are static web pages co-located with the desktop client.
**Source:** Direct `ls`.

**QUESTION** — Are these deployed to the same domain (`pharmacybooks.com`)? Is there a CI step to push them, or are they edited and uploaded manually?

### Q. Vendored / Versioned Helper Backup

**EVIDENCE** — `tools/helpers_payer_type_matcher_Version11.py` looks like a vendored backup of `helpers/payer_type_matcher.py` at "Version11".
**Source:** File listing.

**INFERENCE** — Manual versioning of helpers in a separate folder is a code-smell — it suggests the team was unsure about a refactor and hedged.

### R. Hardcoded External Recipient

**EVIDENCE** — `helpers/sqlite_db_helper.py:121`:
```python
DEFAULT_ALDOI_EMAIL = "pbmcompliance@insurance.alabama.gov"
```
**Source:** Direct read.

**INFERENCE** — Hardcoded regulator email. Updating it requires a code change. Reasonable for a state-specific feature but brittle.

### S. Empty / Stub Files

**EVIDENCE** — `debug_bin_004146.py` at repo root is **0 bytes** (empty file).
**Source:** Bash `ls -la`.

**INFERENCE** — A stub or placeholder. Possibly an artifact of `touch` followed by an abandoned plan.

**EVIDENCE** — `helpers/__init__.py` is just two comment lines. No exports, no `__all__`.
**Source:** Direct read.

**INFERENCE** — Helpers are imported by full path; there's no curated public surface.

### T. Activation/Login Twin-Path

**EVIDENCE** — On startup, after activation, the user must also create a local account (`LocalAccountSetupDialog`) and log in (`LocalLoginDialog`). The local password is stored in SQLite (separate from the API JWT).
**Source:** Architecture-mapper agent.

**QUESTION** — What's the failure mode if the API token expires while the user is logged into the local account? Does the app continue offline-degraded, or prompt for re-auth?

### U. Selenium "turbo_fail" Mode

**EVIDENCE** — `helpers/verify_rate_config.py` exposes a `turbo_fail` setting (per agent).
**Source:** Architecture-mapper agent.

**INFERENCE** — A fast-fail mode for impatient operators — likely skips retries or shortens timeouts. Specific behavior not enumerated.

### V. Tools Module Imported by Helpers

**EVIDENCE** — `helpers/sqlite_db_helper.py:110`:
```python
try:
    from tools.import_row_debug_logger import log_import_row
except Exception:
    log_import_row = None
```
**Source:** Direct read.

**INFERENCE** — The general convention is `helpers/` ↔ `pharmacybooks.py`, with `tools/` as standalone scripts. This is the only `helpers → tools` import seen, and it makes the import structure unsymmetric. Dependency direction reversal (tools shouldn't be a library imported by helpers).

### W. Three "Final" Implementation Summaries

**EVIDENCE** — `BIN_004146_FINAL_SUMMARY.md` and `IMPLEMENTATION_COMPLETE_BIN_004146_PRIORITY.md` and `PBM_KEY_IMPLEMENTATION_FINAL_SUMMARY.md` and `ABOUT_DIALOG_VERSION_FINAL_SUMMARY.md` all coexist. "Final" is used loosely.
**Source:** File listing.

**INFERENCE** — A "Final" doc has been written more than once for the same feature. Reading order ambiguous; freshness ambiguous.

### X. Test File for Visual Styling

**EVIDENCE** — `test_treeview_styling.py`, `test_visual_enhancements.py` exist at repo root — these test UI styling.
**Source:** File listing.

**INFERENCE** — UI styling tests are unusual — typically these would be screenshot/visual regression tests, but I have no evidence the repo has a screenshot harness. Most likely these check that certain `ttk.Style` parameters are configured correctly.

### Y. The `READONLY_REFACTOR_SUMMARY.txt` Is `.txt` Not `.md`

**EVIDENCE** — `READONLY_REFACTOR_SUMMARY.txt` and `IMPLEMENTATION_COMPLETE_SUMMARY.txt` are `.txt`, not `.md`.
**Source:** File listing.

**INFERENCE** — Inconsistent doc-file conventions. Probably no reason — just different copy-paste habits.

### Z. The `examples/IMPORT_TEMPLATES_README.md` Templates

**EVIDENCE** — `examples/sample_template_synonyms.csv` documents alternative column names supported by the importer (e.g., "RX Number" → "Script", "Fill Date" → "Date Dispensed"). Multiple synonyms per canonical name.
**Source:** Docs-review agent.

**INFERENCE** — Synonym mapping is a real, tested feature — and it widens the surface area of the column-mapping logic.

---

## Open Questions (Consolidated for Tony)

(Numbered list spans all 11 docs; here are the items most worth a human eye.)

### Critical / Security

C1. Is the backend deployed with one of the temp-password Fernet env vars set, or is the `_DEFAULT_SECRET` fallback in active use? (`07.A5`)

C2. Does the JWT validation re-check the `exp` claim before each API call, or only at startup? (`07.A3`)

C3. Are local user passwords KDF-hashed (bcrypt/argon2/scrypt) or just SHA-256? (`07` Open Q6 — `helpers/local_auth_helper.py` not directly read)

### High / Code Health

H1. Is `helpers/api_db_helper.py` dead code? (`02` D2)

H2. Are the four "Coming soon" `OpportunitiesDashboard` tabs reachable in production? (`02`/`04` Open Qs)

H3. Has the pytest suite ever passed in its current state, or is the collection error a long-standing issue? (`09` D)

H4. Is SQLAlchemy actually used anywhere, or is the dependency leftover? (`00` Open Q1)

### Medium / Documentation

M1. Should `AGENTS.md` be deleted, renamed, or rewritten? (`06` A1)

M2. Should the README claim about `docs/archive/` be removed? (`01` Section "docs/archive/")

M3. Why does the agent-config tech-stack table not match the actual repo? (`06` A4)

### Low / Cosmetic

L1. Is `site - registration/` (with literal space) intentional?

L2. Is `debug_bin_004146.py` (0 bytes) a stub or accidental commit?

L3. The `User-Agent: 'PharmacyBooks-Desktop/1.0'` hardcoded in `api_client.py:73` should probably interpolate `__version__`.

### Process / Workflow

P1. Are the 80+ loose root scripts maintained, or are they accumulated debris?

P2. Why does the team produce a `_FINAL_` summary for every feature? Are they prescribed by a workflow, or organic?

P3. Are the `grok_test_*.py` files Grok-AI debugging scratchpads, and should they live in `tools/` (or be deleted)?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist (paths + line numbers from direct reads or sub-agent reports)
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included (smells, contradictions, and questions only)
- [x] GAPs explicitly documented (sqlalchemy usage, dead-code confirmation, OpportunitiesDashboard reachability, password KDF)
- [x] User's "shallow flags only" rule honored: no deep dives — surprises captured and moved on
