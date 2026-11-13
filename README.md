<p align="center">
  <h1 align="center">SCAINET Forge</h1>
  <p align="center"><strong>The development tool for the 99% who aren't developers.</strong></p>
  <p align="center">
    <a href="https://github.com/scainet-enterprise/forge-releases/releases/latest"><img src="https://img.shields.io/badge/version-6.79.5-blue" alt="Version 6.79.5"></a>
    <a href="https://github.com/scainet-enterprise/forge-releases/releases/latest"><img src="https://img.shields.io/badge/status-Stable-brightgreen" alt="Stable"></a>
    <img src="https://img.shields.io/badge/platforms-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey" alt="Platforms">
    <a href="https://scainet.io/innovation"><img src="https://img.shields.io/badge/patents-filed-orange" alt="Patents"></a>
  </p>
</p>

---

AI agents are the developers. You provide the vision.

**Voice-first. No coding required. Every action visible and reversible.**

FORGE is a desktop application where AI agents build software while you watch. Speak an instruction. The agent reads your codebase, writes code, runs terminals, manages Git, and reports every action through a real-time thought stream. You see everything. You control everything.

If you can talk, you can build software.

## Download

The buttons below are the latest release, **6.79.5**. A new install follows the stable channel, which is **6.79.5**.

| Platform | Download |
|----------|----------|
| **Windows** (x64) | [SCAINET-Forge_6.79.5_x64-setup.exe](https://github.com/scainet-enterprise/forge-releases/releases/download/v6.79.5/SCAINET-Forge_6.79.5_x64-setup.exe) |
| **macOS** (Apple Silicon) | [SCAINET-Forge_6.79.5_aarch64.dmg](https://github.com/scainet-enterprise/forge-releases/releases/download/v6.79.5/SCAINET-Forge_6.79.5_aarch64.dmg) |
| **macOS** (Intel) | [SCAINET-Forge_6.79.5_x64.dmg](https://github.com/scainet-enterprise/forge-releases/releases/download/v6.79.5/SCAINET-Forge_6.79.5_x64.dmg) |
| **Linux** (AppImage) | [SCAINET-Forge_6.79.5_amd64.AppImage](https://github.com/scainet-enterprise/forge-releases/releases/download/v6.79.5/SCAINET-Forge_6.79.5_amd64.AppImage) |
| **Linux** (Debian) | [SCAINET-Forge_6.79.5_amd64.deb](https://github.com/scainet-enterprise/forge-releases/releases/download/v6.79.5/SCAINET-Forge_6.79.5_amd64.deb) |

Or browse [all releases](https://github.com/scainet-enterprise/forge-releases/releases).

## First Run

1. **Sign in** - email and password, or GitHub
2. **Choose a model** - Anthropic, Google AI Studio (Gemini), Hugging Face (Inference), OpenAI, OpenRouter, xAI (Grok), and Ollama (on this computer). Bring your own key. Ollama runs on this computer.
3. **Open a project** - clone from GitHub or open a local folder
4. **Give an instruction** - type or speak. The agent does the rest.

## What Makes FORGE Different

| | FORGE | Other AI IDEs |
|---|---|---|
| **Built for** | Everyone. Voice-first, no coding needed | Developers only |
| **Agent governance** | 10-Article Constitution, mechanically enforced | None |
| **Agent identity** | Named agents with persistent memory (EGO) | Anonymous, stateless |
| **Output verification** | PNEUMA scans every response for assumptions | None |
| **Voice interaction** | Built-in speech-to-text and text-to-speech | None |
| **Audit trail** | Every action recorded, every session replayable | None |
| **MCP integration** | Built-in marketplace | Varies |
| **Your API keys** | Bring your own keys | Vendor-locked |

## Features

- **250+ agent tools** - file operations, terminal, Git, web search, code analysis, database, memory, MCP, and computer use
- **7 model providers** - Anthropic, Google AI Studio (Gemini), Hugging Face (Inference), OpenAI, OpenRouter, xAI (Grok), and Ollama (on this computer)
- **Voice mode** - press the mic, speak an instruction, watch it happen
- **The Trinity** - HUMAN (input) + EGO (identity) + PNEUMA (checking what was said)
- **10-Article Constitution** - governance rules enforced on every action, not just suggested
- **MCP marketplace** - one-click install of servers such as GitHub, PostgreSQL, Slack, and Brave Search
- **GitHub sign-in** - sign in with a GitHub account
- **Cross-platform** - Windows, macOS (ARM and Intel), Linux
- **Updates** - the app checks for a new version on its channel

## Links

- [Website](https://scainet.io)
- [FORGE Product Page](https://scainet.io/products/scainet-forge)
- [Innovation and patents](https://scainet.io/innovation)
- [Research papers](https://scainet.io/open-source)

## Patents

SCAINET Forge incorporates patented technology. The filings are listed at [scainet.io/innovation](https://scainet.io/innovation).

## About This Repository

This repository contains compiled binaries and update manifests only. Source code is proprietary and maintained in a separate private repository. SCAINET Forge is built with [Tauri](https://tauri.app) (Rust backend) and [Svelte](https://svelte.dev) (frontend).

The feature lines above are generated from the Forge source. Editing them in this file does not last: the next release, stable promotion, or facts publish writes them again.

## Release Channels

| Channel | URL | Description |
|---------|-----|-------------|
| **Beta** | `beta/latest.json` | Development builds. Updated when a version is released. Now 6.79.5. |
| **Stable** | `stable/latest.json` | The build a new install follows. Promoted from beta after the gates. Now 6.79.5. |

Updater manifests are files in this repository, not release assets. The addresses stay the same. The files they point at change. The binaries stay on the GitHub release.

### For developers

A new install checks:

```
https://raw.githubusercontent.com/scainet-enterprise/forge-releases/main/stable/latest.json
```

An install that already had settings, and never picked a channel, checks:

```
https://raw.githubusercontent.com/scainet-enterprise/forge-releases/main/beta/latest.json
```

---

Built by [SCAINET](https://scainet.io).
