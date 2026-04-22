# SCAINET Forge Changelog

> **Maintainers:** This file is copied to forge-releases CHANGELOG.md on every release (at the release tag). Update it **in the same PR as the version bump** so the in-app updater shows current notes. CI requires a top-level `## x.y.z` heading matching the repo-root **`VERSION`** file (see `npm run sync-version` in CONTRIBUTING.md).

## 5.27.11 (2026-04-22)

### Fix: nested Tokio runtime from EGO + `DbHandle::with_blocking` (S4)

**Runtime:** Tauri `async` commands, `tokio::spawn` agent tasks, and the orchestrator run on the async runtime’s worker threads. Calling **`EgoEngine`** (which uses `DbHandle::with_blocking` → `block_on` internally) from those paths caused **`Cannot start a runtime from within a runtime`**, failed DB reads/writes, and the EGO dashboard stalling on “loading identity.”

**Approach:** Prefer **`DbHandle::read` / `write` + `.await`** on async paths. Do not nest `block_on` inside an already running runtime.

#### Backend

- **`ego::build_identity_context_for_conn`** — Builds the same system-prompt EGO block as `EgoEngine::build_identity_context` on a raw `&Connection` (used from async `read`). `EgoEngine::build_identity_context` now delegates to this helper via `with_blocking_read` for sync call sites.
- **`main.rs` — `start_agent_session`** — Naming ceremony, session records, EGO complete/memory/stats/badges, post-session audit, and AEF config for the run use **`db_handle` async APIs**; **`load_config_from_conn`** is bridged with **`config::loader::string_to_rusqlite`** for `read`’s `rusqlite::Result` signature. **`run_agent_loop`** no longer receives **`Arc<Mutex<EgoEngine>>`**, only **`Arc<DbHandle>`**.
- **`agent/orchestrator/mod.rs` — `run_agent_loop`** — PNEUMA and Constitution **`log_action`** paths use **`db_handle.write().await`**. Tool success/failure EGO logging uses **`tokio::spawn` + `db_handle.write().await`** (with **`Option<String>`** for file paths to satisfy `Send + 'static` closures) instead of **`spawn_blocking` + `EgoEngine`**.
- **IPC** — Prior **`ipc/ego.rs`** and **`ipc/config.rs`** work remains on **`db_handle` async** (no `EgoEngine` from async Tauri commands).

#### Documentation

- **`docs/paul-working-docs/TECHNICAL_DEBT.md`** — Notes on (1) “0 iterations” in session summary vs. actual LLM turns, (2) repeated “Seeded 3 standard task(s)” INFO noise.

#### Testing

- **`cargo check`** (release-related validation during development).

## 5.27.10 (2026-04-22)

### SQLite access — audit on `DbHandle` (S4 T11)

Moves **`AuditStore`**, **`AuditEngine`**, and audit-related IPC to **`Arc<DbHandle>`** with async **`record`**, **`verify`**, **`events`**, **`export_session`**, and related export APIs. Startup uses **`AuditEngine::init`**. Orchestrator tool-result audit runs via **`tokio::spawn`** so async `record` is not called inside blocking pools; **`export_project_transcript`** loads audit before holding the EGO mutex so the IPC future stays **`Send`**. Arbiter tests use **`block_on(AuditEngine::test_engine())`** to avoid nested Tokio runtimes with **`LifecycleEngine::new_for_test`**.

#### Backend

- **`audit/`** — `AuditStore` / `export` / `transcript` on `DbHandle`; `#[cfg(test)]` **`test_engine()`** helper.
- **IPC** — `ipc/audit`, `ipc/lifecycle` audit calls, and related commands **`await`** audit APIs.
- **Agent** — Orchestrator audit **`await`**, with EGO + audit split where needed.
- **`main.rs`** — **`tauri::async_runtime::block_on(AuditEngine::init(db_handle))`** at startup.

#### Documentation

- **S4 §5** — `rg` §1.3 total **57** after T11; **`audit/`** app code at **0** lock-site matches in §1.3.

#### Testing

- Audit store/export/transcript and arbiter tests updated for async audit; **`cargo test`** green.

## 5.27.9 (2026-04-22)

### SQLite access — CATALYST, IPC, and orchestrator on `DbHandle` (S4 T9–T10)

Extends the **`DbHandle`** migration to the **CATALYST** stack, **CATALYST IPC**, **W5 / environment / version-control** commands that touch the shared DB, and **orchestrator** paths that assemble CATALYST context. Finishes **T10** by moving **`CatalystConfig`** to **`Arc<DbHandle>`** with an **async** instance API (no `self.db.lock()` in app code). Static `get_or_default_conn` / `set_conn` / `delete_conn` remain for use inside `read` / `write` closures.

#### Backend

- **IPC** — `ipc/catalyst`, `ipc/w5`, `ipc/environment`, and `ipc/version_control/project` use **`db_handle.read` / `write` + `await`** where appropriate instead of holding the legacy mutex connection across async boundaries.
- **Agent** — `orchestrator` updates for CATALYST-related context loading against **`DbHandle`**.
- **CATALYST** — **`ContextInjector`**, **`CatalystDispatcher`**, **`CircuitBreakers` / `HumanOverride`**, **`ClarificationExtractor`**, **`ProgramManager`**, **`PelPipeline`**, **`RefinementEngine`**, and **`test_utils`** (`catalyst_test_db_handle`, etc.) on **`Arc<DbHandle>`**; async methods and **`#[tokio::test]`** as needed.
- **`CatalystConfig`** — **`Arc<DbHandle>`**; async `get` / `get_or_default` / `set` / `delete` / `all_for_project`; `*_conn` helpers unchanged.
- **`main.rs`** — Wiring consistent with the above.

#### Documentation

- **S4 §5** — `rg` §1.3 lock count **62** after T10; **`catalyst/`** app code has **0** `self.db.lock()` matches; test fixture **`catalyst_test_db()`** (mutex) retained where sync tests still use it.

#### Testing

- CATALYST unit and integration tests updated for async **`DbHandle`**; config and cross-instance integration tests on **`DbHandle`**; added coverage for empty/miss paths, PEL, and profile resolution where applicable.

## 5.27.8 (2026-04-21)

### SQLite access — lifecycle on DbHandle only (S4 T5 closeout)

Completes the lifecycle slice of the S4 migration: `LifecycleEngine` no longer exposes `db()`; all lifecycle database work goes through `DbHandle` (`read` / `write` / `transaction` / `with_blocking` / `with_blocking_read`). IPC and other subsystems that still use synchronous `rusqlite` on the shared file now take `state.ego.db()` (legacy `Arc<Mutex<Connection>>`) until ego is migrated in a later phase.

