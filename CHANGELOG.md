# SCAINET Forge Changelog

> **Maintainers:** This file is copied to forge-releases CHANGELOG.md on every release (at the release tag). Update it **in the same PR as the version bump** so the in-app updater shows current notes. CI requires a top-level `## x.y.z` heading matching the repo-root **`VERSION`** file (see `npm run sync-version` in CONTRIBUTING.md).

## 6.18.0 (2026-05-28)

**Grok on xAI Responses API:** Upgrades every Grok text, streaming, and vision call from the legacy Chat Completions endpoint to xAI’s modern **`/v1/responses`** API. You get stronger live-web answers, optional X (Twitter) search and hosted code execution on Grok models, and a new **`grok-build-0.1`** option in the model registry — without changing how Anthropic, OpenAI, Gemini, or other providers work.

**Why:** Chat Completions is deprecated on xAI and cannot expose server-side tools (native `web_search`, `x_search`, `code_interpreter`). Forge previously ran a client-side DuckDuckGo `web_search` for all providers; on Grok that meant extra round-trips, weaker grounding, and no access to xAI-only capabilities. This release keeps multi-provider architecture intact while giving Grok users the API surface xAI actually ships today.

### Grok provider (`src-tauri/src/agent/providers/grok/`)

- **`responses/` translator:** Request shaping, completed-response parsing, and SSE streaming for `/v1/responses`; reasoning-effort whitelist pinned to live probe data (`scripts/probe_xai_responses_api.py`).
- **All Grok LLM paths migrated:** Non-streaming, streaming, and follow-up turns use the new endpoint; legacy `grok.rs` removed.
- **Vision:** `analyze_xai()` image understanding now posts to `/v1/responses` (same API key and transport as chat).

### Server-side tools (Grok only)

- **`web_search`:** On Grok, Forge emits xAI’s built-in web search instead of the local DuckDuckGo scraper — multi-step searches can complete in one model turn with better grounding.
- **`x_search`:** Search recent X (Twitter) posts (no Forge client equivalent).
- **`code_interpreter`:** Sandboxed Python on xAI’s infrastructure for quick calculations without touching your machine (distinct from local `terminal_execute`).
- **Orchestrator guard:** Persona policy is checked _before_ augmenting Grok tool lists so disallowed server-side tools are never offered.
- **Audit trail:** Provider-executed tool usage is recorded with `executed_by: provider` so session history shows what ran on xAI vs locally.

### Model registry & routing

- **`grok-build-0.1`:** New model entry with server-side tool support (live-probed 2026-05-28).
- **`_server_side_tools` metadata:** Capabilities JSON documents which tools each Grok model runs server-side; the router prefers these models for fresh-info / current-events style instructions.

### Developer probes

- **`scripts/probe_xai_responses_api.py`**, **`probe_xai_responses_shapes.py`**, **`probe_xai_websearch_behaviour.py`:** Optional live API shape checks for regression against xAI wire format changes.

## 6.17.0 (2026-05-28)

**Tool Registry Waves 2 & 3 (B-LC-02):** Extracts **14 lifecycle** and **15 job** agent tools from `mod.rs` into domain modules, bringing the registry to **45 registered tools**. Adds job workflow agent tools missing from Wave 1 (`job_lock_plan`, `job_drop_task`), fixes delegation task completion, and hardens job task mutations.

**Why:** Wave 1 (6.16.0) proved the Strangler Fig pattern; Waves 2–3 complete the lifecycle and tiered-work (job) clusters so voice/text agents can run full job plans without falling through to legacy `mod.rs` match arms. UAT on this branch surfaced plan-lock gaps, drop-task asymmetry, and delegated text agents omitting `job_complete_task` — all addressed here.

### Tool registry (`src-tauri/src/agent/tools/`)

- **`lifecycle_tools.rs` (Wave 2):** All 14 `lifecycle_*` execute implementations moved from `mod.rs` (~990 LOC) — list, search, create, status, advance, gates, drafts, documents, steps, tasks.
- **`job_tools.rs` (Wave 3):** All 15 `job_*` / `delegate_task` execute implementations moved from `mod.rs` (~1250 LOC).
- **`registry.rs`:** `REGISTERED_TOOL_NAMES` expanded 17 → 45; dispatch routes lifecycle and job modules before legacy fall-through.
- **`scripts/check-tool-registry.sh`:** Guard arrays and expected count updated for Waves 2–3.

### Job workflow (agent + engine)

- **`job_lock_plan`:** New agent tool — parses Quick Plan draft into job tasks and locks plan (fixes JOB-018 voice/text plan→tasks regression).
- **`job_drop_task`:** New agent tool — hard-delete parity with `daily_flow_drop_task` and IPC `job_drop_task` (replaces incorrect `job_complete_task` + `failed` workaround).
- **`append_job_task_async`:** Atomic sequence + task-id allocation under DB write lock (fixes parallel `job_add_task` race).
- **`resolve_job_engine()`:** AppState fallback when executor field unset (voice dispatch legacy path).
- **F-058 on `job_create`:** Auto-promotes new job into active work context so voice can reach Mode B tools.
- **F-084 guards:** `job_complete_task` / `job_drop_task` verify task lives in `job_tasks` before mutating.

### Delegation

- **`agent/delegation.rs`:** Delegation brief embeds machine-readable `task_id`; orchestrator auto-finalizes `in_progress` job tasks when text agent session ends without calling `job_complete_task`.
- **Delegation brief:** Explicit required final step calling `job_complete_task`.

### Tests

- Job tools: tenant scoping, `job_lock_plan`, `job_add_task`, `job_drop_task`, F-084 rejection, registry routing (RT-D matrix).
- Delegation: brief parsing, completion detection, drop-task exclusion from auto-complete.
- Lifecycle tools: tenant isolation, F-058 status defaults, sync-engine guards.
- Engine: concurrent `append_job_task_async` uniqueness test.

### Docs / debt

- **`TECHNICAL_DEBT.md`:** TD-P2-15 → Partial (`job_drop_task` shipped); new TD-P3-16/17/18 (active-context parity, event source tags, `mod.rs` LOC advisory).
- **`S0-AGENT-STREAM-CHANGED-FILES-SCOPING.md`:** Changed Files panel context-scoping intake.

## 6.16.0 (2026-05-28)

- Introduces **Strangler Fig registry façade** (`registry::try_execute_registered`) in `ToolExecutor::execute` — registered tools dispatch before the legacy match, unmigrated tools fall through unchanged.
- Extracts **Wave 1 domain modules** from `mod.rs`:
  - `daily_flow.rs` — 15 daily flow tools
  - `work_context_tool.rs` — `set_work_context` (full impl move)
  - `lifecycle_create` — registry delegate (body stays in `mod.rs` until Wave 2)
- Adds **`scripts/check-tool-registry.sh`** CI guard — prevents migrated impls/dispatch arms from reappearing in `mod.rs`; cross-syncs registered tool name constants.
- Wires guard into **`pr-quality-gate.yml`**.
- Adds regression tests (RT-B matrix) for registry routing, error propagation, and daily_flow/work_context behaviour.
- **Version 6.15.4** — CHANGELOG updated; `sync-version` propagated.

## 6.15.4 (2026-05-28)

**Tool Registry Scaffold (B-LC-02 Wave 1):** Introduces a Strangler Fig registry façade in `ToolExecutor::execute` and extracts the first domain clusters out of the 8,800+ line `mod.rs` god module — **15 daily_flow tools**, **`set_work_context`**, and **`lifecycle_create`** (delegate). Behaviour-preserving move-only refactor; voice and text dispatch paths unchanged.

**Why:** Every new tool previously required editing the monolithic `execute()` match in `mod.rs`, violating OCP and increasing regression risk. Wave 1 establishes the registry pattern, domain modules, and a CI guard so migrated tools cannot reappear in the god file. Prerequisites B-LC-04 (ToolContextCore) and B-LC-01 (domain events) were already merged in 6.15.3.

### Tool registry (`src-tauri/src/agent/tools/`)

- **`registry.rs`:** `try_execute_registered` — routes registered tool names after entitlements and `requested_by` patching, before legacy match; returns `None` for unmigrated tools.
- **`daily_flow.rs`:** All 15 `daily_flow_*` execute implementations moved from `mod.rs` (~500 LOC).
- **`work_context_tool.rs`:** Full `set_work_context` impl moved from `mod.rs`.
- **`lifecycle_create`:** Registry delegate to existing `run_lifecycle_create` (body stays in `mod.rs` until Wave 2).
- **`pub(crate)` accessors** on `ToolExecutor` for emit/db/context helpers used by domain modules.

### CI guard

- **`scripts/check-tool-registry.sh`:** Fails if migrated tool impls or execute dispatch arms reappear in `mod.rs`; cross-syncs `REGISTERED_TOOL_NAMES` with domain module constants.
- **`pr-quality-gate.yml`:** Tool registry guard step added.

### Tests

- Regression matrix RT-B01–B20 (registry routing, error propagation, daily_flow persistence, work_context validation).
- Existing tool parity tests preserved (`tool_context.rs`, baseline tool inventory).

### Docs

- **`S2-TOOL-REGISTRY-B-LC-02.md`:** Implementation-ready spec (v1.4).
- **`TECHNICAL_DEBT.md`:** TD-P2-24 — voice job ID case sensitivity (pre-existing; not a registry regression).

## 6.15.3 (2026-05-26)

**Sprint 2 maintainability (B-LC-04 + F-LC-02) + voice connect latency fix:** Centralises voice/agent tool wiring via **`ToolContextCore`**, consolidates frontend shell event subscriptions into **`ForgeEventRegistry`**, and fixes Mode B / daily-flow voice connect delays caused by unbounded audit history scans during hindsight fetch.

**Why:** Voice connect in project and daily-flow contexts blocked the UI for 10–15s while `voice_realtime_start` scanned the entire `audit_events` table and serially loaded origin voice sessions. Tool executor wiring in `voice/dispatch.rs` duplicated ~100 lines of deps assembly per call site. Frontend event listeners (`work_context`, `project:deleted`, `agent:file_changed`) were scattered across `agent.ts` and `+layout.svelte` with no single registration point — hard to test and easy to drift. This release fixes the hindsight hot path without changing export/IPC audit semantics.

### Voice connect / hindsight performance (`src-tauri/src/audit/`, `src-tauri/src/voice/hindsight.rs`)

- **Indexed audit queries:** `events_by_project`, `events_by_job`, and `events_by_day` now use `session_id` index + `json_extract(payload, …)` predicates instead of full-table scans; partial expression indexes added via `AuditStore::migrate()`.
- **Hindsight-specific capped reads:** `RowLimit` + `events_for_hindsight_*` APIs cap payload reads at 200 rows (render only needs ~40 user/assistant turns).
- **Parallel daily-flow bundle:** `day_loader` uses `tokio::join!` + `join_all` for current day, user lookup, and prior 2 days.
- **Batch origin-session enrichment:** Single `events_by_sessions_capped` IN-query replaces N serial `audit.events()` calls; max 8 origin sessions.
- **`assembly` module:** DRY merge/enrich/compile pipeline shared by project, job, and day hindsight fetchers.
- **Tests:** 19 store regression tests (legacy parity + edge cases) + 4 hindsight tests.

### Tool context builder (B-LC-04)

- **`src-tauri/src/agent/tools/tool_context.rs`:** `ToolContextCore` + `ToolDeps` builder — single source for voice and orchestrator tool executor wiring.
- **`src-tauri/src/voice/dispatch.rs`:** Migrated to `build_voice_executor()`; `VoiceToolCallOverlay` preserved.
- **`src-tauri/src/main.rs`:** `ToolContextCore::apply_to_orchestrator()` at startup.
- **CI:** `scripts/check-voice-dispatch-wiring.sh` guard.

### Forge Event Registry (F-LC-02)

- **`src/lib/events/forgeEventRegistry.ts`:** Priority-ordered, typed shell event registry with IoC options.
- **`shellEventHandlers.ts`:** Extracted delete-degrade handlers (avoids `agent.ts` circular imports).
- **`+layout.svelte`:** Registry init gates downstream listener setup.
- **`agent.ts`:** Migrated `agent:file_changed` to registry fan-out; removed inline delete listeners.
- **CI:** `scripts/check-forge-event-listeners.sh` + `forgeEventRegistry.test.ts` (180 lines).

### Voice realtime handshake (`src-tauri/src/voice/grok_realtime.rs`)

- **30s wall-clock timeout** on pre-session readiness handshake — prevents infinite "Connecting…" when xAI sends only pings.

### Docs

- **`S2-TOOL-CONTEXT-B-LC-04.md`**, **`S2-FORGE-EVENT-REGISTRY-F-LC-02.md`**, sprint planning docs updated.
- **`TECHNICAL_DEBT.md`:** TD-P2-21 voice connect latency partially addressed; TD-P2-22 backpressure UX remains open.

## 6.15.2 (2026-05-26)

**Fix — legacy jobs invisible after tenant isolation:** Work Hub job lists now include pre-tenant-isolation rows stored with default `tenant_id = 'scainet'`, matching the permissive legacy visibility used for projects.

**Why:** v6.14.0 tightened `list_jobs` to strict tenant equality. Jobs created before auth/tenant stamping remained in `ego.db` but were filtered out when the signed-in user's resolved tenant did not match `'scainet'`. Users reported all jobs "lost" after upgrade; data was present locally but hidden.

### Job listing (`src-tauri/src/lifecycle/jobs.rs`)

- **`list_jobs_async`:** Query now uses `(tenant_id = ?1 OR tenant_id = 'scainet') AND deleted_at IS NULL` so legacy default-tenant jobs remain visible to authenticated users.
- **Test:** `list_jobs_async_includes_legacy_scainet_tenant_rows` locks in the regression guard.

## 6.15.1 (2026-05-26)

**Lifecycle IPC Facade (F-LC-01) + project lifecycle UAT fixes:** Migrates lifecycle-domain UI from scattered direct `invoke()` calls to a typed **`lifecycleApi`** module (queries/commands separation). Closes UAT defects where gate approval locked panels on **Loading**, naming ceremony fired mid-flow, S0 task proceed UI was missing after AgentStream unification, and S0 task 3 received the wrong agent posture.

**Why:** Detail views (`ProjectDetailView`, `JobDetailView`, `DailyFlowDetail`, etc.) each owned their own Tauri IPC wiring — untestable, duplicate, and easy to drift from backend contracts. This release lands the F-LC-01 strangler migration for §4.2 consumers plus CI enforcement so new direct invokes cannot reappear. Lifecycle refresh paths now distinguish initial load from background event refresh so agent sessions are not reset during gate approval and task completion.

### Lifecycle IPC facade (`src/lib/lifecycle/ipc/`)

- **`lifecycleApi`:** Single entry — `queries` (reads) and `commands` (writes) with injectable **`LifecycleIpcTransport`** for unit tests.
- **Typed surfaces:** `types/project.ts`, `types/job.ts`, `types/dailyFlow.ts` mirroring Rust IPC contracts.
- **`surfaceRegistry.ts`:** Command/query inventory aligned with S2 F-LC-01 §2.1.
- **`lifecycleApi.test.ts`:** 230-line transport mock suite — verifies query/command delegation without Tauri runtime.

### Component migration (direct `invoke` → `lifecycleApi`)

Migrated §4.2 registry consumers:

- **`ProjectDetailView.svelte`**, **`JobDetailView.svelte`**, **`DailyFlowDetail.svelte`**, **`DailyFlow.svelte`**
- **`LifecycleDashboard.svelte`**, **`StageView.svelte`**, **`GateApproval.svelte`**, **`LifecycleStepList.svelte`**, **`LifecycleStepContext.svelte`**
- **`EscalationModal.svelte`**, **`NewProjectFlow.svelte`**, **`ProjectPickerWorkspace.svelte`**, **`LiveVoiceController.svelte`**

§2.4 deferred consumers also migrated where in scope: **`LaunchPad.svelte`**, **`DocumentViewerModal.svelte`**, **`NotificationBell.svelte`**, **`HubJobCreateForm.svelte`**, **`entityTypeRegistry.ts`**, **`loadHubSegment.ts`**, **`+page.svelte`**, **`ProjectConversation.svelte`**.

### Project lifecycle UAT fixes

- **`loadData({ silent: true, resetAgentSession: false })`:** All lifecycle event listeners, gate approval `onComplete`, advance stage, task complete, lock draft, and Ship/Publish wrappers — prevents full-panel loading overlay and spurious **`new_conversation`** naming ceremony after gate approval (~~TD-P1-06~~).
- **`projectDetailWorkspace.ts`:** `resetAgentSession` option gates `new_conversation` IPC; test coverage in **`projectDetailWorkspace.test.ts`**.
- **`LifecycleTaskProceedBar.svelte`:** New component — task proceed / complete affordances for unified **AgentStream** path after F-LC-12 **`ProjectConversation`** removal (~~TD-P2-18~~).
- **`lifecycle_posture_for_task` (`lifecycle/mod.rs`):** Task-type-aware agent posture — S0 task 3 (GateApproval) receives **GATE DOCUMENT POSTURE** instead of stage-default listener posture (~~TD-P2-19~~).

### CI, ESLint & guard scripts

- **`scripts/check-lifecycle-ipc-invoke.sh`:** Multiline-aware ripgrep guard for lifecycle-domain `invoke()` in UI files; `--all` enforces §4.2 + §2.4 deferred registry.
- **`.github/workflows/pr-quality-gate.yml`:** `check-lifecycle-ipc-invoke.sh --all` on smoke jobs.
- **`eslint.config.mjs`:** `no-restricted-imports` rule — lifecycle components must use `lifecycleApi`, not `@tauri-apps/api/core` invoke for lifecycle commands.

### Docs & learnings

- **`TECHNICAL_DEBT.md`:** Resolved ~~TD-P1-06~~, ~~TD-P2-18~~, ~~TD-P2-19~~; partial TD-P2-02/05; new open **TD-P2-14** (daily-flow navigate exit policy), **TD-P2-15** (job vs daily-flow agent tools).
- **`PROJECT_LEARNINGS.md`:** #48–#50 (loadData refresh policy, signal_navigate exit vs destination, job/daily tool surface); **Deprecated Patterns** table.
- **`S2-LIFECYCLE-IPC-FACADE-F-LC-01.md`:** Status sync for migration completion.

### Known follow-ups (not in this release)

- **TD-P2-14:** Agent `signal_navigate_ui` → `daily_flow_tab` from project/job detail may land on source list without `dayId`.
- **TD-P2-15:** Job context agent may default to `daily_flow_*` tools; no `job_drop_task` agent wrapper.
- **TD-P2-05 (partial):** Granular in-place task refresh vs full `loadData()` on `tasks:changed`.

## 6.15.0 (2026-05-26)

**Domain Event Emitter (B-LC-01) — canonical backend refresh signals:** Consolidates scattered Tauri emit logic for entity-scoped UI refresh into a single `domain::events` module. Closes the stale-UI class where agent and IPC mutations succeeded on the backend but detail views did not reload until leave/re-enter — especially daily-flow IPC paths that previously emitted nothing (except delegate).

**Why:** Emit calls for `tasks:changed`, `job:changed`, `projects:changed`, and `jobs:changed` were duplicated across `ipc/jobs.rs`, `ipc/lifecycle.rs`, `ipc/daily_flow.rs`, and `ToolExecutor` with inline JSON and no payload contract. Daily-flow IPC mutators had no parity with agent tools. This release centralises emission, locks payload shapes with snapshot tests, and enforces an allowlist in CI so new drift cannot reappear.

### Canonical emitter (`src-tauri/src/domain/events/`)

- **`emit_tasks_changed` / `emit_job_changed` / `emit_projects_changed` / `emit_jobs_changed`:** Single emit site per event name; failures logged via `tracing::warn!`.
- **`emit_*_opt` variants:** No-op when `AppHandle` is absent (unit tests, headless agent dispatch).
- **Typed payloads + builders:** `TasksChangedPayload`, `JobChangedPayload`, `ProjectsChangedPayload`, `JobsChangedPayload` with scoped builders (`tasks_changed_for_day`, `tasks_changed_for_job`, `tasks_changed_for_project`, `tasks_changed_wildcard_subtask`).
- **`source.rs` registry:** Canonical `source` string constants for IPC, agent (`voice:` prefix at call sites), and list-invalidation paths.
- **`lifecycle/events.rs` shim:** Re-exports list emitters for backward-compatible imports during strangler migration.

### IPC migration

- **`ipc/jobs.rs`:** All job task and metadata mutators delegate to `domain::events` with `source::*` constants (9 call sites).
- **`ipc/lifecycle.rs`:** Project task refresh emits use domain builders; wildcard `lifecycle_complete_step` preserved.
- **`ipc/daily_flow.rs`:** **12 mutating IPC commands** now emit `tasks:changed { dayId, source }` after successful DB writes (advance/revert phase, create/update/defer/drop/delegate/escalate tasks, save/lock plan, complete day, roll day). Shared `emit_daily_flow_tasks_changed` helper; emits placed before fire-and-forget sync.

### Agent tools (`ToolExecutor`)

- Private `emit_*_changed` helpers refactored to one-line delegators to `domain::events` `_opt` functions — preserves all existing call sites while routing through the canonical module.

### Frontend contract (`src/lib/domain/events.ts`)

- TypeScript interfaces mirroring Rust payloads; `DOMAIN_EVENTS` constants; `DAILY_FLOW_IPC_SOURCES` registry and `dailyFlowVoiceSource()` helper for F-LC-04 Entity refresh coordinator.

### Testing & CI

- **`domain_event_payload_contract.rs`:** 20 frozen JSON snapshot tests (SN-01–SN-07 variants, job/daily-flow source registries, structural invariants).
- **`domain/events/tests.rs`:** 20 unit/integration tests including mock-runtime delivery (I-01–I-04) and shim re-export verification.
- **`scripts/check-domain-event-emits.sh`:** Allowlist guard — only `src-tauri/src/domain/events` may `.emit()` the four domain events; multiline-aware ripgrep matching.
- **`.github/workflows/pr-quality-gate.yml`:** Inventory + contract test steps on smoke jobs.

