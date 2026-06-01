# SCAINET Forge Release Notes

> **Maintainers:** User-facing release notes mirrored to `forge-releases/RELEASE_NOTES.md` on release.
> Do **not** include internal workstream IDs (B-LC-_, F-LC-_), file paths, or technical-debt references here.
> Engineering detail belongs in `docs/CHANGELOG.md`.

## 6.21.2 (2026-06-01)

**The agent tool system is now fully modular — faster to extend and more reliable under the hood.**

### Agent tools

- The remaining built-in agent tools now run through the modular registry (**169 tools** total), including Board of Directors governance, marketplace search and install, Gmail integration, PDF reading, and environment helpers.
- Tool routing and validation behave the same from your perspective; this release is primarily an architecture and reliability improvement for future agent capabilities.

### Behind the scenes

- CI runs faster on merges to the main branch by skipping duplicate full test suites when changes were already validated on the pull request.

## 6.21.1 (2026-06-01)

**Explore mode now has a persistent workspace, and navigation back from detail views feels correct.**

### Explore workspace

- Explore uses a dedicated sandbox folder under your Forge data directory instead of a throwaway temp path.
- In **Settings → Explore workspace**, you can choose a custom folder or reset to the Forge default.
- The file explorer and agent working directory stay in sync with your Explore sandbox.

### Better navigation when leaving detail views

- When you open a project, job, or daily flow detail and then go back, FORGE restores the workspace you had before — including Explore mode — instead of leaving the explorer pointed at the wrong folder.

### Behind the scenes

- CI runs faster on routine merges by skipping redundant full test suites while keeping PR checks as the quality gate.
- Automated version bumps now go through a pull request so they are tested like any other change.

## 6.21.0 (2026-05-30)

**More agent tools migrated and clearer error handling.**

### Agent tools (Wave 10)

Fifteen additional agent tools now run through the modular tool registry (140 tools total), including:

- **Code intelligence** — extract symbols, map dependencies, analyse complexity, and inspect scope at a line in source files.
- **HTTP requests** — agents can call external APIs directly when needed.
- **GitHub project linking** — connect a lifecycle project to a GitHub repository.
- **Reference documents** — list and read project reference docs from the workspace.
- **Portal tasks** — create and list tasks on the SCAINET Platform from within FORGE.
- **Bug reports** — submit structured bug reports to Nexus with optional screenshots and W5 checkpoint links.
- **Morpheus integration** — import ideas, run auto S0 intake, notify thinkers, and track revenue events.

### Reliability improvements

- Creating a Portal task or submitting a bug report now fails clearly when the upstream service rejects the request, instead of appearing to succeed.
- Bug reports with a missing or blank description are rejected immediately instead of being sent to the server.
- Morpheus notification tools surface database lookup failures instead of failing silently.
- Opening a project no longer fails when lifecycle stage personas need to be synced — duplicate persona name conflicts are reconciled automatically.
- When you ask an agent to delegate a job task, it will retry if it tries to finish without actually calling the delegation tool.
- Constitution Guard blocks on sensitive operations now appear in the audit trail instead of being silently skipped.
- Delegating a job task accepts simpler task references (sequence numbers like `2` or shorthand like `TASK-002`), not only full canonical ids.

## 6.20.0 (2026-05-30)

**Smarter detail views and a complete tool registry migration.**

### Detail views stay up to date

When an agent updates a job, daily flow, or project while you have the detail screen open, the UI now refreshes automatically — task lists, stage rails, and plan content update without navigating away and back. Refreshes are debounced and coalesced so rapid changes do not cause flicker or redundant network calls. Returning to the app after switching windows triggers a stale-data check on open detail views.

### Agent tools (Wave 9)

Twenty-two additional agent tools now run through the modular tool registry (125 tools total), including:

- **W5 cognitive checkpoints** — record and present decisions, generate handovers, and manage context snapshots.
- **Version-control conflict resolution** — show, resolve, abort, and undo merge conflicts; regenerate lockfiles when needed. Tool failures now surface clearly to the agent instead of appearing as successful runs with hidden errors.
- **CATALYST profiles and gates** — set cognitive profiles, run multi-pass refinement, and check or release lifecycle gates.
- **Services integration** — create Portal projects from jobs, present artefacts, track milestones, capture feedback, and back-propagate updates.

### Reliability improvements

- Optimistic UI updates for task changes reconcile correctly when background refreshes complete.
- Audit-driven refresh fallbacks no longer duplicate work already handled by domain events.
- CI guards help prevent regressions that would bypass the new refresh coordinator.

## 6.19.4 (2026-05-29)

- **Stable and Beta update channels** — In Settings → Updates, choose **Stable** (recommended for everyday use) or **Beta** (pre-release builds that may be less stable).
- **Clearer release notes** — Update notifications now link to plain-language release notes instead of the internal engineering changelog.
- **Beta indicator** — The **Beta** badge in the title bar appears only when you have opted into Beta updates; it disappears when you switch back to Stable.
- **Existing installs** — If you have been using FORGE before this release, you remain on the Beta update channel until you choose Stable in Settings → Updates.

## 6.15.4 (2026-05-27)

- Agent briefings are scoped to the assigned task so responses stay focused on what you asked for.

## 6.15.3 (2026-05-26)

- **Faster voice connect** — Starting voice in a project or daily flow no longer stalls while the app loads history.
- **Smoother app updates** — Event handling is consolidated so shell notifications and file-change updates behave more reliably.
- **Jobs visible again** — Work Hub lists jobs created before recent account changes, so nothing appears missing after an update.
- **Voice session timeout** — Voice connect no longer hangs indefinitely if the realtime service is slow to respond.
