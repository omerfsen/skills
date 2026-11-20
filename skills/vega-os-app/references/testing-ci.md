# Testing and CI

Exact sources: `docs/testing-ux-ecosystem.md` (section 1), `docs/toolchain.md` (section 9), `docs/performance.md` (KPI scenarios). Docs version 2.0.

> Vega moves fast: if a command fails, check `vega --help` / `vega <command> --help` and developer.amazon.com/docs/vega before improvising.

## Test layers

| Layer | Tool | Runs on |
|-------|------|---------|
| Business logic (`packages/core`) | Jest | CI, no device |
| Components (`packages/ui`, `apps/vega`) | Jest + React Native Testing Library, with Vega mocks | CI, no device |
| Lint (performance, a11y, OS compatibility) | `@amazon-devices/eslint-plugin-kepler`, `vega project doctor` | CI |
| UI flows | Appium + Vega driver | VVD or stick |
| Performance | App KPI Visualizer (release build) | stick |
| IAP | App Tester, then Live App Testing | VVD/stick, then production |
| Appstore cases | manual checklist (the repo's own `docs/test-checklist.md`) | stick |

## Unit and component tests (RN 0.83)

There is no Vega Jest preset; the template uses `preset: 'react-native'`. You need:

- `@amazon-devices/react-native-kepler` (and `@amazon-devices`) in `transformIgnorePatterns`.
- A mock for `@amazon-devices/react-native-kepler` (e.g. `TVFocusGuideView` → `View`).
- TV mocks: window 960×540 dp at scale 2, `PixelRatio.get()` = 2; mocks returning `{ __esModule: true, default: impl, ...impl }`.
- React 19 `act()` rules and fake timers for the stack navigator.

The full `jest.config.js` and mocks are in `docs/testing-ux-ecosystem.md` section 1.2. Keep logic out of components so most tests need no device.

## UI automation (Appium)

```bash
npm install -g appium@2.2.2
appium driver install --source=npm @amazon-devices/appium-kepler-driver@3.30.0
export PATH="$(dirname "$(vega exec which vda)"):$PATH"
vega exec vda shell "touch /tmp/automation-toolkit.enable"   # turn on the on-device automation toolkit
```

Capabilities: `platformName: "Kepler"`, `appium:automationName: "automation-toolkit/JSON-RPC"`, `kepler:device: "vda://default"`. Only UiSelector and XPath locators work. The same Python scenarios feed the KPI Visualizer (`--test-scenario`).

## Remote control and screenshots from the CLI

```bash
vega exec vda shell inputd-cli --help
vega exec inputd-cli button_press KEY_BACK
vega exec inputd-cli send_text 'hello\ world\!'
vega exec vda shell gwsi-tool-screenshooter /tmp/test.png
vega exec vda pull /tmp/test.png
```

On VVD, input automation needs `vsm developer-mode enable` first (see `setup.md`). Docs show two forms of the `inputd-cli` command: check `--help`.

## On-device checks (every PR that touches UI)

Run on the real stick with the remote: focus order, Back, visible focus, 10-foot readability, cold start time. Before a release also run the Appstore cases in `release-publish.md`.

## In-app purchases

App Tester for sandbox purchases before publishing (setup in `docs/media-iap-integrations.md` section 2), then Live App Testing with real accounts against production services. Verify receipts on the backend.

## CI (GitHub Actions)

Hosts: Ubuntu 20.04/22.04/24.04/26.04 x86_64 or macOS. Pin the runner image (e.g. `ubuntu-24.04`, not `ubuntu-latest`).

1. Checkout; set up Node from `.nvmrc` (22.14.0+). Amazon's sample Dockerfile installs Ubuntu 22.04's distro `nodejs`, which is older than that: install the pinned Node yourself.
2. Install the SDK non-interactively, pinned, telemetry off:

   ```bash
   export NONINTERACTIVE=true
   export SKIP_VVD_INSTALL=true
   export VEGA_SDK_VERSION=<exact SDK version>
   curl -fsSL https://sdk-installer.vega.labcollab.net/get_vvm.sh | bash
   vega -v
   mkdir -p ~/vega/telemetry && echo '{"optIn": false}' > ~/vega/telemetry/config.json   # telemetry off
   ```

   Download the installer to a file and checksum it if you want a stronger supply-chain guarantee. Cache `~/vega` keyed on the SDK version (Amazon gives no cache paths; this is inferred).
3. `vega project install --fix` → `npm ci` → `vega project doctor` (non-zero = fail).
4. Lint (incl. `eslint-plugin-kepler`) → typecheck → `npm test`.
5. Release build: `npx react-native build-vega --build-type Release --build-version <v> --build-number <n>`.
6. Upload `build/armv7-release/*.vpkg` as an artifact; on version tags attach it to a GitHub Release.

Pin every action to a commit SHA. Secrets only in Actions secrets. Add a one-line kill switch for automatic jobs.
