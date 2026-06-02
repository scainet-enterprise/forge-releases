# SCAINET Forge Release Notes

> **Maintainers:** User-facing release notes mirrored to `forge-releases/RELEASE_NOTES.md` on release.
> Do **not** include internal workstream IDs (B-LC-_, F-LC-_), file paths, or technical-debt references here.
> Engineering detail belongs in `docs/CHANGELOG.md`.

## 6.24.0 (2026-05-28)

**Connect Google Workspace and bring calendar and email into Daily Flow.**

### Google Workspace in Settings

- **Connect Gmail and Google Calendar** from Settings → Integrations using one Google sign-in (OAuth).
- **Built-in setup guide** in the credentials panel walks you through creating Google Cloud OAuth credentials and pasting Client ID / Secret.

### Daily Flow

- **Calendar intake** shows upcoming events from your primary Google Calendar for the day you are planning.
- **Email intake** surfaces recent Gmail messages relevant to your flow.
- **Add as task** from a calendar event or email thread to drop it onto your board without retyping.

### Who should update

- Anyone using **Daily Flow** who wants calendar and email context in one place.
- Requires a Google account and OAuth app credentials (free tier; guide in-app).

### Notes

- This release does **not** include the in-app updater toast fix; that ships separately in the next patch.

## 6.23.1 (2026-06-02)

**Fixes a startup hang that could leave the app stuck on “Initializing…” after installing a release.**

### Startup reliability

- **Splash screen no longer blocks** while checking for updates — the app reaches the sign-in screen within a few seconds even on slow or offline networks.
- **More resilient startup** when loading saved API keys and connecting background services.
- **Settings no longer shows empty providers** after upgrade when keys exist in the environment — model discovery keeps working instead of reporting zero models.
- **Saving in Settings is faster and safer** — saving never wipes a working key, a momentary network hiccup no longer clears your discovered models, and discovery can’t hang the dialog.

### Who should update

- **Anyone on 6.19.4 or later** who saw the app freeze on the loading splash after install, or who stayed on 6.19.3 to avoid the issue.
- Recommended for all users on the stable or beta update channel.

## 6.23.0 (2026-06-02)

**Drag files in the explorer, manage them like VS Code, and open a terminal where you’re working.**

### File explorer

- **Drag and drop** files and folders within your workspace — drop onto a folder or empty space to move items to that location, with clear highlights for valid targets.
- **Right-click empty space** in the explorer to create a new file or folder at the workspace root (not only on existing rows).
- **Richer context menu:** copy, cut, paste, duplicate, reveal in Finder (or your platform’s file manager), copy name/path, and **Open in Terminal**.
- **Multi-select** with Shift or ⌘/Ctrl-click for bulk copy, cut, paste, and delete.
- **Inline rename** — press F2 or use Rename from the menu (no popup dialog).
- **Import from Finder** — drag files from macOS Finder into a folder or the explorer background to add them to your workspace.
- When you move or rename a file that’s open in the editor, your tab path updates; unsaved changes are protected with a confirmation before the move.

### Terminal

- **Open in Terminal** from the explorer opens a shell in the right folder and shows the terminal panel if it was hidden.
- Terminal tabs are more reliable when switching workspaces or opening shells from the explorer — fewer blank or stuck terminal panels.

### Who should update

- Anyone who reorganises files in the explorer or uses **Open in Terminal** from the file tree.

## 6.22.1 (2026-06-01)

**Fixes a launch hang where FORGE could stay on “Initializing…” and never reach sign-in.**

### Startup reliability

- FORGE no longer waits for an online update check before showing the sign-in screen. Update checks still run in the background after launch.
- If moving API keys into the secure vault takes longer than usual (first launch after an update), the app continues starting instead of waiting indefinitely.

### Who should update

- Anyone on **6.22.0** who sees the splash screen freeze at **Initializing…** should install **6.22.1**.

## 6.22.0 (2026-06-01)

**See how FORGE builds your agent prompts — and switch views without waiting.**

### Prompt Library (SCAINET team)

If you sign in with a SCAINET tenant account, open **Settings → Prompt Library** to:

- Browse the catalog of prompt injection points and dictionaries.
- Preview the **effective prompt stack** for your current session (project, job, daily flow, voice, or persona).
- Pick a persona and see which profile blocks apply before you start a session.
- In **development builds**, edit persona profile text and reload from disk to iterate quickly (shipped apps still use the embedded profiles until hot-reload ships).

### Voice & session context

- **Live session context** during voice now reflects **voice** prompts, not the text agent defaults — so previews and persona behaviour match what you hear in realtime voice mode.

### Smoother navigation

- Opening **Explore**, a **job**, or a **daily flow** day switches the main view immediately while workspace and agent setup finish in the background — less “stuck on the hub” feeling on slower machines.

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