#### Backend

- **`LifecycleEngine`** — `DbHandle`-only internals; `db_handle()` accessor; `new_for_test()` applies full EGO schema via `ego::db::apply_schema_and_migrations`.
- **`ego::db`** — `apply_schema_and_migrations(conn)` for shared schema + migrations (used by test setup and `initialize()`).
- **IPC** — `lifecycle`, `environment`, `w5`, and `version_control/*` updated to use `ego` for mutex DB access where appropriate; lifetime fixes (`db_arc` binding) where needed.
- **Agent** — Orchestrator W5 context via `db_handle.read().await`; Catalyst context via `ego.db()`; W5 agent tools use `db_handle.write().await`. Optimizer feedback still uses `ego.db().lock()` (expected until T14).
- **`DbHandle` tests** — File-backed round-trip + WAL persistence (`file_backed_write_read_persistence`); concurrent read-pool smoke (`concurrent_reads_do_not_block`).

#### Documentation & learnings

- **S4 §5** — T6+T7 spike closeout (S1 §11a.1 checklist), T8 agent/orchestrator notes.
- **Learning #34** — Updated: lifecycle is DbHandle-only; remaining mutex sites listed by area.

## 5.27.7 (2026-04-21)

### DbHandle — async SQLite wrapper (T1–T7 spike)

Introduces `DbHandle`, a new async SQLite access layer that replaces direct `Arc<Mutex<Connection>>` usage with structured read/write/transaction APIs, automatic tracing instrumentation, and graceful shutdown with WAL checkpoint.

#### New module: `src-tauri/src/db/`

- **`DbHandle`** — Single-owner async wrapper over `tokio-rusqlite`:
  - `read(closure)` — Read-only operations with automatic `db.read` tracing span
  - `write(closure)` — Exclusive write operations with `db.write` span
  - `transaction(closure)` — Multi-statement transactions with `db.transaction` span
  - `shutdown()` — Graceful close with WAL checkpoint and `db.shutdown` span
  - `with_blocking(closure)` — Deprecated shim for mechanical migration of existing sync code
  - `for_test()` — Isolated in-memory database for unit tests (unique URI per instance)
- **`DbError`** — Structured error enum: `Sqlite`, `Driver`, `ShuttingDown`, `ReadPoolTimeout`
- **Connection init** — `open_writer()` / `open_reader()` with PRAGMA settings (WAL, foreign keys, busy timeout)

#### App shell integration

- **`main.rs`** — `AppState` includes `db_handle: Arc<DbHandle>`; graceful shutdown on `RunEvent::ExitRequested` / `RunEvent::Exit` calls `db_handle.shutdown().await`
- **Startup** — `DbHandle::open()` with `block_on` for synchronous initialization before Tauri runs

#### LifecycleEngine hybrid compatibility

- **Backward compatible** — `LifecycleEngine` now holds both `db: Arc<Mutex<Connection>>` (legacy) and `db_handle: Arc<DbHandle>` (new)
- **`db()`** — Deprecated accessor for existing code; emits deprecation warning
- **`db_handle()`** — New async accessor for phased migration
- **`new_for_test()`** — Test constructor using legacy-only path with dummy `DbHandle`
- **Lock count** — 146 → 144 (lifecycle hybrid; remaining subsystems unchanged)

#### Documentation & learnings

- **S4 §5** — Execution findings updated with T7 spike results and hybrid approach notes
- **Learning #34** — Execution note added for `DbHandle` availability and migration status
- **ACTIONABLE_BUG_FIXES.md** — Documented unrelated log anomalies found during smoke testing (persona bootstrap, task seeding, doc watcher, portal 401)

#### Testing

- **Unit tests** — `db::handle::tests` (write/read roundtrip, transaction commits/rollbacks, test isolation)
- **Manual smoke** — macOS and Windows verified: normal lifecycle flow, graceful shutdown with `db.shutdown` span

#### Dependencies

- **`tokio-rusqlite = "0.5.1"`** — Async SQLite driver

## 5.27.5 (2026-04-20)

### Lifecycle DB mutex discipline (follow-up)

- **Rust** — Module and `LifecycleEngine` rustdoc: `std::sync::Mutex` around SQLite is not reentrant; do not call `project_status` / similar while holding `db().lock()` on the same engine; avoid slow I/O or `.await` under the guard.
- **W5 agent tools** — `w5_record_decision`, `w5_present_decision`, and `w5_generate_handover` call `project_status` **before** `db().lock()` inside `spawn_blocking`, fixing a same-thread deadlock class (see RCA).
- **Tests** — `project_status_then_db_lock_same_thread_succeeds` regression on `LifecycleEngine`.
- **Docs** — S0–S4 lifecycle lock-discipline working docs and `ACTIONABLE_IMPROVEMENTS` updated; `PROJECT_LEARNINGS` #34 expanded (`&Connection` helper note, execution summary).

## 5.27.4 (2026-04-19)

### Agent orchestrator modularization

- **Layout** — `agent/orchestrator` is a directory module: `mod.rs` facade plus `display.rs`, `escalation.rs`, `events.rs`, `hints.rs`, and `loop_run/` (`mod.rs` + `stream.rs`). Public `AgentOrchestrator` API and Tauri `agent:*` event names are unchanged (move-only refactor).
- **Extracted helpers** — Display/preview (`strip_internal_tags`, truncation, file meta, tool arg summaries), model escalation (`try_escalate`), serializable event DTOs, lifecycle/PE hints, loop helpers (system prompt assembly, stream buffer + tag stripping, task gates, tool↔LLM mapping, lifecycle tool UI emits and LLM notices, `dispatch_llm_stream_event`).
- **Tests** — 65 focused unit tests under `agent::orchestrator` (was a smaller set pre-split).
- **Docs** — S4 execution log and actionable-improvements entry for modularization milestone; `FURTHER_INVESTIGATION_REQUIRED` note on Project Documents vs filesystem (locked docs + task `linkedDocs`).

## 5.27.3 (2026-04-19)

### Integrated terminal

