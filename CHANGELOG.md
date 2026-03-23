# SCAINET Forge Changelog

> **Maintainers:** This file is copied to [forge-releases `CHANGELOG.md`](https://github.com/scainet-enterprise/forge-releases/blob/main/CHANGELOG.md) on every release (at the release tag). Update it **in the same PR as the version bump** so the in-app updater shows current notes. CI requires a top-level `## x.y.z` heading matching the repo-root **`VERSION`** file (see `npm run sync-version` in CONTRIBUTING.md).

## 5.13.7 (2026-03-22)

- **File Explorer:** Phase 1 fix for "Loading…" blanking the tree during agent file activity — `refresh({ silent: true })` on `agent:file_changed`, sequence guard for overlapping refreshes, header ⟳ badge while updating, full loading only for explicit refresh / mount / `working_dir_changed`.
- **Refactor:** `src/lib/fileExplorer/fetchFileTreeSnapshot.ts` + `entriesToFileNodes` (shared with `loadDirectory`); Vitest coverage for staleness, races, empty/malformed IPC, and `rootPath` preserved when listing fails after `env_status`.
- **Docs:** [ACTIONABLE_BUG_FIXES.md](./ACTIONABLE_BUG_FIXES.md) / [ACTIONABLE_IMPROVEMENTS.md](./ACTIONABLE_IMPROVEMENTS.md) — Phase 1 marked complete; Phase 2 (merge / payload) and Phase 3 (store + fs notify) captured.

## 5.13.6 (2026-03-22)

- **Docs (S0):** [FORGE-ML-DATA-COLLECTION-S0.md](./FORGE-ML-DATA-COLLECTION-S0.md) — training-ready data collection for future fine-tuning (context summarisation, tool trajectories, preferences, consent tiers).
- **Docs (S0):** [S7-DEPLOYGATE-S0.md](./S7-DEPLOYGATE-S0.md) — S7 DeployGate intake (deployment ownership models, Vercel-first web apps, provider-adapter contract).
- **Docs:** [S6-DEVOPS-EVIDENCE-S0.md](./S6-DEVOPS-EVIDENCE-S0.md) — revisions to DevOps evidence / Launch Pad S0.
- **CI:** [auto-version-bump.yml](.github/workflows/auto-version-bump.yml) — smarter post-merge changelog entries (PR Summary / title, duplicate-heading guard, fallback placeholder).

## 5.13.5 (2026-03-21)

- **Docs (S0):** [FORGE-INCIDENT-INTAKE-S0.md](./FORGE-INCIDENT-INTAKE-S0.md) — Forge-native incident intake and AI triage (phased path alongside Sentry).
- **Docs (S0):** [S6-DEVOPS-EVIDENCE-S0.md](./S6-DEVOPS-EVIDENCE-S0.md) — DevOps evidence for S6 Launch Pad (CI/build/artifact signals in Forge; ownership boundaries).

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
