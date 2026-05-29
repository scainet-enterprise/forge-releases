# SCAINET Forge Release Notes

> **Maintainers:** User-facing notes mirrored to `forge-releases/RELEASE_NOTES.md` on every release. Write for end users — no internal codes, file paths, or PR numbers. CI requires a `## x.y.z` heading matching repo-root **`VERSION`**. Update in the **same PR as the version bump** (use `npm run prepare-release` or edit alongside `docs/CHANGELOG.md`).

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
