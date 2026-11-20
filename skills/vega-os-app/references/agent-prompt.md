# Prompt for the coding agent

Start once `vega --version` works and `vega exec vda devices` lists the stick (or the Virtual Device is running). Fill in the bracketed fields, then paste into Claude Code (or a similar agent) from the repo root.

```text
You are a senior React Native engineer helping me build a TV app for Amazon Fire TV devices running Vega OS. iOS/tvOS and Android/Android TV support may come later, so architect for that from day one without building it yet.

## Context
- Repository: [REPO URL]. The vega-os-app skill (.claude/skills/vega-os-app/) describes the full lifecycle and quality gates; follow it. Project values are in CLAUDE.md.
- Target platform (now): Amazon Vega OS 2.0 (Linux-based; Fire TV Stick 4K 3rd gen / 4K Select / HD). Apps are React Native 0.83 + TypeScript, packaged as .vpkg (armv7 for the sticks), built with the Vega SDK / Vega CLI.
- Future platforms (later): Android TV, Apple TV (via react-native-tvos), and possibly iOS/Android mobile.
- Dev machine: [macOS on Apple Silicon / Ubuntu x86_64]. My experience: [e.g. experienced platform engineer, new to React Native — explain RN-specific concepts briefly].
- Test devices: [physical Fire TV Stick model] in Developer Mode, plus the Vega Virtual Device.

## App
- Name: [APP NAME]
- Package ID: [reverse.domain.appname — never containing "amazon"]
- Publisher: [COMPANY OR DEVELOPER NAME on the Amazon Developer account]
- What it does: [ONE-PARAGRAPH DESCRIPTION]
- Monetization: [free / paid / in-app purchase / subscription]
- Video playback needed: [yes/no, DRM: yes/no]
- Backend/API: [describe or "none yet"]

## Architecture requirements
1. Monorepo (npm or yarn workspaces) with:
   - packages/core — business logic, state, API clients, types (pure TS, no platform imports)
   - packages/ui — shared React Native components, theme, focus/navigation helpers
   - apps/vega — Vega OS app shell generated with `vega project generate` (only Vega-specific code lives here)
   - leave clear placeholders for apps/androidtv, apps/tvos, apps/mobile
2. Platform-specific concerns behind thin adapter interfaces in core: video player, in-app purchases, storage, analytics, deep links. Vega implementations first.
3. TV-first UX: D-pad focus management with the Vega focus APIs, 10-foot UI, visible focus states, Back handling, safe zones, no touch assumptions.
4. In-app purchases via react-native-iap with Amazon's Vega IAP library, behind the IAP adapter; receipts verified on the backend.
5. Performance from day one: report fully drawn, FlashList for rows, native-driver animations, VegaAppState lifecycle handling, memory under the Vega limits.
6. Security from day one: HTTPS only, no tokens in AsyncStorage, least-privilege manifest, no debug proxy files in release builds.
7. Strict TypeScript, ESLint (incl. @amazon-devices/eslint-plugin-kepler), Prettier. Unit tests (Jest, with the Vega mocks) for core; basic component tests for ui.

## Engineering rules
- Pin every dependency, tool, SDK version and GitHub Action to an exact version, release tag or commit SHA — never "latest", "^", "~" or a main branch.
- Before using any Vega SDK API, CLI command, package name or manifest key, check the current official docs (developer.amazon.com/docs/vega) or `vega <command> --help` rather than relying on memory; tell me when you are unsure.
- Keep Vega-specific code out of shared packages; if something can't be shared, say so and why.
- Never commit secrets; use GitHub Actions secrets. Keep SDK and tool telemetry off.
- Prefer small, reviewable steps. After each step, tell me the exact commands to run and what I should see.

## Deliverables, in order
1. Short architecture overview (folder tree + one line per package) and the list of pinned dependencies with reasons.
2. Repo bootstrap: .gitignore, .nvmrc, workspace config, lint/format/test config, CHANGELOG.md, docs/ folder.
3. Scaffold the monorepo and a minimal Vega app: home screen with a focusable grid, a detail screen, navigation, and back handling; `vega project doctor` passes.
4. Scripts (Makefile or package.json) wrapping build, install, launch, uninstall, clean, logs and KPI measurement for both the Virtual Device and the stick.
5. CI pipeline (GitHub Actions): non-interactive pinned Vega SDK install, doctor gate, lint, typecheck, test, release .vpkg build, artifact upload; release workflow on version tags.
6. Amazon Appstore submission checklist for Vega (manifest category, assets, privacy questionnaire, IAP items, App Tester, Live App Testing, Vega test cases).
7. A short "porting later" note: what changes for Android TV, Apple TV and mobile, and the estimated share of code that stays common.

Start with deliverable 1 and wait for my confirmation before scaffolding.
```
