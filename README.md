# Cyber Pharma — Frank's Docs Extractions

Reverse-engineered documentation for **Frank's two pharmacy codebases**, extracted by Claude Code on **2026-04-29** from the original (un-documented) repos. Every finding below is tagged with `EVIDENCE` / `INFERENCE` / `CLAIM` / `GAP` and points back to the exact `file:line` it came from in the source repos.

This README is the **central hub** — pick a repo, pick a topic, click through.

---

## The Two Repos

| Folder | Source Repo | What it is | Stack |
|---|---|---|---|
| [`PHARMACY_APIs/`](./PHARMACY_APIs/) | `pharmacybooks-api-main` | Multi-tenant **Flask SaaS API** — onboarding, Stripe billing, prescription ingest, WAC/AAC/FUL/PBM reference data | Python 3.11, Flask, SQLAlchemy, gunicorn, Cloud Run / App Engine, Cloud SQL (MySQL+Postgres), Stripe, GHL, GCS |
| [`PHARMACY_BOOK_DESKTOP_EXTRACTS/`](./PHARMACY_BOOK_DESKTOP_EXTRACTS/) | `pharmacybooks-desktop-main` | The companion **Tkinter desktop client** v1.5.18 — the app the pharmacies actually run on Windows | Python 3.8+, Tkinter (ttkbootstrap), SQLite, Selenium, Inno Setup, GCS auto-update channel |

The API serves the desktop client over `https://pharmacy-books-desktop.ue.r.appspot.com/api`. They are tightly coupled — read both sides for any cross-cutting question (auth, sync, activation, reference data).

---

## The Doc Template (same 11 files per repo)

Each repo was scanned with the same template, so you can read them side-by-side. Numbering is identical across folders.

| # | Topic | Why you'd open it |
|---|---|---|
| 00 | **Repo Profile** | The "what's in the box" intro: language, framework, deps, file tree, deploy targets |
| 01 | **Existing Docs Review** | What the *original authors* wrote in their own `.md` files — confirmed vs. contradicted vs. unverified |
| 02 | **Architecture Map** ⭐ | The system structure — modules, blueprints, models, integrations, data flow |
| 03 | **Request/Response Lifecycle** | The end-to-end flow of a request (API) or a user action (desktop) |
| 04 | **Endpoint / Command Surface** | The full inventory of HTTP routes (API) or user/Selenium/API commands (desktop) |
| 05 | **Context and Memory** | Where state lives — sessions, JWTs, SQLite tiers, caches, correlation IDs |
| 06 | **Prompts and Persona** | The IDE coding-agent config files (`CLAUDE.md`/`AGENTS.md`/`WINDSURF.md`) and templated text |
| 07 | **Guardrails and Sandboxing** ⭐ | Auth, encryption, audit logging, multi-tenant isolation — and where the gaps are |
| 08 | **Error Handling and Recovery** | Try/except patterns, retries, webhook idempotency, recovery markers |
| 09 | **Tests and Evals** | Test layout, frameworks, coverage gaps |
| 10 | **Raw Findings and Questions** | The catch-all — surprises, contradictions, vibe-coded smells, open questions |

⭐ = the two PRIMARY-focus documents in each set.

---

## PHARMACY_APIs — Flask SaaS Backend

**TL;DR:** 22.5K lines of Python across 25 modules and **13 Flask blueprints** behind `/api/*`. Backed by Stripe (live keys, webhook-driven onboarding), Go High Level (CRM), Google Cloud (Secret Manager, Cloud SQL, GCS, Functions, Tasks). Three concurrent deploy paths (Cloud Run, App Engine, Cloud Functions). HIPAA-aspirational; framework controls are sound but app-level gaps exist.

