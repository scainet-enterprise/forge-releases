<p align="center">
  <h1 align="center">SCAINET Forge</h1>
  <p align="center"><strong>The development tool for the 99% who aren't developers.</strong></p>
  <p align="center">
    <a href="https://github.com/scainet-enterprise/forge-releases/releases/latest"><img src="https://img.shields.io/badge/version-5.14.0-blue" alt="Version"></a>
    <a href="https://github.com/scainet-enterprise/forge-releases/releases/latest"><img src="https://img.shields.io/badge/status-Beta-brightgreen" alt="Beta"></a>
    <img src="https://img.shields.io/badge/platforms-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey" alt="Platforms">
    <a href="https://scainet.io/innovation"><img src="https://img.shields.io/badge/patents-6%20filed-orange" alt="Patents"></a>
  </p>
</p>

---

AI agents are the developers. You provide the vision.

**Voice-first. No coding required. Every action visible and reversible.**

FORGE is a desktop application where AI agents build software while you watch. Speak an instruction — the agent reads your codebase, writes code, runs terminals, manages Git, and reports every action through a real-time thought stream. You see everything. You control everything.

If you can talk, you can build software.

## Download

| Platform | Download | Size |
|----------|----------|------|
| **Windows** (x64) | [SCAINET-Forge_5.14.0_x64-setup.exe](https://github.com/scainet-enterprise/forge-releases/releases/download/v5.14.0/SCAINET-Forge_5.14.0_x64-setup.exe) | ~9 MB |
| **macOS** (Apple Silicon) | [SCAINET-Forge_5.14.0_aarch64.dmg](https://github.com/scainet-enterprise/forge-releases/releases/download/v5.14.0/SCAINET-Forge_5.14.0_aarch64.dmg) | ~14 MB |
| **macOS** (Intel) | [SCAINET-Forge_5.14.0_x64.dmg](https://github.com/scainet-enterprise/forge-releases/releases/download/v5.14.0/SCAINET-Forge_5.14.0_x64.dmg) | ~16 MB |
| **Linux** (AppImage) | [SCAINET-Forge_5.14.0_amd64.AppImage](https://github.com/scainet-enterprise/forge-releases/releases/download/v5.14.0/SCAINET-Forge_5.14.0_amd64.AppImage) | ~89 MB |
| **Linux** (Debian) | [SCAINET-Forge_5.14.0_amd64.deb](https://github.com/scainet-enterprise/forge-releases/releases/download/v5.14.0/SCAINET-Forge_5.14.0_amd64.deb) | ~15 MB |

Or browse [all releases](https://github.com/scainet-enterprise/forge-releases/releases).

## First Run

1. **Sign in** — email/password or GitHub OAuth
2. **Enter your API key** — Grok, Claude, GPT, or Ollama (bring your own key)
3. **Open a project** — clone from GitHub or open a local folder
4. **Give an instruction** — type or speak. The agent does the rest.

## What Makes FORGE Different

| | FORGE | Other AI IDEs |
|---|---|---|
| **Built for** | Everyone — voice-first, no coding needed | Developers only |
| **Agent governance** | 10-Article Constitution, mechanically enforced | None |
| **Agent identity** | Named agents with persistent memory (EGO) | Anonymous, stateless |
| **Output verification** | PNEUMA — scans every response for assumptions | None |
| **Voice interaction** | Built-in speech-to-text and text-to-speech | None |
| **Audit trail** | Every action recorded, every session replayable | None |
| **MCP integration** | Built-in marketplace, 10+ curated servers | Varies |
| **Build size** | < 20 MB (native desktop, Tauri + Rust) | 500 MB+ (Electron) |
| **Your API keys** | BYOK — bring your own keys, zero lock-in | Vendor-locked |

## Features

- **39+ Agent Tools** — file operations, terminal, Git, web search, code analysis, database, memory, MCP, computer use
- **4 LLM Providers** — Grok, Claude, GPT, Ollama (bring your own key)
- **Voice Mode** — press mic, speak your instruction, watch it happen
- **The Trinity** — HUMAN (input interpretation) + EGO (persistent identity) + PNEUMA (output verification)
- **10-Article Constitution** — governance rules enforced on every action, not just suggested
- **MCP Marketplace** — one-click install of GitHub, PostgreSQL, Slack, Brave Search, and more
- **GitHub OAuth** — sign in with your GitHub account via Device Flow
- **Tiered Subscriptions** — Free / Pro ($29/mo) / Enterprise
- **Cross-Platform** — Windows, macOS (ARM + Intel), Linux
- **Auto-Updates** — Tauri updater checks for new versions automatically

## Links

- [Website](https://scainet.io)
- [FORGE Product Page](https://scainet.io/products/scainet-forge)
- [Innovation & Patents](https://scainet.io/innovation)
- [Research Papers](https://scainet.io/open-source)

## Patents

SCAINET Forge incorporates technology protected by 6 provisional patent applications filed with IP Australia. See [scainet.io/innovation](https://scainet.io/innovation) for details.

## About This Repository

This repository contains compiled binaries and update manifests only. Source code is proprietary and maintained in a separate private repository. SCAINET Forge is built with [Tauri](https://tauri.app) (Rust backend) and [Svelte](https://svelte.dev) (frontend).

## Release Channels

FORGE uses a channel-based release system for controlled rollout:

| Channel | URL | Description |
|---------|-----|-------------|
| **Beta** | `beta/latest.json` | Current development builds — updated automatically on release |
| **Stable** | `stable/latest.json` | Production-ready builds — promoted from beta after verification (coming soon) |

**Updater manifests** are stored as repo files (not release assets) to enable:
- Multiple channels without asset duplication
- Trivial promotion between channels
- Stable URLs that never change

Assets (binaries) remain in GitHub Releases. Manifests point to those assets.

### For Developers

The Tauri updater checks:
```
https://raw.githubusercontent.com/scainet-enterprise/forge-releases/main/beta/latest.json
```

This URL will remain stable. When stable channel launches, users can switch via:
```
https://raw.githubusercontent.com/scainet-enterprise/forge-releases/main/stable/latest.json
```

---

Built by [SCAINET](https://scainet.io) — Human + AI, building the future together.
