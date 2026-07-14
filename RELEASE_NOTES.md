# SCAINET Forge Release Notes

> **Maintainers:** User-facing release notes mirrored to `forge-releases/RELEASE_NOTES.md` on release.
> Do **not** include internal workstream IDs (B-LC-_, F-LC-_), file paths, or technical-debt references here.

## 6.40.8 (2026-07-14)

### Under-the-hood — cloud briefing worker (headless spike)

Further internal work on **cloud-hosted morning briefings**. There are **no user-visible changes** in the desktop app.

- Cloud worker can now run Patrick (email triage) and Aurora (briefing document) headlessly without the desktop UI
- Staging spike verified end-to-end: job claim, standing tasks, both agent sessions, succeeded terminal state

**What to do:** No action required. Daily Flow on your PC is unchanged.

## 6.40.7 (2026-07-14)

### Under-the-hood — cloud briefing worker (continued)

Continued internal work on **cloud-hosted morning briefings** triggered from the portal. There are **no user-visible changes** in the desktop app.

- Worker can now authenticate with the portal, pull your day state, and load credentials securely for headless briefing execution
- Staging end-to-end test path verified (job claim → session → hydrate → succeed)

**What to do:** No action required. Daily Flow and briefing behaviour on your PC are unchanged.

## 6.40.6 (2026-07-13)

### Daily Flow — Done stays Done

Standing tasks (email briefing, Aurora document, and related standing work) no longer bounce back to ready or re-run after you finish them — including when another PC syncs.

- Completing a standing task sticks across sync and overnight heal
- **Auto-run morning briefing** is a clear checkbox on Daily Flow (on by default) so you control whether Aurora starts without a click

### Agent Stream & Observatory — follow the right agent

Following up on a delegated agent (Patrick, Aurora, Merlin, …) now focuses the Agent Stream on **that session**, binds compose to the follow-up, and keeps both streams stuck to the live edge unless you scroll up.

- No more “Message Clara…” placeholder when you meant Patrick
- Fewer false “agent may be stuck” banners during long model thinking — you see **Waiting on model** first
- Your own voice comments no longer pin in a stack that covers the live stream

### Email tools

Gmail label updates accept the parameter names agents commonly use, and refuse empty label changes that used to retry forever.

**What to do:** Install **6.40.6** on each PC you use for Daily Flow. After update, confirm standing tasks that are Done stay Done after Sync, and try **Follow up** once on a completed delegated task.

## 6.40.5 (2026-07-12)

- deps(npm): bump the production-deps group across 1 directory with 14 updates

_Auto-generated — curate before external comms._

## 6.40.4 (2026-07-12)

### Under-the-hood — cloud briefing preparation

This release lays groundwork for future **cloud-hosted morning briefings** triggered from the portal. There are **no user-visible changes** in the desktop app today.

- Internal worker infrastructure for processing portal briefing jobs
- Shared engine refactor so the desktop app and headless worker share the same core

**What to do:** No action required. Daily Flow and briefing behaviour on your PC are unchanged.

## 6.40.3 (2026-07-11)

### Work data sync — full job and plan parity, reliable large restores

- Job **task lists** now travel with your jobs between devices (not just the job shell)
- Daily Flow **plan line-items** and **phase gate notes** restore on a second PC
- Large Restore / Sync now runs **continue automatically** until the library is complete — you should not need to click Restore again mid-download
- After restore, standing tasks heal and overnight carry catch up so both PCs stay aligned

**What to do:** Install this release on both PCs. On the master PC run **Add a New Device** (or **Sync now** if already Ready). On the second PC run **Restore from account** and wait for the long run to finish.

## 6.40.2 (2026-07-10)

### Work data sync — full Restore actually pulls files and history

If Restore said it succeeded but thread session notes and Agent Stream history were still missing, this release fixes the cause: pagination was skipping most cloud files and agent events after the first batch.

- Restore now pages through the full artifact and agent-history libraries
- The success message shows file and agent-event counts — not only days/jobs/threads
- A restore that gets records but zero files is reported as incomplete

**What to do:** Install **6.40.2**, then on the second PC run Settings → **Restore from account** again and wait for the long run. Expect thousands of files and agent events in the result, matching your primary PC backup.

## 6.40.1 (2026-07-10)

### Work data sync — finish setup on your other PC

If your second computer only got a little data after **Sync now**, or never ran the long restore after sign-in, this release fixes that path.