| Doc | One-liner |
|---|---|
| [00 — Repo Profile](./PHARMACY_APIs/FRANK_API-00-REPO-PROFILE.md) | Flask 3.11 + gunicorn, dual MySQL/Postgres drivers, 62 Alembic migrations, ~38 test files (*claimed* 433 tests), three deploy targets in parallel. |
| [01 — Existing Docs Review](./PHARMACY_APIs/FRANK_API-01-EXISTING-DOCS-REVIEW.md) | 47 author-written `.md` files (17 root + 30 `/docs/`) — heavy redundancy (7 Postgres setup docs), HIPAA checklist still aspirational, several contradictions flagged. |
| [02 — Architecture Map](./PHARMACY_APIs/FRANK_API-02-ARCHITECTURE-MAP.md) ⭐ | All 13 blueprints, 16 SQLAlchemy models, 3 ORM event listeners, and 7 external integrations mapped. The migration dependency graph for any rebuild. |
| [03 — Request/Response Lifecycle](./PHARMACY_APIs/FRANK_API-03-AGENT-LOOP.md) | The webhook-driven onboarding flow: register → Stripe checkout → webhook creates Business → ORM listener auto-issues activation key → user activates → desktop unlocks. |
| [04 — Endpoint Catalog](./PHARMACY_APIs/FRANK_API-04-TOOL-SYSTEM.md) | ~130 routes across 13 blueprints, with auth decorator (`PUBLIC` / `@jwt_required` / `@admin_required` / `@super_admin_required`) and source line for each. |
| [05 — Context and Memory](./PHARMACY_APIs/FRANK_API-05-CONTEXT-AND-MEMORY.md) | JWT cookie sessions, OAuth Flask sessions, contextvars correlation IDs, content-hash checksums for reference-data delta-sync. No Redis. |
| [06 — Prompts and Persona](./PHARMACY_APIs/FRANK_API-06-PROMPTS-AND-PERSONA.md) | The "Stark Industries" agent-config triplet (`CLAUDE.md`/`AGENTS.md`/`WINDSURF.md`) plus email/HTML templates and API response strings. |
| [07 — Guardrails and Sandboxing](./PHARMACY_APIs/FRANK_API-07-GUARDRAILS-AND-SANDBOXING.md) ⭐ | The HIPAA-migration gating doc. What works (JWT, HSTS, audit log, Stripe HMAC) vs. what doesn't (column encryption is comment-only, `/api/pharmacy_profile` is public mass-assign, GHL webhook signature optional, hardcoded Fernet fallback). |
| [08 — Error Handling and Recovery](./PHARMACY_APIs/FRANK_API-08-ERROR-HANDLING-AND-RECOVERY.md) | Three-layer error handling, webhook returns 200 even on internal errors (anti-retry), activation-key 3×2s retry loop, no global exception middleware. |
| [09 — Tests and Evals](./PHARMACY_APIs/FRANK_API-09-TESTS-AND-EVALS.md) | pytest, integration-heavy, SQLite in-memory. Gaps: no perf, no fuzzing, no real-Stripe E2E, no OpenAPI contract tests, no MySQL/Postgres parity tests. |
| [10 — Raw Findings and Questions](./PHARMACY_APIs/FRANK_API-10-RAW-FINDINGS-AND-QUESTIONS.md) | Catch-all: two parallel Stripe webhook handlers, two registration paths, committed binaries (`cloud-sql-proxy.exe`, `upload-trace.log`), live Stripe key in `app.yaml`, `session.json` at repo root. |

---

## PHARMACY_BOOK_DESKTOP_EXTRACTS — Tkinter Desktop Client

**TL;DR:** A **monolithic procedural** Python desktop app — a single 13,200-line `pharmacybooks.py` orchestrator + ~63 `helpers/` modules + ~32 `tools/` scripts + ~80 loose `*.py` and **127 loose `*.md`** at the repo root. Tk event loop + 4 daemon threads (sync, auto-update, subscription poll, scrapers). SQLite locally, REST API remotely, GCS for downloads. Inno Setup builds the Windows installer.

