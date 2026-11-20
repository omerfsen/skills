# Ecosystem: samples, help, AI tooling

Checked 2026-11-20. Sources: `docs/testing-ux-ecosystem.md` (section 7), `docs/toolchain.md` (section 10).

## Official sample apps (github.com/AmazonAppDev)

| Repo | Use it to learn |
|------|-----------------|
| [vega-video-sample](https://github.com/AmazonAppDev/vega-video-sample) | W3C media player, search, focus, IAP, Content Launcher, Account Login, Live TV |
| [vega-tv-interfaces-sample](https://github.com/AmazonAppDev/vega-tv-interfaces-sample) | Focus, navigation, i18n, scrolling patterns |
| [vega-scrolling-sample](https://github.com/AmazonAppDev/vega-scrolling-sample) | Home grid layout and scroll performance (also a KPI Visualizer setup) |
| [vega-seekbar-sample](https://github.com/AmazonAppDev/vega-seekbar-sample) | TV seek bar and focus |
| [vega-sports-app](https://github.com/AmazonAppDev/vega-sports-app) | Theming, content catalog |
| [vega-audio-sample](https://github.com/AmazonAppDev/vega-audio-sample) | Music player |
| [vega-epg-sample](https://github.com/AmazonAppDev/vega-epg-sample) | TV guide (EPG) components |
| [react-native-multi-tv-app-sample](https://github.com/AmazonAppDev/react-native-multi-tv-app-sample) | One monorepo for Vega, Android TV and Apple TV |
| [vega-native-component-sample](https://github.com/AmazonAppDev/vega-native-component-sample) | C++ native UI components with D-pad focus and VoiceView |
| [vega-periodic-task-sample](https://github.com/AmazonAppDev/vega-periodic-task-sample) | Periodic headless tasks via a Turbo Module |

Read a sample's code; don't copy its version ranges (they use `~`/`^`).

## Where to get help

- Docs: https://developer.amazon.com/docs/vega/ (pick the version that matches your SDK).
- Release notes, including breaking changes and **known issues**: https://developer.amazon.com/docs/vega/2.0/vega-release-notes
- Troubleshooting index: https://developer.amazon.com/docs/vega/2.0/troubleshoot-overview
- Forum: https://community.amazondeveloper.com/c/vega/6
- Supported React Native libraries: https://developer.amazon.com/docs/vega-api/2.0/supported-libraries and https://reactnative.directory/packages?vegaos=true

## Amazon Devices Builder Tools (ADBT)

Amazon's official AI tooling for Vega: an MCP server (`@amazon-devices/amazon-devices-buildertools-mcp`), about a dozen `amazon-devices-vega-*` skills (manifest, performance, focus, media player, navigation, RN upgrade, setup…) plus caption, multi-TV and Android skills, and an optional `adbt-dev` agent. Amazon calls it "the recommended way to use AI coding agents for Vega development". Tools include `search_documentation`, `read_document`, `symbolicate_acr`, `analyze_perfetto_traces`, `get_app_hot_functions`.

Before installing:

- **Telemetry:** it sends documentation search queries to Amazon and collects anonymous telemetry; the docs say it can be turned off "in the settings" without naming the setting. Find and disable it, or decide consciously.
- **Pinning:** Amazon's command uses `@latest`. Pin an exact version instead:

  ```bash
  npx -y @amazon-devices/amazon-devices-buildertools-mcp@<exact-version> init-context --agent claude-code-cli
  ```

- It writes user-level MCP config and skills; `check-status` and `clean-context` inspect or remove them.

This skill complements ADBT: ADBT gives live API docs and diagnosis; this skill gives the end-to-end workflow, quality gates and release discipline.

## Community agent skills

| Skill | Source | Use it for | Notes |
|-------|--------|------------|-------|
| `rn-tv-ui-best-practices` | [AmazonAppDev/devices-agent-skills](https://github.com/AmazonAppDev/devices-agent-skills) (MIT) | TV UI: focus, D-pad, swimlanes, safe zones, 10-foot type, remote input; Vega pitfalls file and an `audit.js` helper | Written for SDK 0.22 / RN 0.72; check APIs against 2.0 docs |
| `vega-multi-tv-migration` | same repo | Vega app → monorepo with Android TV and Apple TV (Expo TV) | Later stage; its folders differ from `apps/vega` + `packages/*` |
| `react-native-tv-best-practices` | [callstackincubator/agent-skills](https://github.com/callstackincubator/agent-skills) (MIT) | TV review: focus, layout, playback, low-memory, a11y | Detects the stack first, so fine on Vega; overlaps the Amazon UI skill |
| `argent` (incl. `argent-vega`) | [software-mansion/argent](https://github.com/software-mansion/argent) | MCP server: drive VVD/TVs, e2e record/replay, logs, profiling | Telemetry on by default (`argent telemetry disable`); bundles proprietary binaries |

The two Amazon skills had no release tags when checked; commit `f31e0e6357d5987e003b69b263dd5bdc6748d6ab` (2026-11-17) was current. Pin to a commit and review a skill before installing it.