- Signing in on a new or wiped install starts the full restore when your account already has a backup
- **Restore from account** stays available even when this PC already has some days — use it to download the rest (safe merge)
- **Export diagnostics** opens a real Save dialog so support files actually save

**What to do:** Install **6.40.1** on the second PC → Settings → Work data sync → **Restore from account** → let it run as long as the original backup (thousands of files). Do not rely on **Sync now** for a first full setup.

## 6.40.0 (2026-07-09)

### Work data sync — truth you can trust across devices

This release hardens multi-device Work sync so backup and restore tell the truth about what actually landed in the cloud.

- **Migrate before upload** — older day and thread folders are copied into the correct Forge locations before backup, so sync no longer misses files that lived in legacy paths
- **Honest completion** — backup and restore only report complete when workspace files actually uploaded or downloaded; you see **Ready**, **Partial**, or **Never** instead of a false success
- **Full round-trip** — task dependencies, prompts, plans, and live thread sessions survive the trip between machines
- **Daily Flow on every device** — standing tasks, prefs, and delegation restore cleanly; duplicate standing tasks are healed after overnight roll

**Recommendation:** Update to **6.40.0** on every machine you use with Work sync. On your primary computer, run **Sync all Work to cloud** or **Add a New Device** once, then sign in on the second machine and confirm restore shows **Ready**.

### Daily Flow — restore and tools

Daily Flow preferences, delegation, and tool access are more reliable after a restore so briefing and standing work behave the same on a new device.

### Clara — better handoff from typed chat into live voice

When Clara is already listening, text you type in the stream is injected into the live voice conversation so she can answer with that context — not only from a short instruction summary.

- Typed context is flushed into the voice turn when you stop speaking
- Freeform compose no longer auto-binds to Aurora when Clara is the active badge

Full mid-session “one Clara” continuity across voice and text is still evolving; this release improves the catch-up path for production sync testing.

### Models — Grok 4.5 in the picker

Grok 4.5 is available in the model matrix with long context and cost-aware ranking. Forge can use conversation cache keys and report cache usage in the audit trail where the provider supports it.

## 6.39.0 (2026-07-09)

### Agent Stream — scroll-aware turn context

When you scroll through a long agent reply, your instruction for that turn stays pinned at the top — similar to Cursor's conversation UX.

- **Compact prompts** — long messages show a short preview in the stream
- **Click to expand** — open the full text when you need it; click outside to collapse
- **Smoother scrolling** — less jumpiness when reviewing delegated briefs and past turns

## 6.38.2 (2026-07-08)

- business launch plan and CTO portal briefing brief

_Auto-generated — curate before external comms._

## 6.38.1 (2026-07-07)

### Work data sync — reliable backup on every device

If **Add a New Device** or **Sync all Work to cloud** appeared stuck part-way through (often around step 41), this release fixes that:

- Legacy daily folders are scanned correctly — Forge no longer walks your entire user profile when an old day points at the wrong folder
- Backup continues past Windows permission blocks instead of stopping
- Background sync no longer interrupts the restore progress screen

**Recommendation:** Update to **6.38.1** on every machine you use with Work sync, then run **Add a New Device** or **Sync now** once on your primary computer.

## 6.38.0 (2026-07-07)

### Add a New Device — one click, then just sign in

Moving to a second computer is now a single click:

- **Add a New Device** (Settings → Account → Work data sync) backs up everything — Daily Flow, Jobs, Threads, workspace files, your full agent history, **and your model API keys**
- On the new machine, **just sign in** — setup starts automatically and finishes with "This device is ready." Your models work immediately; nothing to re-enter

### Everything stays in sync automatically

Work between a PC at the office, a laptop in transit, and a machine at home in the same day — no manual syncing:

- Changes push as you work, including live agent activity
- Other devices pick them up on window focus and every minute while Forge is open
- **API keys follow your account** — change or remove a key on one device and the others apply it within a minute. Keys are encrypted on your device before upload; the cloud only ever stores ciphertext

### Restore fixes for multi-device users

- Days restored to a second machine now open in the **correct working folder** (previously some restored days looked empty even though the files were on disk)
- **Agent Stream history** now restores to a new device — your full activity record follows your account

## 6.37.1 (2026-07-05)

### Work data sync — progress and complete restore

Multi-device Work sync is easier to follow and more reliable when moving between computers.