### Docs & learnings

- **S2-DOMAIN-EVENT-EMITTER-B-LC-01.md** — implementation-ready F&F (v1.3).
- **S0 backend/frontend SOLID architecture reviews** — B-LC-01 / F-LC-04 context.
- **PROJECT_LEARNINGS.md** — domain event patterns and CI gate notes.

## 6.14.5 (2026-05-25)

**Navigation Coordinator — symmetric enter/exit shell policy (F-NC-01–05):** Closes the architecture gap where backend work-context updates (voice agent, signal navigate, lifecycle create) updated the SessionBar pill but left the Work panel on the wrong view. Introduces a single Navigation Coordinator that reacts to `forge:work_context_changed` and shared enter bundles for day/job/project drill-in, mirroring the existing F-OE-01 exit policy.

**Why:** Enter logic was scattered across 8+ handlers across three state planes (backend `ActiveWorkContext`, explorer cwd, shell drill-in locals). Exit was unified in F-OE-01; this release completes symmetric consolidation and closes **TD-P2-02** (agent daily-flow drill-in) and **TD-P2-06** (stale drill-in after hub delete).

### Navigation Coordinator (`workContextNavigation.ts`)

- **`bindNavigationCoordinator()`:** Single policy owner in `initAgentEventListeners` — subscribes to `forge:work_context_changed`, `job:deleted`, and `project:deleted`.
- **Enter policy (F-NC-02):** Reacts to `voice_tool`, `lifecycle_create`, and `signal_navigate` sources; invokes canonical enter bundles with `syncBackend: false` (backend already mutated).
- **Exit policy (F-OE-01):** Full-clear shell navigation for agent/human clears, signal list targets, and delete sources; respects `ipc_drill_in_release` (backend sync only, no shell teardown).
- **Delete-aware teardown (F-NC-05):** Tears down jobs/projects drill-in (including launch pad) when the actively viewed entity is deleted.
- **Idempotency:** Skips redundant enter when shell tab + selected ID already match.
- **Init race handling:** Queues enter payloads (max 10) when `+page.svelte` callbacks not yet registered; flushes on mount.
- **Payload validation:** Type guards for malformed `work_context_changed` and delete event payloads.
- **98 frontend unit tests** across enter bundles, signal state, and coordinator policies.

### Canonical enter bundles (`lifecycleEnterNavigation.ts`, F-NC-01)

- **`enterDailyFlowDay` / `enterJob` / `enterProject`:** Ordered shell steps — clear drill-in, set tab/selection, optional backend sync, lifecycle pill update, explorer cwd switch, app machine events.
- **`EnterOptions`:** `syncBackend` (prevents IPC feedback loops), `emitAppMachineEvent`, hub job agent priming, cwd stash before switch.
- **`+page.svelte`:** Human list clicks and hub handlers delegate to enter bundles; `onSignalNavigate` thinned to bundles + list-only local tab/clear.

### Signal navigate parity (F-NC-03)

- **`dayId`** in `SignalNavigateDetail`, `signal-navigation.ts` bridge, and Rust `signal_navigate_ui` / orchestrator paths.
- **`next_context_for_signal_navigate`:** Pure Rust resolver — day > job > project priority (CR-3); list targets clear all context.
- **`set_work_context` tool:** Accepts `day_id` with mutual-exclusivity validation (at most one of project/job/day).
- **Coordinator enter** enabled for `signal_navigate` source after `onSignalNavigate` migration.

### Backend work context (`work_context.rs`)

- **`WorkContextSource::IpcDrillInRelease`:** Drill-in back clears backend context without triggering shell list navigation or explorer teardown.
- **`next_context_for_signal_navigate`:** Shared resolver for voice dispatch and orchestrator signal paths.
- **Unit tests:** List-target clear, entity navigation, day context stamping, voice tool `day_id`.

### Lifecycle mode store (F-NC-04 SRP)

- **Shell navigation extracted** from `lifecycleModeStore.bindBackendEvents` — store now mirrors SessionBar pill IDs only.
- **`userSessionStarted` preserved** on work-context clear (SessionBar ✕, drill-in back, agent clear) so in-progress chat is not interrupted by Start Text/Voice splash.

### Project context

- **`clearProject({ syncBackend })`:** Optional Rust cwd reset on clear (F-OE-02 / INTRO-01 parity).

### Lifecycle events (`lifecycle/events.rs`)

- **`emit_projects_changed` / `emit_jobs_changed`:** Canonical Tauri emitters with `source` + entity ID payload for Work Hub list invalidation.

### Job / project delete integration

- **`job:deleted` / `project:deleted`:** Coordinator listens and tears down matching drill-in; complements existing work-context clear from delete handlers.

### Docs

- **S2-NAVIGATION-COORDINATOR-ARCHITECTURE.md** — full F&F spec (v1.3, NC-01–05 complete).
- **S0/S1 navigation review docs**, **OPERATOR-EXPERIENCE-FOLLOWUP-WORK.md**, **TECHNICAL_DEBT.md** triage updates.

### Manual UAT (pending)

- UAT-NC-07: Agent navigate to job (signal) → job detail + Let's Go state
- UAT-NC-08: Historical day drill-in via `daily_flow_tab` + `dayId`
- UAT-NC-10: Delete active job from hub while drilled in → jobs list
- UAT-NC-11: Voice open daily flow from Work Hub → IDE + detail

## 6.14.4 (2026-05-25)

- deps(cargo): bump the production-deps group in /src-tauri with 2 updates

## 6.14.3 (2026-05-25)

- deps(npm): bump the production-deps group with 5 updates

## 6.14.2 (2026-05-25)

**Data access Train 3 — `JobRepository` + `DailyFlowRepository` + delete lifecycle:** Completes the repository-layer migration for jobs and daily flow. All production mutating SQL on `forge_jobs` / job link tables and `day_*` tables now lives in canonical repo modules; engines and IPC are thin delegates. Active-project and active-job delete paths emit frontend events and clear work context without leaving stale SessionBar state.

