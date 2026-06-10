# SCAINET Forge Release Notes

> **Maintainers:** User-facing release notes mirrored to `forge-releases/RELEASE_NOTES.md` on release.
> Do **not** include internal workstream IDs (B-LC-_, F-LC-_), file paths, or technical-debt references here.

## 6.29.2 (2026-06-10)

### Bug reports go to Service Desk

- When an agent files a bug with **Create bug report**, it is sent to the **Portal Service Desk** so you get a real issue key and can triage it in the inbox.
- Bug reports now require **severity** and **tags** so issues are easier to filter and prioritize on the Service Desk.

### Who should update

- Anyone using **agent bug reports** or **Service Desk intake** from Forge.

## 6.29.1 (2026-06-10)

**Critical fix: FORGE failed to open at all after updating from 6.27 to 6.28/6.29. Also fixes the Jobs tab showing no jobs.**

### App would not start after updating

- If you updated from **6.27.x** to 6.28.0 or 6.29.0, FORGE could **fail to launch entirely** — no window, no error. An internal database upgrade step tried to apply a change your database already had, and the app stopped before opening.
- 6.29.1 makes the upgrade system self-healing: it recognises what your database already has, applies only what is missing, and starts normally. **No data was lost** — your days, jobs, and projects are intact.
- Because the broken version cannot open to auto-update, affected users need to **download and install 6.29.1 manually** from the releases page (one time only).

### Jobs are back

- A related issue made the **Jobs tab show 0 jobs** on some databases even though all jobs were safe. The same self-healing upgrade restores them on next launch.

### Who should update

- **Everyone on 6.28.0 or 6.29.0**, and especially anyone whose FORGE **won't open** after updating — install 6.29.1 manually and launch.

## 6.29.0 (2026-06-09)

**Daily Flow completes the morning briefing — Patrick and Aurora work in the background while Clara walks you through the day, without cutting you off mid-sentence.**

### Morning briefing pipeline

- **Patrick** triages your inbox automatically when you open today's day.
- **Aurora** drafts your briefing document from your interests and news sources.
- **Clara** delivers the briefing section by section and discusses it with you **before** planning starts — briefing and planning are clearly separate steps.

### Briefing quality

- Briefing documents use clear sections per topic, with **links to sources** and one story per bullet.
- When you approve your plan, Clara knows the plan is locked — she won't ask again.
- Planning avoids duplicating tasks already on your day.

### Voice reliability

- Clara **waits until you finish speaking** before relaying briefing updates from Patrick or Aurora.

### Daily Flow stage gates

- Formal **stage artifacts** for briefing and wrap-up phases, with visible gate controls on the day view.
- **Plan tracking** improvements including dependencies and overnight task rollover.

### For SCAINET developers (dev builds)

- **Clear Day (dev)** resets today's day for UAT — admin-only, development builds only.

### Who should update

- Anyone using **Daily Flow on voice** for morning briefings or **delegated persona work** (Patrick, Aurora).

## 6.28.0 (2026-06-09)

**Promote Service Desk issues straight into coding jobs — with a repo worktree and Quick Plan ready to review.**

### My Issues → Service Desk job

- **Promote** an assigned issue from **My Issues** to a **Service Desk coding job** in one step.
- Forge **infers the target repository** from linked events, projects, and stack traces — or lets you pick from your tenant’s repo list.
- A **git worktree** is provisioned automatically so the agent can work in the right codebase.
- The job opens at **Plan (J1)** with a **Quick Plan draft** seeded from the issue — review it, add numbered tasks, then approve when ready.

### My Issues polish

- **Selection highlight** in the issue list now shows clearly which ticket you’re viewing.

## 6.27.1 (2026-06-08)

- deps(npm): bump the production-deps group across 1 directory with 9 updates

_Auto-generated — curate before external comms._

> Engineering detail belongs in `docs/CHANGELOG.md`.

## 6.27.0 (2026-06-08)

**Daily Flow planning works the way you intended, voice stays responsive while Patrick works, and Work opens where you actually start your day.**

### Quick Plan ceremony

- **Clara drafts the Quick Plan in voice** (`save plan`) — not ad-hoc “add task” shortcuts during planning.
- **Lock for Approval** in the Quick Plan panel (or Clara’s submit) stages the plan without creating tasks.
- **Approve Plan** (UI only) turns each line into a task and moves the day to **Executing**.
- **Skip planning anytime** — advance phase without a plan if you want to rush through; add tasks later.
- **Quick Plan** is now collapsible and **tucks away automatically** when you enter Executing so task work stays front and centre.
- Planning guidance **refreshes when the day changes phase** so Clara’s instructions stay current.