- **Sync now**, **Restore from account**, and **Sync all Work to cloud** each show phase, a progress bar, and batch detail in Settings
- A **floating banner** continues in the background if you close Settings while sync is running
- **Restore from account** now downloads **all workspace files** (briefings, plans, session notes) across large libraries — not only metadata from the first sync batch

If you use more than one device, update to **6.37.1** on each, then run **Restore from account** on any machine that was missing files after an earlier restore.

## 6.37.0 (2026-07-05)

### Audit Trail — three views on the right rail

The **Audit** tab is now a full trace explorer for agent history:

- **Context mode** — see the same scoped events as your Agent Stream (project, job, day, or thread)
- **Session mode** — tamper-evident chain with verify badge and **Export JSON** (native save dialog on desktop)
- **Tree, Timeline, and Graph** — list, timing, and flow-map views with a shared details drawer
- **Command palette** — open Audit Trail in Context or Session mode directly
- **Jump to Stream** — from an audit event back to the matching stream row

### Export JSON

Session export opens a **Save** dialog and writes a complete audit JSON file to disk (chain metadata + all events). Cancel the dialog with no error banner.

### Timeline

Timeline tracks scroll together with the time ruler — individual rows no longer have their own horizontal scroll.

### Details drawer

Close selected event details with **×** in the top-right corner.

Agent Stream rows are unchanged — payload expanders remain on stream cards for now.

## 6.36.5 (2026-07-05)

### Work across machines — sync reliability (second-device restore)

If **Restore from account** failed with **Duplicate field 'dayId'**, or a full **Sync all Work to cloud** timed out on large libraries, **6.36.5** fixes the underlying causes:

- **Cloud restore** — handles Portal day records that carry both `id` and `dayId` without error
- **Large backups** — splits big libraries into smaller upload batches so sync completes instead of timing out
- **Artifact uploads** — fixes rare storage upload failures on some file types
- **Clearer errors** — retry and failure detail in logs when a sync step fails

**Update both machines** to **6.36.5** (or run a dev build with this patch). On your source PC, run **Sync all Work to cloud** once; on the new PC, sign in and use **Restore from account**.

## 6.36.4 (2026-07-04)

### Agent Stream — long instructions no longer hide agent output

If a task delegation brief or other long prompt appeared as your message in the stream, it could cover the whole view and make tool calls hard to read. **6.36.4** keeps your latest instruction visible at the top but limits it to a scrollable area so agent activity stays on screen below.

### Compose — clears after you send

The message box now clears reliably after you submit, including when using @mentions. If sending fails, your draft is put back so you can retry without retyping.

### Compose — roomier default

The input area starts at about two lines tall so it is easier to write longer messages before it grows upward.

## 6.36.3 (2026-07-03)

- Merlin persona

_Auto-generated — curate before external comms._

## 6.36.2 (2026-07-02)

### Dependency updates

Routine security and maintenance updates to bundled npm dependencies (including Sentry and js-yaml). No feature changes — install if you are already on **6.36.1** and want the latest dependency pins.

### Release notes correction

**6.36.1** also shipped the work state sync verification fix below; that section was missing from the original 6.36.1 release notes and is now documented under **6.36.1**.

## 6.36.1 (2026-07-01)

### Work state sync — verified before it runs

If **6.36.0** showed a cryptic `401` or "Missing or invalid X-Forge-Key" when syncing, **update to 6.36.1 or later**:

- **Settings shows Portal connectivity status** before sync — you'll see whether this build can reach the cloud, not a silent failure after you click sync
- **Sync is blocked when misconfigured** — install the verified release; no manual Windows environment variables required
- **Release builds are gate-checked** — CI proves the baked Portal key matches production before installers ship

### Agent Stream — clearer conversation layout

This patch improves how the centre **Agent Stream** and compose area feel day to day:

- **Your instructions stay visible** — when you scroll through a long agent reply, the user message for that turn remains pinned at the top so you always know what you asked
- **Smoother scrolling** — less jitter when reading history near the bottom of the stream; auto-follow only engages when you are actually at the latest messages
- **Compose stays at the bottom** — the message box is anchored to the bottom of the window and expands **upward** as you type, similar to Cursor and other chat tools
- **Softer cards** — message and event blocks use slightly rounder corners for a cleaner read

No settings changes required. Drag the handle between the stream and compose area if you want more room for typing.