- **PTY ↔ xterm sizing** — After `fit()`, always resizes the backend PTY when `cols`/`rows` are valid (not only after `ready`), so wrapping and backspace match the visible grid.
- **Project root CWD** — `resolveTerminalCwd()` prefers `projectContextStore.workingDir`, falls back to `env_status.working_dir`, then `"."`. Vitest coverage in `terminalCwd.test.ts`.
- **Project switch (single tab)** — On `working_dir_changed`, if exactly one terminal tab is ready and the path changed, the tab is closed and recreated so the shell starts in the new directory without injecting a visible `cd`.
- **IPC `terminal_execute`** — One-shot commands use the orchestrator working directory when it exists on disk (`execute_command_in_dir`); Rust unit test for cwd on Unix and Windows.
- **PTY idle thread** — Shared shutdown flag and `Drop` on `PtyInstance` so the idle-detection loop exits when a tab is closed (avoids orphaned threads).
- **Frontend** — `Terminal.svelte` uses typed `@xterm/*` instances and constructors; clearer separation from dynamic import.

### Editor & workspace UI

- **Editor tabs** — Tab bar and editor store support for tab-style navigation alongside **CodeEditor** / **EditorPanel** wiring.
- **Project picker** — Freeform working-directory flow, folder picker helpers, and navigation back to the picker home.
- **`+page.svelte`** — Layout and integration updates for the above.

### Documentation

- Working docs reorganized under `completed/`, `partial-or-blocked/`, and `raw-research/`.
- New or updated S0 notes: agent orchestrator modularization, GitHub org/repo OAuth, settings BYOK phase B / N5 / phase D.

## 5.27.2 (2026-04-18)

File explorer **Git / version-control decorations**, **`.gitignore` integration**, and **merge-safe silent refresh** — plus everything from **5.27.1** below when comparing this branch to `main` (still at **5.27.0**).

### Critical fix — W5 orchestrator deadlock

- **Same-thread deadlock in W5 prompt injection** — The W5 Cognitive Checkpoints feature (`2cf93e6`) called `lifecycle_engine.project_status()` while already holding `lifecycle_engine.db().lock()`. Since `std::sync::Mutex` is non-reentrant, this caused the agent session to hang immediately after "Lifecycle mode active" with no LLM call ever made.
- **IPC lock contention** — `list_user_projects` and `get_project_state` held the DB lock during slow filesystem/git operations, extending contention. Now both release the lock immediately after the SQL query.
- See `docs/paul-working-docs/RCA-AGENT-LOOP-BLOCKING.md` for full root-cause analysis.

### File explorer — Git working tree

- **Porcelain v2** — IPC `get_git_working_tree_status` (`git status --porcelain=v2`), `gitDecorationStore`, debounced refresh on `forge:file-saved`, `agent:file_changed`, focus, terminal idle, and VC events.
- **Row styling** — Modified / staged / untracked badges and accent classes; **unsaved editor buffer** (`dirtyPathKeys`) shows distinct “unsaved” styling before disk matches Git.
- **Rust** — Parser fix for real-world modified lines (9-token porcelain lines); non-repo and spawn failures handled without breaking the tree load.

### File explorer — `.gitignore` and “show hidden”

- **IPC** `check_git_ignore_paths` (`git check-ignore -z --no-index --stdin`) for batch ignore checks — **`--no-index`** applies `.gitignore` rules even for **tracked** files (default `check-ignore` skips them), so muted styling and “hidden when gitignored” match editor expectations.
- **Default hidden** — Build/cache dir names (e.g. `node_modules`, `.git`, `target`) and junk filenames are hidden with the same toggle as dotfiles; **gitignored** paths default hidden; all can be shown with the eye control.
- **Muted rows** — Gitignored entries use `forge-explorer-gitignored` when visible.
- **`patchTreeGitIgnoreFlags`** — Immutable tree clone so Svelte updates ignore styling immediately after `.gitignore` saves; **immediate** reapply on `forge:file-saved` (plus short debounced full silent refresh).

### File explorer — merge snapshot (Phase 2)

- **`mergeFileTreeSnapshot`** — Merges structural `list_directory` snapshots into existing UI state (expanded folders, loaded children) so silent refresh does not reset the tree.
- **`refresh` race** — `try`/`finally` clears loading only for the latest `refreshSeq`; avoids stuck “Loading…” when overlapping refreshes occur (e.g. non–git repos).

### Tests & docs

- Vitest coverage for `fetchFileTreeSnapshot`, `mergeFileTreeSnapshot`, `gitDecorationUi`, path utilities, and related stores; working docs under `docs/paul-working-docs/` (S2–S4 file explorer merge / decorations).

## 5.27.1 (2026-04-17)

Extensible BYOK (bring-your-own-key) LLM providers, dynamic discovery, and OpenRouter — plus fixes so the model list and counts stay aligned when toggling providers on or off.

### Providers & discovery

- **Catalogue & validation** — IPC `get_llm_provider_catalogue` with stable provider ids; key validation per provider (e.g. Anthropic/Google/OpenAI/xAI list-models, Hugging Face `whoami`, OpenRouter models API). `save_api_keys` extended for OpenRouter (`OPENROUTER_API_KEY`), env status, `.env` migration/strip, and Stronghold wiring on the frontend.
- **Gemini & Hugging Face** — Rust providers and discovery paths integrated into the dynamic registry (`registry.rs`).
- **OpenRouter** — OpenAI-compatible chat (`openrouter.rs`), model discovery from `GET https://openrouter.ai/api/v1/models` (with sensible filters and cap), streaming with reasoning/tool-call handling, catalogue row + UI.
- **model_capabilities.json** — Bumped to **1.1.0** with expanded score entries for Gemini, OpenRouter-routed models, and Hugging Face curated ids; `_meta.sources` for maintainers.

### Settings UI

- **SettingsDialog** — Unified Providers section: add/remove catalogue providers, API keys via Stronghold, Ollama URL, enabled toggles, save flow tied to `save_api_keys` + discovery.
- **ApiKeySetup** — Help link, accessibility, discovery messaging updates.
- **TitleBar** — Provider health chip with popover, OpenRouter in status list, **FolderInput** icon for switch project, Escape/outside-click behavior.

### Model list & prefs (fixes)

- **`buildSaveApiKeysParamsFromStore` / `hydrateRustFromStronghold`** — Single source for which keys hit process env; toggling a provider re-invokes discovery and refreshes the registry.
- **`syncLlmEnvAndRegistry`** — Used after provider/Ollama toggles, remove-provider, and Models panel refresh.
- **`filterModelsByLlmPrefs`** — UI lists and title-bar counts respect `llmProviders.enabled` and `ollamaEnabled` (incl. `grok` → `xai` mapping); reactive `displayModels` in **ModelPanel** replaces `{#each filteredModels()}` for reliable updates.
- **`reconcileActiveModelAfterRegistryChange`** — Reconciles active model against prefs-filtered models when appropriate.

### Tests & docs