**Why:** Train 2 centralised `lifecycle_projects`; Train 3 applies the same pattern to P4 (jobs) and P5 (daily flow), adds CI guards so sprawl cannot return, and closes [#218](https://github.com/scainet-enterprise/scainet-forge/issues/218) / [#219](https://github.com/scainet-enterprise/scainet-forge/issues/219) delete-degrade gaps for projects and jobs.

### JobRepository (`job_repo.rs`, `jobs.rs`)

- **`JobRepository`:** Canonical async repo on `DbHandle` — insert/update/delete, stage transitions, `link_project_on_conn`, `touch_last_opened`, task/draft helpers.
- **`JobEngine` refactor:** Business rules and audit emission stay in engine; **zero production DML** left in `jobs.rs`.
- **`job_deleted.rs`:** On delete, clears active work context when the deleted job was active; emits **`job:deleted`** / **`jobs:changed`** for UI refresh.
- **Agent promote:** `services_create_project_from_job` uses `link_project_on_conn` inside existing transactional promote path ([#220](https://github.com/scainet-enterprise/scainet-forge/issues/220)).
- **IPC:** `ipc/jobs.rs` delegates to `AppState.job_repo` / `job_engine`.
- **20+ unit tests** on repo insert, link, stage update, delete cascade shapes.

### DailyFlowRepository (`day_repo.rs`, `mod.rs`, `tasks.rs`)

- **`DailyFlowRepository`:** All `day_records` / `day_tasks` / dependency mutating SQL consolidated in `day_repo.rs`.
- **Engine/module refactor:** Phase orchestration and read helpers in `mod.rs` / `tasks.rs`; **zero production DML** outside repo.
- **IPC:** `ipc/daily_flow.rs` delegates to `AppState.daily_flow_repo`.
- **Unit tests** on day create, task CRUD, cascade delete, plan lock paths.

### Project delete degrade ([#218](https://github.com/scainet-enterprise/scainet-forge/issues/218))

- **`project_deleted.rs`:** When the active project is deleted (hub or Portal tombstone), clears work context and emits **`project:deleted`** so SessionBar / explorer do not reference a removed workspace.
- **Sync pull:** Tombstone path integrated with repo cascade delete (Train 2 parity preserved).

### Frontend event wiring

- **`agent.ts`:** Listeners for **`job:deleted`**, **`jobs:changed`**, **`project:deleted`** — refresh agent context and clear stale pills.
- **`ProjectPickerWorkspace.svelte`**, **`LifecycleDashboard.svelte`:** Hub list refresh on job/project lifecycle events.

### CI guards (Train 3)

- **`check-forge-jobs-sql.sh`** + allow-list (**1 path:** `job_repo.rs`) — enforced in `pr-quality-gate.yml`.
- **`check-daily-flow-sql.sh`** + allow-list (**2 paths:** `day_repo.rs`, `ego/db.rs`) — enforced in CI.
- **`check-lifecycle-sql-allowlist.txt`:** Trimmed sprawl entries; **3 paths** remain (`project_repo.rs`, `ego/db.rs`, `ipc/w5.rs`).

### Catalyst, voice, W5

- **Catalyst / agent tools:** Job and daily-flow dispatch paths use repo injection; reduced `engine.db().lock()` on lifecycle mutations.
- **Voice session:** Work-context hints aligned with repo-backed job list reads.

### Docs & UAT

- **S4 Train 3 DAP**, **DATA-ACCESS-INVENTORY** §3.3–§3.5, Work Surface feature-flag S0–S4 docs updated.
- **`TECHNICAL_DEBT.md`:** Job/Daily Flow UAT (A/B) + Project/context UAT (C-series) findings — no Train 3 regressions; C3 agent daily-flow UI drill-in logged as pre-existing gap.

## 6.14.1 (2026-05-24)

**Data access Train 2 — sync façade + agent promote paths:** Completes the `ProjectRepository` migration by moving all remaining production mutating SQL on `lifecycle_projects` out of sync and agent-tool modules. Portal purge/tombstone/upsert, recent-projects backfill, and job/idea promotion now delegate to canonical repo helpers with explicit transaction boundaries.

**Why:** Train 1 centralised IPC and engine paths but left sprawl in `sync/pull.rs`, `migrate_recent_projects.rs`, and agent promote tools — violating the DA-P0 guard goal and risking partial commits (project row without link). Train 2 makes `project_repo.rs` the single owner of lifecycle project mutations, shrinks the CI allow-list from 7 paths to 3, and aligns agent dispatch with `DbHandle` async read/write (no nested `db().lock()` deadlocks).

### Repository extensions (`project_repo.rs`)

- **`delete_cascade_on_conn` / `delete_cascade`:** Composed 21-table cascade delete (sync purge + user delete parity).
- **`upsert_from_portal_batch_on_conn` / `upsert_from_portal_batch`:** Portal batch upsert with frozen merge policy (`COALESCE` preserves local `working_dir`, empty-string `repo_url`, and null `tenant_id` on resync).
- **`insert_backfill_row_on_conn`:** Startup `recent-projects.json` backfill INSERT shape.
- **`insert_project_on_conn`:** Canonical INSERT for agent promote and engine create paths.
- **DTOs:** `PortalProjectEntry`, `BackfillProjectRow`, `DeleteCascadeResult`, `UpsertBatchResult`.
- **20+ new unit tests** for cascade delete, portal upsert edge cases, and backfill shape.

### Sync façade migration

- **`sync/pull.rs`:** Purge, tombstone enforcement, and portal upsert call `delete_cascade_on_conn` and `upsert_from_portal_batch_on_conn`; zero inline mutating SQL in production code.
- **`sync/migrate_recent_projects.rs`:** Production INSERT via `insert_backfill_row_on_conn`.
- **Tests extracted** to `pull_sync_test.rs` and `migrate_recent_projects_test.rs` (DA-SYNC guard exempt path).

### Agent promote paths (transactional)

- **`services_create_project_from_job`:** Single `db.write` transaction — project INSERT + `forge_job_project_links` + `forge_jobs.project_id` update; rolls back on link failure (#220).
- **`morpheus_auto_s0`:** Same transactional pattern — project INSERT + `forge_morpheus_idea_projects` link.
- **Dispatch (`agent/tools/mod.rs`):** Promote tools use `engine.db_handle().write/read` directly; `exists()` pre-check prevents duplicate project ids; W5 and services read paths migrated off `spawn_blocking` + mutex.

### Engine & catalyst

- **`lifecycle/mod.rs`:** `delete_project_async` delegates to `project_repo.delete_cascade()` (21-table parity with sync purge).
- **`catalyst/flows.rs`:** Transition lookup via `db_arc().read` (async-safe).

### CI guard

- **`check-lifecycle-sql-allowlist.txt`:** Reduced from 7 to 3 paths — `project_repo.rs`, `ego/db.rs`, `ipc/w5.rs` only.
- **`check-lifecycle-sql.sh`:** Passes clean on this branch.

### Docs

- **`DATA-ACCESS-INVENTORY.md`, S1/S2/S4 DAP docs:** Train 2 phases marked complete; production DELETE/mutating counts updated.

## 6.14.0 (2026-05-22)

**Project List Parity (PLP / WS-B) + sync integrity:** One authoritative project-list path via `ProjectRepository::list_projects` and shared SQL in `project_list.rs`, so the Work Hub picker, lifecycle IPC, agent tools, voice, and Arbiter no longer diverge on auth, filters, or tombstones. Fixes stale Work Hub search after archive, Arbiter IPC deadlocks, and false “at risk” health on new S0 projects.

**Why:** Grep-verified UAT showed the same user could see different project sets in Work Hub vs Work panel; portal sync and list filters were fixed in one path but not others. PLP centralises listing so membership, tenant, job exclusion, and archive state apply consistently — while keeping intentional **UserVisible** (picker) vs **TenantWide** (Arbiter board view) policies.

### Shared list layer (`project_list.rs`, `project_repo.rs`)

- **`ProjectListAuth` / `ProjectListParams` / `ProjectListPolicy`:** Explicit auth (uid, tenant, admin) and flags (`include_archived`, `include_jobs`, `UserVisible` vs `TenantWide`).
- **`ProjectRepository::list_projects`:** Canonical async list API; shared `build_list_query`, visibility exists, tombstone guard, job exclusion.
- **`rows_to_project_summaries`:** Moved to `lifecycle/mod.rs` to break module cycle with `project_repo`.
- **Branch 6 tenant isolation:** Non-admin with tenant but no uid resolves correctly in `build_from_where` (legacy list filter parity).
- **11+ unit tests** on list visibility (member, non-member, admin, guest, archived, jobs, tenant-wide).

### IPC & consumers migrated to repo

- **`list_user_projects` / `lifecycle_list_projects`:** Thin wrappers over repo + policy (picker vs lifecycle enrichment preserved).
- **Agent tools, voice dispatch, orchestrator hints:** `UserVisible` + membership-scoped lists.
- **Arbiter monitor:** `TenantWide` for board-style tenant snapshots; IPC via `spawn_blocking` to avoid async-runtime deadlock with nested `block_on`.
- **Lifecycle backfill / version-control project IPC:** Auth-aware list and touch paths.

### Arbiter health & briefing

- **Health assessment:** Gate counts scoped to **current stage** only (fixes false `at_risk` on new S0 projects counting all unapproved G1–G5 gates).
- **`Blocked`:** Only when pre-approval tasks are complete and the current gate is unapproved.
- **`all_gate_statuses`:** Includes **G0** through G5 for pattern detection and gate lookups.
- **Dev UAT:** `window.forgeUat` helpers (`plpParityCheck`, `arbiterSnapshots`, `arbiterBriefing`, `logArbiterBriefing`) wired in dev layout.

### Lifecycle UI sync

- **Archive / unarchive:** `lifecycle_archive_project` and `lifecycle_unarchive_project` now emit **`projects:changed`** so Work Hub picker search refreshes without full reload (same as delete path).

### Portal sync & auth (WS-A follow-through)

- **`pullChanges`:** Idle sync when unauthenticated; guarded portal purge (`should_purge_local_project_missing_from_portal`) preserves linked `working_dir`.
- **Job IPC:** Tenant resolution from authenticated user for create/list.
- **Tests:** Portal sync `working_dir` integrity (WS-A A2/A3) in `pull.rs`.

### Docs & debt

- **PLP S1/S2/S4 DAP, UAT guide, WORKING-PROJECT-LIST-SYNC-INTEGRITY, TECHNICAL_DEBT:** WS-B progress, UAT scripts, arbiter pattern inflation and follow-ups tracked.

## 6.13.0 (2026-05-21)

**Data access Train 1 — `ProjectRepository`:** Centralises all `lifecycle_projects` mutating SQL behind an async repository layer. Version-control IPC, engine lifecycle helpers, and project creation ownership stamping now route through `ProjectRepository` instead of scattered inline SQL.

### Repository layer (`src-tauri/src/lifecycle/project_repo.rs`)

- **`ProjectRepository` (new):** Async CRUD and domain updates for `lifecycle_projects` + owner `project_members` rows via `DbHandle`.
- **Worktree / workspace:** `persist_worktree_and_stamp_owner`, `update_download_workspace`, `update_worktree_columns` (factory parity — preserves tier, `lifecycle_kind`, PI v2).
- **VC / GitHub:** `update_after_save`, PR metadata/status, conflict snapshot, `get_vc_context`, `get_github_pr_context`, github linkage, manifest, backfill ownership.
- **Engine helpers:** `archive`, `unarchive`, `set_working_dir`, `stamp_project_ownership`, `delete_project`.
- **21 unit tests** covering save/push counters, PR round-trips, conflict flags, archive idempotency, download vs worktree column contract, and ownership stamping edge cases.

### IPC & engine migrations

- **VC IPC (`ipc/version_control/*`):** `workspace`, `github_repo`, `save`, `publish`, `pr_status`, `ship`, `conflict`, `project` — zero inline `lifecycle_projects` SQL remaining.
- **`ipc/environment.rs`:** Auth backfill uses `backfill_local_project_ownership`.
- **`lifecycle/mod.rs`:** Engine create/archive/unarchive/working-dir/stage-advance delegates to repo; `ProjectRepository` wired on `AppState` and `LifecycleEngine`.
- **`project_creation.rs`:** `create_project_full` ownership stamp → `stamp_project_ownership`; persistence verify uses `project_repo.exists()`.
- **`version_control/conflict/summary.rs`:** Conflict snapshot persistence via repo.

### CI guard & docs

- **`scripts/check-lifecycle-sql.sh` (new):** Fails CI when mutating `lifecycle_projects` SQL appears outside an allow-list (`pr-quality-gate.yml`).
- **`DATA-ACCESS-INVENTORY.md`, `S4-DATA-ACCESS-REPOSITORY-AND-PATH-STRATEGY.md`:** Train 1 marked **Done**; inventory tracks remaining Train 2+ sprawl (agent tools, sync).
- **`TECHNICAL_DEBT.md`:** Logged follow-ups (silent IPC error ignores, workspace TOCTOU messaging, PR number typing).

### Tests

- **`project_creation_tests.rs` (new):** 35 integration tests extracted from `project_creation.rs` (exempt `*test*.rs` guard path).
- **`github_flow` / `project_worktree` parity tests** remain green (tier + subtask preservation).

## 6.12.1 (2026-05-21)

**Work Hub Train 2 + Train 3:** Mixed Recent list, initiative (artifact-only) projects, and a cleaner Projects | Jobs browse experience on the Work Hub picker.

### Train 2 — Mixed Recent (WS-C)

- **`ego/db.rs`:** Migration V49 — `forge_jobs.last_opened_at` column + index.
- **`ipc/jobs.rs`:** `touch_job_last_opened` IPC; job list stub icon updated to 💼.
- **`+page.svelte`:** Touches job `last_opened_at` when opening from hub or Lifecycle dashboard.
- **`loadHubSegment.ts`:** `loadHubRecentSegment()` merges projects and jobs by `last_opened_at`.
- **`HubWorkItemRow.svelte` (new):** Unified row for mixed Recent items.
- **`projectPickerUtils.ts`:** `HubWorkItem` type and `sortHubWorkItemsByLastOpenedDesc`.
- **`ProjectPickerHome.svelte`:** Unified search placeholder; **Projects | Jobs** tabs below Recent (replaces stacked “All projects” + “Jobs” sections).

### Train 3 — Initiative projects (WS-E)

- **`initiative.rs` (new):** `InitiativeProjectKind` — artifact-only, no Git repo required.
- **`registry.rs`:** Registers `initiative` alongside `coding`; registry unit tests.
- **`project_creation.rs`:** Initiative create tests; unknown `lifecycle_kind` rejected before DB write (AC-WH-14).
- **`lifecycle.rs`:** `lifecycle_create_project` returns full `CreateProjectResult` (`project` + `working_dir`) so artifact-only kinds open without `create_project_workspace`.
- **`HubInitiativeCreateForm.svelte` (new):** Name-only create wizard; opens explorer at artifact root with `isWorktree: false`.

### Refactors & tests

- **`hubCreateForm.css` (new):** Shared styles for `HubJobCreateForm` and `HubInitiativeCreateForm`.
- **`entityTypeRegistry.test.ts`, `projectPickerUtils.test.ts` (new):** Vitest coverage for kind registry cache and Recent sort.

## 6.12.0 (2026-05-21)

**Work Hub Train 1 — Projects, Jobs, Daily:** The Work Surface Hub now provides a unified entry point for creating and opening projects, jobs, and daily flows. New action cards replace the previous project-only picker with a 2×2 grid: **New Project**, **New Job**, **Daily**, and **Just Start Creating**. Jobs are listed alongside projects with delete support, and clicking **Daily** opens (or creates) today's daily flow using the host's local timezone.

### Frontend — Work Hub (`src/lib/workHub`, `src/lib/components/projectPicker`)

- **`EntityTypePicker.svelte` (new):** Generic picker for project/job types with ARIA `radiogroup`, loading/error states, and "Coming soon" disabled entries.
- **`HubJobCreateForm.svelte` (new):** Job creation form with title/reason inputs; wrapped in `<form>` for Enter-key submission.
- **`entityTypeRegistry.ts` (new):** Frontend registry for project/job kinds; fetches from `list_project_kinds` / `list_job_kinds` IPCs with session-scoped cache; `invalidateKindCache()` called on sign-out.
- **`ProjectPickerHome.svelte`:** Refactored to 2×2 action grid; renders job list with delete button; terminology updated ("Create a job" not "task").
- **`ProjectPickerWorkspace.svelte`:** Routes hub views (`home`, `pick_project_type`, `new_project`, `pick_job_type`, `new_job`); manages `deletingJobId` state; wires `onJobSelected` / `onDailyOpened` callbacks.
- **`ProjectPickerDeleteModal.svelte`:** Generalized with `entityLabel` prop for project/job deletion; ARIA `role="presentation"` on overlay.
- **`ProjectPicker.svelte`:** Forwards `onJobSelected` / `onDailyOpened` to workspace.

### Frontend — App State (`src/routes`, `src/lib/stores`)

- **`+page.svelte`:** `handleHubJobSelected` / `handleHubDailyOpened` handlers; `clearLifecycleDrillIn()` ensures single active drill-in (day vs job vs project); `JOB_SELECTED` / `DAILY_OPENED` XState events; sign-out invalidates kind cache.
- **`appState.ts`:** Added `JOB_SELECTED` and `DAILY_OPENED` event types to `AppEvent` union; transitions from `work_surface_hub` → `ide`.

### Frontend — Daily Flow (`src/lib/dailyFlow`, `src/lib/components`)

- **`dayDisplay.ts` (new):** Helpers for daily flow date formatting — `calendarDateFromDayId`, `dailyFlowDayLabel`, `formatLocalDate` (uses `'en-CA'` locale for ISO format), `formatLocalDateTime`.
- **`dayDisplay.test.ts` (new):** 9 tests covering valid/invalid day_id parsing, fallback behavior, and date formatting.
- **`DailyFlowDetail.svelte`:** Uses `dailyFlowDayLabel` / `formatLocalDateTime` for consistent date display in header and metadata.

### Rust — Project Kind Registry (`src-tauri/src/lifecycle/project_kind`)

- **`mod.rs`, `registry.rs`, `coding.rs` (new):** Backend-first extensible pattern for project types; `ProjectKindRegistry` held on `AppState`; `CodingProjectKind` implements the `ProjectKind` trait.
- **`lifecycle/mod.rs`:** Embedded SQL migration for `lifecycle_kind` column with index.
- **`project_creation.rs`:** `CreateProjectParams` includes `lifecycle_kind`; `create_project_full` validates against registry.

### Rust — Daily Flow (`src-tauri/src/daily_flow`)

- **`mod.rs`:** `local_calendar_date()` uses `chrono::Local::now()` (host timezone) for deterministic `day_id` generation — fixes off-by-one day issue for users ahead of UTC.

### Rust — IPC (`src-tauri/src/ipc`)

- **`mod.rs`:** `KindEntry` struct with `#[serde(rename_all = "camelCase")]` for frontend-backend consistency.
- **`lifecycle.rs`:** `lifecycle_create_project` accepts optional `lifecycle_kind`; `list_project_kinds` IPC returns registry entries.
- **`jobs.rs`:** `list_job_kinds` IPC returns `general` (enabled) and `code` (disabled, FD-WH-01 deferred until WS-D).
- **`main.rs`:** Registers `list_project_kinds` / `list_job_kinds` IPCs; initializes `project_kind_registry` on `AppState`.

### Rust — Agent Tools (`src-tauri/src/agent`)

- **`orchestrator/mod.rs`, `tools/mod.rs`:** `lifecycle_create_tool` passes `lifecycle_kind` and registry to `create_project_full`.

### Frontend — Sync (`src/lib/stores`)

- **`sync.ts`:** `pullChanges()` calls `pushRustAuthTokenIfLoggedIn()` before `sync_pull_changes` to address Firebase ID token expiry (R3 fix from Portal sync audit).

### Documentation (`docs/paul-working-docs`)

- **`TECHNICAL_DEBT.md`:** Added "Work Hub: naming inconsistency" section documenting `lifecycle_kind` vs `job_type`, "task" vs "job" terminology, and "IDE" vs "workspace" UI copy.
- **`S0-PORTAL-SYNC-HTTP-FANOUT-AND-QUOTA.md`:** Updated to v2 with cross-repo senior review summary; R3 marked done.
- # **`S4-WORK-SURFACE-PICKER-PROJECTS-JOBS-DAILY.md`:** Updated to v7 reflecting Train 1 completion (WS-M, WS-K, WS-A, WS-B mostly done); outlines WS-C, WS-E, WS-D for future trains.

## 6.11.0 (2026-05-20)

- generate_spreadsheet tool — native .xlsx output

## 6.10.1 (2026-05-20)

- bump protobufjs from 7.5.7 to 7.6.0 in the npm_and_yarn group across 1 directory
  > > > > > > > main

## 6.10.0 (2026-05-19)

**Workspace roots + Portal sync hardening:** Resolves project and worktree locations through a single **`workspace_resolver`** (workspace-root discipline) with **`project_worktree_persist`** for durable, single-writer worktree metadata; lifecycle **`project_creation`** and version-control **`workspace`** IPC paths updated accordingly, including optional **`skipFilesystemInit`** for creation flows. Repo script **`scripts/check-workspace-paths.sh`** (also **`npm run check:workspace-paths`**) guards against ad-hoc **`join("projects" | "jobs" | "daily")`** patterns in **`src-tauri/src`**.

**Portal / scainet.io client:** **`sync/http::request_with_retry`** now logs a **`request_label`** (full URL), captures a truncated error body on **5xx**, and **does not retry** when the response body suggests **quota / rate exhaustion** (e.g. Firestore **`RESOURCE_EXHAUSTED`**, “quota exceeded”, common rate-limit phrasing), reducing load amplification when the backend is already failing. All Portal **`request_with_retry`** call sites pass the URL as the label.

**Frontend — lifecycle + picker:** **`LifecycleDashboard`** uses the same **60s** global cooldown as the work-surface picker (**`shouldThrottlePortalProjectSync`** / **`markPortalProjectSyncRan`**) so **`lifecycle_sync_portal`** (which fans out to **`/api/tasks`** and **`/api/subtasks`** per project) is not re-fired on every dashboard visit right after a picker sync. **`projectContext`** / **`FileExplorer`** / **`NewProjectFlow`** / **`+page.svelte`** updated for workspace-root and freeform entry behaviour; **`ProjectPickerWorkspace`** / **`ProjectPickerHome`** adjusted (including disabled **Just Start Creating** styling **`.action-card:disabled`**).

**Rust — supporting changes:** **`workspace_resolver.rs`** (+ tests); **`lifecycle/jobs`**, **`daily_flow`**, and related IPC tweaks; **`git_fixtures`** / **`workspace_test`** updates.

**Documentation:** **`docs/paul-working-docs/TECHNICAL_DEBT.md`** — Portal sync **1 + 2N** HTTP fan-out, mitigations, and long-term items (batch APIs, **429** semantics, incremental sync); **`S4-WORKSPACE-ROOTS-PROJECTS-JOBS-DAILY.md`** for the workspace-roots epic.

## 6.9.8 (2026-05-19)

**Work surface hub refactor + IPC integration tests:** frontend **`ProjectPicker`** split into a thin shell (**`ProjectPickerWorkspace`** + child panels/modals), Portal reconcile logic consolidated in **`createPortalScheduler`** with Vitest coverage, XState hub state renamed **`project_picker` → `work_surface_hub`** with **`loadSnapshot`** migration for persisted snapshots. Backend **`src-tauri`**: mock-runtime wiring in **`main`** for integration-style IPC tests, incident-cache coverage, audit IPC contract tests (**`record_ui_action`**, **`audit_get_events`**, **`audit_verify_chain`**), orchestrator loop / stream tweaks and **`level_b_harness`** (**`read_file`** + **`write_file`** Level-B scenarios), lifecycle and work-context touch-ups. **PR quality gate** now also runs filtered **`scainet-forge`** tests (**`tauri_invoke_contract`**, **`level_b`**) on Windows/macOS. Repo **`TESTING.md`** documents **`#[ignore]`** policy and opt-in commands; **`INTEGRATION_TESTING_AUDIT`** §11c / §12 updated. Working docs for the hub epic (S0–S4) and integration audit.

### Frontend — work surface picker (`src/lib/components`)

- **`ProjectPicker.svelte`:** Shell only — forwards **`projectSelected`**, **`freeformMode`**, **`resumePreviousProject`**, **`signOut`**, **`upgrade`** from **`ProjectPickerWorkspace`**.
- **`projectPicker/ProjectPickerWorkspace.svelte`:** Orchestrator (list load, Portal chain, modals, navigation).
- **`projectPicker/ProjectPickerHome.svelte`**, **`ProjectPickerHeader.svelte`**, **`ProjectPickerFreeformSetup.svelte`**, **`ProjectPickerDeleteModal.svelte`**, **`ProjectPickerLimitModal.svelte`:** Extracted UI.
- **`projectPicker/portalScheduler.ts`:** Single surface for throttle + **`schedulePortalReconcile`** (replaces **`src/lib/stores/portalProjectSyncThrottle.ts`**, deleted).
- **`projectPicker/portalScheduler.test.ts`:** Scheduler behaviour tests (force, throttle, unmount, follow-up coalesce, announcements, etc.).
- **`projectPicker/loadHubSegment.ts`:** Hub segment façade — real projects loader + jobs/daily stubs (**`@internal`**).
- **`projectPicker/projectPickerTypes.ts`**, **`projectPicker/projectPickerUtils.ts`:** Shared types and pure helpers.

### Frontend — app state (`src/lib/stores`, `src/routes`)

- **`appState.ts`:** Machine state **`work_surface_hub`**; **`loadSnapshot()`** runs **`migrateAppStateSnapshotForMachine`**.
- **`migrateAppStateSnapshot.ts`** + **`migrateAppStateSnapshot.test.ts`:** Legacy snapshot **`value: 'project_picker'` → `'work_surface_hub'`** (no other context mutation).
- **`+page.svelte`:** Renders hub when **`appState === 'work_surface_hub'`**.

### Frontend — tests

- **`ProjectDetailView.gate-approval-cta.test.ts`:** Gate approval CTA coverage.

### Rust — IPC and integration harness (`src-tauri`)

- **`main.rs`:** **`#[cfg(test)]`** mock **`Runtime`** + command dispatch helpers for integration-style IPC tests.
- **`ipc/incidents.rs`:** Incident cache IPC tests (`invoke` round-trips).
- **`ipc/config.rs`:** Config-related IPC tests with mock runtime.
- **`ipc/audit.rs`:** Audit IPC contract tests — **`record_ui_action`**, **`audit_get_events`**, **`audit_verify_chain`**.
- **`audit/store.rs`:** **`ChainVerification`** derives **`Deserialize`** (IPC JSON round-trip in tests); other visibility tweaks aligned with audit IPC tests.

### Rust — agent orchestrator (`src-tauri`)

- **`agent/orchestrator/mod.rs`**, **`loop_run/mod.rs`**, **`loop_run/stream.rs`:** Agent loop / stream handling updates.
- **`agent/orchestrator/level_b_harness.rs`:** Orchestrator **Level B** — deterministic **`read_file`** and **`write_file`** round-trips (queued mock LLM, real **`ToolExecutor`**, temp workspace, conversation assertions).

### Rust — lifecycle and work context

- **`lifecycle/mod.rs`**, **`work_context.rs`:** Adjustments alongside IPC / orchestrator work.

### Rust — integration test crate (`src-tauri/tests`)

- **`integration/harness.rs`:** Doc pointer to repo **`TESTING.md`** for **`#[ignore]`** Chromium lifecycle hook.

### CI (`.github/workflows`)

- **`pr-quality-gate.yml`:** After **`cross_platform`**, runs **`cargo test --bin scainet-forge tauri_invoke_contract`** and **`cargo test --bin scainet-forge level_b`** (Windows + macOS required path; Linux job remains diagnostics-only).

### Documentation (`docs/paul-working-docs`)

- **`INTEGRATION_TESTING_AUDIT.md`:** **`#[ignore]`** policy §11c → **`TESTING.md`**; §12 adds filtered **`cargo test`** for P1-g + Level-B; prioritized backlog row **4** (ignore policy) marked done; §11a–c section order cleanup.
- **`S2-TEST-COVERAGE-P2-BREADTH-AND-UX.md`:** P2 coverage notes.
- **Work surface / hub refactor chain:** **`S0-WORK-SURFACE-PICKER-REFACTOR.md`**, **`S1-WORK-SURFACE-PICKER-REFACTOR.md`**, **`S2-WORK-SURFACE-PICKER-REFACTOR.md`**, **`S4-WORK-SURFACE-PICKER-REFACTOR.md`**, **`S0-WORK-SURFACE-PICKER-PROJECTS-JOBS-DAILY.md`**, **`S0-WORKSPACE-ROOTS-PROJECTS-JOBS-DAILY.md`**; **`WORK-WORKSPACE-ROOTS-PROJECTS-JOBS-DAILY.md`** touch-up.

### Documentation (repo root)

- **`TESTING.md`:** **`#[ignore]`** inventory table (Chromium headless, live Grok latency, deferred tool-count gate), opt-in **`--ignored`** commands, policy for new ignores; notes PR quality gate coverage for P1-g / Level-B filters.

## 6.9.7 (2026-05-18)

**Version control stability + P2 test breadth:** offload synchronous **libgit2** / **`git_ops`** work from Tokio async IPC handlers via **`tokio::task::spawn_blocking`** (mitigates **nested-runtime** / worker blocking seen in production). New **Vitest** coverage for onboarding/settings UI and **Rust** integration-style tests for **EgoEngine** session ledger + memory persistence. Working docs for Sentry RCA, S2 VC IPC blocking scope, test-coverage S0/S2, Tauri IPC performance, and voice transcript UX.

### Rust — version control (`src-tauri`)

- **`version_control/operations.rs`:** **`save_project`** split into blocking **`save_project_phase1`** / **`save_project_phase2_sync`** with **`spawn_blocking`** around each phase; async **lifecycle test** step remains between phases.
- **`ipc/version_control/project.rs`:** **`get_project_state`**, **`check_project_staleness`**, **`sync_project_remote_on_open`** run heavy git/state work on the blocking pool (**`sync_project_remote_on_open_blocking`** helper).
- **`ipc/version_control/publish.rs`:** Status, push, fetch, conflict check, and commit enumeration moved into blocking closures / **`publish_blocking_finish_local_git`**.
- **`ipc/version_control/save.rs`:** **`discard_project_changes`** wraps **`discard_changes`** in **`spawn_blocking`**.
- **`ipc/version_control/workspace.rs`:** Worktree create/checkout/list/remove wrapped in **`spawn_blocking`**.
- **`ipc/version_control/branch.rs`:** **`list_stale_branches`** runs **`current_branch`** on the blocking pool after GitHub HTTP listing.
- **`ipc/version_control/manifest_sync.rs`:** **`apply_manifest_drift`** tail (stage / commit / push) in **`apply_manifest_drift_git_tail`** + **`spawn_blocking`**.
- **`ipc/version_control/conflict.rs`:** Conflict IPCs that touch the repo use **`spawn_blocking`** (detect, AI prelude, resolve, undo, abort, complete, lockfile regeneration, sync with staging/main); **`resolve_high_confidence_conflicts`** already used blocking.

### Rust — tests (`src-tauri`)

- **`ego/mod.rs`:** **`#[cfg(test)]`** **`ego_engine_persistence_tests`** — session start + tool action ledger round-trip, memory **`store_memory`** / **`recent_memories_async`**, failure outcome persistence (traces to audit-trail doc + **S2-TEST-COVERAGE-P2**).

### Frontend — tests (`src/lib`, `src/tests/mocks`)

- **`ApiKeySetup.test.ts`, `GateApproval.test.ts`, `InstructionPanel.test.ts`, `SettingsDialog.test.ts`:** P2 breadth coverage (mocks per **S2-TEST-COVERAGE-P2-BREADTH-AND-UX**).
- **`auditEvents.test.ts`:** Adjustments aligned with store behaviour.
- **`tests/mocks/tauri.ts`:** Minor mock updates for new tests.

### Documentation (`docs/paul-working-docs`)

- **`SENTRY-NESTED-RUNTIME-GIT-PANIC.md`:** Expanded incident / RCA / inventory.
- **`S2-VC-IPC-LIBGIT2-SPAWN-BLOCKING.md`:** S2 scope for VC IPC offload (**G2**).
- **`S0-TEST-COVERAGE-P2-BREADTH-AND-UX.md`**, **`S2-TEST-COVERAGE-P2-BREADTH-AND-UX.md`:** P2 test breadth S0/S2.
- **`S0-TAURI-IPC-PERFORMANCE-AND-BINARY-PAYLOADS.md`**, **`S0-VOICE-LIVE-USER-TRANSCRIPT-UX.md`:** Working notes.
- **`TEST_COVERAGE_AUDIT.md`:** P2 pointer updates.

## 6.9.6 (2026-05-18)

**CI, quality gates, and Rust hygiene:** stable **`CI Passed`** aggregation for branch protection; **`cargo clippy --all-targets`** on the Ubuntu PR gate; **`pr-quality-gate`** workflow for Windows + macOS cross-platform smoke (Linux diagnostics only, non-blocking); **`mirror-release`** only when the multi-platform **`build`** matrix fully succeeds. Rust dead-code / Clippy fixes, more reliable voice audit tests, architecture and S0 operator docs.

### GitHub Actions

- **`build.yml`:** After **`cargo test`**, run **`cargo clippy --all-targets`** (fails on deny-by-default lints; warnings allowed per **`docs/paul-working-docs/S0-RUST-DEAD-CODE-WARNINGS.md`**). New **`ci-passed`** job (**name: CI Passed**) requires **`test` (Ubuntu)** **`success`** — use this as the stable required check for branch protection. **`auto-tag`** now **`needs: [ci-passed]`**. **`mirror-release`** runs only when **`needs.build.result == 'success'`** (no mirror on partial matrix failure); see S0 §13 for recovery.
- **`pr-quality-gate.yml` (new):** On **`pull_request`** to **`main`** (+ **`workflow_dispatch`**): **`Smoke (Windows)`** / **`Smoke (macOS)`** run **`cargo test --test cross_platform`**; **`Smoke (Linux — diagnostics only)`** uses **`continue-on-error: true`**.
- **`cross-platform-smoke.yml` (removed):** superseded by **`pr-quality-gate.yml`**.

### Contributor docs

- **`CONTRIBUTING.md`:** CI overview table ( **`build.yml`** vs **`pr-quality-gate.yml`** ), required checks (**`CI Passed`** + Win/mac smoke, not Linux diagnostics), mirror-only-on-success pointer to S0.

### Rust (`src-tauri`)

- **`version_control/manifest_sync.rs`:** Test assertion covers real manifest drift (removes tautology; fixes **`clippy::overly_complex_bool_expr`**).
- **`integrations/gmail/api.rs`:** Unused deserialized fields kept for Serde with **`#[serde(rename = …)]`** and **`_`-prefixed** Rust field names.
- **`db/error.rs`:** Drop unused **`ReadPoolTimeout`** enum variant (nothing constructed it).
- **`config/mod.rs`:** Remove unused **`ConfigEngine::db_handle`**.
- **`ego/mod.rs`:** Remove unused **`db_handle`** and superseded async helpers; canonical paths remain for session / ledger behavior.
- **`audit/store.rs`:** **`append`** / **`last_event`** limited to **`#[cfg(test)]`**; production continues to use **`record_chained`**.
- **`voice/session.rs`:** Test **`AUDIT_BARRIER_WAIT`** **5s → 10s**; F-091 coalesce test waits **`UserMessage`** then **`AssistantMessage`** before assertions; **`flush_pending_user_message_audit`** **`tracing::warn!`** when **`AuditEngine::record`** errors.

### Documentation

- **`docs/paul-working-docs/S0-CI-RELEASE-GATING-AND-SMOKE.md`:** CI charter / runbook (jobs, branch protection, mirror policy).
- **`docs/paul-working-docs/S0-RUST-DEAD-CODE-WARNINGS.md`:** Rust warning / Clippy triage.
- **`docs/paul-working-docs/INTEGRATION_TESTING_AUDIT.md`**, **`TECHNICAL_DEBT.md`**, **`TEST_COVERAGE_AUDIT.md`**, **`FRONTEND_COMPONENT_SIZE_AND_REFACTOR_CANDIDATES.md`:** Updates and additions.
- **`docs/architecture/`:** Large expansion — C4 views, system stack, conduit / VC / W5 Mermaid diagrams, EGO ER, sequences, walkthroughs; **`FORGE_ARCHITECTURE_DIAGRAMS.md`** and **`README.md`** refactored.
- **`docs/FORGE_UNDERSTANDING_COURSE.md`**, **`scainet-test/docs/FORGE_*.md`**, **`raw-research/W5-VERSION-CONTROL-CONTEXT-S0.md`:** Small edits.

### Housekeeping

- Removed an erroneously tracked screenshot under **`src-tauri/`** with an invalid Windows path segment.

## 6.9.5 (2026-05-17)

**Free tier unblocked:** the `start` HUMAN command (and the lifecycle navigation set) is now available on the **Free** plan. Previously a free user clicking **Start Text** — or typing any HUMAN command trigger — received `"start" requires a Basic or Pro subscription. Available commands: \`status\`, \`who\`, \`version\`.`, even though the **Basic** tier no longer exists in the codebase. Free users could not get past the entry splash, could not run the naming ceremony, and could not experience FORGE at all without paying — directly contradicting the **Free forever** pricing message.

This release opens the door to the product. Pro+ differentiation now lives where it always belonged — automation (Daily Flow), scale (unlimited projects, Catalyst dispatch), and infrastructure-touching git/release commands (`stage it`, `ship it`, `quick save`, `handover`) which are also actively being migrated to dedicated toolbar buttons (Save / Publish / etc.). The HUMAN command surface will be revisited once those buttons are dialled in; until then the gate is kept on commands whose UX is mid-migration.

### `src/lib/stores/entitlements.ts`

- **`FEATURE_MATRIX.scainet.free.humanCommands`:** expanded from `['status', 'who', 'version']` to include `start`, `rules`, `tasks`, and the lifecycle navigation commands `advance_to S0` through `advance_to S6`. Free now has 13 HUMAN commands (was 3) — covering the full onboarding path (`start` → naming ceremony → lifecycle navigation → discovery) without leaving anything that costs us infrastructure exposed.
- Kept Pro+ gated: `stage it`, `ship it`, `quick save`, `handover`, `framework version`. These touch git/deploy infrastructure AND are being replaced with toolbar buttons — no churn on their gate until that migration completes.

### `src/lib/components/InstructionPanel.svelte`

- **Stale message text removed:** the blocked-command warning hardcoded `"Basic or Pro subscription"` and an `allowed` string of `` `\`status\`, \`who\`, \`version\`` ``. Both were stale — `Basic`tier was removed long ago, and the`allowed`list no longer matches the (now expanded) free command set. Replaced with`"X" requires a Pro subscription. Upgrade to unlock the full HUMAN command set.` — accurate, non-stale, no per-tier interpolation.

### Tests

- **`entitlements.test.ts`:** all 29 tests pass unchanged. Existing assertions (`canRunCommand('handover') === false` for free, `canRunCommand('ship it') === true` for pro) still hold — the change is additive to free, not subtractive from gated commands.

### Why now (release context)

A user-facing Stripe Cloud Function regression earlier today produced the misleading error `Invalid tier: pro. Valid: pro` for paying customers attempting to upgrade — root cause was missing `STRIPE_PRICE_PRO` / `STRIPE_SECRET_KEY` env in the deployed `billingCreateCheckout` function, fixed server-side independent of this release. While investigating, the `start` gate surfaced as a secondary blocker — even users who _could_ pay couldn't experience the product enough to want to. This release unblocks the door.

## 6.9.4 (2026-05-17)

**Work tab inner IA (S4):** unified **Projects | Jobs | Daily Flow** chrome on **`LifecycleDashboard`**, **`signalNavigateState`** helper for **`signal:navigate`** / **`+page`** navigation parity, **SessionBar** + backend **work context** cleared when leaving project, job, daily, or **LaunchPad** drill-in views, **project-archive** UI scoped to **Projects** only, and **Daily Flow** creation from the shared toolbar.

### Navigation and work context (`+page.svelte`)

- **`signalNavigateState` integration:** Centralized patch application for lifecycle panel state (project/job/day/launch-pad selection, **`lifecycleDashboardTab`**, **`projectWorkingDirBeforeJob`**, refresh keys) while preserving documented edge cases (e.g. job_tab without **`jobId`**).
- **`handleLifecycleBack`:** **`lifecycleModeStore.clearWorkContext()`** + **`clear_active_work_context`** when leaving **ProjectDetailView**.
- **Job back:** same clear after restoring the pre-job working directory.
- **DailyFlowDetail back:** **`clear_active_work_context`** (aligned with other exits).
- **LaunchPad back:** clear store + **`clear_active_work_context`** when returning to the dashboard list.
- **`handleLifecycleBack`:** **`console.warn`** on **`clear_active_work_context`** failure.

### `LifecycleDashboard.svelte`

- **Single primary band:** segment **tablist** + trailing toolbar (overflow **More**, per-tab primary CTA).
- **CTAs:** **+ New Project** / **+ New Job** / **+ Start Today** per segment (fixes **+ New Job** / **+ Start Today** disappearing when **showArchived** was true on Projects).
- **Archived projects:** **“N archived”** toggle, **Sort**, and archived banner only when **Projects** is active.
- **Daily Flow:** **`bind:this`** to **`DailyFlow.startTodayFromToolbar()`**.
- **Stable `data-testid` hooks** for segments and panels (tests / a11y).

### `DailyFlow.svelte`

- Removed redundant in-panel header (**toolbar** owns **+ Start Today**).
- Exported **`startTodayFromToolbar`**; **`startToday`** ignores re-entrancy while **`creating`**.

### `signalNavigateState.ts` (new)

- Pure **`applyNavigatePatch`** + **`createLifecycleNavigateState`** for **`CustomEvent<NavigateDetail>`** handling; used by **`onSignalNavigate`**.
- Tests in **`signalNavigateState.test.ts`**.

### Accessibility and lint

- **`eslint.config.mjs`:** a11y-related adjustments.
- **Components:** removed unnecessary **`svelte-ignore` a11y** suppressions where roles/handlers are correct (**AuthGate**, **BranchCleanupDialog**, **TabBar**, **McpSettings**, etc.).

### Documentation

- **`docs/paul-working-docs/S4-WORK-TAB-IA-DAP.md`:** expanded DAP traceability and execution notes.

### Tests

- **`LifecycleDashboard.work-tab.test.ts`:** panel visibility, **`aria-selected`**, archived chip vs tab, CTA labels.
- **`signalNavigateState.test.ts`:** navigation patch behaviour.

## 6.9.3 (2026-05-17)

**Rust dependency security:** bump the HTTP, error-reporting, and WebSocket stacks so TLS resolves through maintained **`rustls` / `rustls-webpki`** versions, and known-vulnerable transitive crates (including older **`rustls-webpki`** and **`rand` 0.7**) are no longer pulled in via Forge’s direct dependencies.

### `src-tauri`

- **`Cargo.toml` / `Cargo.lock`:** **`reqwest`** `0.11` → **`0.12`** (rustls TLS unchanged); **`sentry`** `0.34` → **`0.48`**; **`tokio-tungstenite`** `0.21` → **`0.29`** (`connect`, `handshake`, **`rustls-tls-native-roots`**).
- **`voice/grok_realtime.rs`:** Adapt to **`tungstenite`** **`Message::Text`** using **`Utf8Bytes`** (`.into()` on send, **`text.as_str()`** when deserializing JSON).
- **`voice/reconnect.rs`:** Tests and mock servers store **`Utf8Bytes`** text as **`String`** where the harness expects owned **`String`** logs.

## 6.9.2 (2026-05-17)

ESLint tightening across the frontend, safer unknown-error handling, EGO repair for drifted **`prompt_package_metadata`** schemas, and MCP settings/marketplace UX plus **persisted “don’t auto-start”** when the user disconnects a server.

### ESLint and TypeScript

- **`eslint.config.mjs`:** **`@typescript-eslint/no-explicit-any`** is **error**; `reportUnusedDisableDirectives` enabled; continues gradual **warn** posture for selected Svelte a11y / reactivity rules.
- **`eslint-any-report.txt`:** Reference sweep of `any` usage after the rule change (for reviewers and follow-up cleanup).
- **`src/app.d.ts`:** App-wide typing / `Window` augmentations aligned with stricter linting.
- **`src/lib/utils/unknownError.ts` (new):** Shared helpers to normalize **`unknown`** caught values for logging and UI (`cause.message` chaining, `AggregateError`).
- **`src/lib/types/sliderPanelValues.ts` (new):** Typed helpers for slider panel values instead of loose `any`.

### Components and stores (lint + correctness)

Wide touch across Svelte components and TS modules for explicit types, safer `unknown` handling, and minor logic cleanups, including: **`AgentStream`**, **`AuthGate`**, **`AuthPanel`**, **`FileExplorer`**, **`InstructionPanel`**, **`DailyFlow`** / **`DailyFlowDetail`**, **`VoiceButton`**, **`SliderPanel`**, **`LifecycleDashboard`**, **`JobDetailView`**, audit / gate / persona modals, agent-stream rows, **`+page.svelte`**, and stores **`agent`**, **`auth`**, **`sync`**, **`updater`**, **`protocols`**, plus **`firebase`** and Firestore modules (**`scoped-query`**, **`documents`**, **`projects`**, etc.).

### MCP — Rust (`src-tauri/src/protocols/mod.rs`)

- **`McpServerStatus.enabled`** exposed to the UI; **`start_all`** only connects servers with **`enabled: true`** in persisted config.
- **User Connect** sets **`enabled: true`** and saves; **user Disconnect** stops the client, sets **`enabled: false`**, and saves so the server stays off across restarts until re-enabled.
- **`protocol:server_disconnected`** includes a **`reason`** (`user_requested` | `server_removed` | `app_shutdown`) so the frontend only treats **explicit user disconnect** as disabling auto-start.

### MCP — Frontend

- **`src/lib/stores/protocols.ts`:** **`serversListHydrated`** / **`serversListLoading`** with refcounted in-flight **`loadServers`**; **`finally`** always marks hydrated after the first completed fetch (fixes perpetual skeleton when **`mcp_get_servers`** succeeds); **`initProtocolListeners`** wrapped in **try/catch** so listener setup failures still hydrate; merge preserves **event-ahead** connected servers.
- **`McpSettings.svelte`:** Add/edit in a **modal**; connect/disconnect feedback (**toasts**); **“manual start”** indication when **`enabled === false`**; list **skeleton** until first load completes.
- **`McpMarketplace.svelte`:** **Skeleton** until hydrated; **`onMount`** triggers **`loadServers()`**.

### EGO database

- **Migration v47** **`prompt_package_metadata_repair`:** Re-applies the PI v2 batch so **`prompt_package_metadata`** exists on databases that partially applied earlier migrations (fixes **`no such table`** / persist failures).

### Documentation

- **`PROJECT_LEARNINGS.md`** updates.
- **`docs/paul-working-docs/`:** Work tab IA refactor series (S1–S4), **S0** right-rail / work-surface notes, **S0-VC-BUGS**, **S1-AUDIT-WIRING-AND-ENTRY-POINTS**.

### Tests

- Adjustments in **`auditEvents.test`**, **`vcCiPolling.test`**, **`vcStateMachine.integration.test`**, **`voice/toggle.test`**.

## 6.9.1 (2026-05-16)

**Projects tab navigation parity** with the Project Picker (Path A): opening a lifecycle project from **Projects → project row** now applies the same frontend workspace bundle as **picker → `projectSelected`** (explorer, recents, lifecycle mode, backend work context, agent session context), with guards for races and legacy folder linking.

### Path B parity (`ProjectDetailView` + shared module)

- **`src/lib/lifecycle/projectDetailWorkspace.ts` (new):**
  - **`pickFolderAndTryPersistWorkingDir`** — native `pick_folder`, then persists via **`link_folder_to_project`** (same IPC as `ProjectPicker.svelte` for portal/legacy rows without git), with trim/cancel/error shaping (`path`, `persistedToDb`, `pickerFailed`).
  - **`applyLifecycleWorkspaceParity`** — ordered bundle: `projectContextStore.setProject` → `recentProjectsStore.addProject` → `lifecycleModeStore.setActiveProject` → `set_active_work_context` → `new_conversation` + `enterContext`; optional **`isStale`** for overlapping `loadData` calls; lifecycle/agent steps remain best-effort if `setProject` fails (matches `+page.svelte` pattern).
- **`ProjectDetailView.svelte`:**
  - **`loadData`** — stale-flight epoch (`loadDataGeneration`); legacy **`working_dir` null** flows prompt for folder, persist with picker-aligned IPC, show success vs **“Folder selected but not saved”** when DB write fails but session path is still applied; **`loadReferenceDocs()`** after workspace attempt; **`workspaceReady`** flag.
  - **`workspaceReady`** — starts `false`, set `true` only after **`parity.kind === 'ok'`** so **`ProjectStateIndicator`** mounts **after** `projectContextStore` reflects the current project (fixes **AutoLink / Link Conflict** firing on stale context while the folder dialog was still open).
  - **`vcStateMachine.init(projectId)`** — removed from **`onMount`**; called from **`loadData`** only after successful parity (avoids **PROJECT_SWITCHED** / VC/automation racing the folder picker for legacy projects).
  - **`changeProjectDirectory`** — uses the same pick + parity helpers; after success calls **`vcStateMachine.init(project.id)`** so VC state tracks the new working directory (fixes stale Save/Publish/Ship / CI affordances after directory change).
  - Local **`project.working_dir`** updated when the user selects a folder even if persist fails (explorer + UI stay aligned for the session).

### Tests

- **`src/lib/lifecycle/projectDetailWorkspace.test.ts`** — picker cancel/throw, persist success/fail, trim, parity order, stale checks, `setProject` failure still runs downstream best-effort steps, no direct `set_working_directory` invoke in parity.

### Documentation

- **`docs/paul-working-docs/S0-PROJECTS-FEATURE-FLAG.md`** — **§10** lists remaining low-severity Path A vs Path B differences (recents ordering on `setProject` failure, picker portal refresh after link, early `onMount` work-context, DB uniqueness vs `lifecycle_set_project_working_dir`) and summarizes fixes shipped in this release.

### Related planning

- **`docs/paul-working-docs/S0-PROJECT-LIFECYCLE-NAVIGATION-PARITY.md`** — scope and acceptance context for lifecycle navigation parity (Path A / Path B).

## 6.9.0 (2026-05-15)

GitHub lifecycle integration hardening: **app install status**, **automatic repository linking (Flow C)**, **smarter link prompts**, **settings-based GitHub sign-in / reconnect**, and clearer UX when the **signed-in GitHub user does not match the repository owner**.

### GitHub App & repository linking

- **Settings — GitHub App:** Installation check with clearer **installed / checking** feedback and a heuristic **installed** badge when API status is ambiguous but a prior install hint exists.
- **Flow C auto-link:** Attempts to auto-link when opening a project whose remote is already covered by the installed app; **`no_access`** surfaces a warning toast with actionable guidance.
- **Retry after browser changes:** Auto-link retries on **window focus** and **document visibility** (returning from GitHub after granting repo access) without wiping unrelated cached `no_access` state — fixes focus retry that previously invalidated the cache too early.
- **Link prompt modal (feature-flagged):** Shows when the app is **known not installed**, or when the app is installed but **lacks access to this repo** (`no_access`), without racing a successful auto-link (modal gated on skip / `no_access` outcome).
- **Access verification:** Helpers and UI paths for verifying GitHub access (`verify-access`, `AccessLostModal`), Firestore VC listener merge behaviour, and branch normalization utilities with tests.
- **Project UI:** Link repository control and inline link affordances when **Save / Publish / Ship** are allowed for unlinked projects (Phase B flags).

### Settings & account identity

- **GitHub account card:** **Sign in with GitHub** (device flow) when not authenticated, **Disconnect** when connected, and inline **user code / cancel** during device flow — not only after disconnect.
- **Removed** the in-app **“remove project webhook / Firestore linkage”** control; users manage hooks and installation from GitHub.
- **Identity tracking:** `identity-change` and **last-known username** persistence; **`githubAppStore`** centralizes installation/repo list/account login state; GitHub device-auth wiring updates.
- **Rust (`github_auth`):** Device-flow / persistence improvements used by the frontend.

### Account mismatch warning (project detail)

- **Banner** when the lifecycle remote is GitHub, **`repoOwner`** is known, and the signed-in GitHub user (from the app store or last-known username) **differs** from that owner — explains why linking / VC may fail.
- **Save / Publish / Ship** are **non-interactive** (disabled styling + tooltip) when there is a mismatch **and** webhook linkage is not complete, avoiding failed operations with no explanation.

### Tooling & docs

- **`tauri:dev:quiet:mac`** and **`tauri:dev:quiet:windows`** npm scripts for reduced log noise during local dev.
- **Planning / working docs** under `docs/paul-working-docs/` for GitHub app lifecycle, auto-link, MCP-to-settings, manifest gitops, and related S1/S2/S4 references; **EGO_REGISTRY** and **TECHNICAL_DEBT** touch-ups.

### Tests

- New / extended tests for **link-repository** (including flow cases), **webhook-linkage**, **forge-vc-listener** merge, **branch-normalize**, **polling**, **verify-access**, **projectState** effective PR state, and related modules.

## 6.8.1 (2026-05-15)

Settings-first MCP, a slimmer project header, stage context on the title, and a self-healing database migration.

### MCP in Settings

- **MCP Servers** is now under **Settings** with **My Servers** and **Marketplace** sub-tabs (the former right-panel MCP tab is removed).
- **Command Palette → MCP Servers** opens **Preferences** on that section via the new `settingsNavigation` store (`requestOpenSettings('mcp')`).
- **TitleBar** no longer shows a live MCP tool-count indicator; onboarding copy points to **Settings → MCP Servers**.

### Project detail header

- **Overflow menu (⋮)** replaces the wide experience dropdown and separate refresh control: **View mode** (Guided / Standard / Professional), **Refresh**, and **Settings…** (global preferences).
- **Stage** is shown as an emoji before the project name, with a rich **InfoPopover** tooltip (headline + stage coaching text). **InfoPopover** adds **`triggerInline`** for compact triggers.

### Reliability

- **EGO SQLite migration v44** (`day_id` on `w5_conductor_transitions`) is **idempotent**: ensures the table exists (per earlier schema) before altering, so databases that skipped a prior migration no longer fail on startup.

### Documentation

- **`docs/paul-working-docs/S1-MCP-TO-SETTINGS.md`** — implementation spec for the Settings move.

## 6.8.0 (2026-05-14)

A large feature drop covering five connected workstreams: a new **Daily Flow** operating layer with a **named Persona Cast**, the internal voice **Hindsight Protocol** rename, the **Grok F-070** provider work (`reasoning_effort` + `tool_choice` + `grok-4.3` auto-routing + image-gen + 1M context window), **frontend agent-stream regressions** fixed (voice chat history leak, freeform thinking disappearance, voice-pill accumulation), and a round of conflict / W5 / Gmail / audit polish.

### Daily Flow & Persona Cast (PROJ-DAILYFLOW)

A day-scoped operating layer built on top of the existing lifecycle infrastructure. Each day progresses through five phases — **Briefing → Planning → Executing → Wrapping → Closed** — with a named cast of personas handling specific roles.

**Persona Cast — five named personas with hard rules and emotional sliders:**

- **Atlas (Research)** — sources every claim, surfaces uncertainty explicitly, never presents inference as fact. Output format: Finding → Source → Confidence Level.
- **Aurora (Briefing)** — voice-first morning persona. Routes to voice mode automatically when a `voice_assignment` exists; falls back to text otherwise.
- **Lens (Review)** — end-of-day review and pattern surfacing.
- **Quill (Writing)** — prose synthesis, leveraging the new `writing_voice` per-persona overrides.
- **Sigma (Strategy)** — multi-day pattern analysis and strategic framing.

Each persona is defined under `src-tauri/src/persona/cast/{atlas,aurora,lens,quill,sigma}.rs` following the `voice_conductor.rs` structural pattern (constants for ID/NAME/ROLE, sliders function, `CreatePersonaRequest` builder, `PersonaDefinition` factory). FLINT-38 invariant tests guard the contracts.

**Calibration overlays.** Per-user slider tuning without modifying canonical persona definitions. Overlays are partial JSON (only the fields the user changed); merge applies the overlay on top of the canonical sliders. An LRU-backed `CalibrationCache` keys rendered prompts by `(persona_id, user_id, slider_hash)` so the same calibrated state never re-renders. Stored in the new `persona_calibrations` and `slider_presets` tables.

**Output signal tracking.** Every persona-generated output carries a `slider_state_hash` linking it back to the exact slider configuration that produced it. The audit hook (`record_signal`) writes to `persona_output_signals` with optional `quality_score`, enabling quality correlation across calibration variants over time.

**Voice assignment.** New `voice_assignments` table maps persona × user → voice provider + voice ID + optional `voice_clone_source_ref` and `consent_grant_id`. Briefing routes Aurora to voice mode when an assignment exists; clean text fallback otherwise.

**Daily Flow domain.** New SQLite tables `day_records` (day-level state, doc refs, opened/closed timestamps) and `day_tasks` (per-day tasks with status / persona assignment / dependencies). Phase transitions through `DayPhase::{Briefing, Planning, Executing, Wrapping, Closed}` with `next()` / `prev()` enforcement.

**Frontend.** New `DailyFlow.svelte` (day-level dashboard) and `DailyFlowDetail.svelte` (task-level view), plus generic `EscalationModal.svelte` and `PermissionModal.svelte` reused across the cast for human-in-the-loop checkpoints.

**EGO migrations:** v39 (`daily_flow_calibration`), v40 (`daily_flow_signals`), v41 (`daily_flow_domain`), and follow-on migrations for `voice_assignments` and writing-voice overrides. All idempotent (`CREATE TABLE IF NOT EXISTS`), forward-only.

**New code:** `src-tauri/src/daily_flow/{mod,delegation,tasks}.rs`, `src-tauri/src/ipc/daily_flow.rs`, `src-tauri/src/persona/{calibration,signal,voice_assignment,writing_voice}.rs`, `src-tauri/src/persona/cast/`, `src/lib/components/{DailyFlow,DailyFlowDetail,EscalationModal,PermissionModal}.svelte`.

**Supporting refactors:** `lifecycle/{jobs,mod,project_creation}.rs`, `ipc/{jobs,lifecycle}.rs`, `work_context.rs` extended for day-scoped context. `persona/{templates,process_scales,mod,voice_conductor}.rs` updated to slot into the cast model. `data/persona_profiles.toml` extended (+104 lines) with the five-persona seed data.

**Dependency:** `lru = "0.12"` added to `src-tauri/Cargo.toml` for `CalibrationCache`.

**Planning docs (committed):** `docs/S3-HLP-DAILY-FLOW-AND-PERSONA-CAST.md` (HLP V2, DIAMOND-hardened) and `docs/S4-DAP-DAILY-FLOW-AND-PERSONA-CAST.md` (DAP V3, DIAMOND-hardened + UX wiring pass) — both source-of-truth for the build.

### Voice — Hindsight Protocol (internal rename + barge-in fix + conductor wiring)

**Hindsight Protocol** is the new internal name for the bounded recent-conversation injection that makes "close voice session, reopen in the same project" feel like the same conversation. The mechanism, behaviour, hardening history (F-065, HELIX-38), and on-disk audit shape are unchanged — this is a naming alignment so the in-code vocabulary matches the customer-facing product name.

**Code-side renames:**

- File: `src-tauri/src/voice/recent_context.rs` → `src-tauri/src/voice/hindsight.rs`
- Module path: `crate::voice::recent_context` → `crate::voice::hindsight`
- Constants: `RECENT_CONTEXT_ROW_LIMIT` (40) → `HINDSIGHT_ROW_LIMIT`; `RECENT_CONTEXT_TOKEN_BUDGET` (6 000) → `HINDSIGHT_TOKEN_BUDGET`; `RECENT_CONTEXT_KEEP_LAST` (10) → `HINDSIGHT_KEEP_LAST`
- Struct: `RecentContext` → `Hindsight`
- Functions: `render_recent_context()` → `render_hindsight()`; `render_voice_conductor_system_prompt_with_recent_context()` → `render_voice_conductor_system_prompt_with_hindsight()`

**Preserved unchanged (Article 1 PRESERVE — durable contracts):**

- `VoiceTransport` audit row `frame_type = "conversation.item.create.recent_context"` is intentionally NOT renamed — every existing voice-session audit chain in any user's audit DB continues to verify, and replay tooling that greps for the historical `frame_type` keeps working.
- Audit JSON keys `recent_context_rows`, `recent_context_byte_size`, and `resumed_from_session` are preserved on disk for the same reason.
- `agent/system_prompt.rs` keeps its generic `recent_context: Option<&str>` parameter — that file is the generic system-prompt builder used by non-voice paths too; "Hindsight" is the voice-product naming, not the generic primitive.

**Voice barge-in (F-069 — Owner-reported regression).** `RealtimeEvent::SpeechStarted` is now wired through `voice/session.rs` to send `response.cancel` to xAI mid-stream, clear pending text, and audit the `VoiceTransport` cancel + W5 conductor transition. Previously `voice/conductor.rs` had the logic but no caller — interrupting the assistant mid-utterance failed silently.

**Conductor + dispatch.** `voice/dispatch.rs` (+227 lines) and `voice/session.rs` (+557 lines) extended for cleaner state-machine integration with the conductor. `voice/over_promise.rs` and `voice/tools.rs` updated to align.

### Grok provider — F-070 (`reasoning_effort` + `tool_choice` + `grok-4.3` auto-routing + image-gen routing + context windows corrected)

**`reasoning_effort` plumbing.** Forge previously left two `/v1/chat/completions` parameters at xAI defaults: `reasoning_effort` (defaulted to `"low"` for every reasoning model) and `tool_choice` (defaulted to `"auto"` with no override path). Owner uses `grok-4.3` in manual mode daily; the lack of an effort knob meant complex multi-step problems silently got the same thinking-token budget as a "say hi" prompt. Both parameters are now plumbed end-to-end so the orchestrator can opt in per call.

- New types in `agent/providers/mod.rs`: `ReasoningEffort` (`None` / `Low` / `Medium` / `High` — serializes lowercase as xAI expects), `ToolChoice` (`Auto` / `Required` / `None` / `Function(name)` — `to_json()` renders the OpenAI-compatible shape), and `RequestOptions { reasoning_effort, tool_choice }` with `Default::default()` = both `None`.
- New trait method `LLMProvider::complete_conversation_streaming_with_options` whose default implementation **delegates to the existing `complete_conversation_streaming`** — every non-Grok provider keeps working unchanged with zero touches.
- `GrokProvider` refactored: both streaming trait methods route through one private `streaming_call(...)` so the legacy entry point and the new opt-in entry point share exactly one HTTP path. No code duplication, no risk of drift.

**Reasoning-effort whitelist verified live against the xAI API (`scripts/probe_xai_reasoning_effort.py`, committed).** Probed 2026-05-14:

- **ACCEPTS:** `grok-4.3`, `grok-3-mini`.
- **REJECTS:** `grok-3`, `grok-4`, `grok-4-0709`, `grok-4-latest`, `grok-4.20-reasoning`, `grok-4.20-non-reasoning`, `grok-4.20-0309-reasoning` / `-non-reasoning`, `grok-4-1-fast-reasoning` / `-non-reasoning`, `grok-4-fast-reasoning` / `-non-reasoning`, `grok-code-fast-1`, `grok-4.20-multi-agent-0309` ("Multi Agent requests are not allowed on chat completions").

**Counterintuitive lesson** (now documented in `grok.rs` and pinned by the `whitelist_excludes_models_proven_to_reject_via_live_api` regression test): a model id containing `"reasoning"` does **NOT** imply the `reasoning_effort` knob is accepted. The "reasoning" / "non-reasoning" suffix is xAI's signal for internal reasoning, not for the user-visible effort knob.

**Effort mapping** (`router::classifier::compute_complexity_score` → effort):

| Score    | Effort   | Best for                                                |
| -------- | -------- | ------------------------------------------------------- |
| 0 – 60   | `Low`    | xAI's untuned default; general agentic + tool-call work |
| 61 – 85  | `Medium` | Complex data analysis, multi-file refactors, debugging  |
| 86 – 100 | `High`   | Challenging problems, deep architecture, math proofs    |

`None` is deliberately not used — it disables reasoning entirely on xAI's side, too aggressive for general chat.

**`tool_choice` wire — RESERVED.** The wire flows end-to-end and is testable, but the orchestrator passes `None` today, preserving xAI's `"auto"` default (Article 1 PRESERVE). Investigation in this branch confirmed the existing **reject-after-call delegation policy** + **system-prompt tool enumeration** (`agent/system_prompt::get_tool_documentation`) handle stray tool calls in lived experience — Owner explicitly verified the model accurately recognises "I do not have that tool" from the prompt and the post-call gate catches the residual misses. The wire stays so a future `Function(name)` pin (e.g. when the user explicitly names a tool) can be added without re-plumbing every provider. Doc on `providers::ToolChoice` documents the RESERVED status so the next agent doesn't try to "fix" the dormant wire or assume the allowlist is doing more than it actually does.

**`grok-4.3` auto-routing enabled.**

- New entry in `src-tauri/resources/model_capabilities.json` (bumped to **v1.4.0**): `grok-4.3` at perception 85, reasoning 92, code 90, instruction 90, creativity 80 — ranked above `grok-4.20-reasoning` so the router prefers it for hard CloudHeavy work.
- `classify_grok_capabilities` in `registry.rs`: reasoning flag matches `grok-4.3`; vision flag expanded across the `grok-4.x` family (per xAI docs all modern grok-4 models accept `image_url` content parts; the previous code only matched legacy `grok-2-vision-1212`).
- `classify_tier` correctly promotes `grok-4.3` to `CloudHeavy`.

**Image-gen routing fixed (Owner-reported: "model picker selects grok 3 for image generation etc which is obviously wrong").** Root cause: every Grok model in `model_capabilities.json` had `generation: 0`. On a Grok-only registry, no model passed the dimension threshold (50), the router fell through to tier-default selection, and tier default returned the first/weakest Grok in the tier (typically `grok-3`). The `generation` dimension is **TOOL-MEDIATED** in Forge — chat models invoke the `image_generate` tool (`agent/tools/image.rs`), which posts to xAI's `/v1/images/generations` with `model: "grok-imagine-image"`. The chat model's job is just to recognise intent and call the tool with a sensible prompt — basic tool-calling competency.

Calibrated generation scores reflecting tool-mediated capability:

| Model                          | Score | Rationale                                                                                                    |
| ------------------------------ | ----- | ------------------------------------------------------------------------------------------------------------ |
| `grok-4.3`                     | 72    | Top instruction (90), Owner's daily driver, slight edge over gpt-4o (70) so it wins ties on mixed registries |
| `grok-4.20-reasoning`          | 65    | Strong tool-caller, instruction 88                                                                           |
| `grok-4.20-non-reasoning`      | 60    | Same family, slightly lower reasoning                                                                        |
| `grok-4` / `-latest` / `-0709` | 55    | Older flagship, retiring 2026-05-15                                                                          |
| `grok-3`                       | 35    | Below threshold by design — basic tool-calling, falls back to tier default if it's all that's available      |
| `grok-3-mini`                  | 25    | Below threshold by design — lighter                                                                          |

The `dimensions.generation` description and `_meta.scoring_note` were both updated to document the tool-mediated semantics so future contributors understand why a model with no native image output (grok-4.3) can score above one with native DALL-E integration (gpt-4o).

**Article 1 PRESERVE — what was NOT done.** Owner initially green-lit removing the `id.contains("image")` filter from `GrokDiscovery` so `grok-imagine-*` models would surface in the registry. Investigation revealed this would expose **broken endpoints** to the chat orchestrator: imagine models live at `/v1/images/generations`, not `/v1/chat/completions`, and would 400 every time the auto-router picked one. Filter retained; defence-in-depth `None` return added to `classify_grok_context_window` for any imagine id that ever leaks through.

**Grok context windows — corrected against authoritative xAI source.** `agent::providers::registry::classify_grok_context_window` returned wrong values for half the Grok family. The most consequential miss: **`grok-4.3` was reported as 256k when xAI's pricing table lists it as 1M.** Owner noticed: "the context we quote is totally wrong. please correct it, grok 4.3 is 1m for example but many are wrong". Refetched `https://docs.x.ai/docs/models` (2026-05-14) and rebuilt the lookup against the actual pricing table:

| Model                                     | Was                           | Now                                                          |
| ----------------------------------------- | ----------------------------- | ------------------------------------------------------------ |
| `grok-4.3`                                | 256,000                       | **1,000,000**                                                |
| `grok-4.20-0309-reasoning`                | 256,000 (via grok-4 fallback) | **2,000,000**                                                |
| `grok-4.20-0309-non-reasoning`            | 256,000                       | **2,000,000**                                                |
| `grok-4.20-multi-agent-0309`              | 256,000                       | **2,000,000**                                                |
| `grok-4-1-fast-{reasoning,non-reasoning}` | 2,000,000                     | 2,000,000 (unchanged — was right)                            |
| `grok-4-fast-{reasoning,non-reasoning}`   | (no entry)                    | **2,000,000**                                                |
| `grok-4` / `grok-4-0709`                  | 256,000                       | 256,000 (unchanged — last published value before retirement) |
| `grok-code-fast-1`                        | 256,000                       | 256,000 (unchanged)                                          |
| `grok-imagine-*`                          | 256,000 (matched grok-4!)     | **None** (defence in depth — not chat-completions)           |

**Heads-up — xAI model retirement 2026-05-15 12:00 PT.** `grok-4-1-fast`, `grok-4-fast`, `grok-4`, `grok-code-fast-1`, and `grok-imagine-image-pro` retire on the day this release ships. xAI server-side redirects deprecated text-model slugs to `grok-4.3` (charged at grok-4.3 pricing). Forge keeps the retired entries in the lookup so any late requests still get a sensible context size; the auto-router's tier classification already favours `grok-4.3` when available.

**Tests (Owner UAT 2026-05-14 — all green):** 16 new `grok::tests` (whitelist coverage, ReasoningEffort/ToolChoice serialization, forbidden-params regression guard, `RequestOptions::default()` mutates nothing); 7 new `agent::providers::request_options_tests` (complexity-score-to-effort mapping at every band boundary, `ToolChoice::to_json()` shape pinning); 5 new `classify_grok_context_window` tests (1M for grok-4.3, 2M for grok-4.20 family, 2M for grok-4-1-fast family, PRESERVE for unaffected models, `None` for imagine variants); 8 new `classify_grok_capabilities` tests; `bundled_capability_scores_grok_generation_above_threshold` pinning the image-gen score floor.

### Frontend — agent-stream regressions fixed (F-067 + F-068 + voice-pill resume)

Three connected Owner-reported regressions were diagnosed and fixed in this branch:

**F-067 — voice chat history leaking across jobs.** The `auditDrivenAgentState` filter in `src/lib/stores/agent.ts` was too broad in freeform mode. Restructured the logic so the no-tag filter only applies in true freeform-voice mode, restoring correct bucketing for project / job contexts. `src/lib/stores/auditEvents.ts` updated to cross-bucket non-voice freeform events under the active voice session ID so freeform text-agent activity remains visible in the stream.

**F-068 — text-agent thinking disappears on reasoning models.** Diagnosed that reasoning models (e.g. `grok-grok-3`) put conversational content in `reasoning_content`, leaving `delta.content` empty. The frontend's `promote_thinking_to_response` was for the legacy store, causing the audit-driven UI to show blank messages where the prior session showed thinking. Fix: backend `CompletionResponse` extended with `reasoning_content`; providers accumulate it; `is_empty_response` updated; orchestrator `select_assistant_text` uses `reasoning_content` as a fallback so the model's actual conversational output reaches the UI.

**Voice / chat pills — resume the latest matching pill instead of accumulating new ones.** Owner UAT showed four pills accumulating in a single day from just navigating in and out of the same project. New `enterContext({ projectId, jobId })` function in `src/lib/stores/session.ts`:

1. Filters existing in-memory sessions by `(projectId, jobId)` match (both nullable; `null === null` so freeform vs project don't collide).
2. If one or more match, picks the highest `Session.number` (a monotonic creation counter — deterministic tiebreaker even when multiple sessions share a wall-clock start time), marks any other active session `completed`, flips the matched session back to `running`, and reuses its existing `id`.
3. If nothing matches, falls back to `startNewChat(context)` — old behaviour for first-time entry.

Wired in `src/routes/+page.svelte`. Freeform-entry path is intentionally NOT switched — it still calls `startNewChat() + agentStore.clearForNewChat()` because `clearForNewChat()` invokes `endConversation()` and the two don't compose; revisiting freeform clears the active conversation by design. The "create a brand-new chat" intent (HUMAN command, explicit Plus button) still calls `startNewChat()` directly. Audit chains stay intact — pills are a frontend display concept.

**Tests:** new `src/lib/stores/session.test.ts` with 9 regression tests (same-context resume keeps id, different-context creates new, freeform → project marks freeform completed, project → project resume picks the higher `number` deterministically when timestamps collide, no-match falls back to `startNewChat`). Frontend Vitest suite: **623 / 624 pass** (the single failure is a pre-existing `vcStateMachine.timers.test.ts` flake under parallel-build CPU pressure — passes 8/8 in isolation, file unchanged in this branch).

**Misc UI updates:** `LifecycleDashboard.svelte`, `ModelPanel.svelte`, `SessionBar.svelte`, `agentStream/{PromptPackageRow,SystemPromptRow}.svelte` aligned with the above. `lib/types/conflict.ts` extended for the conflict-system updates below.

### Version control — conflict enrichment polish + W5 commit metadata refinements

Continued cleanup from the per-hunk AI conflict enrichment work that landed in 6.7.0:

- `version_control/conflict/{analysis,bulk_resolve,commit_context,enrichment,enrichment_session,feature_flags,mod,summary,types}.rs` — touch-ups across the conflict pipeline: tighter `enrichment_session` lifecycles, cleaner `bulk_resolve` IPC contract, additional metadata on conflict summaries.
- `version_control/{git_ops,operations_test,state}.rs` — small fixes shaken out by the conflict-system polish.
- `ipc/version_control/{conflict,helpers,save,workspace}.rs` — IPC surface aligned with the new conflict types.
- `w5/commit_trailer.rs` — minor tightening on the `forge:w5_*` trailer parsing.
- `lib/types/conflict.ts` — frontend types extended (+62 lines) to mirror the backend changes.

### Gmail integration — auth + tool surface tightening

`integrations/gmail/{api,mod,oauth,tools}.rs` updated for cleaner OAuth flow, additional API coverage, and a tighter tool surface for the agent. Internal-only — no user-facing flow change.

### Audit + IPC + Sync infrastructure

- `audit/{events,mod,store}.rs` — new event types (Daily Flow phase transitions, persona output signals) and storage extensions to support them.
- `ipc/{arbiter,audit,environment,git_working_tree,incidents,mod}.rs`, `cs/profiles.rs` — IPC plumbing aligned with new domain events; misc fixes.
- `sync/{mod,pull,types}.rs` (+115 / +29 / +60) — sync layer extended for Daily Flow data.
- `agent/orchestrator/{display,escalation,events,hints,mod}.rs` — orchestrator updates for persona / Daily Flow integration; `display.rs` extended (+153 lines) for richer in-stream rendering of persona events.
- `agent/{system_prompt,tools/{baseline,mod,vc_conflict}}.rs` — tool surface extended (`agent/tools/mod.rs` +832 lines) for Daily Flow / persona tools; system prompt updates aligned with persona cast.
- `router/mod.rs`, `main.rs` — minor wiring changes for the above.

### Repo hygiene

- `.gitignore` extended with `.tmp_*` and `audit_*_dump.txt` patterns to prevent throwaway probe scripts and debug dumps from being staged accidentally (cleaned three such files from the working tree as part of this release).
- New utility scripts: `scripts/probe_xai_reasoning_effort.py` (read-only xAI capability probe — re-run before adding any new model id to the `reasoning_effort` whitelist) and `scripts/audit_today_full.py` (local-only audit DB dump utility).

### Project learnings captured

`PROJECT_LEARNINGS.md` extended (+69 lines) with two new entries:

- **Learning #41 — xAI modern API (`/v1/responses`) — strategic gap; X integration is a near-term marketing offering.** Frames the legacy `/v1/chat/completions` → modern `/v1/responses` migration as a strategic workstream blocked behind the marketing-management offering's spec. Sequencing recommendation locked by Owner.
- **Learning #42 — Verify docs by FETCHING, not by recall.** Article 6 (VALIDATE) failure analysis from this session: a docs claim about `grok-4.3`'s context window was asserted from recall instead of fetching `https://docs.x.ai/docs/models`. Discrepancy was 256k vs the actual 1M. Captured so the next agent doesn't repeat it.

## 6.7.2 (2026-05-12)

> **Note on 6.7.1:** A 6.7.1 tag was auto-created during a CI race but no release was ever published (the release-build job was cancelled by an immediately-following push). 6.7.2 is the first published successor to 6.7.0 and supersedes the orphan 6.7.1 tag. There are no 6.7.1 artifacts on `forge-releases`.

### Agent tooling — `read_file` returns full content (no truncation)

- **Removed the hardcoded 100 KB cap** in **`agent/tools/mod.rs::read_file`**. The previous behaviour silently truncated any file larger than 100 KB and appended a `[File truncated. Total size: ... bytes. ...]` tail with no agent-facing opt-out. This forced agents to assume content past the cut point — a direct **Article 2 (VERIFY)** violation that produced exactly the failure mode of agents inventing content they could not see.
- **New contract (Cursor-style):** full file by default, no truncation. **`offset`** (0-based starting line) and **`limit`** (max lines) remain available as **opt-in** pagination for agents that genuinely only want a slice of a very large file.
- **Tool description updated** so callers understand the new contract (no behavioural surprise; descriptions surface in the tool registry sent to LLMs).
- **Two latent panics fixed in the same function:**
  - **UTF-8 boundary panic:** the old `&content[..100_000]` byte-slice could panic on a multi-byte character boundary. Removed entirely with the cap.
  - **Out-of-range panic:** `lines[start..end]` panicked when `offset > lines.len()`. Now returns an empty string safely via `start >= total` short-circuit.
- **Tests (all green):**
  - **`test_read_file_returns_full_content_no_truncation`** — round-trips a ~250 KB file byte-exact, asserts no truncation marker.
  - **`test_read_file_offset_limit_returns_requested_range`** — `offset=5, limit=3` returns exactly the three requested lines.
  - **`test_read_file_offset_past_end_returns_empty_not_panic`** — `offset=999` on a 1-line file returns `""`, no panic.
  - **`test_read_file_offset_only_reads_to_end`** — `offset` without `limit` reads to EOF.
  - The three pre-existing `read_file` tests (param required, nonexistent file, schema) all still pass.
- **PR:** [#189](https://github.com/scainet-enterprise/scainet-forge/pull/189).

### Release pipeline — `ship-it` test stability

- **`FORGE_AI_CONFLICT_ENRICHMENT` env-var tests serialized:** `version_control::conflict::feature_flags` now guards temporary process-env mutation with a test-local mutex and restores the prior value even if an assertion panics. This removes the race where `env_falsey_values_disable_enrichment` could fail under the full parallel Rust test suite while passing in isolation.
- **Why it matters for this release:** `npx agent-excellence ship-it` runs the Rust suite before opening the release PR. The flaky env test was blocking the proper Ship It process, so this release includes the root fix instead of bypassing the guard.
- **Verification:** `cargo test --bin scainet-forge` passes locally (`1580 passed; 0 failed; 9 ignored`).

### Documentation — Daily Flow & Persona Cast S2 (G2-stamped)

- **`docs/S2-FF-DAILY-FLOW-AND-PERSONA-CAST.md` revision 2 — G2 (F&F Lock) stamped 2026-05-12.** Internal planning artifact, not a user-facing change, but included here so the release record reflects what shipped to `main`.
- Cuts at G2: Echo persona deleted (cast 7→6), reflective wrap-up survey removed, calibration suggestion engine deferred to v1.1+, avatars cut entirely from v1, voice-sample selection UX simplified to top-3 confirmation.
- Locks at G2: Daily Flow data is local-first SQLite synced via the existing **`src-tauri/src/sync/`** channel (no new transport); tier scope = paid Forge tiers only; Daily Flow is a Forge feature distinct from SCAINET tenant portals (`/staff`, `/customers`).
- **PR:** [#190](https://github.com/scainet-enterprise/scainet-forge/pull/190).

## 6.7.0 (2026-05-11)

### Merge conflicts — per-hunk AI enrichment and VC planning docs

#### Backend (Rust)

- **Per-hunk AI enrichment:** Structured conflict hunks feed a bounded hunk table in the LLM prompt (soft cap on hunks shown, snippet size caps, rationale length cap); model may return **`regions[]`** aligned to hunk indices with per-hunk strategy, confidence, and rationale; file-level rollup vs per-hunk guidance clarified in the prompt contract.
- **Types & parsing:** **`HunkAiRecommendation`** and related conflict analysis wiring; caps such as **`AI_CONFLICT_REGIONS_MAX_PARSE`**, **`AI_CONFLICT_HUNKS_PROMPT_SOFT_CAP`**, **`AI_CONFLICT_HUNK_RATIONALE_MAX_CHARS`**.
- **IPC:** Minor conflict / enrichment session hookups as needed for the new shape.

#### Frontend

- **`ConflictHunkPanel` / `ConflictDialog` / `ConflictSidebar`:** Per-hunk model-assisted rationale and confidence UX; clearer AI vs pattern badges and reading flow for rationales.
- **Telemetry:** **`conflictAiHunkDivergence`** (+ tests) for hunk/file recommendation drift visibility.
- **`TreeNode` / file explorer:** **`gitDecorationStore`** (ancestor roll-up for changed paths) and **`pathUtils`** (compare normalization, e.g. backslashes) with tests; row lookups stay in sync with absolute paths returned from **`list_directory`** and **`get_git_working_tree_status`**.

#### Git

- **Working tree (`get_git_working_tree_status`):** `git status --porcelain=v2` is invoked with **`-uall`** so untracked **directories** are expanded to **per-file** paths in porcelain (default Git only emits a single `? docs/` line, which left nested explorer rows undecorated). Rust unit test covers nested untracked paths in **`-uall`** form.

#### Documentation (working docs)

- **GitHub App → Firestore VC mirror:** New / updated **`S0`–`S4`** under **`docs/paul-working-docs/`** (`S*-FORGE-GITHUB-WEBHOOK-FIRESTORE-VC.md`) — shared **`repoMirrors`** model, **`branchDocId`** normalization (§4.7), resolver, security, and DAP tasks **GH-VC.\*** .
- **Per-hunk AI conflict enrichment:** **`S0`–`S4`** track (`S*-PER-HUNK-AI-CONFLICT-ENRICHMENT.md`); **enrichment cache / session reset** notes in **`S0-CONFLICT-AI-ENRICHMENT-CACHE.md`**.
- **AI conflict resolution enhancement** and **membership / RBAC** working docs refreshed; **`ACTIONABLE_IMPROVEMENTS`** and partial **VC abstraction** pointers updated.

## 6.6.0 (2026-05-10)

### Merge conflicts — AI-assisted resolution enhancement (S0–S4 working docs)

End-to-end improvements for smart conflict suggestions, manifest-driven guardrails, bulk resolve safety, and a clearer conflict UI. Work is aligned with **`docs/paul-working-docs/S0-AI-CONFLICT-RESOLUTION-ENHANCEMENT.md`** and the S4 DAP track.

#### Backend (Rust)

- **Conflict AI enrichment:** Per-file LLM enrichment after analysis; bounded prompts via **`complete_structured_short`** (max tokens / low temperature) to avoid runaway generation; configurable per-file timeout (**`AI_CONFLICT_FILE_TIMEOUT_SECS`**); racing/cancellation support; session token budget and enrichment progress in analysis.
- **Process kill switch:** **`FORGE_AI_CONFLICT_ENRICHMENT`** env gate to disable enrichment at the IPC layer.
- **`forge.project.json` — `ai.conflictResolution`:** Optional **`neverAutoResolve`** path globs (globset `**` semantics on the backend) and **`instructions`** merged into enrichment prompts; manifest parsing tests.
- **Bulk high-confidence resolve:** Respects manifest **`neverAutoResolve`** globs; optional post-bulk lifecycle test via **`run_lifecycle_test_command`**; frontend/backend types for follow-up test results.
- **Confidence for bulk apply:** **`merged_confidence_for_bulk_apply`** prefers model confidence when the model source succeeded (mirrors UI **`mergedConfidenceForDisplay`**).

#### Frontend

- **Settings:** New **Git** section for conflict AI — modern toggles; defaults **on** for suggestions, consent, and run-tests-after-bulk; user-facing copy (no internal S4/AI.x jargon).
- **Conflict dialog:** Unified suggestion card with **AI-assisted** vs **pattern-based** badge; AI-first displayed confidence with pattern fallback in the info popover; **`selectedFile` re-synced from store** when enrichment arrives so the detail pane updates without re-selecting the file; single global enrichment banner when no file is selected; modal stacking / scrim fixes for batch dropdown; **InfoPopover** fixed below the trigger and viewport clamped.
- **Progressive UX:** Header progress and loading copy while enrichment runs; sidebar AI loading affordance.
- **Types:** **`picomatch`** for manifest never-auto-resolve on the client; tests for bulk exclusion helpers.

#### Dependencies

- **`picomatch`** (+ **`@types/picomatch`**) for path glob matching in TS.

## 6.5.7 (2026-05-08)

Preserve the 507-line S2 Features & Functions draft for **Daily Flow & Persona Cast** that was sitting on a local-only branch ( + daily-flow-design + , originally committed 2026-05-04). Docs-only, single new file.

## 6.5.6 (2026-05-08)

### VC rail — tamper-evident audit (Phase 1)

- **`AuditEventType::VcRailOperation`:** Records outcomes for **`save_project`**, **`publish_project`**, and **`ship_project`** with a bounded JSON payload (operation, outcome, `result_type`, `error_class`, truncated `message_preview`, conflict path samples, counts — no full test output, PR URLs, or raw conflict file lists beyond caps).
- **`merged_tool_definitions` path unchanged** for VC; auditing is **`record_vc_operation`** (best-effort, non-blocking) from IPC wrappers after **`try_*`** inner functions return.
- **`ipc/version_control/helpers.rs`:** Payload builders, **`infer_vc_error_class`** heuristics (ordering fixes for GitHub vs validation), and unit tests for truncation, caps, and schema subset.
- **`audit/transcript.rs`:** User actor, summary line, and markdown icon for VC rail events in project transcripts.
- **Working docs:** `WORKING-VC-RAIL-AUDIT-INTEGRATION.md`, `S1` / `S2` / `S4` VC rail audit integration notes under **`docs/paul-working-docs/`**.

### Live Voice — macOS microphone (Info.plist)

- **`NSMicrophoneUsageDescription`** via **`src-tauri/Info.plist`** merged through **`bundle.macOS.infoPlist`** in **`tauri.conf.json`**, so WKWebView can expose **`navigator.mediaDevices`** for **`getUserMedia`** (fixes “Microphone unavailable” when the usage string was missing).
- **`webview_permission.rs`:** Comment clarifies macOS plist requirement vs WebView2 pre-grant on Windows.

### Hygiene

- **Unused import / re-export cleanup:** `catalyst/flows.rs` (test-only `Arc`/`Mutex`), `version_control/conflict/mod.rs` (unused hunk parser re-export), `w5/mod.rs` + `ipc/w5.rs`, migration and lifecycle tests; removes a batch of compiler warnings without behaviour changes.

## 6.5.5 (2026-05-07)

- Voice project integration

## 6.5.4 (2026-05-07)

This release **replaces the hand-rolled VC rail reducer with an XState v5 machine** (`vcRailMachine` + `createActor`) while preserving the existing Svelte store API (`send`, `init`, `refresh`, affordances). The migration spans **XS1–XS6** (model parity, production actor cutover, operation timeouts, CI polling extraction, centralized listener teardown); **6.5.4** also includes Rust publish/branch parity and the editor dirty-bridge follow-up below.

### Version control rail — full XState v5 migration (XS1–XS6)

- **XS1–XS2 — Machine + reducer parity:** `vcStateMachineModel.ts` holds **`transitionVC`** and related guards (`acceptsFileChangedState`, `hasCommitsReadyForPublish`, etc.). `vcRailMachine.ts` uses **`setup().createMachine`** with an **`active`** node; rail events **`assign`** through the same transitions as before.
- **XS3 — Actor + store adapter:** `vcStateMachine.ts` uses **`createActor(vcRailMachine)`**; public **`send`** → **`dispatchRail`**; **`REFRESH_MERGE`**, **`SHOW_CI_MODAL`**, **`HIDE_CI_MODAL`** avoid spurious history. **`destroy()`** stops the actor; **`dispatchRail`** restarts if stopped (Vitest lifecycle).
- **XS4 — Operation timeouts:** Parallel **`opTimers`** region, numeric **`after`** delays, **`raise`** for **Save / Publish / Ship / shipped** timers; **`OP_TIMER_SYNC`** in the model; history for actor-driven timeout events (**`historyEventForActorDrivenRailTransition`**).
- **XS5 — CI polling:** **`vcCiPolling.ts`** (`createVcCiPolling`) — interval, **`pollInFlight`**, max duration, **`ci:status-changed`**; store **`startPolling`** / **`clearTimers`** / **`reset`** / **`destroy`** wired.
- **XS6 — Teardown:** **`teardownCiAndExternalListeners()`** centralizes detach for CI schedule, editor-dirty, project context, Tauri listeners; **`destroy()`** uses it.
- **Tests:** **`vcRailMachine.test.ts`**, **`vcStateMachine.test.ts`** (incl. XS4 ordering constraints), **`vcStateMachine.timers.test.ts`** (`vi.resetModules()` isolation), **`vcCiPolling.test.ts`**, destroy/reset lifecycle coverage.
- **Developer tooling (XState):** **`@statelyai/inspect`** (devDependency); **`forwardVcRailInspect`** from **`createActor(..., { inspect })`**; **`window.__forgeStatelyVcConnect?.()`** opens the browser inspector (dev, allow popups). **`src/app.d.ts`** declares the hook.

### Version control — publish / project-state parity (Rust)

- **`get_commits_since_from_branch`:** Commits-ahead vs **`origin/staging`** can target the **canonical** branch (`lifecycle_projects.working_branch`) instead of only worktree **`HEAD`**, so the pre-publish gate matches the GitHub PR head (avoids **422** _No commits between staging and forge/…_ when checkout and DB branch differ).
- **`get_project_state`:** Reads **`working_branch`** from SQLite and passes it into **`detect_project_state`** when **`refs/heads/<branch>`** exists so **`commits_ahead_of_staging`**, unpushed counts, and Publish affordances align with the same branch **`publish_project`** uses.

### Editor → VC — dirty bridge after save / ship

- **`forge:editor-dirty`:** If the post-**`forge:file-saved`** suppress window drops a **`dirty: false`** dispatch, a **deferred retry** re-reads **`anyDirtyBuffer`** so **`dirtyBridgeLastDispatched`** cannot stay stuck; **Save** can appear again after **Ship** / **Save** flows when the user edits.

### Documentation

- **`WORKING-VC-PIPELINE-FSM-ANALYSIS.md`:** XS1–XS6 XState migration, timers, history, CI polling, teardown, and follow-on items.
- **`S4-VC-FSM-XSTATE-MIGRATION.md`:** Detailed action plan / execution summary for the migration phases.

## 6.5.3 (2026-05-05)

### Version control — affordances, editor dirty state, and manifest naming

- **Affordances façade (`vcAffordances`):** Derived store consolidates Save / Publish / Ship enablement with IPC-backed `projectStateCache` and explicit `blockedBy` reasons.
- **P2c-B03 — Publish:** `canPublish` combines FSM state with backend `hasUnpublishedWork` via `canPublishWithIpc`; `CI_FAILED` remains actionable per FSM rules.
- **Rail idempotency:** `railLocked` / early-return guards on VC actions to avoid duplicate in-flight IPC.
- **`forge:editor-dirty`:** Debounced (`500ms`) `window` events from the editor bridge; FSM listens and forwards **`FILE_CHANGED`** when the state accepts local changes; suppression during **`SAVING` / `PUBLISHING` / `SHIPPING`** and a short window after **`forge:file-saved`**.
- **`unsaved_buffers`:** `anyDirtyBuffer` adds **`blockedBy`** and gates **`canPublish`**; **Publish** tooltips for **`unsaved_buffers`**, **`open_pr`**, **`squash_merged_no_new_save`**.
- **Save before Git:** **`flushDirtyTabsUnderWorkingDir`** (path-safe under working dir) runs before **`save_project`** so disk matches open buffers.
- **Tests:** `isPathUnderWorkingDir`, `acceptsFileChangedState`, `vcAffordances` dirty-buffer behaviour.
- **Docs:** `S0`–`S4` for **forge:editor-dirty**; **`docs/architecture/vc-affordances.md`** — editor dirty buffers; **`WORKING-VC-PIPELINE-FSM-ANALYSIS.md`** updated for completed items.

### Project creation — `forge.project.json` (shared repo)

- **Workspace manifest:** If **`forge.project.json`** already exists (e.g. cloned repo), it is **loaded and preserved** instead of overwritten.
- **`name` field:** New manifests use the **GitHub repository name** parsed from **`repo_url`** (stable when several Forge projects share one repo); falls back to the user **project name** when the repo cannot be parsed. Avoids UUID worktree folder names and conflicting **`name`** values across branches.
- **`parse_github_url`** is evaluated before manifest creation so owner/repo metadata and manifest naming stay aligned.

### New project flow

- **Custom repository naming** and related UX in the new-project flow.

## 6.5.2 (2026-05-05)

### Project picker — Recent by last opened

- **DB:** Migration adds `last_opened_at` on `lifecycle_projects` (schema v38).
- **IPC:** `list_user_projects` returns `lastOpenedAt`; new **`touch_project_last_opened`** updates open time with tenant visibility checks.
- **Frontend:** `isTouchEligibleProjectId` helper; **`projectContextStore`** and resume path call **`touch_project_last_opened`** for lifecycle project IDs (not freeform / `local-*`).
- **`ProjectPicker.svelte`:** **Recent** section shows up to five projects with a recorded last-open, sorted by **`lastOpenedAt`** (tie-break by id); **All projects** lists the rest; **`openProject`** always dispatches **`projectId`** for consumers.
- **Just start building:** Enters freeform immediately using the same default working directory as the existing “use default folder” path; **Start in a different folder…** opens custom folder setup.
- **Docs:** Working docs **`S0` / `S1` / `S2` — Project Picker recent by last opened** under **`docs/paul-working-docs/`**; lifecycle doc reorg (moves completed SQLite/router artefacts; **`S4-SQLITE-ACCESS-ARCHITECTURE`** retired from active stack).

### Ship — recoverable FSM (no stuck “Shipping…” spinner)

- **`ShipButton.svelte`:** Every **`ship_project`** outcome exits **`SHIPPING`** via **`SHIP_SUCCESS`** or **`SHIP_FAILED`** (no reliance on **`refresh()`** or **`CI_POLL_RESULT`** while shipping, which the FSM ignores).
- **`no_pr`:** Optional **`get_project_state`** for **`prStatus`** — toast **“Review not merged yet”** when review is **open**; generic **“No active review”** otherwise; always **`SHIP_FAILED`**.
- **`ci_pending` / `ci_failed`:** **`SHIP_FAILED`** + toasts (parity for future backend variants).
- **`conflict_with_main` / `syncWithMain`:** **`updated`** path sends **`SHIP_FAILED`** instead of **`refresh()`**; **up_to_date** after sync sends **`SHIP_FAILED`** so the user can retry; defensive **`default`** arm for new Rust result types.
- **Tests:** **`vcStateMachine.test.ts`** documents **SHIPPING** ignoring stray events; **`ShipButton.svelte.test.ts`** covers ship result → FSM contract.
- **Docs:** **`S0` / `S1` / `S2` — Ship spinner / FSM stuck** under **`docs/paul-working-docs/`**.

## 6.5.1 (2026-05-04)

- Continuous improvement 2026 05 04

## 6.5.0 (2026-05-04)

### Non-blocking Portal sync (Project Picker)

- **`ProjectPicker.svelte`:** Separates cheap local **`list_user_projects`** from slow **`lifecycle_sync_portal`** so opening the picker, deleting a project, or linking a folder is no longer blocked on a full N-project Portal pull.
- **`reloadLocalProjects`**, **`schedulePortalReconcile`**, **`flushPortalSegments`:** Serialized promise tail so **`lifecycle_sync_portal`** never overlaps in-flight; **`forcePortalSync`** coalesces after the current segment; **`pickerMounted`** + **`onDestroy`** prevent stale mutations after teardown.
- **UI/a11y:** Header **“Syncing with cloud…”** / **offline** badges, **`prefers-reduced-motion`**, debounced **`aria-live="polite"`** announcements, **`aria-busy`** on the local-loading skeleton (**Article 1 preserved:** row delete **`disabled`** while **`deletingId`** is set).

### Lifecycle documentation

- Working docs **`S0`/`S1`/`S2`/`S4` — Project Picker Portal sync UX** under **`docs/paul-working-docs/`**; **`FORGE_ARCHITECTURE_DIAGRAMS.md`** — picker local list vs reconcile path.

### Dependencies & Dependabot reliability

- **Tauri 2.11 alignment:** **`tauri`** (Rust crate) **→ 2.11.0** with **`@tauri-apps/api`** and **`@tauri-apps/cli`** **→ 2.11** so **`npm run tauri build`** no longer fails on **minor-version mismatch** (regression from NPM-only bumps).
- **Dependabot:** Ignores **`@tauri-apps/plugin-*`** (npm) and **`tauri-plugin-*`** (cargo) alongside core **`tauri`** / **`@tauri-apps/api`** / **`@tauri-apps/cli`**; **`tauri-plugins`** auto-merge groups removed so coordinating upgrades stays manual **in one PR**.

## 6.4.0 (2026-05-04)

### W5 commit metadata in Git and conflict brief 1.1

- **Save → Git:** When a project has a lifecycle stage and W5 snapshot or recent decisions, Forge loads a compact **W5 trailer draft** in the same DB read as project save prep and writes **`forge:w5_*`** trailers (mission, stage, decisions, snapshot timestamp). **`forge:intent`** is not written on that path when W5 trailers are present; the subject line stays the file-based summary so chat intent does not override W5 semantics.
- **Merge conflicts:** The JSON conflict **`brief`** is now **`version` `1.1`**. Each of **`local_commit`** / **`incoming_commit`** may include **`context_kind`** (`w5` \| `intent` \| `none`) plus optional **`w5`** / **`intent`** parsed from Forge trailers. **`vc_show_conflicts`** tool description documents these fields for models.
- **Implementation:** New modules **`w5/commit_trailer`** (**`load_commit_w5_draft`**, Unicode-safe caps) and **`version_control/conflict/commit_context`** (**`extract_commit_context`**); writer/reader folding rules marked with **`// SYNC:`**. **`truncate_message`** in the conflict brief uses scalar-aware truncation.

### Orchestrator reliability

- **xAI tooling:** **`llm_error_suggests_conversation_repair`** no longer treats **“maximum tools”** / tooling-limit errors as conversation-shape failures that merit an automatic repair retry.

### Documentation

- Lifecycle working docs **S1–S4** and raw research for W5 ↔ version-control integration; **FORGE_ARCHITECTURE_DIAGRAMS** and related working-doc updates.

## 6.3.4 (2026-05-03)

Removes `.github/workflows/mirror-release-manual.yml` — a `workflow_dispatch`-triggered release workflow I introduced in #167 and refined in #168 without it being raised, planned, or approved as a new piece of release infrastructure.

## 6.3.3 (2026-05-03)

Direct follow-up to #167. The v6.3.1 manual mirror surfaced the exact edge case I called out as known-follow-up in that PR: a previous `gh release create` succeeded server-side but its response was eaten by a 5xx, leaving the public release present with **zero assets uploaded**. The retry's "release exists, exit success" check then preserved that empty shell on every subsequent re-run.

## 6.3.2 (2026-05-03)

- Adds a 5-attempt exponential-backoff retry around `gh release create` in the `mirror-release` step of `build.yml`. Detects "succeeded but response lost in flight" via `gh release view` between attempts.
- Adds a new `mirror-release-manual.yml` workflow (`workflow_dispatch`-only) that re-runs the mirror for any tag, downloading assets from the matching private release. Permanent escape hatch for runs that failed under the old (no-retry) workflow — re-running a failed Actions job uses the YAML at the original commit, so this manual path is the only way to recover historical failures.

## Unreleased — continuous improvement (forge-continuous-improvement-2026-05-04)

### UI palette: replace silently-broken Tailwind class names

**What changed.** Two CSS class names referenced across 18 Svelte components did
not exist in `tailwind.config.js`:

- `bg-forge-bg-primary` (intended: page / modal background)
- `bg-forge-bg-secondary` (intended: elevated surface — input fields, list rows, panels)

The actual palette tokens defined under `theme.extend.colors.forge` are:

- `forge.bg` (`#0D1117`) — page background
- `forge.surface` (`#161B22`) — elevated surface
- `forge.border` (`#30363D`)
- `forge.text.{primary,secondary,muted}`
- `forge.accent.{blue,cyan,purple,green,yellow,red}`

Because Tailwind silently produces no CSS for undefined utility class names,
every `bg-forge-bg-primary` / `bg-forge-bg-secondary` was a no-op. On `<div>`
elements this rendered as transparent (visually acceptable over the dark
backdrop), but on `<input>` / `<textarea>` / `<select>` the user-agent default
white background won — making fields appear as solid white rectangles with
black-on-near-white text that was unreadable in the dark theme. Most visible
in the Job Detail "Edit task" / "Add task" / "Task detail" modals, but also
present in: AgentStream, LifecycleDashboard, ProjectDetailView,
NotificationBell, DocumentViewerModal, UndoToast, TechniqueLibrary, LostButton,
BriefingView, ConfirmDialog, GuidedTour, LaunchPad, McpSettings,
PostStageReview, GateApproval, ActionCardModal, TelemetryConsent.

**Fix.** Global rename across all 18 files:

- `bg-forge-bg-primary` → `bg-forge-bg`
- `bg-forge-bg-secondary` → `bg-forge-surface`
- `hover:bg-forge-bg-secondary` → `hover:bg-forge-surface`

No new tokens were introduced — fix uses the existing palette.

**Downstream watch list.** Components that previously rendered transparent on
`<div>` will now have their intended dark surface colour. If any UI looks
visually different after this change (slightly darker panel where there used
to be no fill), trace back to this CHANGELOG entry. The `<div>` cases are
expected to look _more_ like the original developer's intent. Form fields
(inputs / textareas / selects) are the cases where the change is functional,
not just cosmetic.

**Out of scope.** A handful of CSS `var(--forge-bg-elevated, #1e293b)`
references in `ConflictDialog.svelte`, `ConflictDiffViewer.svelte`,
`CIFailureModal.svelte`, `BranchCleanupDialog.svelte` — those use a CSS-custom-
property fallback that already renders the intended colour without touching
the Tailwind palette. Left as-is to avoid unrelated visual churn.

### Job Detail UX

- Per-task 3-dot menu on every task row (was `pending_review`-only).
  Status-aware actions: Approve / Start / Mark complete / Mark failed /
  Re-open / Retry, plus universal Edit / Drop. Menu uses `bg-forge-bg` and
  a thicker cyan border to clearly occlude the task list behind it.
- Display labels: `pending` → "ready to action", `in_progress` → "in progress"
  in badges, status dropdown, menu actions, confirm dialogs and toasts. DB
  status values are unchanged (still `pending` / `in_progress`); display
  layer only.
- Lock-failed regression after job revert (F-008 follow-up). `job_lock_plan`
  IPC was unconditionally re-INSERTing draft-derived tasks, colliding on
  `JOB-XXX::TASK-NNN` PK because revert preserves the original tasks (now
  in `pending_review`). IPC now only seeds tasks from the draft when the job
  has zero existing tasks; re-approval after revert trusts the curated rows
  and `lock_job_plan_async` flips any `pending_review` back to `pending`.
- Backend additions: `job_approve_task`, `job_update_task_content`,
  `job_drop_task` IPC commands + `update_job_task_content_async` /
  `delete_job_task_async` engine methods. All emit `tasks:changed` for F-015
  UI auto-refresh.

### Voice persona (F-017 reopened)

- Added a single rule under "Conversational Style" in
  `persona/voice_conductor.rs` instructing Clara not to overuse the user's
  name (introductions / formalities only — once or twice per conversation).

## 6.3.1 (2026-05-03)

- voice connect ipv4-first + bounded TCP, eliminates 43s startup freeze

## 6.3.0 (2026-05-03)

### FORGE Incident Store (local-first telemetry)

This release ships the **FORGE-INCIDENT-STORE** MVP: structured crash/incident capture with a **local SQLite queue** (`incident_cache` in `ego.db`), optional **Firestore** writes to `incidents/{tenantId}/events/{eventId}`, and a **Sentry mirror** only after a successful Firestore write. **Logger** paths with an `Error` argument route through **`captureIncident`** (non-throwing); direct **`captureException`** from the logger is removed in favour of the incident pipeline.

#### Pipeline and storage

- **`$lib/incidents`:** normalise + scrub, SHA-256 **fingerprint**, envelope build, size caps, **Firestore adapter**, **TauriIncidentLocalStore** (`invoke` with **`payload`** wrappers matching Rust IPC), **FirestoreIncidentStore**, **SentryIncidentMirror**.
- **`IncidentService`:** consent (`crashReportsEnabled`), tenant / **`auth_required`**, **`no_consent`**, exponential backoff and permanent Firestore failure handling; **`processQueue`** for retries.
- **Sync worker:** interval + **`visibility`** handling, HMR guard, registered service getter to avoid circular imports.
- **SQLite migrations:** **v34** `incident_cache`; **v35** idempotent repair for **`lifecycle_projects`** prompt-injection v2 columns when schema drifted; **v36** rebuild so **`sync_status`** may be **`dev_disabled`** (CHECK constraint).
- **Tauri:** `incident_cache_insert`, `incident_cache_list`, `incident_cache_update_status`, `incident_cache_list_needing_sync`.

#### Product and developer experience

- **Settings → Diagnostics:** local crash report list and **Retry sync now**.
- **Dev cloud gate:** In development, **Firestore and Sentry** are **off** unless **`VITE_FORCE_SENTRY=1`** (same opt-in as Sentry SDK init). Local rows use **`dev_disabled`** so shared staging/prod incident data is not filled by default dev smoke tests. Production behaviour is unchanged (consent + tenant gate cloud).
- **`window.forgeDevSmokeIncident()`** (dev builds): console-friendly smoke trigger without **`$lib`** imports.
- **`captureException`** returns **`boolean`** so the mirror can record when Sentry did not actually send; **`__sentryStatus`** helper updated for clarity.
- **`ProjectPicker`:** **`portalProjectSyncThrottle`** — **`lifecycle_sync_portal`** at most once per 60s unless a mutation forces refresh (link folder, delete project, etc.).
- **`tryGetTenantId`** (non-throwing) on **`scoped-query`** for incident gating.

#### Docs and ops

- Working docs: **FORGE-INCIDENT-STORE-S2/S4** updates (`sync_status` **`dev_disabled`**, §1.5 dev telemetry, acceptance **A10**); **FURTHER_INVESTIGATION_REQUIRED** observability section; **TECHNICAL_DEBT** incident follow-ups.
- **Portal `firestore.rules`:** tenant-scoped **`incidents/{tenantId}/events/{eventId}`** rules are tracked in the Portal repo (separate PR); Forge assumes rules allow authenticated tenant writes and deny client **delete** (immutability / Admin SDK for erasure).

#### Verification (pre-merge spot-check)

- `npm run test:run -- src/lib/incidents` — Vitest incident suite green.
- `cargo test --bin scainet-forge incident_cache` — SQLite **`incident_cache`** / v36 migration tests green.
- Full repo verification: `npm run test:run`, `cargo test --bin scainet-forge`, `npm run verify-version` as required by your release gate.

#### Release files changed

- `VERSION` → 6.3.0
- `package.json` / `package-lock.json` → 6.3.0
- `src-tauri/Cargo.toml` / `Cargo.lock` / `tauri.conf.json` → 6.3.0

## 6.2.0 (2026-05-02)

### Voice Conductor "Clara" + Single-Mutator Lifecycle Architecture

Forge 6.2.0 is a substantive UX release built around three architectural shifts plus sixteen issue closures from a full UAT day on the `forge-ux-improvements-2026-05-02` branch. The minor-version bump reflects the new voice persona ("Clara"), the new context-delivery mechanism for the live xAI realtime model, the new single-mutator lifecycle context architecture, and the new recent-conversation injection system that finally gives the voice agent durable memory across context switches and app restarts.

This release is the answer to the entire 6.1.x next-branch list: F-037, F-036, F-034 (architecture in place — pill scoping itself stays open), F-021, F-009, F-027 stragglers, and the voice prompt overhaul the prior handover called out as "the one you wanted to discuss".

#### Voice Realtime Context Delivery (F-037 — Critical, Closed)

The flagship fix. The orchestrator already emitted fresh `system_prompt` + `prompt_package` audit rows on project/job switches, but the live xAI realtime model continued answering from the previous context — only acknowledging the change after explicit user correction. Root cause: `session.update.instructions` updates the model's _configuration_ but does not insert anything into the conversation history the model is actually attending to.

- **Context-reset injected as `conversation.item.create`** — `voice/session.rs::apply_mode_flip` now sends three frames on every context flip: (1) `session.update` with the rebuilt prompt, (2) `conversation.item.create.context_reset` with a clear "[CONTEXT UPDATE] You are now in <project/job label>" system-style item, (3) `response.create` to acknowledge. Verified live by Clara responding to context updates without prompting ("Thanks, I've absorbed the context...").
- **Mode-suffix removed from the reset message** — `build_context_reset_message` no longer hardcodes "in job mode" / "in project mode" (F-054), which previously crossed wires when a lifecycle project was framed as a job. Negative assertion added to the F-037 regression test.
- **Voice transport audit attestation** — Every WebSocket frame is now audited as a `voice_transport` event with payload type, byte budget, and rendered hash so future regressions in this class are diagnosable from the audit chain alone (F-042 plumbing partially landed).

#### Clara — Voice Conductor Persona Overhaul (F-062 — High, Partial/Verified)

The voice prompt was bloated with text-agent Constitution tables, Rust crate references, contradictory Mode B / JOB MODE blocks, and PNEUMA tells that hurt spoken flow. Replaced with a structured ~700-word Clara persona authored with the owner.

- **`voice_conductor.rs`** — `VOICE_CONDUCTOR_NAME = "Clara"`, `VOICE_CONDUCTOR_ROLE = "FORGE Voice Conductor"`. New `VOICE_CONDUCTOR_DESCRIPTION` covers Identity & Stance, Hard Rules, Verbal Formatting (read project names, not codes), Verification, Conversational Style, and Boundaries. Tests updated.
- **`agent/system_prompt.rs`** — `build_voice_system_prompt` restructured. Mode sections renamed `Active Mode: Freeform` and `Active Mode: Lifecycle Project`. Mode-A explicitly tells Clara _"Do not invent or assume a project context"_ — fixes F-041 (freeform fabrication) on the voice surface to match the existing text-side `freeform_preamble`.
- **Placeholder-owner normalization extended** — `dispatch.rs` and `agent/tools/mod.rs` now treat `"clara"` and `"clara (forge)"` as placeholder owners so audit rows attribute correctly when the model self-identifies.

#### Recent Conversation Injection — Durable Voice Memory (F-063 + F-065 — High, Closed)

A voice session that didn't remember what was said five minutes ago, let alone last week, was the single largest reason users felt the agent was not really an agent. Fixed by injecting the last N audit-chain exchanges into every voice session as a `conversation.item.create.recent_context` frame.

- **`voice/recent_context.rs`** — `RECENT_CONTEXT_TOKEN_BUDGET` raised from 2 000 to 6 000; new `RECENT_CONTEXT_KEEP_LAST = 10` floor guarantees a minimum recall window even under tight budgets.
- **`fetch_for_active_project` rewritten** — Was querying by `session_id` (which dies on app close) and returning 0 rows for any cross-session recall. Now uses `audit::events_for_project` (payload-based query). UAT 12:33:18 PROJ-MORPHEUSINT recalled an S0 lock from April directly, no tool call needed (F-065).
- **`fetch_for_active_job` added** — Mirrors the project path. UAT 12:09–12:13: 4 job switches all hit `rows_injected=40` with Clara recalling specific prior content ("we were analyzing the images Grok 3 created", "you were frustrated last time").
- **Cold-start + mid-session symmetry** — `ipc/voice.rs::voice_realtime_start` injects on startup; `voice/session.rs::apply_mode_flip` injects on every mid-session switch. The audit emits a third `voice_transport` frame so injection is independently observable.

#### Single-Mutator Lifecycle Context Architecture (F-058 — Critical, Fix Landed)

The `set_work_context` agent tool reported success but did not actually clear the lifecycle backend. Five different code paths could mutate `ActiveWorkContext` independently — store-only updates, IPC-only updates, tool-only updates — leading to drift between the UI, the backend, and the voice session. Replaced with a single mutator.

- **New `work_context::apply` is the sole writer** — `work_context_changed` audit variant + `forge:work_context_changed` Tauri event are emitted _before_ the function returns, with `src` distinguishing `ipc_ui`, `tool`, `voice`, `internal` callers.
- **5 call sites refactored** — `set_active_work_context` IPC, `clear_work_context` IPC, `set_work_context` tool, project/job navigation, session-pill ✕ button.
- **F-059 (SessionBar pill ✕)** and **F-060 (project click)** verified in UAT 10:13–10:28: every interaction fires the correct `work_context_changed src=ipc_ui` row; user verdict 10:28:42: _"working as expected"_. F-053 closed structurally by the same change.
- **F-036 (project entry should auto-trigger context)** is now literally how it works — project card click _is_ the lifecycle context update.
- **F-048 (mode-flip waits for Let's Go)** is closed because mode-flip fires on the project click itself, not on a follow-up button.

#### Agent Stream UX Fixes (F-039, F-044, F-045, F-047, F-049 — Critical/High/Medium)

- **F-039 (Critical, Closed)** — Start Text no longer leaves the user stuck on the splash. `lifecycleModeStore.userSessionStarted` flag introduced; `InstructionPanel.isFreshSession` now also gates on `!userSessionStarted`. Both Start Text and Start Voice handlers call `markUserSessionStarted()` for defensive symmetry. Reset by `clearWorkContext()` so "New Chat" brings the splash back. 3 new unit tests, all 30 lifecycleMode tests green.
- **F-044 (Critical, Closed)** — Voice session text (both user and assistant) now renders in the Agent Stream when freeform. `auditEventsStore.clearActiveContext` added. UAT 08:23–08:26 verified end-to-end.
- **F-045 (Critical, Fix Landed)** — "New Chat" no longer produces a blank window in freeform voice mode. Closed structurally by F-044 + F-047, verification deferred to first 6.2.0 freeform UAT.
- **F-047 (High, Closed)** — Project→project navigation no longer renders a blank stream until the first interaction. Per-bucket loading state added to `auditEventsStore`; `AgentStream.svelte` distinguishes loading-replay / no-prior-conversation / error empty states. UAT 08:37–08:41 verified across three projects, including one untouched for ~3 weeks.
- **F-049 (Medium, Fix Landed)** — `SystemPromptRow` and `PromptPackageRow` rewritten to surface persona, mode, context, size, and short hash as a parenthetical instead of bare hash strings. `FILTER_STORAGE_KEY` bumped `v1 → v2` to reset stale filter values from prior builds. Visually verified in UAT screenshot — rows now read _"System prompt — Clara (voice) · Mode B · project PROJ-MORPHEUSINT · 12.9 KB (8511c7d9)"_.

#### "Let's Go" Button Cleanup (F-061 — Medium, Fix Landed)

After F-058 + F-060 made project click the canonical context update, the "Let's Go" button became a redundant double-fire and then, after stripping the redundancy in voice mode, a no-op the user could click without feedback.

- **Option B implementation** — Redundant `set_active_work_context` IPC call removed from `ProjectDetailView.svelte::startWorking` and `JobDetailView.svelte::startWorking`. The button is retained for now because it still primes the lifecycle persona and submits stage-start instructions for text agents.
- **No-op-suppression CTA arm** — In voice mode on a project detail view, the CTA renders as a disabled cyan pill (_"Voice Active — talk to begin"_ / _"Voice Active — speak to start working"_ / _"Voice Session Active — interact via voice"_ by experience level) instead of a clickable button that does nothing. JobDetailView already handles this via its existing `workStarted` swap to _"Working via Voice — speak to the conductor"_.
- **F-068 design captured** — Auto-prime on first text entry, mandatory persona, full button removal. Top priority next branch alongside F-035.

#### Closures Earned Without Code (F-009, F-017, F-021, F-033, F-036, F-041, F-048, F-050, F-055)

Eight items closed structurally because the architectural work above subsumes them:

- **F-009** (cross-session voice continuity) — F-062 cold-start + F-063 mid-session + F-065 payload-based query give continuity across switches AND across app launches.
- **F-017** (voice overuses user's name) — Won't Fix (test artifact); user confirmed cause was explicit self-introduction priming the model.
- **F-021** (system messages unanswered) — superseded by F-037 + F-063; Clara now visibly responds to context updates.
- **F-033** (bottom-bar "Project Start S0" artefact) — closed pending re-report; not seen since F-058/F-060 landed; user 2026-05-02: _"I think it is gone"_.
- **F-036** (project card auto-context) — implemented by F-058 + F-060.
- **F-041** (freeform fabrication) — both surfaces fixed (text-side `freeform_preamble`, voice-side new "Active Mode: Freeform" block).
- **F-048** (mode-flip waits for Let's Go) — gone by construction (F-058 + F-060 + F-061).
- **F-050** (voice conductor has no formal name) — superseded by F-062 (Clara). Customisable-persona-with-sliders direction split off as F-067 (Low) so that scope is explicit without holding F-050 open.
- **F-055** (Let's Go button rename) — subsumed by F-061 + F-068; renaming becomes irrelevant once the button goes.

#### Verification Snapshot

- `npm run verify-version` → OK, all anchors match `VERSION = 6.2.0`.
- `cargo test --bin scainet-forge` (1 thread) → **1459 passed, 0 failed, 9 ignored**.
- `npx vitest run` → **489 passed, 0 failed** across **44 test files**.
- `agent-excellence validate` → PASSED with 17 framework-drift warnings (no functional impact).
- Live UAT day 2026-05-02 across freeform voice, lifecycle voice, project switching, job switching, session pills, text input, "Let's Go" button, audit-stream rendering, and recent-context recall (PROJ-MORPHEUSINT, PROJ-OPENCLAWGAP, PROJ-LIGHTHOUSE, JOB-009, JOB-013, JOB-015 + multiple ad-hoc projects).

#### Release Files Changed

- `VERSION` → 6.2.0
- `package.json` / `package-lock.json` → 6.2.0
- `src-tauri/Cargo.toml` / `Cargo.lock` / `tauri.conf.json` → 6.2.0
- `docs/CHANGELOG.md` → 6.2.0 release notes
- `docs/HANDOVER_2026-05-02.md` → updated with end-of-branch state
- `docs/KNOWN_ISSUES_2026-05-02.md` → final status snapshot

#### Known Follow-ups (Next Branch)

The following are intentionally **not** fixed in 6.2.0 and form the recommended order for the next branch:

1. **F-035 (Critical)** — Text input dropped in voice Mode A / no-context. Top priority. Pairs with F-068.
2. **F-068 (High)** — Auto-prime stage on first text entry; remove "Let's Go" button entirely; mandatory-persona architecture. Pairs with F-035 (same input handler).
3. **F-046 reopened (Medium)** — Voice agent still reads stage/gate codes verbatim from `lifecycle_status` / `job_status` tool output. Structural fix is to reshape tool output to translate codes before they reach the model.
4. **Job CRUD pack** — F-006 (create from UI) / F-007 (delete) / F-008 (move stage backwards) deferred as a focused mini-pack.
5. **F-064 (High)** — `lifecycle_list` returns empty + voice agent has no project/job SEARCH tool. Add `lifecycle_search(query)` + `job_search(query)`.
6. **F-004 (Critical)** — Document scope leak. Carried forward.
7. **F-005 (Critical)** — GitHub-optional local-first project creation. Carried forward.
8. **F-034 (Critical)** — Session pill switching (filter wiring); F-056 + F-057 (architecture: pills are freeform-only + freeform→project/job conversion tools) are the structural answer.

## 6.1.2 (2026-05-01)

### Mirror Recovery Republish

Forge 6.1.2 republishes the 6.1.1 stabilization release with all four build platforms intact on the public `forge-releases` mirror. The 6.1.1 mirror job ran while a transient Apple `timestamp.apple.com` outage was still preventing the macOS ARM64 codesign step from completing, so the public release was created with only Linux, Windows, and macOS Intel artifacts. Apple Silicon users on the in-app updater would not have received a working `aarch64` entry for 6.1.1.

This release contains the same product changes as 6.1.1 (see notes below) and re-runs the full build/mirror pipeline so all four platforms publish together.

#### What 6.1.2 fixes vs 6.1.1

- **Apple Silicon (`aarch64`) installer published** — `SCAINET-Forge_6.1.2_aarch64.dmg`, `SCAINET-Forge_aarch64.app.tar.gz`, and `SCAINET-Forge_aarch64.app.tar.gz.sig` are now part of the public release alongside the Intel, Linux, and Windows artifacts.
- **`latest.json` now lists `darwin-aarch64`** — The auto-updater manifest on `forge-releases` now resolves correctly for Apple Silicon users running 6.1.0 and earlier.
- **No code changes vs 6.1.1** — Functional product surface, agent-stream behaviour, project usability fixes, and orchestrator context isolation are identical to 6.1.1.

#### Carried forward from 6.1.1

All 6.1.1 fixes ship in 6.1.2 unchanged. See the 6.1.1 section below for the detailed breakdown of agent-stream reliability, project view scrolling, text-agent context isolation, regression coverage, and the next-branch handover.

#### Known Follow-ups

The 6.1.1 follow-up list still applies to 6.1.2:

- **F-037 — Voice realtime context delivery**: context updates are audited correctly, but the live voice model can still answer from its prior job. Next-branch fix.
- **F-036 — Project entry should replace "Let's go"**: project-card click should auto-serve lifecycle context to the live voice session.
- **F-034 — Session-pill scoping**: pills need a `conversation_id` discriminator so each pill opens its own stream.
- **F-004 / F-005**: document-scope isolation and GitHub-optional local-first project creation remain the top product work after F-037.

A separate CI follow-up will harden the mirror job so a transient platform-build failure can never publish a partial public release again.

## 6.1.1 (2026-05-01)

### Continuous Improvement Stabilization

Forge 6.1.1 is a focused stabilization release following the first production UAT pass on v6.1.0. It fixes the highest-impact regressions discovered in job-mode agent streams, project task scrolling, new-chat clearing, and stale text-agent context. It also captures a detailed handover for the next continuous-improvement branch so follow-up work can continue without rediscovery.

#### Agent Stream Reliability

- **Job-mode text output restored** — Text-agent audit payloads now carry `job_id` alongside `project_id`, so job-scoped text sessions render in the correct job stream instead of falling through to a hidden session bucket. This fixes the family of issues where the Start Text button appeared to hang, job-mode text replies disappeared, and voice-delegated text work looked successful but produced no visible output.
- **Voice stream leakage fixed** — Voice events are now merged into the visible stream only when they match the active project/job context, preventing a prior voice session from appearing in a newly created project.
- **New Chat now starts empty** — Starting a new chat clears the sticky voice session pointer and the relevant event bucket, so the visible thread actually resets instead of re-rendering the previous conversation.

#### Project View Usability

- **Task list scrolling restored** — The stage-work task/reference panel now scrolls correctly when content extends below the viewport, so late-stage tasks such as `STEP-032` are reachable.

#### Text Agent Context Isolation

- **Stale CATALYST briefing invalidated on context switch** — The text orchestrator now tracks which project its cached CATALYST briefing belongs to and clears that cache when the user moves to another project, a job-only context, no context, or Mode A. This prevents deleted or previous projects from being reintroduced into new text-agent sessions.
- **Conversation history reset on project/job switch** — The text orchestrator now resets `ConversationHistory` when `(project_id, job_id)` changes, while preserving legitimate multi-turn memory inside the same project/job. This prevents the text agent from opening a new project with a prior job's persona or memory.
- **Regression coverage added** — New Rust and Vitest regression tests cover job-id audit payloads, new-chat clearing, voice-event scoping, project task scrolling, CATALYST cache invalidation, and conversation-history resets.

#### Handover For Next Branch

- **Known-issues backlog expanded** — `docs/KNOWN_ISSUES_2026-05-01.md` now includes F-031 through F-038 with live audit evidence and recommended next actions.
- **Next-branch handover added** — `docs/HANDOVER_2026-05-01.md` consolidates shipped fixes, outstanding critical/high/polish work, verification results, and the recommended next-branch priority order.
- **Top remaining issue identified** — The next branch should start with F-037: the voice realtime session emits correct prompt-package audit rows on context change, but the live voice model can still answer from the previous job context. The handover includes the exact audit timeline and proposed fix.

#### Known Follow-ups

The following UAT findings are intentionally **not** fixed in 6.1.1 and are ready for the next branch:

- **F-037 — Voice realtime context delivery**: context updates are audited correctly, but the live voice model can still answer from its prior job. Recommended fix: inject an explicit conversation-history context marker on every voice mode/context flip, not only `session.update`.
- **F-036 — Project entry should replace "Let's go"**: entering a project should automatically serve lifecycle context to the live voice session instead of requiring a second button click.
- **F-034 — Session-pill scoping**: session pills need their own `conversation_id` discriminator so each pill opens its own stream rather than all pills in a job collapsing into one job-level bucket.
- **F-004 / F-005**: document-scope isolation and GitHub-optional local-first project creation remain the highest-priority product work after the voice realtime context fix.

## 6.1.0 (2026-04-27)

### Unified Agent Stream & UI Polish

This release consolidates the agent interaction experience into a single, filterable stream and refines the voice and job workflows introduced in 6.0.

#### Unified Agent Stream (Phase B)

- **Persona labels** — Agent stream messages now display persona name and function. Voice conductor messages show as `conductor (voice)` in purple, text operator messages as `operator (text)` in green.
- **Stream filters** — New filter bar in the agent stream header. Toggle visibility of Voice Chat, Text Chat, Tool Calls, Tool Responses, System Prompts, Prompt Packages, and Events. Filters persist across sessions via localStorage.
- **Tenancy-gated filters** — System Prompt and Prompt Package filters are only visible to SCAINET-tenancy users, protecting trade secrets from external users.
- **Removed ProjectConversation** — The embedded chat box in ProjectDetailView is removed. All agent interaction now flows through the unified AgentStream panel, eliminating the split-attention UX.

#### Job File Structure (Phase A)

- **Directory scaffolding** — Jobs now get a local `jobs/{id}/documents/`, `artifacts/`, `references/` directory tree on creation, mirroring the project file structure pattern.
- **Job document panel** — JobDetailView now includes a documents section showing files from all three subdirectories with document type icons and a document viewer modal.
- **IPC commands** — New `job_list_documents` and `job_working_dir` Tauri commands for frontend access to job files.

#### Voice Agent Improvements (Phase E)

- **Human-readable names in prompts** — Voice agent now refers to projects and jobs by their display name, never by UUID. Mode flip acknowledgments resolve the name from the database, and the system prompt includes an explicit directive to use human-readable names.
- **Job prompt title-first** — Job context sections now lead with the job title (`"My Task" (JOB-001)`) instead of the ID.
- **Context replay on session renewal** — When the voice session auto-renews after a 5-minute xAI timeout, the system prompt and lifecycle context are fully re-injected via `apply_mode_flip`, preserving conversation continuity.

## 6.0.0 (2026-04-28)

### Voice-Native Conductor — A New Way to Build Software

Forge 6.0 introduces the **Voice-Native Conductor**: a bidirectional realtime voice interface powered by xAI's Grok Voice API. This is not a voice-to-text wrapper — it is a persistent conversational presence that hears, thinks, acts, and speaks in real time while the user works. The conductor can take screenshots, create jobs, navigate the UI, advance lifecycle stages, and discuss project context as naturally as a human colleague.

This release represents a paradigm shift from text-agent IDE to voice-native orchestration platform.

#### Voice Architecture (Phase 2B)

- **`GrokRealtimeClient`** — WebSocket client against xAI's `wss://api.x.ai/v1/realtime` endpoint with bearer-token auth, server-VAD turn detection, and PCM16 16 kHz bidirectional audio.
- **`ReconnectingProvider`** — Audio-buffer-and-replay reconnection layer (90-second buffer, exponential backoff, auto-replay on disconnect). New in 6.0: **session auto-renewal** resets the reconnect budget when the server enforces a session duration limit, keeping conversations alive across server disconnects without user intervention.
- **`VoiceSession` actor model** — Single tokio task owns the provider; audio uplink, event downlink, tool dispatch, mode commands, and shutdown are independent channels. No shared mutex over network I/O. Biased `select!` guarantees shutdown wins over message processing.
- **`MicCapture`** — AudioWorklet-based 16 kHz PCM capture with hardware mute detection (`MediaStreamTrack.onmute`), first-quantum diagnostic logging, and graceful rollback on permission denial.
- **`StreamingAudio`** — Low-latency base64 PCM playback via Web Audio API with interrupt-on-speech (gain ramp to zero within one render quantum when VAD fires).
- **Tool dispatch** — Voice-triggered function calls (`forge_screenshot`, `job_create`, `lifecycle_create`, `job_advance`, `signal_navigate_ui`, `take_screenshot`, and more) execute server-side, emit results back to the realtime model, and auto-navigate the frontend UI to reflect the outcome.
- **Duplicate call-id guard** — xAI's realtime stream emits `function_call.done` twice per invocation; the actor deduplicates by `call_id` to prevent double tool execution and double audit rows.

#### Interaction Modes

- **Mode A (Conversational)** — Generic conductor persona with boundary detection. Redirects lifecycle actions to the appropriate UI surface.
- **Mode B (Transactional)** — Receives detailed project/job context (stage, tasks, documents, history). Directly handles lifecycle actions, references project documents by name, and advances workflow stages via tool calls.
- **Dynamic mode switching** — `set_active_work_context` triggers a mid-session `session.update` to the realtime model, flipping persona and tools without dropping the WebSocket. Deduplication prevents redundant prompt rebuilds when context hasn't changed.
- **Proactive acknowledgement** — After a mode flip to Transactional, the conductor proactively greets the user with the new context ("I've switched to JOB-008. How can I help?") so the user has immediate confirmation.

#### Audit-Driven Agent Stream

- **Job-aware context keying** — Audit events are stored and retrieved under `project_id` or `job:{jobId}` keys, ensuring voice and text agent activity renders in the correct panel regardless of which agent produced it.
- **`audit_get_events_for_job`** — New IPC command fetches historical audit events for a specific job, enabling the Agent Stream to show voice conductor activity within job views.
- **Reactive refresh** — `JobDetailView` listens for `audit:event_written` events and reloads data when job-mutating tools (`job_advance`, `job_complete`, `skip_job_stage`) fire, so the UI reflects stage changes without manual refresh.

#### Frontend UX

- **"Let's Go" voice awareness** — When a voice session is active, clicking "Let's Go" updates the work context and shows a status indicator ("Working via Voice — speak to the conductor") instead of spawning a competing text agent.
- **Mic state badge** — Real-time visual indicator showing idle/listening/speaking/muted/error states, driven by a derived Svelte store.
- **Mode flip toast** — Brief notification when the conductor switches between conversational and transactional modes.
- **New Chat fix** — Restored functionality that had regressed during session management changes.
- **Session pill rationalisation** — Reduced the bottom-bar session pill count from displaying all historical sessions to a focused recent set.

#### Data Integrity

- **Project deletion with tombstones** — New `project_delete_tombstones` SQLite table tracks deletions locally. The sync engine respects tombstones during Portal sync, preventing deleted projects from reappearing after Firebase pull.
- **Tenant filtering** — Projects from other tenants (identified by Firebase custom claims) are excluded from local display, preventing cross-tenant data leakage in the UI.
- **Reference document scoping** — The reference documents panel now shows only documents associated with the active project, not all documents in the storage directory.

#### Performance & Stability

- **Mode flip deduplication** — `set_active_work_context` compares incoming context against the current state before queuing a mode flip, eliminating redundant `system_prompt` and `prompt_package` audit events on every reactive component mount.
- **Deadlock prevention** — All `LifecycleEngine::new()` calls within async contexts are wrapped in `tokio::task::spawn_blocking`, preventing `block_on` inside the Tokio runtime thread pool.
- **Session auto-renewal** — When the realtime server enforces a session duration limit (observed at 5 minutes), the actor resets the reconnect budget and establishes a fresh session transparently. The model receives a continuity instruction to resume naturally.

#### Testing

- **`cargo check`** — Clean with no new warnings beyond pre-existing dead-code notices.
- **Voice session unit tests** — Actor lifecycle, drain semantics, mode flip queuing, cost-unit accounting, audit chain integrity, and tool dispatch all covered.
- **UAT validated** — Screenshot tool, job creation, stage advancement, and natural conversation verified in live testing sessions.

---

## 5.27.13 (2026-04-23)

### SQLite access — S4 T15 program closeout + audit chain hardening

Completes the **S4 Detailed Action Plan** (T15): **S2 §8** definition-of-done evidence in **S4 §5.2**, **S0–S3** SQLite architecture docs moved to **`docs/paul-working-docs/completed/`** with thin redirect stubs at the old paths, **`TESTING.md`** (macOS/Windows manual matrix vs Linux CI), and **Learning #34** retitled per **S1 §15.2** (with **Historical context** for the W5 mutex incident). **Removes `DbHandle::with_blocking`** and migrates **lifecycle** / **IPC** / **orchestrator** paths to **`read` / `write` / `transaction` + `.await`** (or narrow `block_on` at sync boundaries).

#### Backend

- **`AuditStore::record_chained`** — parent read + insert in a **single `db.transaction`** (S2 §7); **`AuditEngine::record`** delegates; **in-process cache** for last event removed.
- **Tests** — **`eight_concurrent_records_same_session_pass_verify_chain`** (8 concurrent `record` + `verify_chain`).
- **`AuditEventType`** — **`Copy`** for clean transaction closures.
- **`db/handle.rs`** — delete **`with_blocking`** / **`with_blocking_read`**; **`lifecycle`**, **`ipc`**, **`catalyst`**, **`orchestrator`** — async facades / no blocking shims on the hot paths covered in this branch.

#### Documentation

- **`S4-SQLITE-ACCESS-ARCHITECTURE.md`** — status **Complete**, **§5.2** DoD table, **§6** acceptance.
- **`TESTING.md`** (repo root); **`.github/workflows/build.yml`** — comment linking S2 §11 / DoD #11 to manual matrix.
- **`PROJECT_LEARNINGS.md`** — Learning **#34** final form; **Review History** row.

#### Testing

- **`cargo test`** — full suite green.

## 5.27.12 (2026-04-22)

### SQLite access — agent tools on `DbHandle`, remove dead EGO escape hatches (S4)

Finishes the **agent runtime** slice of the S4 migration: the **tool executor** uses the shared **`Arc<DbHandle>`** for VC conflict tools, long-term memory (LTM) tools, and **tool metrics**—no ad-hoc `Connection::open` to `ego.db`. Removes unused **`agent::session_manager`** (never wired; distinct from `session::manager` in `main`) and legacy **`ego::db::get_db_connection`** / **`initialize()`** now that startup and tools go through **`DbHandle::open`** + **`apply_schema_and_migrations`**.

#### Backend

- **`ToolExecutor`** — `set_db_handle`, `require_db()`; `record_tool_metric` uses **`db_handle.write().await`** in a spawned task; **memory** store/recall/context use **`write`** with a shared **`map_agent_memory_err`** (boxes `anyhow::Error` into `rusqlite::Error::ToSqlConversionFailure` for the closure contract; `UserFunctionError` is behind rusqlite `features = ["functions"]`, not enabled).
- **`vc_conflict` tools** — `vc_show_conflicts`, `vc_resolve_file`, `vc_abort_merge`, `vc_mark_resolved`, `vc_undo_resolution`, `vc_regenerate_lockfile` take **`&Arc<DbHandle>`** and use **`read` / `write` + `await`** instead of `get_db_connection()`.
- **`AgentOrchestrator::set_db_handle`** and **`AgentPool::with_db_handle`** — spawned agents get the same DB wiring as the primary orchestrator; **`main`** calls **`agent.set_db_handle(db_handle.clone())`** and builds the pool with **`with_db_handle`**.
- **`ego::db`** — delete **`get_db_connection`** and **`initialize()`**; keep **`db_path`**, **`ensure_db_parent_dir`**, **`apply_schema_and_migrations`**.
- **Removed** — **`src-tauri/src/agent/session_manager.rs`** and **`pub mod session_manager`** from **`agent/mod.rs`**.

#### Maintenance

- **`db/mod.rs`** — remove unused re-exports of PRAGMA constants; clarify module docs for sync tests vs **`with_blocking`**.
- **`hidden_command` (`agent/tools/mod.rs`)** — `#[cfg(windows)]` / `#[cfg(not(windows))]` so non-Windows builds avoid **`unused_mut`**.
- **`version_control/git_ops`** — drop unnecessary **`mut`** on `repo.index()` in merge-state handling.
- **`lifecycle/documents.rs` / `gates.rs`** — remove unused **`rusqlite::Connection`** imports.
- **`lifecycle` tests** (`play_mode_no_injection`, `active_project_selects_correct_prompt`) — comments explaining why they stay sync **`#[test]`** + deprecated **`with_blocking`** (nested **`block_on`** with **`build_v3_lifecycle_prompt_section`** / **`project_status`**).

#### Documentation

- **`docs/paul-working-docs/TECHNICAL_DEBT.md`** — resolved multi-tab `session_manager` + **`get_db_connection`**; new item **S4: `LifecycleEngine` / `JobEngine` still use `with_blocking` / `with_blocking_read`** (epic: async facades, **`read` / `write` / `.await`**).
- **`docs/paul-working-docs/TEST_COVERAGE_AUDIT.md`** — drop row for removed **`agent/session_manager.rs`**.

#### Testing

- **`cargo test`** — full suite green after changes.

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

## 5.28.0 (2026-04-23)

PROJ-TOOLENHANCE (partial) — Voice-native conductor, document engine, headless CLI, and 9 net-new agent tools. This release ships the PROJ-TOOLENHANCE critical path (Phase 1, Phase 2A, Phase 2B, plus Tasks 3.1, 3.4, and 4.1). Phase 3 breadth tasks (3.5–3.10) and Phase 4 polish tasks (4.2–4.6, 4.8) are deferred to a subsequent release; see CORRECTION-19 in `strategy/S4-PROJ-TOOLENHANCE-CORRECTIONS.md` for the full scope reconciliation.

### New tools (9 net new, 88 → 97 baseline)

| Tool                   | Source task | Purpose                                                                |
| ---------------------- | ----------- | ---------------------------------------------------------------------- |
| `w5_create_checkpoint` | Task 2.8    | Create a W5 cognitive checkpoint snapshot tied to a session            |
| `w5_get_checkpoint`    | Task 2.8    | Fetch a single checkpoint by id                                        |
| `w5_query_checkpoints` | Task 2.8    | Query checkpoints by session / agent / time window                     |
| `w5_get_latest`        | Task 2.8    | Fetch the most recent checkpoint for a project                         |
| `catalyst_set_profile` | Task 2.9    | Set the active CATALYST refinement profile for the current session     |
| `catalyst_multi_pass`  | Task 2.9    | Trigger a multi-pass refinement cycle (planning surface)               |
| `gate_evaluate`        | Task 2.9    | Evaluate a lifecycle gate against the current document state           |
| `gate_scope_extract`   | Task 2.9    | Extract the in-scope item set for a given gate                         |
| `bug_report_create`    | Task 3.4    | File a structured bug report with optional W5 link, attachments, Nexus |

### Voice-native conductor (Phase 2A, Tasks 2.1–2.7)

Realtime voice provider trait, Grok WebSocket client, PCM16 streaming with reconnection, Tauri IPC bridge, Svelte streaming UI, conductor persona with interruption handling, voice-aware narration on `forge_act` / `forge_perceive`, and an equivalent text/CLI conductor for accessibility. The pipe model remains the default; realtime mode is opt-in via the new voice-mode toggle in the agent stream header.

### Document engine (Task 3.1)

Headless `chromiumoxide` HTML→PDF rendering with a global `tokio::sync::OnceCell` engine pool, two HTML fixtures, and a Criterion benchmark. `forge_pdf_render` is callable from the bug-report tool to attach screenshots or pre-rendered documents.

### Headless CLI (Task 4.1)

New `forge-cli` binary alongside the existing `scainet-forge` GUI binary. Two modes:

- `--pipe` reads one JSON request from stdin and writes one JSON response to stdout, suitable for shell pipelines.
- `--batch <script>` runs a script file (one command per line, `#` comments) and emits one chained audit event per command into the EGO audit log.

Shared initialization (`.env` loading, tracing setup, audit DB path) lives in `src/cli_init.rs` so both binaries call the same code path. See CORRECTION-18 for the `#[path]` design rationale.

### Final compliance gate (Task 4.7)

New integration test (`src-tauri/tests/final_compliance.rs`) and binary-size validators (`scripts/validate-binary.sh`, `scripts/validate-binary.ps1`) enforce the tool-count baseline (>= 97), the Article 1 (PRESERVE) regression set (88 pre-DAP tools must remain), the Article 8 (GUARD) naming convention, and the S0 §4 success criteria. The DAP-original `test_tool_count_142plus` is shipped as `#[ignore]` and acts as a forcing function for the deferred Phase 3.5–3.10 work.

### Deferred from PROJ-TOOLENHANCE

- Phase 3 breadth: Tasks 3.5–3.10 (compliance docs, patent templates, brand kit, etc.) — would have added ~39 more tools to reach the DAP-original 136 built-in target.
- Phase 4 polish: Tasks 4.2–4.6 (CLI flags expansion, telemetry export, etc.) and 4.8 (docs site updates).
- The `[lib]` extraction of `scainet-forge` — deliberately deferred per CORRECTION-18 to avoid touching ~30 internal modules; the `#[path]` include pattern shipped in this release is the surgical alternative.

### Post-critical-path additions to 5.28.0 (continued PROJ-TOOLENHANCE work)

After the 5.28.0 critical path landed (Tasks 1.x → 2.x → 3.1, 3.4 → 4.1 → 4.7 above), the resumed-DAP sequence was completed on the same `feat/proj-toolenhance-dap` branch before the public release cut. The deliverables below ship in the same 5.28.0 binary; the version was not re-bumped because no public release intervened.

#### Additional tools (30 net new — bringing the total from 97 to 127)

| Tools                                                                                                                                                      | Source task | Count |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | ----- |
| `services_create_project_from_job`, `services_present_to_portal`, `services_track_milestone`, `services_capture_feedback`, `services_back_propagate`       | Task 3.5    | 5     |
| `morpheus_import_idea`, `morpheus_auto_s0`, `morpheus_notify_thinker`, `morpheus_track_revenue`                                                            | Task 3.6    | 4     |
| `pdf_read`, `archive_extract`, `archive_create`, `env_read`, `env_set`, `audio_transcribe`, `git_log`, `git_merge`, `voice_mode_switch`, `headless_status` | Task 3.9    | 10    |
| `bod_request_briefing`, `bod_trigger_deliberation`, `bod_get_decision`, `bod_cast_vote`, `bod_dispatch_decision`, `bod_ceo_veto`                           | Task 3.7    | 6     |
| `marketplace_search`, `marketplace_install`, `marketplace_publish`, `marketplace_pneuma_scan`, `marketplace_update`                                        | Task 3.8    | 5     |

The full inventory (with grouping by domain and tier) lives in [`docs/TOOLS.md`](TOOLS.md), which is parity-tested against the runtime via `cargo test --test docs test_tool_doc_count_matches_runtime`.

#### Document templates (Tasks 3.2, 3.3)

Versioned template registry with a starter `minimal` template embedded via `include_str!`, plus the patent template + USPTO-style `patent-document.css` overlay. Templates render through the Task 3.1 chromiumoxide engine.

#### Composability flows (Task 3.10)

Two scripted compositions of existing primitives that emit chained audit events:

- `voice_present_approve_advance` — voice presents a draft, human approves, W5 snapshot, gate signoff, stage advance.
- `bug_then_support_spawn` — file a bug, then spawn the support persona.

Additive `LifecycleEngine::db_arc()` accessor introduced for flow-level reads without widening the engine's mutating surface (CORRECTION-30).

#### Cross-platform smoke matrix (Task 4.3)

Smoke tests under `tests/cross_platform/` cover Windows, macOS, and Linux conditional paths via `#[cfg(target_os = ...)]`. CI workflow runs the matrix on the appropriate runners.

#### Performance budgets (Task 4.4)

Four Criterion benchmarks (cold-start, audio-roundtrip, PDF render, dispatcher) with cross-platform validators that compare results against documented budgets.

#### Patent evidence pipeline (Task 4.5)

Filing-grade JSONL artefacts for CPA-007 (adaptive orchestration), CPA-008 (friction-controlled gates), CPA-009 (board-to-build workflow). Generated by tests under `tests/patent_evidence/`.

#### Tier and entitlement matrix (Task 4.2)

Three-tier model (Free / Pro / Enterprise) gated at `ToolExecutor::execute`. Default tier is Enterprise so existing callers see no behaviour change. See `src-tauri/src/agent/tools/entitlements.rs` and `docs/TOOLS.md` for the tier mapping.

#### Day-2 operations docs (Task 4.6)

Three runbooks (`voice_outage`, `cdp_session_leak`, `headless_chromium_crash`) plus monitoring + rollback playbooks under `docs/operations/`. Cross-platform validators (`scripts/validate-docs.{sh,ps1}`) and an integration test (`tests/docs.rs::test_runbooks_referenced_from_code`) keep the runbooks discoverable from the production source tree.

#### Docs site updates (Task 4.8)

New `docs/TOOLS.md` (the human-readable view of `BASELINE_TOOLS`) and updated README tool-count claims (`40+` → `127`). Parity is asserted by `cargo test --test docs test_tool_doc_count_matches_runtime`.

The deferred-list above (`Phase 3 breadth`, `Phase 4 polish`) is therefore now complete. The remaining open item is the `[lib]` extraction, which stays deferred per CORRECTION-18.

---

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