| Doc | One-liner |
|---|---|
| [00 — Repo Profile](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-00-REPO-PROFILE.md) | v1.5.18, Python 3.8+, no `pyproject.toml`/`setup.py`/`Makefile`, GCS auto-update manifest at `downloads.pharmacybooks.com/latest-version.json`, 6h check interval. |
| [01 — Existing Docs Review](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-01-EXISTING-DOCS-REVIEW.md) | **142 markdown files** total (127 root + 15 `docs/`). README claims `docs/archive/` exists — **it doesn't**. `AGENTS.md` is a byte-identical duplicate of `WINDSURF.md`. |
| [02 — Architecture Map](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-02-ARCHITECTURE-MAP.md) ⭐ | The 13K-line orchestrator + flat helpers package, Tk + ttkbootstrap, SQLite `user_data.db`, App Engine REST API, GCS, Selenium for Alabama Medicaid Portal. No service layer, no DI, module-level globals coordinate state. |
| [03 — Lifecycle / Event Loop](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-03-AGENT-LOOP.md) | The Tk main thread + 4 daemon-thread loops, cross-thread results via `root.after(0, callback)`. Sequential startup phase machine (activation → login → reference data → main window). |
| [04 — Command Surface](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-04-TOOL-SYSTEM.md) | The Tk widget tree + `helpers/api_client.APIClient` HTTP methods + GCS downloaders + Selenium scraper commands. No client-side permission layer (server-side JWT enforces). |
| [05 — Context and Memory](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-05-CONTEXT-AND-MEMORY.md) | 3-tier state: SQLite `user_data.db` (truth) + Python globals/Tk widgets (mutable) + on-disk caches (column maps, device identity, sync state, recalc markers). Secrets in OS keyring. |
| [06 — Prompts and Persona](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-06-PROMPTS-AND-PERSONA.md) | No in-app LLM. Three coding-agent config files describe an "AI App Factory" Next.js/FastAPI/Vertex-AI stack that **does not match** this app's actual Tkinter+AppEngine reality. |
| [07 — Guardrails and Sandboxing](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-07-GUARDRAILS-AND-SANDBOXING.md) ⭐ | Activation key + local SQLite login + JWT in OS keyring + support-override hash + Fernet temp-passwords + single-instance lock + device UUID + SQL column whitelist. **Critical:** hardcoded Fernet fallback `"pharmacybooks-fallback-secret"` in `helpers/temp_password_crypto.py:18`. |
| [08 — Error Handling and Recovery](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-08-ERROR-HANDLING-AND-RECOVERY.md) | Two well-developed subsystems (auto-update with hash-verified rollback, sync worker with exp-backoff). Network errors degrade to local SQLite-only. Recalc policy survives restarts via `pending_recalculate.json` markers. |
| [09 — Tests and Evals](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-09-TESTS-AND-EVALS.md) | Tests scattered across `tests/` (9 files) AND repo root (**106 loose `test_*.py`**). No `conftest.py`, no `pytest.ini`. Suite **fails at collection** due to relative-import bug. 50% of helpers untested. |
| [10 — Raw Findings and Questions](./PHARMACY_BOOK_DESKTOP_EXTRACTS/FRANK_DESKTOP-10-RAW-FINDINGS-AND-QUESTIONS.md) | The sprawl indictment: 13K-line monolith, 127 root MDs, 80+ loose root scripts, 12-file BIN-004146 debug fleet, three duplicate agent-config files. Several documented features contradict current code. |

---

## How to Read This

- **Onboarding a new collaborator?** Start with both `00`s (profiles) and both `02`s (architecture maps).
- **Planning a Supabase / rebuild migration?** `02` is the dependency graph. `07` is the gating list of HIPAA / security gaps that must close before cutover.
- **Touching auth or activation?** Read API `03` + Desktop `03` together — the flow crosses both repos.
- **Auditing security?** Both `07`s are the primary focus docs. Both `10`s surface the smells.
- **Hunting contradictions / surprises before quoting any author doc?** `01` (verification status of every existing doc) and `10` (catch-all surprises).

Every doc opens with a Summary, lists Findings with evidence tags + `file:line` source, and closes with Open Questions. No invented function names; no synthesis or recommendations — extraction only.

---

**Extraction methodology:** Brain-drain skill, Claude Code, 2026-04-29. Source repos were *not* in this repository — only the extraction outputs were copied here.