### Who should update

Anyone who spends time in the **Agent Stream** and found the old compose area awkward or the stream hard to scroll — especially on longer sessions with multiple turns.

## 6.36.0 (2026-07-01)

### Work state sync — production ready on installed Forge

**6.35.0** introduced work state sync; **6.36.0** makes it work out of the box on downloaded installers — no manual `.env` setup.

Signed-in users can keep **Daily Flow days**, **Jobs**, and **Threads** aligned across machines. This release hardens the path from dev to production:

- **Installed builds authenticate automatically** — the Portal client key is baked in at compile time, so sync works the moment you sign in
- **Fresh Firebase tokens before sync** — Settings and restore actions refresh your session token so long-lived installs don't fail with silent auth errors
- **Smarter reconciliation** — last-write-wins timestamps on days, tasks, and jobs; scoped job backfill avoids pulling unrelated history
- **Clearer errors** — push failures surface in the UI instead of failing quietly
- **Offline-safe queue** — Daily Flow changes made offline enqueue and drain when you're back online

**Getting started (two machines):**

1. Sign in with the **same SCAINET account** on each machine
2. On your primary machine: **Settings → Account → Work data sync → Sync all Work to cloud**
3. On the second machine: **Restore from account**, then **Sync now** as needed

Ask Clara _"how does work state sync work?"_ — updated in-app guides cover setup, troubleshooting, and what syncs (and what doesn't).

### Operator Browser — smoother shared browsing

The **Operator Browser** (right rail → **Browser**) is a real browser you share with your agent. This release polishes the experience:

- **No more tab traps** — keyboard focus stays where you expect when switching between FORGE and the live browser panel
- **Headed navigation** — agent-driven browsing can open a real Chromium window when a full browser is needed (sign-in, pop-ups, OAuth)
- **Loading feedback** — clearer state while pages load so you know the agent is working
- **Profile recovery** — if the persistent browser profile gets into a bad state, FORGE can recover without a full reinstall

### In-app guides expanded

Human and agent coaching docs were overhauled for **Threads**, **Work state sync**, **Daily Flow**, **Operator Browser**, **My Issues**, and **Agent Observatory**. Find them in **Settings → Help** or the title bar **?** button; Clara uses the agent guides automatically in context.

### Who should update

- **Everyone on 6.35.0** who uses **work state sync** on an installed build — this is the release that makes cross-device sync work without manual configuration
- **Multi-device users** — update all machines to 6.36.0, run **Sync all Work to cloud** once on your primary machine, then **Restore from account** on others
- **Operator Browser users** — tab-focus, sign-in, and loading improvements
- **6.34.x users** still on an older line — skip straight to 6.36.0 for Threads, sync, and Daily Flow improvements from 6.35.0 as well

## 6.35.0 (2026-06-29)

### Threads — a new first-class Work surface

**Threads** are ongoing life context — separate from Projects and Jobs. Pick a template, name your thread, and Clara helps you capture what matters over weeks and months.

**Ten templates, each with its own Clara guidance:**

| Template             | What it's for                                                        |
| -------------------- | -------------------------------------------------------------------- |
| **Medical**          | Appointments, symptoms, care team — organised, never clinical advice |
| **Dream**            | Fluid dream journal — recall, feelings, patterns when you want them  |
| **Food & Cooking**   | Tastes, skills, recipes, family meals over time                      |
| **Health & Fitness** | Movement and wellbeing goals — not medical care                      |
| **Goals**            | Life goals and values — long horizon, not work projects              |
| **Learning**         | Explore a topic with Clara as tutor                                  |
| **Gratitude**        | Notice what matters — moments, people, small wins                    |
| **Mindfulness**      | Breath, presence, and reflection practice                            |
| **Relationships**    | People who matter — conversations, milestones, care                  |
| **Creativity**       | Writing, art, music, making — capture the process                    |

**How Threads work:**

- **Threads tab** in Work — create from the template picker, open for detail, archive when done
- **Session notes** — Clara writes dated session documents (`thread_write_session`); nothing is remembered without an explicit tool write
- **Care & profile panel** — intake summary visible in the UI (medical-first design, works for every template)
- **Medical disclaimer** — acknowledged once, persisted and audited; Clara will not give clinical advice
- **Morning briefing** — Aurora can mention active threads; Dream threads stay quiet unless you enable briefing per thread
- **End-of-day wrap** — Aurora sees threads you created or updated today, even before the first session note
- **Escalation (Pro+)** — turn a thread into a Job or link to a Project when work needs a formal track
- **Free tier** — one active thread; **Pro+** — unlimited threads

### Work state sync — Days, Jobs, and Threads across devices

Signed-in users can keep Daily Flow days, Jobs, and Threads aligned with the cloud — so a second machine or a fresh install can pick up where you left off.

- **Settings → Account → Work state sync** — status, last pull time, pending queue, and actions: **Sync now**, **Backfill**, **Hydrate**
- **Cloud is the reconciliation layer** — local changes push after mutations; pull applies last-write-wins metadata
- **Threads and artifacts** — thread workspace files sync with your work-state payload
- **Offline-safe** — changes queue when disconnected and drain when you're back online

### Daily Flow — clearer task control

- **Roll count** — carried tasks show how many times they've rolled forward
- **Mark complete** from the ⋮ menu without starting the task first
- **Do today (clear deferral)** — bring a deferred task back to today's plan in one action

### What's New in FORGE — morning mini-news

- Aurora may include up to three benefit-led bullets when you have unseen releases (from bundled release notes, not engineering changelogs)
- **Pro+** can turn this off in Settings → Daily Flow; **Free** always sees it when updates are available
- Ask Clara _"what's new?"_ any time via `forge_get_release_news`

### My Issues — included in this release

- Agent navigation to **Submitted by me** / **Assigned to me** tabs
- Fixed title resolution when staff ask for a submitted issue by name
- Settings shows the correct working directory on the My Issues list
- Agent-selected issues update the UI panel; Jobs tab no longer snaps back from issue drill-in
- Agent can read issue comments and activity; conversation stream scopes to the focused issue

### Who should update

- **Everyone on 6.34.x** who wants **Threads**, **work-state sync**, or the Daily Flow task improvements
- **Multi-device users** — run Backfill once after updating, then Sync now on each machine
- **Free users** trying Threads — you get one active thread; archive or complete before starting another

## 6.34.4 (2026-06-27)

**My Issues improvements**

- Agent can now navigate you to "Submitted by me" or "Assigned to me" tabs correctly
- Fixed: asking the agent to select a submitted issue by title no longer opens the wrong (assigned) copy
- Fixed: Settings panel shows correct working directory when on the My Issues list
- Fixed: agent-selected issues now properly update the UI panel for staff users
- Fixed: Jobs tab snap-back when navigating from an issue
- Agent can now read issue comments and activity
- Reduced voice-mode log noise
- Agent conversation stream now correctly scopes to your focused issue

## 6.34.3 (2026-06-25)

- Operator browser

_Auto-generated — curate before external comms._

## 6.34.2 (2026-06-23)

### My Issues — reliable assignee inbox

- **Assigned to me** now loads your **open** tickets first — closed work no longer pushes active assignments off the list when you have many resolved issues
- **My Requests** still shows **resolved** submissions so you can track closure
- Agent issue lists follow the same rules as the panel

### Who should update

- Staff with large My Issues history who saw missing open tickets
- Anyone on **6.34.1** using My Issues assignee or submitter lenses

## 6.34.1 (2026-06-23)

### My Issues — agent navigation and calmer account switching

- The agent can **open My Issues** and **drill into a specific ticket** by title or id — free users see **My Requests**, staff see the assignee inbox
- **Account switching** no longer floods you with dozens of “project removed” warnings — you get at most one summary toast
- Clicking a request you submitted now shows an **Issue** context pill and resumes prior chat about that ticket (read-only — you still cannot mutate tickets you did not assign)
- **Fix:** agent navigation to an issue now opens the detail panel reliably

### Who should update

- Anyone on **6.34.0** using **My Issues** with the agent, or switching Firebase accounts on one machine
- Strongly recommended if the agent listed jobs instead of your requests, or issue drill-in did not open the UI row

## 6.34.0 (2026-06-23)

### PDF deliverables — `document_render_pdf` agent tool

Agents can now produce real PDF files from HTML or markdown using Forge's built-in document engine (headless Chromium). Use it for quotes, briefs, patent drafts, and any PDF the user asks for — no browser print workarounds.

- **`document_render_pdf`** — pass `html` or `markdown` + optional `template` (`minimal` or `patent`) and a required `output_path`
- Works alongside existing **`pdf_read`** (extract text from PDFs)
- **Pro tier** and above
- Requires Chromium, Chrome, or Edge installed on the machine

### Who should update

- Anyone on **6.33.x** who wants agents to generate PDF deliverables in the workspace
- Pairs with the tool surface registry from 6.33.0 — search for `document_render_pdf` if it is not in the hot tool set

## 6.33.0 (2026-06-22)

### Tool surface registry — smarter tool selection for large catalogs

Forge can send a **curated hot set** (~49 common tools) directly to the model and expose the rest via **tool_search → invoke_tool**, staying under provider tool caps (e.g. Grok 200). **Enabled in shipped Forge builds** from this release; local release builds without the CI compile flag stay off until you opt in.

- First-turn system prompt matches the inline tool list (hot tools documented in full; cold tools listed as searchable names)
- Discovery instructions guide the agent to search before assuming a tool is unavailable
- Voice sub-sessions keep the legacy full tool surface
- Disable at runtime with `FORGE_TOOL_SURFACE_REGISTRY=0` if you need the legacy full inline surface

**Known limitation:** MCP tools from a server connected _after_ the session started require a **new session** to appear in the catalog.

### My Issues — clearer Service Desk vocabulary, manual intake, agent tools, and session context

- Promoted Service Desk jobs label the J1 section **Issue Summary** (not Quick Plan)
- My Issues list and detail show **tags** and intake kind (Defect, Improvement, etc.)
- Daily Flow and My Issues tabs no longer show **+ New Project** in the toolbar
- **+ New Issue** on the My Issues tab submits a ticket to Portal Service Desk for team triage — track it under **Submitted by me** (staff) or **My Requests** (all users)
- The agent can list assigned and submitted issues, check status, add comments, resolve tickets, and create new issues
- **Selecting an issue** now updates session context like projects and jobs: an amber **Issue** pill in the session bar, chat resumes per issue when you switch rows, and clearing context returns you to the My Issues tab

### Who should update

- Anyone on **6.32.x** using **My Issues**, **Service Desk promoted jobs**, or **Grok/OpenAI** with the full tool catalog
- Recommended if you hit provider tool-cap errors or want the agent to work issues with proper session context when switching between tickets

## 6.32.1 (2026-06-18)

### Daily Flow — dependency removals that stay removed

Removing a task dependency now sticks. Previously, navigating away (or restarting Forge) could bring the chip back — especially on carried plan tasks after overnight roll. Removals are recorded on the day and honoured across navigation, list refresh, and startup carry.

- **Add and remove dependencies** from the task menu — blocked-by chips show on the task list with satisfied/blocked indicators.
- **Standing briefing gates** respect your choice — if you remove the email → briefing-doc dependency, it will not be re-seeded when you return to the day.
- **Smarter overnight carry** — completed tasks no longer roll forward; tasks you explicitly drop stay dropped; standing process tasks are not duplicated on carry.

### Agent Observatory — rows that hydrate correctly

- **Task numbers and titles** populate on Observatory rows as soon as Daily Flow tasks load — no more anonymous delegated lanes after navigation.
- **Session-scoped rows** — switching context no longer bleeds events between days or jobs.

### Voice — natural speed control

- **Pitch-preserving speed** (WSOLA) — Clara and cast speech stay natural at faster or slower playback, not chipmunk/slurred.
- **Coarser speed slider** — easier to land on a comfortable listening pace.

### Who should update

- Anyone on **6.32.0** who uses **Daily Flow task dependencies**, **Agent Observatory**, or **voice playback speed**.
- Strongly recommended if you removed a dependency and saw it reappear after leaving the day or restarting Forge.

## 6.32.0 (2026-06-13)

### Agent Observatory — finally, a window into what your agents are doing

Delegated tasks and direct persona conversations no longer fight for space in the main Agent Stream. **Agent Observatory** (right rail) gives each agent its own lane.

- **One row per agent** — delegated tasks show as **Task N · title**; direct chats show as **Direct · Patrick** (or Aurora, Lens, etc.). Expand a row to read that agent's stream: your messages, replies, tool calls, and live thinking — without cluttering Clara's conversation.
- **Talk to personas with `@`** — type `@patrick`, `@aurora`, `@lens`, `@architect`, and more in the compose box. Resolved mentions highlight in cyan so you always know who you're addressing. No more mystery routing.
- **STREAM badges** above compose filter what appears in the main Agent Stream. Observatory holds the delegated work — clean separation, no duplicate persona banners.
- **Follow up** on any Observatory row to continue that thread in compose. The faint **cyan row highlight** shows which persona is currently active.
- **All / Direct** toggle — focus on direct persona threads when you want to, or see every tracked lane.
- **Clara knows the Observatory** — ask her how to watch Patrick classify your inbox or follow up with Lens; she can coach you through it.

### Daily Flow polish

- **Skip Plan** during planning when you want to move on without drafting a Quick Plan first.

### Who should update

- Anyone on **6.31.0** or later who delegates Daily Flow tasks or uses **`@persona`** direct messages.
- Recommended after one Observatory UAT pass — expand rows, send `@` messages to two personas, confirm each row only shows its own **You** messages.

> **Note:** Daily Flow task dependencies, inbox cleanup, and process templates shipped in **6.31.0** (12 Jun 2026). This release is **Observatory-only** on top of that baseline.

## 6.31.0 (2026-06-12)

### Daily Flow — task order that actually means something

- **Link tasks on your day** — mark what must finish before something else can start. Blocked tasks show clearly; satisfied dependencies unlock automatically.
- **Jobs can gate your day** — a task can wait until a FORGE job completes, so cross-repo work and daily tasks stay in sync.
- **Add standing processes from the task picker** — morning briefing, inbox cleanup, and other process templates are one click away when you add a task.

### Inbox cleanup — Patrick classifies the whole inbox, you approve once

This is the big one. Blue-moon inbox cleanup is now a **plan → review → confirm** flow instead of a chat roulette.

- **Delegate “Full email clean up”** — FORGE fetches every message in your Gmail inbox (`in:inbox`, not just the Primary tab count) and Patrick classifies each one in **audited batches** you can watch in Agent Stream (`patrick-prep` sessions).
- **Live progress** — while Patrick works, Daily Flow shows how many emails were found and batch progress (e.g. `2/3`), not an endless spinner.
- **Review before anything moves** — Trash, Archive, and **Keep** counts are all inspectable. Click a subject to **read the full email** in-app; pull individual messages out of a batch before you approve.
- **Two confirmations after approve** — only Trash and Archive need Gmail bulk confirm (Keep stays in inbox). When you confirm the last batch, the task completes automatically.
- **No more 48-iteration Patrick loops** on delegate — preparation runs as a server pipeline, not a text agent told to “do nothing.”

### Gmail reliability

- Large inbox surveys and bulk-action previews **finish in seconds**, not minutes — concurrent metadata fetch and sensible timeouts.
- **Drafts include your Gmail send-as signature** (HTML) when you create them from FORGE.

### Who should update

- Anyone using **Daily Flow task dependencies**, **process tasks** (especially **inbox cleanup**), or **Gmail bulk approve** workflows.
- Recommended after a heavy inbox-cleanup UAT pass — this release is the polished pipeline Simon validated on 2026-06-12.

## 6.30.0 (2026-06-10)

### Daily Flow — close your day properly

- **Evening wrap-up ceremony** mirrors the morning briefing: Aurora writes the wrap document, Clara walks you through it section by section, then you **Close day** when you're done.
- **No more typing a wrap-up summary** — the wrap document is the record. Clara won't ask you to repeat what Aurora already wrote.
- **Close day** button replaces "Advance Phase" while you're in Wrapping — enabled once the wrap document is ready.
- **Day notes** — optional scratch pad on the day view for quick reminders (separate from the wrap-up).
- **Reopen today's day** if you wrapped up early and more work appears — same calendar day only.
- **Local Weather** is on by default for morning briefings and wrap-up outlook; turn it off in briefing interests if you prefer.

### Gmail — trash and approvals actually work

- **Trash and untrash** in Needs Attention now use the correct Gmail API path — fixes messages that silently failed or returned errors.
- When trash fails for specific messages, you see **which ones** failed instead of a false "all done".
- **Clara and text agents know when you approve or reject** a draft email or bulk action — no more re-asking after you've already decided in the UI.

### Editor

- Markdown and other previewable files open in **preview** first so you read documents without staring at raw source.

### Who should update

- Anyone using **Daily Flow** (especially voice wrap-up), **Gmail Needs Attention**, or the **in-app editor** for day documents.

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
