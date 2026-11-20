# Repo bootstrap and app generation

Exact sources: `docs/toolchain.md` (sections 2, 6, 7, 8) and `docs/testing-ux-ecosystem.md` (section 6). Docs version 2.0.

> Vega moves fast: if a command fails, check `vega --help` / `vega <command> --help` and developer.amazon.com/docs/vega before improvising.

## 1. Create the repo (once)

```bash
gh repo create <repo-name> --private --clone
cd <repo-name>
```

Commit from day one:

- `.nvmrc` with the exact Node version (22.14.0 or later for SDK 2.0)
- `.gitignore` with at least: `node_modules/`, `build/`, `*.vpkg`, `.env*`, `generated/` (KPI reports), `*.acr`
- `README.md`, `CLAUDE.md` (project values for this skill), `CHANGELOG.md`
- `docs/adr/` for short decision records, `docs/test-checklist.md`

Protect `main` (PRs + green CI required). Keep secrets (Amazon credentials, API keys, signing material) only in CI secrets.

## 2. Generate the app (once)

Pick the package ID now; it is permanent once published (see the hard rules in `SKILL.md`).

```bash
vega project list-templates
# helloWorld        → React Native 0.83 (React 19, React Compiler, Hermes v1) — use for new apps
# helloWorld-rn72   → React Native 0.72
# basic-turbo-module, idl-turbo-module → native modules
# vegaWebview       → WebView app

vega project generate \
  --template helloWorld \
  --name "<AppName>" \
  --packageId "<package-id>" \
  --outputDir "apps/vega"
```

`--name` cannot contain spaces or special characters.

Then, from `apps/vega`:

```bash
vega project install --fix --os-min 2.0 --os-version 2.0   # aligns @amazon-devices/* and creates [os.version]
npm install
vega project doctor                                        # non-zero exit = not compatible; use as CI gate
npx react-native build-vega
```

- `[os.version]` with `min` and `target` is required (build fails without it). `min = "1.2"` also reaches OS 1.2 devices; guard newer APIs with `isPresentOnOS(...)` from `@amazon-devices/kepler-compatibility`.
- `vega project update-manifest` removes tool-owned module entries from the source manifest: commit the result, don't restore them by hand.
- Pin the SDK per project in `vega-sdk-requirements.json`:

  ```json
  { "vegaSdkVersion": "<exact SDK version>" }
  ```

- Pin every dependency to an exact version and commit the lockfile. `vega project install` writes `~` ranges for `@amazon-devices/*`; replace them with the exact resolved versions.

## 3. `manifest.toml` essentials

Full schema: `docs/toolchain.md` section 7. The keys you always need:

```toml
schema-version = 1

[package]
id = "<package-id>"
title = "<App title>"              # or "@text/<id>" from assets/text/puff.json
icon = "@image/<icon>.png"          # 512x512 PNG, max 1 MB, in assets/image

[os.version]
min = "2.0"
target = "2.0"

[components]
[[components.interactive]]
id = "<package-id>.main"           # this is the "app ID" used by launch commands
runtime-module = "/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0"
launch-type = "singleton"
categories = ["com.amazon.category.main"]   # exactly one interactive component must carry it; required for the Appstore

[needs]
[[needs.module]]
id = "/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0"
```

Add these when you use the feature (else the call errors at start, logged, or media playback never starts):

- Remote key events: `[[wants.service]] id = "com.amazon.inputd.service"` (from the media player FAQ)
- On-screen keyboard: `[[wants.service]] id = "com.amazon.inputmethod.service"`
- Media, DRM, IAP, Content Launcher, Account Login: entries in `docs/media-iap-integrations.md`.
- Validate: `vega exec vpt validate <path>/manifest.toml`

## 4. Monorepo and other platforms

- Layout: `apps/vega` (Vega-only), `packages/core` (pure TS logic, no platform imports), `packages/ui` (shared components, focus helpers); placeholders for `apps/androidtv`, `apps/tvos`, `apps/mobile`.
- On Vega `Platform.OS` is `'kepler'`; platform files use `.kepler.tsx`.
- npm/Yarn workspaces are supported (Vega Studio handles monorepos). Amazon's reference: `AmazonAppDev/react-native-multi-tv-app-sample`.
- On RN 0.83, import `@amazon-devices/*` packages directly (e.g. `@amazon-devices/react-navigation__*`) and don't add the Vega module resolver preset: the 0.83 template drops it and several 0.83 ports don't support it. On RN 0.72 the preset is `@amazon-devices/kepler-module-resolver-preset` (`@vega-tv/...` does not exist).
- Platform concerns go behind adapters in `packages/core`: player, IAP, storage, analytics, deep links.