### Briefing & cast personas

- **Structured briefing ceremony** for Daily Flow — email triage (Patrick), calendar snapshot, task rollover, and a clear path into Planning.
- **Cast personas** (Patrick, Lens, Quill, …) load their tool policies correctly without backend warnings.
- **Delegated briefing** uses cast-specific instructions without cluttering the main agent identity.

### Voice reliability during delegation

- **Voice tools no longer freeze** for the full duration of a delegated briefing or inbox review while a text agent runs in the background.

### Email-linked tasks

- **Create tasks from Needs Attention emails** via voice with the same source linkage as the UI.
- **Needs attention** starts **collapsed** so your task board is the focus until you expand intake.

### Work tab

- **Daily Flow** is now the **first tab** and opens **by default** — Jobs in the middle, Projects on the right.
- **Super admins and admins:** filter projects by **My projects** or **All tenancy** (Work tab and project picker). Defaults to **My projects**.

### Who should update

- Anyone running **Daily Flow** on voice (planning → lock → approve) or **delegated briefing**.
- **Tenant admins** who want a focused project list without losing tenancy-wide visibility.

## 6.26.0 (2026-06-08)

**My Issues — work assigned tickets inside Forge, without leaving your desk.**

### My Issues (SCAINET staff)

- New **My Issues** tab in Work Hub shows incident tickets **assigned to you** from Portal Service Desk.
- Read linked error events, add comments, **resolve** tickets, or **promote** to a local Job with a quick plan and starter tasks.
- Real-time inbox updates when triagers assign work on Portal.

### Smarter error priority

- New Forge-captured errors get an automatic **severity** rating so triage queues are easier to prioritise.

### Work Hub polish

- Clearer tab icons; My Issues uses distinct inbox styling so it is easy to spot.

## 6.25.0 (2026-06-05)

**Google Workspace goes deep inside Daily Flow — read mail, manage calendar, and let Patrick clean your inbox with your approval.**

### Email in FORGE

- **Click any Needs Attention email** to read the full message (and load the whole thread) without leaving FORGE.
- **Filter intake** by Today, All Unread, Drafts, and more; section expands and collapses so it does not dominate the day view.
- **Draft replies** go through an approval queue — agents draft, you approve and send.
- **Patrick** is your email specialist: delegated email tasks route to him; he can search, label, archive, filter, and stage bulk cleanups for your confirmation.

### Calendar in FORGE

- **Tap a calendar item** to open an in-app event sheet — view details, edit, **reschedule** (if you created the event), and **RSVP**.
- Create tasks from events with suggested titles and email/event details pre-filled.

### Safer inbox cleanup

- **Bulk trash, archive, and label** actions from Patrick are staged for your review — trash always needs a confirm; large batches do too.
- **View the list** of emails in a staged batch and remove any you want to keep before approving.

### Audit and transparency

- Human approvals (send, trash, bulk confirm) and persona handoffs are now recorded in the audit trail.
- Tool calls and results show in the Agent Stream by default so you can see what agents did.

### Who should update

- Anyone using **Google Workspace** in Settings and **Daily Flow** who wants to stay in FORGE for mail and calendar instead of switching to the browser.

## 6.24.2 (2026-06-04)

**Smarter task delegation and more reliable Job / Daily Flow views while the agent is working.**

### Task delegation

- When you delegate a task to a worker agent, that worker **stays focused on the assigned task** — it can no longer re-delegate, jump to other jobs, or mark the wrong task complete.
- **Delegate all remaining** and single-task delegation both produce cleaner worker sessions that finish with the right completion step.

### Job and Daily Flow UI

- **Job and Daily Flow detail screens stay in sync** when the agent navigates or updates tasks, plans, or board items during a session — no more stale panels after the agent moves on.
- **Navigation is more consistent** whether you use voice commands or let the agent drive the UI.

### Who should update

- Anyone using **Jobs**, **Daily Flow**, or **task delegation** who saw wrong screens, stale task lists, or workers that tried to orchestrate instead of execute.

## 6.24.1 (2026-06-02)

**Stability Improvements**

- Fixed an issue where the app could fail to check for updates with an internal error
- Improved reliability of AI provider connections during temporary network issues
- Fixed a startup delay that could occur when checking for updates
- Update notifications now link to public release notes for easier access

**Under the Hood**

- Various internal improvements to error handling and resilience
- Updated testing infrastructure

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
