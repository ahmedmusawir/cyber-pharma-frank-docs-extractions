# 06-PROMPTS-AND-PERSONA — Coding-Agent Configuration (Reframed)

**Repo:** pharmacybooks-desktop-main
**Extraction Date:** 2026-04-29
**Extracted By:** Claude Code
**Status:** FINAL — REFRAMED (no in-app LLM persona; this doc describes the coding-agent configuration files used by humans editing this repo)

---

## Summary

This application has **no in-app LLM** and **no system prompts**. It does, however, ship three repo-root files that look like agent configuration: `CLAUDE.md`, `AGENTS.md`, and `WINDSURF.md`. They are instructions for **AI coding tools** (Claude Code, Cascade/Windsurf) operating on this codebase under "Tony Stark's" supervision (per the files' framing). `AGENTS.md` is **byte-identical** to `WINDSURF.md` — a mislabeled duplicate. `CLAUDE.md` is mostly parallel to `WINDSURF.md` but adds Claude-Code-specific tool semantics (e.g., the mechanical Plan Mode tool). All three describe an "AI App Factory" tech stack (Next.js, FastAPI, Vertex AI/ADK) that does **not match this app's actual stack** (Tkinter desktop, App Engine API). In-app "templated text" surfaces (PDF reports, email bodies via Jinja2) were not deeply enumerated in this pass.

---

## Findings

### A. The Three Coding-Agent Config Files

#### A1. File Identity

**EVIDENCE** — `CLAUDE.md` (root) — 685 lines, version `3.0 | March 2026`. Header: `# CLAUDE.md — Claude Code Configuration`.
**Source:** `CLAUDE.md:1-5`.

**EVIDENCE** — `AGENTS.md` (root) — 687 lines, version `2.0 | March 2026`. Header: `# .windsurfrules — Windsurf/Cascade Configuration`.
**Source:** `AGENTS.md:1-5`.

**EVIDENCE** — `WINDSURF.md` (root) — 687 lines, version `2.0 | March 2026`. Header: `# .windsurfrules — Windsurf/Cascade Configuration`.
**Source:** `WINDSURF.md:1-5`.

**EVIDENCE** — `diff AGENTS.md WINDSURF.md` returns no output (files are byte-identical).
**Source:** Bash `diff` invocation 2026-04-29.

**INFERENCE** — `AGENTS.md` is mislabeled. Its content matches WINDSURF.md exactly. Tooling that auto-loads `AGENTS.md` (a common cross-tool convention) will receive Windsurf-specific guidance.

#### A2. Common Persona ("Tony's Hands")

**EVIDENCE** — All three files share the same role definition (verbatim across CLAUDE.md / AGENTS.md / WINDSURF.md, lines 12-14 of each):

> *"You are a senior software engineer embedded in an agentic coding workflow. You write, refactor, debug, and architect code alongside Tony Stark, who reviews your work in a side-by-side IDE setup."*

**EVIDENCE** — All three define the same operational philosophy (lines 17-19):

> *"You are the hands; Tony is the architect. Move fast, but never faster than Tony can verify."*

**Source:** `CLAUDE.md:9-19`, `AGENTS.md:9-19`, `WINDSURF.md:9-19`.

#### A3. Common Workflow Rules

**EVIDENCE** — All three enforce the same general rules (catalogued by docs-review agent, sample below):
- Plan Mode protocol — write plan to session file as PENDING_APPROVAL → present in CLI/chat → wait for approval → execute → mark complete
- Maintain a daily session file `session_YYYY-MM-DD.md` and a `RECOVERY.md` at repo root
- Follow TDD flow `Build → Unit Test → Integrate → Block Test → System Test → Finalize`
- "Ironman Rule": don't break working features when fixing one
- Scope discipline: surgical changes only; no unsolicited refactors
- Sycophancy is a failure mode; push back when warranted
- 20+ enumerated failure modes to avoid (sample: skipping plan mode, killing working features, calling `os.getenv()` directly instead of via `config_service`)

**Source:** Docs-review agent + cross-references to `CLAUDE.md`/`AGENTS.md`/`WINDSURF.md` mid-file sections.

#### A4. Substantive Deltas (CLAUDE.md vs WINDSURF.md/AGENTS.md)

| Delta | CLAUDE.md | WINDSURF.md/AGENTS.md |
|---|---|---|
| Plan Mode enforcement | Mechanical via `EnterPlanMode` tool — harness physically removes write tools (`CLAUDE.md:33-36`) | Behavioral only — session file write is the substitute mechanical anchor (`WINDSURF.md:33-40, 620-651`) |
| Announcement banner | `🔵 ENTERING PLAN MODE` in CLI (`CLAUDE.md:64-67`) | `🔵 PLANNING` in chat (`WINDSURF.md:70-75`) |
| RECOVERY.md emphasis | Mentioned as required (`CLAUDE.md:209-223`) | Same content, phrasing varies (`WINDSURF.md:189-202`) |
| Tool-specific notes section | (none; tool-agnostic within Claude Code model) | "Windsurf-Specific Notes" (`WINDSURF.md:620-651`) describing Cascade capabilities + lack of EnterPlanMode |

**Source:** Docs-review agent's diff.

**EVIDENCE** — Both files include identical "Tech Stack Context" tables describing an "AI App Factory" stack:
| Layer | Stack listed |
|---|---|
| Frontend | Next.js 15, TypeScript, Tailwind, ShadCN, Zustand |
| Backend | FastAPI + Uvicorn (Python), Supabase |
| AI/Agents | Google ADK, Vertex AI, Gemini 2.5 Flash/Pro, LangGraph |

**Source:** `CLAUDE.md:611-613`, `WINDSURF.md:582-586`.

**CONTRADICTED** — This stack does NOT match the present codebase. Verification:
- No `package.json`, no `next.config.js`, no React/Tailwind anywhere — **CONTRADICTED** Frontend.
- No `FastAPI` import anywhere; no `uvicorn`; no Supabase client. App backend is on App Engine (per `config.ini.template:13`) and the desktop app uses `requests` against it — **CONTRADICTED** Backend.
- No `google-cloud-aiplatform`, `vertexai`, `langgraph`, or any LLM/agent framework dependency in `requirements.txt`. The app has no LLM features — **CONTRADICTED** AI/Agents.

**Source:** Direct read of `requirements.txt` (no Next.js, FastAPI, or LLM deps); listing of repo root (no Next.js scaffolding).

**INFERENCE** — These files appear to belong to a multi-project "Stark Industries / AI App Factory" working environment whose canonical stack docs got copied verbatim into a repo whose actual stack is something different. The general workflow rules (Plan Mode, session files, scope discipline) still apply, but the tech-stack snippets are misleading for anyone using the file to understand this specific project.

#### A5. Purpose Confirmation

**EVIDENCE** — All three files self-describe as configuration for *coding agents that work on this repo* (CLAUDE.md:5, AGENTS.md:5, WINDSURF.md:5 read as "System prompt/rules for [Claude Code|Windsurf] agentic coding sessions"). They are NOT in-app personas, characters, or LLM system prompts consumed by the running application.
**Source:** Header text of each file.

### B. In-App Templated Text

**INFERENCE** — The application uses templated text in three places (none of which are LLM prompts):

1. **PDF Reports** — `helpers/pdf_helpers.py` uses `reportlab` (per `requirements.txt`) and `pypdf` for fillable PDF templates. Templates ship under `inclusion_lists/pdf_templates/` (per `docs/INCLUSION_LISTS_README.md`). These are static PDF forms with fillable fields, not narrative templates.
2. **Email bodies** — `helpers/email_helpers.py` (`EmailHelper`) sends SMTP. Per `requirements.txt`, `Jinja2` is a declared dependency — likely used for HTML email body templating (CLAIM — not code-level verified in this pass).
3. **Compliance / state-form pre-fill** — `pharmacybooks.py` defines `DEFAULT_ALDOI_EMAIL = "pbmcompliance@insurance.alabama.gov"` (`helpers/sqlite_db_helper.py:121`) as a hardcoded recipient for Alabama compliance reports.

**Source:** `requirements.txt` (Jinja2), `helpers/sqlite_db_helper.py:121` (direct), agent + docs.

**GAP** — A full enumeration of Jinja2 template files in the repo was not performed in this extraction. They likely live near `helpers/email_helpers.py` or in a `templates/` folder if one exists.

### C. User-Facing Static Strings (Non-Templated)

**EVIDENCE** — Hardcoded strings present in code (sample, not exhaustive):
- `User-Agent: 'PharmacyBooks-Desktop/1.0'` (`helpers/api_client.py:73`) — stale version
- `DEFAULT_ALDOI_EMAIL = "pbmcompliance@insurance.alabama.gov"` (`helpers/sqlite_db_helper.py:121`)
- `_DEFAULT_SECRET = "pharmacybooks-fallback-secret"` (`helpers/temp_password_crypto.py:18`) — see 07-GUARDRAILS
- Theme name `"flatly"` (`pharmacybooks.py:9937` per agent)
- Various dialog titles, error messages, "Coming soon" placeholders in `OpportunitiesDashboard`

**Source:** Direct reads.

### D. Session-Memory Mechanism for Coding Agents

**EVIDENCE** — Per `CLAUDE.md` and `WINDSURF.md`, a coding agent should maintain:
- `session_YYYY-MM-DD.md` at repo root (per-day session log with PENDING_APPROVAL → APPROVED → COMPLETE markers)
- `RECOVERY.md` at repo root ("Last action / Pending / Next step" 3-line state)

**Source:** `CLAUDE.md:201-280` region (per docs-review agent).

**EVIDENCE** — As of 2026-04-29, this extraction is being conducted under that protocol; `session_2026-04-29.md` exists at repo root, `RECOVERY.md` does not yet (will be created at completion).

**Source:** Filesystem listing.

---

## Open Questions

1. **Should `AGENTS.md` be deleted, or rewritten as a tool-agnostic agent file?** Currently it carries a misleading `.windsurfrules` header.
2. **Why does the tech-stack table in CLAUDE.md/WINDSURF.md not reflect this repo?** Is this repo expected to migrate to that stack (the future "web app" the User mentioned), or did factory-default text leak into a project where it doesn't belong?
3. **Are there Jinja2 templates anywhere?** `Jinja2` is in `requirements.txt` but no template files were enumerated in this pass.
4. **Does the app have any LLM-driven features at all?** The `requirements.txt` and code grep show none, but the agent-config files mention "Google ADK / Vertex AI / Gemini 2.5" — could there be a hidden integration?
5. **Is `pbmcompliance@insurance.alabama.gov` correct?** A hardcoded recipient address feels brittle — what's the path to update it without a code change?

---

## Verification Checklist

- [x] All findings labeled with evidence tags
- [x] All file references verified to exist
- [x] No invented function names or file paths
- [x] No synthesis or recommendations included
- [x] GAPs explicitly documented (Jinja2 template enumeration, LLM-feature audit)
- [x] Reframed correctly: not an in-app LLM persona; documented coding-agent config files + in-app templated-text surfaces instead
