---
name: vega-os-app
description: Use when building, running, testing, releasing or publishing a React Native app for Amazon Fire TV on Vega OS (Fire TV Stick 4K 3rd gen, 4K Select, HD), including Vega SDK, Vega CLI, VDA, .vpkg and Amazon Appstore questions.
---

# Vega OS app development

A workflow for a secure, stable and fast Vega OS TV app, from an empty Mac to the Amazon Appstore and beyond. Checked against Amazon's Vega docs **2.0** (SDK, OS and Developer Tools 2.0, GA) on 2026-11-20.

- `references/*.md`: the workflow, step by step.
- `references/docs/*.md`: sourced fact sheets (every line cites the Amazon page and doc version). Use them for exact commands, package names, manifest keys and limits.

> Vega moves fast. Before running or recommending a `vega`/`npx` command, a package version or a manifest key, check `vega --help` / `vega <command> --help` or developer.amazon.com/docs/vega. When the docs disagree with these files, the docs win; say so.

## Project values

This skill is generic. Read the project's own values (package ID, publisher account, test device, dev machine) from the repo's `CLAUDE.md` or project profile. If a value is missing, ask; never guess a package ID.

## Lifecycle at a glance

| # | Step | How often | Key commands / actions | Read |
|---|------|-----------|------------------------|------|
| 1 | Set up the dev machine | once | Apple Silicon Mac or Ubuntu x86_64, Node 22.14.0+, `get_vvm.sh` installer, telemetry off, `vega --version`, `vega devmode login` | `references/setup.md` |
| 2 | Set up the Fire TV Stick | once | Developer options, `vega devmode enable-device --code <code>`, `vega exec vda devices` | `references/setup.md` |
| 3 | Create the repo | once | day-one files, branch protection, secrets in CI only | `references/repo-and-app.md` |
| 4 | Generate the app | once | `vega project generate --template helloWorld ...`, `[os.version]`, `vega project install --fix`, `vega project doctor` | `references/repo-and-app.md` |
| 5 | Dev loop | daily | `npx react-native build-vega`, `vega run-app <vpkg> <app-id> -d <device>`, Fast Refresh, logs | `references/dev-loop.md` |
| 6 | Quality gates | every feature | KPI targets, memory limits, lifecycle, security, TV UX | `references/quality-gates.md` |
| 7 | Test | every change | Jest with Vega mocks, Appium Vega driver, on-device checks, App Tester | `references/testing-ci.md` |
| 8 | CI | every push / PR | non-interactive pinned SDK, `doctor` gate, lint, test, release `.vpkg` | `references/testing-ci.md` |
| 9 | Release | per version | `--build-version` / `--build-number`, `armv7` release `.vpkg`, smoke test on the stick | `references/release-publish.md` |
| 10 | Publish | per version | Developer Console, assets, privacy questionnaire, Vega test cases, review | `references/release-publish.md` |
| 11 | Maintain / retire | ongoing | SDK upgrades, OS updates, monitoring, unpublish | `references/release-publish.md` |

## Which file to read for what

| Question or task | Read |
|------------------|------|
| Host requirements, SDK install, telemetry, VVD, Vega Studio, stick + Developer Mode | `references/setup.md` |
| Templates, `manifest.toml`, `[os.version]`, compatibility profile, SDK pin, monorepo | `references/repo-and-app.md` |
| Build, install, launch, Fast Refresh, debugger, logs, crash reports | `references/dev-loop.md` |
| What "fast, stable, secure" means on Vega, with numbers | `references/quality-gates.md` |
| Unit tests, UI automation, remote simulation, CI | `references/testing-ci.md` |
| Versioning, signing, Appstore assets and review, updates | `references/release-publish.md` |
| Video, DRM, IAP, Content Launcher, Account Login, deep links, ads | `references/docs/media-iap-integrations.md` |
| Official samples, forum, Amazon's MCP server, community skills | `references/ecosystem.md` |
| Kick off a new app with a coding agent | `references/agent-prompt.md` |
| Exact CLI flags, manifest keys, device specs | `references/docs/toolchain.md` |
| KPI tooling, lifecycle and memory APIs, ANR limits | `references/docs/performance.md` |
| Privileges, crypto, storage, signing, submission detail | `references/docs/security-appstore.md` |
| Jest config, Appium, focus APIs, navigation, a11y, i18n | `references/docs/testing-ux-ecosystem.md` |

## Hard rules

- **Package ID:** take it from the project values. Reverse DNS, at least 2 segments, `A-Za-z0-9._` only. Never use "amazon" in it. It can never change once published.
- **Pin everything:** every dependency, tool, SDK version (`vega-sdk-requirements.json`, `VEGA_SDK_VERSION`) and GitHub Action to an exact version, release tag or commit SHA. Never "latest", "^" or a main branch. Amazon's own examples often use `@latest` or `~`: pin them anyway.
- **Verify before use:** Vega commands, packages and manifest keys change between SDK versions. Check `vega --help` / `vega <command> --help` or the docs before running or recommending one, and say when unsure. Never invent a command, package or key.
- **Layout:** Vega-specific code in `apps/vega`; shared logic in `packages/core` and `packages/ui`; platform concerns behind adapters (player, IAP, storage, analytics, deep links).
- **Secrets:** never commit secrets; use CI secrets. Before any release build, delete Network Proxy files (`proxy-config.json`, `assets/raw/ca.pem`).
- **Telemetry:** the Vega SDK sends telemetry by default. Turn it off (`"optIn": false` in `~/vega/telemetry/config.json`) and check every other tool (MCP servers, CLIs) before adding it.
- **Developer Mode:** enabling it reboots the stick and opens it to VDA connections and sideloading; keep it on a trusted network and say so before suggesting it. It can be turned off again in Developer Options (another reboot); a factory reset also disables it but signs out and deregisters the device.
- **Real device:** performance numbers and Appstore readiness come from a release build on a physical Fire TV Stick, not the Virtual Device.

## Before running any vega / npx command

1. Confirm the command and flags with `vega <command> --help` for the installed SDK (`vega --version`, `vega sdk ls`).
2. Run app commands from the app directory (`apps/vega`).
3. Match the `.vpkg` architecture to the target: `armv7` for Fire TV Sticks, `aarch64` for the VVD on Apple Silicon, `x86_64` for the VVD on Intel/Ubuntu.
4. Device commands need the device connected over VDA: `vega exec vda devices` should list it.
5. The app ID for launch commands is the interactive component ID (for example `<package-id>.main`), not the package ID.