- Unit tests for settings helpers and catalogue selection utilities; lifecycle test alignment.
- Working docs updates under `docs/paul-working-docs/` (CONFIGURABLE_SETTINGS, S2–S4 extensible LLM providers).

## 5.27.0 (2026-04-12)

PROJ-W5COGNITIVE: W5 Cognitive Checkpoints — a decisions-first architecture for structured, auditable reasoning. This is the foundation for on-chain decision attestation and cross-session context persistence.

### W5 Decision Records

Structured, individually hashable records that capture every decision an agent makes or presents to a human. Each record contains the question, all options with rationale, the selected option, rejection reasons for alternatives, who decided (human or agent), confidence score, and a SHA-256 attestation hash.

Three decision severity tiers:

- **Strategic** — Human-decided, individually chain-attestable (architecture, business model, scope)
- **Tactical** — Logged, batch-attestable (implementation approach, library choice)
- **Micro** — Log-only (variable names, formatting decisions)

### Inline Decision Cards

When an agent needs a human decision, it calls `w5_present_decision` which renders an interactive card directly in the chat stream. The card shows the question, all options with rationale, the agent's recommendation and confidence. The user clicks **Select**, **Defer**, or **Discuss** — the choice is recorded as a W5Decision with `decided_by_type = human`.

New Svelte component: `DecisionCard.svelte` with severity-aware styling (amber/strategic, blue/tactical, slate/micro), reactive status tracking, and full IPC integration via `w5_decide`, `w5_defer`, `w5_discuss` Tauri commands.

### W5 Agent Tools (3 new tools)

| Tool                   | Purpose                                                                    |
| ---------------------- | -------------------------------------------------------------------------- |
| `w5_record_decision`   | Agent records its own autonomous decision with full structured audit trail |
| `w5_present_decision`  | Agent presents a decision to the user as an interactive inline card        |
| `w5_generate_handover` | Auto-generates a structured handover document from W5 context + decisions  |

All three tools are registered in the tool executor with full JSON Schema definitions, routed through the lifecycle engine's database connection, and emit Tauri events where needed.

### PNEUMA Decision Detection (6 new patterns)

Extended `pneuma/scanner.rs` with decision-language detection patterns that flag when agents make decisions without using the structured W5 tools:

- `decision_recommendation` — "I recommend", "I suggest we"
- `decision_selection` — "let's go with", "I'm choosing"
- `decision_rejection` — "I'm ruling out", "not suitable"
- `decision_tradeoff` — "the trade-off is", "pros and cons"
- `decision_assumption` — "I'll assume", "defaulting to"
- `decision_question_posed` — "which one do you think", "your call"

Each pattern produces a soft flag guiding agents toward structured capture. `IssueSeverity::Low` penalty adjusted from 3.0 to 2.0 to reflect the advisory nature.

### W5 Context Snapshots & System Prompt Injection

Context snapshots capture the enriched W5 dimensions (Who + stakeholders, What + outcomes, Where + research provenance, When + stage timing, Why + mission statement). The latest snapshot + recent decisions are injected into the agent's system prompt between the lifecycle and PNEUMA sections, giving every agent immediate awareness of the project's cognitive state.

### Database Migration V19

New tables `w5_decisions` and `w5_context_snapshots` in the lifecycle database, with full schema for all decision and context fields including hash storage and chain_tx placeholder for future on-chain attestation.

### W5 Design Document

Comprehensive architecture document (`strategy/W5_DESIGN.md`) capturing all 8 locked design decisions, detailed schemas, integration points, build order, and verification criteria. Created collaboratively during a structured design discussion before any code was written.

### PROJ-TOOLENHANCE Gap Analysis

Full audit of FORGE's tool inventory (85 existing tools) against strategic documents and patents. Identified 3 remaining gaps and proposed 51 new tools across 8 categories. Output captured in `strategy/TOOLENHANCE_GAP_ANALYSIS.md` and used to create a new S0 in the lifecycle database via VOLTAIC-18.

### Housekeeping

- **`.gitattributes`** — Added to normalize line endings (LF in repo, native on checkout), eliminating persistent CRLF ghost diffs on Windows
- **`AppHandle` wiring** — Tool executor now receives the Tauri app handle for event emission, enabling tools to communicate with the frontend
- **`W5_FORGE_COMPLETION_PROMPT.md`** — FORGE agent prompt for progressing PROJ-W5COGNITIVE through remaining lifecycle stages to project completion

### Fixes

- **`w5Store` logger calls** — Fixed TypeScript type errors: `logger.info`/`logger.error` require `(tag, message)` format, not single-string; removed unused `get` import from svelte/store

### Test Coverage

869 tests passing, 0 failures, 0 regressions. New tests cover W5 decision CRUD, SHA-256 hashing, context snapshot generation, all 6 PNEUMA decision-language patterns, and score calculation adjustments.

## 5.26.0 (2026-04-12)

PROJ-PROMPTAUDIT deliverables. This release ships the findings from a full S0–S7 lifecycle audit of FORGE's prompt injection system — the first time a FORGE project was used to audit FORGE itself. The audit identified 8 findings (AF-001–AF-008), 16 Prompt Effectiveness Ledger entries (PEL-001–PEL-016), and produced 2 new features, 1 critical bug fix, and 5 new Rust modules.

### Feature 8: Stage-Aware Agent Posture Model (PEL-015 fix)

The highest-impact change from the audit. Replaces the static lifecycle Rule #1 ("draw it out gradually") in `lifecycle/mod.rs` — which caused agents to default to an interviewer posture at every stage — with 7 context-sensitive postures:

- **S0 (Listener):** Capture the user's vision faithfully, one question at a time
- **S1 (Analyst):** Evaluate feasibility with evidence, don't defer analysis to the user
- **S2 (Co-Designer):** Propose features with rationale and traceability to S0
- **S3 (Technical Lead):** Present trade-offs with recommendations, lead the technical conversation
- **S4 (Planner):** Decompose the HLP into precise, verifiable steps with PRESERVE blocks
- **S5–S7 (Builder):** Execute the plan, diagnose issues, report progress — don't request direction
- **Default:** Single-question conversational fallback

Also adds explicit **stage boundary rules** (Rule 2) that constrain each stage's scope — S0 captures ideas only, S3 does architecture only, S5 builds only, etc. — preventing agents from leaking work across stage boundaries.

### Feature 10: Conditional Prompt Inclusion

Reduces token cost on repeat messages by making large prompt sections conditional on `is_first_message`:

- **First message:** Full tool documentation (~3,000 tokens), complete project structure (~500 tokens), FORGE capabilities overview (~400 tokens), and work management details are included
- **Subsequent messages:** Concise one-line summaries replace these sections, saving ~3,900 tokens per message

Wired through `system_prompt::build_system_prompt(is_first_message)` → `orchestrator.rs` derives state from `self.conversation.is_none()`.

### Fix: Gateless Stage Advancement (AF-007)

S5 has no gate in the backend (by design — it's a build stage), but the UI assumed every stage required `lifecycle_approve_gate` before advancement. This caused the "Approve & Advance" button to fail silently for S5, blocking project progression.

Three changes:

1. **`lifecycleTokens.ts`:** `GATE_IDS` correctly omits S5 (G5 is the S6→S7 gate, not S5→S6)
2. **`ProjectDetailView.svelte` `onComplete`:** Detects gateless stages via `GATE_IDS[current_stage]` and calls `lifecycle_advance` directly, skipping `lifecycle_approve_gate`
3. **`ProjectDetailView.svelte` UI:** New `needsGatelessAdvance` computed property renders a "Complete & Advance" button for stages without gates, and the `isAlreadyApproved` error handler now precisely detects "already approved" states instead of treating all errors as approvals

### CATALYST Module: Phase 3+4 — Prompt Quality Data Layer

Five new Rust modules in `src-tauri/src/catalyst/` plus three data files in `src-tauri/data/`:

- **`registry.rs` + `patterns.toml`:** Pattern Registry — 16 prompt injection quality patterns with PEL cross-references, injection point mapping, per-stage filtering, and confidence weights. Loaded via `include_str!` with TOML schema validation.
- **`pipeline.rs`:** PEL Pipeline — reads raw PEL entries from `ego.db`, categorises them by injection point, computes weights from severity × frequency, and exports structured TOML for downstream consumption.
- **`token_budget.rs`:** Ceiling Manager — per-slot and global token budgets with `truncate_to_ceiling` (per-section) and `reconcile` (cross-section budget balancing that drops low-priority sections when overbudget).
- **`quality.rs` + `quality_rubric.md`:** Quality Harness — PEL regression testing (checks for mitigation markers), token compliance checks against CeilingManager, structural validation, and composite scoring. Rubric defines objective (60%) and subjective (40%) criteria with "expert-calibre = consistent with Simon's manual guidance" anchor.
- **`exemplars.rs` + `exemplars.toml`:** Exemplar Library — 12 gold-standard prompt injection templates across 4 injection points (system_prompt, v3_lifecycle, tool_docs, identity) with quality scores and PEL coverage references. `best_exemplar()` returns the highest-scoring template for a given injection point and stage.

**106 catalyst tests passing** (Phases 1–4 + integration). `toml = "0.8"` added to `Cargo.toml` dependencies.

### Audit Artifacts

- **Prompt Effectiveness Ledger:** 16 entries (PEL-001 through PEL-016) tracking every identified prompt injection gap, root cause, and fix status
- **Audit Findings:** 8 findings (AF-001 through AF-008) covering agent posture, knowledge injection, inter-agent referral, gateless advancement, S5 task types, ancillary document storage, and project-type awareness

## 5.25.2 (2026-04-14)

- **Document revision cycle**: New `DocumentRevision` task type for review-to-revision tasks in S2/S3/S4. Agents now receive explicit step-by-step instructions to read review findings, read current draft, apply changes, and present a summary of revisions in conversation.
- **Task-type-specific V3 prompt guidance**: The lifecycle prompt now injects behavioural instructions based on task type — `DocumentDraft` tasks instruct agents to present draft summaries in chat (PEL-014 fix); `DocumentRevision` tasks include the review document stage reference and a 5-step revision workflow.
- **Enhanced review approval injection**: `[REVIEW APPROVED]` instruction now includes the specific review document stage (e.g., `S2-DIAMOND`) so agents know exactly which document contains the findings.
- **PEL-014 logged**: Agent behaviour of saving drafts without presenting content in conversation now tracked and systemically addressed.

## 5.25.1 (2026-04-14)

- **Prompt injection delivery restored**: Gate approval, task completion, and review approval actions now send context-rich instructions to FORGE agents instead of content-free `[proceed]` tokens. Agents receive stage transition details, completed/next task info, and tool guidance (PEL-012, PEL-013).
- **New agent tools**: `lifecycle_get_draft` (read saved drafts), `lifecycle_get_document` (read locked stage documents), `lifecycle_reject_document` (delete locked doc for revision).
- **Document rejection flow**: Review modal "Request changes" and ActionCardModal rejection now call `lifecycle_reject_document` and instruct the agent to revise.
- **Fire-and-forget submitInstruction**: All UI-to-agent instruction calls are now non-blocking to prevent misleading errors when the underlying operation succeeded.
- **Prompt Effectiveness Ledger**: New tracking document (PEL-001 to PEL-013) for systematic agent communication analysis.

## 5.25.0 (2026-04-14)

This PR merges **`feat/version-control-abstraction-part-2`** into `main`, including version control / merge conflict work, documentation, and release **5.24.1**.

## 5.24.1 (2026-04-14)

- **Conflict resolution UI:** File-level block is context-only (region count, confidence, Preview defaults, disabled “Refine with AI” placeholder). Removed duplicate Apply from the top card; single Apply remains with per-region choices. Removed “Accept all suggested” from the hunk panel; “Restore defaults” appears only after the user changes a region. Planning doc updates in `S2-CONFLICT-UI-P1-P2.md`. Minor working-docs touch-ups for settings LLM provider notes.

## 5.24.1 (2026-04-14)

- Project ownership model

## 5.24.0 (2026-04-12)

- **Project Ownership Model**: Adds `project_members` table with role-based access (owner/editor/viewer), backfills existing projects on sign-in, and filters `list_user_projects` by user identity. Admins see all projects; regular users see only owned/membered projects. Prevents cross-user project visibility.
- **UX Flow Fixes**: Fixes the broken project selection flow — selecting a project from the picker now correctly opens the Projects tab with that project loaded, shows the correct project in the session bar, and starts a fresh agent conversation (no stale chat from previous sessions).
- **Freeform Mode**: Adds a "Just Start Building" option to the project picker so users can open FORGE without selecting a project — for experimentation, exploration, and non-project work.

## 5.23.1 (2026-04-12)

- Lifecycle personas ui refinements

## 5.23.0 (2026-04-12)

This release delivers the **version control abstraction** (Save / Publish / Ship), a **unified VC state machine**, **Git worktree workspace** support, **GitHub** publish/ship flows, **conflict and manifest** tooling, a major **IPC refactor** out of `main.rs`, the **project context store**, merged **lifecycle personas UI** work, and **Portal sync** hardening. It also bumps the app to **5.21.0** with changelog and synced semver across Tauri/npm/Cargo.
**Stats vs `main`:** ~173 files changed; branch includes 36 commits ahead of `main` (including merge of `feat/lifecycle-personas-ui-refinements`).

---

## 5.22.0 (2026-04-12)

Bug fixes, test reliability, UI layout improvements, and GitHub auth hardening.

- **Tier defaults:** All new projects now default to Standard tier instead of Enterprise. Existing Enterprise-defaulted projects migrated. Gold Review subtasks marked completed for Standard tier.
- **Document visibility:** Review & Approve button only appears when the document actually exists in the database, preventing phantom approval prompts.
- **Agent retry:** When `lifecycle_lock_document` or `lifecycle_save_draft` fails, the agent receives an explicit failure notice and retries automatically. User sees a clear error message instead of false success.
- **UI layout:** Stage description inlined into the lifecycle rail row (dynamic per selected stage). Chat area fills remaining screen space via flex layout. Tasks/docs sections capped at 25vh when chat is active. Removed duplicate "Approve Plan" CTA banner.
- **GitHub auth:** Detects organization OAuth third-party app restrictions and shows actionable warnings. User-friendly error message for DNS/network failures instead of raw reqwest errors.
- **Project type selector:** New Project flow now includes project type radio (Job / Standard / Enterprise).
- **Test fixes:** Windows path separators in `project_root` skip-dir checks. `file://` URL generation for workspace tests on Windows. Unique project IDs in conflict resolution tests to prevent parallel timestamp collisions.

## 5.21.0 (2026-04-09)

Version control abstraction (Save / Publish / Ship), unified VC state machine, workspace and Git worktree support, GitHub integration, conflict and manifest tooling, IPC modularization, project context store, lifecycle UI merge, and Portal sync hardening.

- **Version control UX:** User-facing Save, Publish, and Ship replace raw git operations; `vcStateMachine` drives button labels, loading, offline, CI, and conflict states with integration tests.
- **Workspace & worktrees:** Bare-repo worktrees, project initialization, migration paths, and New Project / Project Picker flows aligned with per-project branches.
- **Save flow:** Staging merge on save, commit intent from conversation context, manifest and discard paths, local change detection (explorer, terminal idle, window focus).
- **Publish & Ship:** GitHub API client; PR creation to `staging`; ship toward `main`; PR / CI status polling; cancel review and branch cleanup hooks; network checks before push-with-publish.
- **Conflicts:** Detection, resolution UI, merge completion, conflict brief generation; FSM returns to a sane state after resolution (including publish retry).
- **Manifest:** Drift detection, nested project roots, removal detection, manifest sync and CI template awareness where wired.
- **IPC refactor:** Large extraction from `main.rs` into domain modules (version control, lifecycle, jobs, terminal, sync, personas, etc.) for maintainability.
- **Project context store:** Shared reactive project context for UI and agent surfaces; related S1/S2 working docs.
- **Lifecycle UI (merged):** Human-confirmed task progression with inline proceed controls; lifecycle list sync fixes; review-before-gate behavior and tier-aware seeding refinements as integrated from `feat/lifecycle-personas-ui-refinements`.
- **Portal & sync:** Wait for Firebase auth before Portal pull; push ID token into Rust for Bearer calls; refresh token before `lifecycle_sync_portal`; Portal-only projects (`is_worktree` off) with **Link folder** and `link_folder_to_project`.
- **Save button:** `no_changes` from `save_project` now emits `SAVE_SUCCESS` so the UI does not stay stuck on “Saving…”.
- **Publish guard:** Invalidate network cache before online check; more resilient connectivity probing; logging around unpushed commits before publish.
- **CI:** Path filters on workflows to cut unrelated runs.
- **Docs:** Version control abstraction pipeline (S1–S4) and project context store triage/F&F; assorted planning and investigation notes.

## 5.20.3 (2026-03-31)

EGO database migration fix, Firebase emulator reliability, documentation for data-sync testing, and version-control planning notes.

- **Fix:** EGO migration `routing_decisions_extended` (v3_017) no longer runs duplicate `ALTER TABLE` statements. The extended columns already exist in the base schema; rerunning the migration on a fresh database caused `duplicate column name: model_changed` and a startup panic.
- **Firebase:** Import `$lib/firebase` from `+layout.ts` so emulator connection runs early in the client lifecycle (fixes Auth emulator sign-in when components loaded `firebase` late).
- **Firebase:** Dev-mode sign-in errors suggest enabling `VITE_USE_EMULATORS=true` when using seeded Auth emulator accounts.
- **Docs:** `FURTHER_INVESTIGATION_REQUIRED.md` — new section on Forge data sync inconsistency (Firestore vs Portal API vs local `ego.db`) and impact on local testing.
- **Docs (working):** `S0-FILE-EXPLORER-GIT-STATUS-OVERLAY.md` (S0 idea for explorer git status overlay). Version-control abstraction pipeline: `S1-VERSION-CONTROL-ABSTRACTION.md` through `S4-VERSION-CONTROL-ABSTRACTION.md` (triage through detailed action plan for a unified VCS layer in Forge).

## 5.20.2 (2026-03-30)

File Explorer polish, editor reveal, MCP protocol tool counts, and security documentation.

- **File Explorer:** VS Code–style file-type icons (CDN SVGs), chevron-only folders with Lucide `ChevronRight` rotation, unified muted label color for files and folders.
- **Editor reveal:** Opening a file from the explorer dispatches `forge:reveal-editor` so the right panel switches to the editor tab.
- **MCP:** Protocol store exposes per-server tool counts; TitleBar and MCP UI show tool numbers where relevant.
- **Docs:** S0 note `FORGE-FIRESTORE-RULES-TENANT-ISOLATION-S0.md` and related updates to `FURTHER_INVESTIGATION_REQUIRED.md` and `AUDIT_TRAIL_AND_LIFECYCLE_FLOW.md`.
- **Tests:** `fileIcons.test.ts` for icon URL and mapping behavior.

## 5.20.1 (2026-03-30)

- deps(npm): Bump the production-deps group with 8 updates

## 5.20.0 (2026-03-29)

Per-project working directory and automatic document-to-disk on lock.

- **Document-to-disk:** When a lifecycle document is locked (immutable in SQLite), a markdown copy is now written to `{working_dir}/docs/{stage-folder}/{STAGE}-{PROJECT_ID}.md`. Disk write is best-effort — failure does not block the authoritative SQLite lock or Portal sync.
- **Per-project working directory:** `lifecycle_projects` table gains a `working_dir TEXT` column (migration v18). Stored at project creation time and used by `lock_document()` to resolve the output path.
- **Stage folder mapping:** `Stage::folder_name()` maps each stage to its docs subdirectory (e.g., S2 → `features`, S3 → `high-level-plan`).
- **Caller wiring:** Both the IPC `lifecycle_create_project` command and the agent `lifecycle_create` tool now pass the working directory to the engine.
- **Tests:** 9 new tests covering folder mapping, file writing, path creation, error handling, and graceful degradation. 535 total passing.

## 5.19.0 (2026-03-29)

Lifecycle stage personas, Mode B tool enforcement, shared engine wiring, and UI/UX refinements.

- **Lifecycle Personas:** 8 stage-specific personas (SPARK through HERALD, S0-S7) with tailored system prompts, mandates, and per-stage DelegationPolicy tool permissions. Bootstrap upserts on startup keep SQLite in sync with source definitions. Persona descriptions use "operating persona" framing.
- **Mode B DelegationPolicy Enforcement:** When a lifecycle persona is active and mode is not Conversational (Mode A), the persona's tool restrictions are now enforced at runtime. Mode A remains fully unrestricted. Policy is cleared after each agent loop.
- **Shared LifecycleEngine:** All 6 lifecycle tool methods now use the shared AppState engine instead of creating a new DB connection and running full schema init per call. Fixes `lifecycle_status` timeout caused by SQLite lock contention.
- **Per-Project Conversations:** Multi-session conversation store with archive and new-session support. Conversation header shows active persona name and role description.
- **System Prompt Context:** Agents are told they are "operating under" a persona (not "You are"). Greeting directives reference the active project and stage, preventing generic responses.
- **UI/UX:** Terminal and Agent Stream panels minimizable to session bar buttons; card view removed from Projects tab; duplicate gate coaching card removed; session bar controls centered.
- **Housekeeping:** Rust warning cleanup, unused export removal, gitignore for test artifacts, deprecated `process_engine` standalone functions.
- **Fix:** `sync-version.mjs` Cargo.lock regex now handles CRLF line endings.

## 5.18.0 (2026-03-28)

Extended router decision logging and UI polish.

- **Router decision logging:** Extended `routing_decisions` table with 9 new fields for comprehensive model selection analysis — tracks stickiness behavior, override reasons, detected capability dimensions, conversation depth, vision requirements, and fallback chains.
- **Every decision logged:** Now records all routing decisions (not just model changes) for complete optimizer training data.
- **Stickiness observability:** New `evaluate_stickiness()` returns override reasons (`tier_upgrade`, `vision_required`, `context_capacity`, `low_confidence`, `shallow_conversation`, `model_unavailable`).
- **Dimension detection:** Router logs which capability dimension triggered model selection (`code`, `reasoning`, `perception`, `creativity`, `generation`).
- **Bug fix:** Firestore undefined error in action-cards.ts — omit `previousCardHash` field when absent instead of passing `undefined`.
- **UI:** Added Beta pill to TitleBar and ProjectPicker.
- **Docs:** Updated AUDIT_TRAIL_AND_LIFECYCLE_FLOW.md with §1d Routing Decisions; cleaned up completed items from ACTIONABLE_BUG_FIXES.md; added terminal workspace opportunities doc.

## 5.17.0 (2026-03-28)

Agent reliability and audit trail improvements.

- **Empty response guard:** Detect and surface silent LLM failures (empty content with EndTurn) instead of showing misleading "Task complete" messages.
- **Router stickiness:** Prevent unnecessary model bouncing mid-conversation by preferring the current model when confidence is moderate and no tier upgrade is needed.
- **Vision routing:** Router now checks for pending images and overrides stickiness to select vision-capable models when needed.
- **Audit trail:** Added model_id and project_id to all orchestrator audit events (SessionStart, ToolCall, ToolResult, FileChange, EscalationRequest, SessionEnd) for training data and cross-chain correlation.
- **Bug fix:** tool_metrics.session_id was in schema but never populated - now correctly recorded for per-session performance analysis.
- **Grok diagnostics:** Added logging when Grok API returns empty content with no tool calls.
- **Docs:** Comprehensive update to AUDIT_TRAIL_AND_LIFECYCLE_FLOW.md with gap analysis; new S2 for router decision logging.

## 5.16.1 (2026-03-28)

- Lifecycle personas ui refinements

## 5.16.0 (2026-03-28)

Test coverage expansion and MCP configuration fixes.

- **Test coverage:** +34 Rust tests for ToolExecutor, PNEUMA scanner, sync pull, and lifecycle operations. Total Rust tests now ~500.
- **MCP import fix:** Auto-split command strings when no args provided (supports Tavily documentation format).
- **MCP test fix:** Prevent test from polluting production MCP config file.
- **Docs:** Updated TEST_COVERAGE_AUDIT.md with P1 completion status; added startup log issues to ACTIONABLE_BUG_FIXES.md.

## 5.15.0 (2026-03-28)

Implements full MCP Streamable HTTP transport support (2025-03-26 spec) and enhanced server configuration management, enabling direct connection to modern MCP servers like Tavily without requiring the `mcp-remote` stdio bridge.

## 5.14.0 (2026-03-25)

Complete lifecycle project flow overhaul for FORGE — delivering a seamless, intuitive experience for users from project creation through stage approval and advancement.

## 5.13.10 (2026-03-25)

- **Docs (S0):** S7-DEPLOYGATE-S0.md — exit/portability (Vercel, GitHub, Firebase/GCP), "automate in their accounts," OAuth token lifecycle, GitHub App orchestration, novice UX, database/Sentry notes, enterprise BYO / IdP / connection modes; open questions Q11–Q13; Phase 1-ALT framing.
- **Project picker:** Replace `catch (err: any)` with `unknown` + `formatTauriError`; log GitHub/clone/init/folder errors via `$lib/logger` (`ProjectPicker` tag).

## 5.13.9 (2026-03-24)

- **CI:** `auto-version-bump.yml` and `build.yml` authenticate with **`actions/create-github-app-token`** using org secrets **`SCAINET_APP_ID`** / **`SCAINET_APP_PRIVATE_KEY`** (scoped to `scainet-forge` / `forge-releases` as configured).
- **CI:** `.github/backup/` — PAT-based rollback copies of `auto-version-bump.yml` and `build.yml` (`VERSION_BUMP_TOKEN` path).
- **CI:** `.github/actions/setup-rust-with-retry` — local composite for `rustup` with retries (avoids flaky `static.rust-lang.org` and org allowlist issues with marketplace retry actions).
- **Docs / learnings:** `PROJECT_LEARNINGS.md`, `docs/FURTHER_INVESTIGATION_REQUIRED.md`, `docs/ACTIONABLE_BUG_FIXES.md`, `docs/ACTIONABLE_IMPROVEMENTS.md` updates.
- **UI:** `+page.svelte` and `LifecycleDashboard.svelte` adjustments (FREE tier / upgrade flow context).

## 5.13.8 (2026-03-23)

- **CI:** `build.yml` mirror job — authenticate to **`forge-releases`** with the **SCAINET** org GitHub App (`SCAINET_APP_ID`, `SCAINET_APP_PRIVATE_KEY`) instead of a long-lived `FORGE_RELEASES_TOKEN` PAT (git push to `main` + `gh release create` unchanged).
- **CI:** `github-app-forge-releases-probe.yml` — optional read-only check for the same app credentials (`workflow_dispatch` and/or branch push).

## 5.13.7 (2026-03-22)

- **File Explorer:** Phase 1 fix for "Loading…" blanking the tree during agent file activity — `refresh({ silent: true })` on `agent:file_changed`, sequence guard for overlapping refreshes, header ⟳ badge while updating, full loading only for explicit refresh / mount / `working_dir_changed`.
- **Refactor:** `src/lib/fileExplorer/fetchFileTreeSnapshot.ts` + `entriesToFileNodes` (shared with `loadDirectory`); Vitest coverage for staleness, races, empty/malformed IPC, and `rootPath` preserved when listing fails after `env_status`.
- **Docs:** ACTIONABLE_BUG_FIXES.md / ACTIONABLE_IMPROVEMENTS.md — Phase 1 marked complete; Phase 2 (merge / payload) and Phase 3 (store + fs notify) captured.

## 5.13.6 (2026-03-22)

- **Docs (S0):** FORGE-ML-DATA-COLLECTION-S0.md — training-ready data collection for future fine-tuning (context summarisation, tool trajectories, preferences, consent tiers).
- **Docs (S0):** S7-DEPLOYGATE-S0.md — S7 DeployGate intake (deployment ownership models, Vercel-first web apps, provider-adapter contract).
- **Docs:** S6-DEVOPS-EVIDENCE-S0.md — revisions to DevOps evidence / Launch Pad S0.
- **CI:** auto-version-bump.yml — smarter post-merge changelog entries (PR Summary / title, duplicate-heading guard, fallback placeholder).

## 5.13.5 (2026-03-21)

- **Docs (S0):** FORGE-INCIDENT-INTAKE-S0.md — Forge-native incident intake and AI triage (phased path alongside Sentry).
- **Docs (S0):** S6-DEVOPS-EVIDENCE-S0.md — DevOps evidence for S6 Launch Pad (CI/build/artifact signals in Forge; ownership boundaries).

## 5.13.4 (2026-03-21)

- **Infra:** Upgrade to Node.js 22 LTS (Node 20 EOL is April 2026).
  - Updated `.nvmrc`, CI workflows, and `package.json` engines field.
  - See README for setup instructions (nvm/fnm/nvm-windows).

## 5.13.3 (2026-03-21)

- **Sentry:** Production-ready error tracking with release health and source maps.
  - Events tagged with `scainet-forge@{VERSION}` for release tracking.
  - Source maps uploaded in CI via `@sentry/vite-plugin` (readable stack traces in production).
  - Automatic session tracking for crash-free % per release.
  - Dev testing: `VITE_FORCE_SENTRY=1` env var + `window.__sentryTest()` helper.
- **CI:** Added `SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, `SENTRY_PROJECT` env vars for source map upload.

## 5.13.2 (2026-03-20)

- **CI:** Single source of truth for app version — repo-root `VERSION` file + `npm run sync-version`; CI verifies consistency and that CHANGELOG has a matching heading.
- **CI:** Changelog enforcement — build fails if `docs/CHANGELOG.md` lacks a `## x.y.z` heading for the current version.

## 5.13.1 (2026-03-20)

- **CONDUIT — PR #96 (project → Firestore, Portal visibility):** Rust lifecycle emits `project:created` and `project:advanced` Tauri events; the frontend `project-listener` writes to Firestore so Portal sees FORGE-created projects. `startProjectListener` / `startDocumentListener` wired in app layout. New IPC `lifecycle_backfill_firestore` to backfill existing local projects. Lifecycle Dashboard syncs archive/unarchive to Firestore and uses **Sync (backfill)** for alignment.
- **Note:** Document hash → Firestore (`lockDocument`) still respects `useDirectFirestore_documents`; project sync above is not gated by that flag.

## 5.13.0 (2026-03-20)

- **CONDUIT — PR #95 (direct Firestore path):** New `src/lib/firestore/*` service layer with tenant-scoped queries and dual-path `routed*` APIs (Firestore vs Portal) controlled by **`conduit-flags`** in localStorage (defaults keep Portal path for reads). Firestore **offline persistence** via `persistentLocalCache`. Doc watcher refactored to emit **Tauri events** (`document:hash_computed`) instead of Portal HTTP for hash propagation. Firestore connection health + migration metrics; **`ConduitMigrationPanel`** (dev UI for flags — mount where needed). Auth store tenant cache for scoped queries.

## 5.12.1 (2026-03-20)

- **Updater (macOS):** Clearer error when **Install & Restart** fails from a read-only location (e.g. app still on the `.dmg`); install to **Applications** and launch from there.
- **Releases:** Public changelog link fixed; `docs/CHANGELOG.md` is synced to forge-releases each release. Targeted URL rewrite in mirror job (no blind `sed` over `latest.json`).
- **Docs / process:** Learnings and CONTRIBUTING updated for changelog and release hygiene.

## Auto-Generated Entries (2026-02-24)

### Fix: Suppress PowerShell Window Popups on Windows

**Description:** Introduced a hidden_command() helper to set the CREATE_NO_WINDOW flag for agent tool process spawns, preventing console windows from flashing during agent sessions. This addresses issues like unwanted UI interruptions.

**Impact:** Improves user experience by making the agent interface more seamless and professional, reducing distractions in desktop operations, which could enhance productivity in agent development workflows.

**Commit Details:**

- Hash: 66b3b8a63cd4b782bcf0714e9019b7d2f36496cb
- Author: Simon <simoncase78@gmail.com>
- Date: 2026-02-23 14:04:15 +1030
