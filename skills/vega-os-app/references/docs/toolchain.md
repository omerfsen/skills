# Vega OS toolchain and project structure

> Fact sheet. Collected 2026-11-20 from Amazon's Vega docs (latest version: 2.0) and the sources listed at the end; every line names its source page and doc version. Vega changes between SDK versions: re-check a fact against the current docs (or `vega <command> --help`) before relying on it, and prefer the docs when they disagree with this file.


Latest docs version: **2.0** (sidebar says "Version 2.0 (LATEST)"; `/docs/vega/2.1/`, `/2.2/`, `/3.0/`, `/1.0/`, `/0.25/` all return 404). Older trees still online: 0.21, 0.22, 0.23, 0.24. Everything below is from 2.0 pages unless tagged otherwise. Vega SDK 2.0 / Vega OS 2.0 is GA ("no longer in open beta") [release notes 2.0](https://developer.amazon.com/docs/vega/2.0/vega-release-notes).

Note for scrapers: the docs site is compiled MDX. Version-conditional blocks and callouts (system requirements, Node version, notes) are missing from a plain HTML-to-text conversion and from TinyFish output. They are in the raw HTML as `_jsx(...)` strings guarded by `[`0.24`,`2.0`].includes(docVersion_0)`. Read the raw HTML to get them.

## 1. System requirements and install

- Disk space: "20 GB available for VDT installation and development environment"; check with `df -h ~`. [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- OS (2.0/0.24 variant): "macOS: 10.15+ — Apple Silicon arm64 is the recommended platform." "Ubuntu: 20.04, 22.04, or 24.04 — x86_64 is fully supported." "Windows: Not supported. Windows and Windows Subsystem for Linux WSL aren't tested". [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- Intel Mac end of life: "Starting with SDK v0.24, support for Intel Mac (x86_64) reaches end of life on December 15, 2026." [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk), [target-os-version](https://developer.amazon.com/docs/vega/2.0/target-os-version) (2.0)
- Supported build hosts per SDK (matrix): 0.21 to 2.0 all list "Apple Silicon Mac, Intel Mac, Ubuntu x86_64". "Windows is not a supported build host." No Linux arm64 host is listed. [target-os-version](https://developer.amazon.com/docs/vega/2.0/target-os-version#version-compatibility-matrix) (2.0)
- CI host list (wider Ubuntu list than the install page): "Ubuntu 20.04, 22.04, 24.04, or 26.04 — x86_64"; "macOS X — arm64 (M-series) — recommended; x86_64 (Intel)". [continuous-integration](https://developer.amazon.com/docs/vega/2.0/continuous-integration) (2.0)
- Node.js: "version 22.14.0 or later" for docs 0.24 and 2.0; older doc versions say "18.x or later" (0.21 said 16.x). [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0, conditional block)
- "The installer verifies your Node.js and VS Code versions are compatible and prompts you to upgrade if needed." [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- Ubuntu dependency list (2.0): jq ("The script stops with `jq is required` if jq isn't installed"); build-essential ("Required to build Turbo Modules and other native code. The Vega SDK doesn't bundle these host tools."); "python3.8, lz4 for crash report symbolication"; KVM virtualization (KVM Hypervisor guide); Watchman; Node.js. Do not use Snap curl: "The Snap version has sandbox limitations that prevent the installation script from working." [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)

macOS dependencies (2.0):
```
[[ $(arch) == "arm64" ]] && softwareupdate --install-rosetta --agree-to-license; brew update && brew install binutils coreutils gawk findutils grep jq lz4 gnu-sed watchman
```
Ubuntu dependencies (2.0):
```
sudo apt remove curl
sudo apt install curl
(command -v jq >/dev/null || sudo apt install -y jq) && \
(dpkg -l | grep -q build-essential || sudo apt install -y build-essential) && \
(dpkg -l | grep -q lz4 || sudo apt install -y lz4) && \
sudo add-apt-repository -y ppa:deadsnakes/ppa && \
sudo apt update && \
(dpkg -l | grep -q libpython3.8-dev || sudo apt install -y libpython3.8-dev)
```
Installer (close VS Code first; "takes 10–15 minutes"):
```
curl -fsSL https://sdk-installer.vega.labcollab.net/get_vvm.sh | bash && source ~/vega/env
```
- What it installs: "Vega CLI → ~/vega/bin", "Latest SDK version including Vega Virtual Device", "Vega Studio extension in VS Code", "Shell configuration updates (.bashrc, .zshrc, .bash_profile, .zprofile, and .profile)". If VS Code is missing the extension step is skipped. [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- The interactive installer also offers to set up Amazon Devices Builder Tools (ADBT); it skips ADBT "when it runs noninteractively, such as in a CI pipeline, or without a terminal". [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- VDA binary path: `$HOME/vega/sdk/vega-sdk/main/{SDK version}/bin/tools/vda`. [vda-tools](https://developer.amazon.com/docs/vega/2.0/vda-tools) (2.0)
- Config file default `~/vega/config.json`, override with `VEGA_CONFIG_FILE`. [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager) (2.0)
- Verify: `vega --version` prints `Active SDK Version: ...` and `Vega CLI Version: ...`. Installing a new SDK "doesn't always make it the active version"; check with `vega sdk ls`. [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- Clean build after an SDK switch: `rm -rf ./build && rm -rf ./node_modules`, reinstall, `npm start -- --reset-cache`, `npm run build:app`. "Prior build artifacts may be incompatible with the updated SDK." [install-vega-sdk](https://developer.amazon.com/docs/vega/2.0/install-vega-sdk) (2.0)
- Example SDK version strings seen in docs: `2.0.13377`, `2.0.10945`, `0.24.12112`, `0.22.5850`; CLI `1.2.0` in examples. [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager), [continuous-integration](https://developer.amazon.com/docs/vega/2.0/continuous-integration) (2.0)

## 2. SDK version management (`vega sdk`)

Works "like `nvm` for Node.js or `pyenv` for Python". [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager) (2.0)
```
vega sdk install                      # latest
vega sdk install 1.2.3                # specific
vega sdk list-remote                  # alias: vega sdk lr (newest first)
vega sdk list-installed               # alias: vega sdk ls
vega sdk use <SDK_VERSION>            # set global default; alias: vega sdk sv <SDK_VERSION>
vega sdk uninstall <version>
vega sdk config setup [--non-interactive] [--component-path /opt/vega-sdk]
vega sdk config doctor                # health check
vega sdk link --discover              # auto-discover from KEPLER_SDK_PATH
vega sdk link --path /path/to/existing/sdk
vega update                           # update the Vega CLI itself
```
Global options: `--json`, `--verbose`, `--version`. [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager) (2.0)

Per-project pin, `vega-sdk-requirements.json` in project root (searched in cwd then parents):
```
{
   "vegaSdkVersion": "1.2.3"
}
```
Resolution order: (1) `vegaSdkVersion` from the requirements file, (2) global default from `vega sdk use`, (3) most recently installed SDK. [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager) (2.0)

Non-interactive: `--non-interactive` flag beats `NONINTERACTIVE=true` env, which beats the interactive default. [sdk-manager](https://developer.amazon.com/docs/vega/2.0/sdk-manager) (2.0)

## 3. `vega` CLI reference (2.0)

Source: [cli-tools](https://developer.amazon.com/docs/vega/2.0/cli-tools) (2.0, modified 2026-11-13) unless noted. Exit code `0` on success, non-zero on failure; errors go to stderr. "Don't use `amazon` as your company name or package ID."

Help/version:
```
vega --version        # short: vega -v
vega --help
vega <command> --help
vega build --help
```
Setup:
```
vega setup-studio [--install] [--verbose]   # "isn't currently listed in vega --help"; `kepler setup-studio` is equivalent; without --install it installs nothing
```
Build/clean/templates:
```
npm install
npx react-native build-vega        # compiles app, produces .vpkg
vega clean                          # removes <projectroot>/build artifacts
vega project list-templates
vega project generate --template helloWorld --name "<APPNAME>" --packageId "com.<COMPANY NAME>.<APPNAME>" --outputDir "<APPNAME>"
```
- `--name` can't contain whitespace/special chars; `--packageId` required for React Native apps, not Turbo Modules; `--outputDir` optional.
- `npx react-native build-vega` flags documented elsewhere: `--build-type Release|Debug`, `--build-version 1.2.3` (freeform string), `--build-number 101` (increasing, 1 to 2^63-1). [app-version](https://developer.amazon.com/docs/vega/2.0/app-version) (2.0)
- `vega build` exists (docs say `vega build --help`, and "When you build your app with `vega build`, the build process generates the `.vpkg`"), but its flags are not documented on the reference page. [cli-tools](https://developer.amazon.com/docs/vega/2.0/cli-tools), [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- "Vega command-line interface (CLI) doesn't build JavaScript code automatically, and only builds the native portion and the app package (.vpkg files)." Default package manager npm; Yarn and pnpm supported. [setup-overview](https://developer.amazon.com/docs/vega/2.0/setup-overview) (2.0)

Run:
```
vega run-app <Vpkg path> [App ID] -d <device serial number>   # -d = --deviceId; installs + launches; omit App ID to only install
vega run-app build/aarch64-release/sampleapp_aarch64.vpkg com.amazondeveloper.sampleapp.main -d VirtualDevice
```
Project compatibility commands (write `package.json`/`manifest.toml` only; "never run `npm`, `yarn`, or `pnpm` for you"):
```
vega project install [packages..] [--os-version] [--os-min] [--rn-version] [--fix] [--dry-run]
vega project install --fix                               # re-align all @amazon-devices/* to the compatibility profile
vega project install --fix --os-min 2.0 --os-version 2.0 # also creates [os.version] if missing
vega project update [--os-version] [--os-min] [--rn-version] [--dry-run]
vega project doctor [--os-version] [--os-min] [--rn-version]   # exits non-zero on fail -> CI gate
vega project update-manifest [--os-version] [--os-min] [--dry-run]
vega project update-manifest --os-min 2.0 --os-version 2.0
```
- `doctor` checks: compatibility manifest present; config resolves; min/target are valid minted OS versions; min <= target; `@amazon-devices/*` versions match profile; lint rules `no-unguarded-os-api`, `no-removed-api`, `warn-deprecated-api`. `--rn-version` default "Detected from your `@amazon-devices/react-native-kepler` dependency".
- `--os-min` on `install` "takes effect only together with `--fix`", needs `--os-version`; on a project without `[os.version]` and no `--os-version` it fails with `Missing OS Version`.
- `update-manifest` also removes tool-owned module entries (`/com.amazon.vega.os@IVega_*`, versioned `@amzn/<pkg>@<version>` and `@amazon-devices/<pkg>@<version>`) from the source manifest; "Commit it, and don't restore the entries by hand."

Device commands (need VDA on PATH; `--device`, short `-d`):
```
vega device list
vega device info --device <serial>              # JSON: architecture, profile, product, simulated, inDeveloperMode...
vega device is-connected --device <serial>
vega device install-app -p build/x86_64-release/sampleapp_x86_64.vpkg --device <serial>
vega device install-app --dir . [-b Debug|-b Release]
vega device install-app --packagePath <vpkg-path>          # form used on run-apps page
vega device -d <serial> install-app --packagePath <vpkg-path>
vega device launch-app --device <serial> --appName <App ID>   # or --dir <project directory>
vega device terminate-app --device <serial> --appName <App ID>
vega device uninstall-app --device <serial> --appName <App ID>
vega device installed-apps --device <serial>
vega device installed-packages --device <serial>
vega device is-app-installed --device <serial> --appName <App ID>
vega device is-app-running --device <serial> --appName <App ID>
vega device running-apps --device <serial>
vega device shell --device <serial>
vega device run-cmd --device <serial> --command '<command to run>'
vega device reboot --device <serial>
vega device copy-to   --device <serial> --source <f> --destination <f>
vega device copy-from --device <serial> --source <f> --destination <f>
vega device copy-logs --device <serial> --artifact <artifact/provider> --dir <dir>   # e.g. system/var_log; dir must exist
vega device get-log-info --device <serial>
vega device start-log-stream --device <serial>
vega device stop-log-stream --device <serial>
vega device start-port-forwarding --device <serial> --port <port> --forward <true|false>   # -p, -f short forms
vega device stop-port-forwarding  --device <serial> --port <port> --forward <true|false>
```
Virtual device:
```
vega virtual-device status
vega virtual-device start [--timeout 120] [--no-gui] [--no-gl-accel] [--display-res=1920,1080]
vega virtual-device stop
```
`--no-gl-accel` is "Ubuntu x86 only". [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)

Developer mode on a Fire TV (Amazon developer account OAuth):
```
vega devmode login
vega devmode enable-device --code <6-digit code>
```
[run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps), [developer-mode](https://developer.amazon.com/docs/vega/2.0/developer-mode) (2.0). Turn it off from the same Developer Options setting (the device reboots), or `vega exec vda shell vsm developer-mode disable` ([developer-mode](https://developer.amazon.com/docs/vega/2.0/developer-mode), [fire-tv-stick](https://developer.amazon.com/docs/vega/2.0/fire-tv-stick), 2.0, checked live 2026-11-20). `vega exec vda connect <host>[:<port>]` connects over the network (default port 5555); the vda-tools page also describes `vega exec vda tcpip <port>` over USB first, and it is unconfirmed whether a Fire TV Stick in Developer Mode needs that step ([vda-tools](https://developer.amazon.com/docs/vega/2.0/vda-tools), 2.0). On the device: Settings > My Fire TV > About > press center button seven times > Developer options > Developer Mode > Continue shows the 6-digit code.

`vega exec` wrappers for SDK tools:
```
vega exec vda devices [-l]
vega exec vda -s <serial-or-qualifier> shell     # -s overrides VEGA_SERIAL; VVD is emulator-5554
vega exec vda shell vsm developer-mode enable    # enable dev mode on VVD
vega exec vda shell cat /etc/os-release | grep VERSION_NUMBER
vega exec vda shell ace mw ota forceUpdate
vega exec vda shell vfr reset --type shallow     # factory reset
vega exec vpt validate /<path-to-manifest>/manifest.toml
vega exec perf kpi-visualizer --app-name <id>.main --iterations 10 --kpi ui-fluidity --test-scenario <file.py>
```
Sources: [vda-tools](https://developer.amazon.com/docs/vega/2.0/vda-tools), [fire-tv-stick](https://developer.amazon.com/docs/vega/2.0/fire-tv-stick), [app-manifest](https://developer.amazon.com/docs/vega/2.0/app-manifest) (2.0); the `perf kpi-visualizer` line is from the sample's package.json [vega-scrolling-sample](https://github.com/AmazonAppDev/vega-scrolling-sample/blob/main/package.json).

Other CLI tools listed (pages not read in detail): Vega Studio Performance CLI, vpm/vlcm (app management), loggingctl, inputd-cli, VPT, strict ABI check, VDA. [cli-overview](https://developer.amazon.com/docs/vega/2.0/cli-overview) (2.0)

## 4. Vega Virtual Device (VVD)

- Start/stop: `vega virtual-device start` (prints "Launching virtual device. / Waiting for virtual device to boot. / Virtual device ready.") and `vega virtual-device stop`; or Vega Studio > Devices > `VirtualDevice:Tv` > play. Device ID in commands is `VirtualDevice`. [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- vpkg arch must match host: "Mac M-series" uses `aarch64`, "x86_64 devices" use `x86_64`. [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- Linux requires KVM: error on `/dev/kvm` permissions means KVM is disabled; if Ubuntu itself is a VM, enable nested hypervisor support. [kvd-issues](https://developer.amazon.com/docs/vega/2.0/kvd-issues) (2.0)
- WebView apps: "VVD doesn't support Webview on Ubuntu or Intel-based Mac platforms." Use Apple Silicon Mac or a Fire TV 4K Select. Error: `Module dependency not found: /com.amazon.vega.webview_3@IWebview_3`. [kvd-issues](https://developer.amazon.com/docs/vega/2.0/kvd-issues) (2.0)
- Low memory: crashes when host free memory "is less than 2GB". Crash with some USB hubs/docks and audio interfaces used for both input and output. [kvd-issues](https://developer.amazon.com/docs/vega/2.0/kvd-issues) (2.0)
- Keyboard: Select=ENTER, D-pad=arrows, Back=ESC, Home=F1, Menu=F2, Rewind=F3, Play/Pause=F4, FF=F5. Context menu: Dump Render Tree (XML, same as Appium App Source), Dump Rendered Text. Register via Settings > Account Settings > Amazon Account (code at amazon.com/code). Disable CSAT prompt: `export KEPLER_CSAT_DISABLED=true`. [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- Limited `inputd-cli` until dev mode: `vega device shell -d VirtualDevice` then `vsm developer-mode enable` (VVD reboots). [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- Limits vs real device (SDK 2.0) [vvd-limitations](https://developer.amazon.com/docs/vega/2.0/vvd-limitations) (2.0):
  - Not on VVD: Simple Sign-In (`/com.amazon.dms.ssi@ISimpleSignIn`), DIAL (`/com.amazon.kepler.net.dial`), Login with Amazon, the Fire TV launcher/Home, other Fire TV middleware; Alexa voice services, native ad measurement, privacy consent, device telemetry.
  - Content Launcher/Account Login handlers register but the system features that call them (Home, search, Alexa) are absent.
  - If missing component is in `[needs]`, install fails (`Module dependency not found: ...`); in `[wants]` it installs and `vega run-app` can report success for an app that then crashes. Use `[wants]` for optional features.
  - DRM: clear playback works; Widevine L3 only, and only on Mac M-series hosts; "On other host machines, DRM-protected playback isn't available." No Widevine L1, no PlayReady; `SECURE_DECRYPT`/`HW_SECURE_ALL` fail with `InvalidStateError: KeySystem com.widevine.alpha is not supported`.
  - Task/service components may run without GL preloaded: link `GLESv2`/EGL explicitly (`target_link_libraries(<lib> PRIVATE GLESv2)`).
  - KPI Visualizer warm-start can report no value on VVD.
- Older VVD images lack newer native modules (e.g. `kepler-ui-components` on SDK 0.21 and earlier). [kvd-issues](https://developer.amazon.com/docs/vega/2.0/kvd-issues) (2.0)
- Docs advise testing on simulators at both `min` and `target` OS versions before submission. [target-os-version](https://developer.amazon.com/docs/vega/2.0/target-os-version) (2.0)

## 5. Vega Studio (VS Code / Kiro extension)

- Extension for VS Code and Kiro. Marketplace ID `amazon-devices.kepler-studio`; Open VSX `amazon-devices/kepler-studio`; or `vega setup-studio --install`. [setup-extension](https://developer.amazon.com/docs/vega/2.0/setup-extension) (2.0)
- Panel sections: Projects, Build Modes (Debug/Release), Devices, SDK Manager, App Performance Tools, Help and Feedback. [setup-extension](https://developer.amazon.com/docs/vega/2.0/setup-extension) (2.0)
- Debug mode enables Metro and Fast Refresh automatically; Release mode disables Fast Refresh and debug info. CLI users must set Fast Refresh up by hand. [setup-extension](https://developer.amazon.com/docs/vega/2.0/setup-extension), [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app) (2.0)
- Performance tools: Activity Monitor, Memory Monitor, App KPI Visualizer, Chrome Dev Tools; "Some tools are available only when your app runs on a physical device." [setup-extension](https://developer.amazon.com/docs/vega/2.0/setup-extension) (2.0)
- Debugger: GDB-based native debugging of apps and Turbo Modules; Chrome DevTools; for RN 0.83 use React Native DevTools. [debugger](https://developer.amazon.com/docs/vega/2.0/debugger) (2.0)
- A11y Inspector (new in 2.0) lints .tsx/.jsx as you edit, same rules as the Vega ESLint plugin. [release notes](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Settings: Automatic Device Detection; Build Target Architecture Optimization (e.g. Fire TV Stick target builds only armv7); Vega Launch Options; React Native Port (default 8081). Monorepo via npm/Yarn workspaces. Bug Report tool collects system/device/logs/ACR (10 MB attach limit). [setup-extension](https://developer.amazon.com/docs/vega/2.0/setup-extension), [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app) (2.0)
- Command palette: "Vega SDK: Validate Vega SDK", "Vega Project: Create a new Vega project", "Vega Project: Build and run Vega project", "Vega Project: Build Vega project", "Vega App: Installs/Uninstalls/Run the current Vega App", "Vega App: Reload Application", "Vega App: Restart App", "Vega Device Management: Copy ACR from device / Copy logs from device / Symbolicate JS ACR", "Vega Virtual Device: Start simulator / Stop simulator", "Vega: Start Activity Monitor", "Vega: Open Recording View", SDK management commands. [vega-studio-commands](https://developer.amazon.com/docs/vega/2.0/vega-studio-commands) (2.0)

Fast Refresh with CLI on VVD:
```
vega virtual-device start
npm install
npm run build:debug
npm start
vega device start-port-forwarding --device VirtualDevice -p 8081 --forward false
vega run-app build/x86_64-debug/sampleapp_x86_64.vpkg com.amazondeveloper.sampleapp.main -d VirtualDevice
# custom Metro port
npm start -- --port=<new_port>
vda reverse tcp:8081 tcp:<new_port>
```
[fast-refresh](https://developer.amazon.com/docs/vega/2.0/fast-refresh) (2.0)

## 6. Templates, generated layout, build outputs

- `vega project list-templates` output (2.0): `helloWorld` (RN v0.83), `helloWorld-rn72` (RN v0.72), `basic-turbo-module`, `idl-turbo-module` ("uses native OS APIs defined in IDL"), `vegaWebview`. [cli-tools](https://developer.amazon.com/docs/vega/2.0/cli-tools), [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app) (2.0)
- Package name format `<top-level domain>.<company_name>.<app_name>`. [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app) (2.0)
- Build: `npm install` then `npm run build:app` ("install necessary NPM dependencies, build the native code, and then bundle the JavaScript using the Metro bundler"). [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app) (2.0)
- vpkg path: `build/<ARCH>-<BUILD_TYPE>/<APP_NAME>_<ARCH>.vpkg`; build types `debug`/`release` (dirs like `x86_64-debug`, `armv7-release`). [cli-tools](https://developer.amazon.com/docs/vega/2.0/cli-tools), [fast-refresh](https://developer.amazon.com/docs/vega/2.0/fast-refresh) (2.0)
- Architectures: `x86_64-release/..._x86_64.vpkg` "for the virtual device for x86 machines"; `armv7-release/..._armv7.vpkg` "to run your app on the Fire TV Stick"; `aarch64-release/..._aarch64.vpkg` "for the virtual device for M-series machines". "For Fire TV devices, use the `armv7` architecture." Upload for submission: `build/armv7-release/<app>_armv7.vpkg`. [build-an-app](https://developer.amazon.com/docs/vega/2.0/build-an-app), [run-apps](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- Typical file layout (from an official 2.0 sample, not a template listing): `manifest.toml`, `package.json`, `index.js`, `app.json`, `babel.config.js`, `metro.config.js`, `tsconfig.json`, `jest.config.json`, `eslint.config.mjs`, `src/`, `assets/` (icon at `assets/image/`, localized strings at `assets/text/puff.json`), `test/`. [vega-scrolling-sample](https://github.com/AmazonAppDev/vega-scrolling-sample), [manifest-package](https://developer.amazon.com/docs/vega/2.0/manifest-package)
- Sample package.json scripts: `"build": "react-native build-vega --build-type Release"`, `"build:debug": "react-native build-vega --build-type Debug"`, `"build:app": "npm-run-all build:release build:debug"`, `"start": "react-native start"`. Dependencies: `@amazon-devices/react-native-kepler ~4.0.0`, `react 19.2.0`, `react-native 0.83.0`; devDependency `@amazon-devices/kepler-cli-platform ~0.22.0`, `@react-native-community/cli ^20.0.0`. [vega-scrolling-sample package.json](https://github.com/AmazonAppDev/vega-scrolling-sample/blob/main/package.json)
- Hello World build success log line: `info build-kepler completed` (and `kepler ktbuild exited with code 0`). [hello-world](https://developer.amazon.com/docs/vega/2.0/hello-world) (2.0)

## 7. `manifest.toml` (schema-version 1)

Root of package; validated by VPT: `vega exec vpt validate /<path-to-manifest>/manifest.toml`. [app-manifest](https://developer.amazon.com/docs/vega/2.0/app-manifest) (2.0)

| Section | Req | Keys |
|---|---|---|
| `schema-version = 1` | required, must be 1 | |
| `[package]` | required | `id` (reverse DNS, >=2 segments, chars `A-Za-z0-9._`, must never change), `title` (string or `@text/<id>` resolved from `assets/text/puff.json`), `icon` (`@image/<file>` under `assets/image`, 512x512 PNG, max 1 MB) |
| `[os.version]` | required since SDK 0.24 (build fails without it) | `min`, `target` - quoted `"M.n"` minted OS versions, `min <= target` |
| `[components]` | required | `[[components.interactive]]`, `[[components.service]]`, `[[components.task]]` |
| `[tasks]` | optional | `[[tasks.work]]` `id` (task component id; overview page shows `component-id`), `mode = "install"` (only value) |
| `[needs]` | optional | `[[needs.module]] id`, `[[needs.privilege]] id` |
| `[wants]` | optional | `[[wants.module]] id`, `[[wants.privilege]] id`, `[[wants.service]] id` |
| `[[message]]` | optional | `uri`, `sender-privileges`, `receiver-privileges` (`"*"`, `"self"`, or privilege ids) |
| `[offers]` | optional | `[[offers.service]]` (`id`, `required-privileges`), `[[offers.module]]` (`id` `/<name>@I<Tag>`, `includes-messages`), `[[offers.message-target]]` (`uris`, `uses-component`), `[[offers.interaction]]` legacy (`id`, `launch-uris`, `required-privileges`) |
| `[processes]` | optional | `[[processes.group]] component-ids = [...]` |
| `[[extras]]` | optional | `key`, `value` (string or table), `component-id` |

Sources: [app-manifest](https://developer.amazon.com/docs/vega/2.0/app-manifest), [manifest-package](https://developer.amazon.com/docs/vega/2.0/manifest-package), [manifest-components](https://developer.amazon.com/docs/vega/2.0/manifest-components), [manifest-needs](https://developer.amazon.com/docs/vega/2.0/manifest-needs), [manifest-wants](https://developer.amazon.com/docs/vega/2.0/manifest-wants), [manifest-tasks](https://developer.amazon.com/docs/vega/2.0/manifest-tasks), [manifest-message](https://developer.amazon.com/docs/vega/2.0/manifest-message), [manifest-offers](https://developer.amazon.com/docs/vega/2.0/manifest-offers), [manifest-processes](https://developer.amazon.com/docs/vega/2.0/manifest-processes), [manifest-extras](https://developer.amazon.com/docs/vega/2.0/manifest-extras), [manifest-os-version](https://developer.amazon.com/docs/vega/2.0/manifest-os-version) (all 2.0)

Component fields [manifest-components](https://developer.amazon.com/docs/vega/2.0/manifest-components) (2.0):
- interactive: `id` (`<package_id>.<component_name>`), `categories`, `launch-type` (`new-instance` default, `new-container`, `singleton`, `singleton-per-display`), `timeout-secs`, `lifespan` (`permanent` default, `long`, `short`, `transient`), `runtime-module`.
- categories: `com.amazon.category.main` (exactly one interactive must have it; "required to launch your app or submit to the Appstore"), `com.amazon.category.caf` (Alexa Capability Agent), `com.amazon.category.kepler.media`, `com.amazon.category.launcher` (system only).
- service: `id`, `categories` (`com.amazon.category.kepler.media`), `launch-type` (`singleton` default, `singleton-per-application`), `runtime-module`.
- task: `id`, `categories`, `runtime-module`.

Runtime modules [vega-runtime-module](https://developer.amazon.com/docs/vega/2.0/vega-runtime-module) (2.0):
- RN 0.83 interactive: `/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0`; headless: `/com.amazon.kepler.runtime.react_native_kepler_headless_4@IReactNativeKeplerHeadless_0`.
- RN 0.72 interactive: `/com.amazon.kepler.keplerscript.runtime.loader_2@IKeplerScript_2_0`; headless: `/com.amazon.kepler.headless.runtime.loader_2@IKeplerScript_2_0`.
- Same process group must share a runtime module; don't mix runtime versions.

Minimal RN 0.83 manifest (official sample, 2.0):
```
schema-version = 1
[package]
title = "..."
version = "0.1.0"
id = "com.amazondeveloper.keplerscrollingapp"
icon = "@image/default_app_icon.png"
[os.version]
min = "2.0"
target = "2.0"
[components]
[[components.interactive]]
id = "com.amazondeveloper.keplerscrollingapp.main"
runtime-module = "/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0"
launch-type = "singleton"
categories = ["com.amazon.category.main"]
[needs]
[[needs.module]]
id = "/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0"
```
[vega-scrolling-sample manifest.toml](https://github.com/AmazonAppDev/vega-scrolling-sample/blob/main/manifest.toml). Note `[package] version` appears in the sample but not on the `[package]` doc page.

OS version modules: build generates `[[needs.module]] id = "/com.amazon.vega.os@IVega_<min>"` and `[[wants.module]]` for each minted version above min up to target into the validation/staged/packaged manifests, never into source. For SDK 2.0 set both to `2.0`; `min = "1.2"` to also support OS 1.2. [manifest-os-version](https://developer.amazon.com/docs/vega/2.0/manifest-os-version) (2.0)

Example ids from docs: privileges `com.amazon.media.secureplayback`, `com.amazon.network.privilege.net-info`, `com.amazon.camera.privilege.access`, `com.amazon.kepler.tv.privilege.data_provider`; services `com.amazon.inputmethod.service`, `com.amazon.mediabuffer.service`, `com.amazon.tv.developer.dataservice.service`, `com.amazon.alexa.datastore.service`; module `/com.amazon.kepler.media@IContentLauncher1`. [app-manifest](https://developer.amazon.com/docs/vega/2.0/app-manifest), [vvd-limitations](https://developer.amazon.com/docs/vega/2.0/vvd-limitations) (2.0)

## 8. React Native for Vega, packages, compatibility

- Matrix: SDK 0.21/0.22/0.23 -> RN 0.72 (OS 1.1); 0.24 -> RN 0.72 and 0.83 (OS 1.2); 2.0 -> RN 0.72 and 0.83 (OS 2.0). RN 0.83 needs `min >= 1.2`. Runtime is inferred from `@amazon-devices/react-native-*` versions in package.json, not set explicitly. [target-os-version](https://developer.amazon.com/docs/vega/2.0/target-os-version) (2.0)
- RN 0.83 GA in 2.0 "brings React 19, the React Compiler, and Hermes v1". Breaking in 2.0: deep imports no longer resolve for core and ported libs; `@amazon-devices/react-linear-gradient` replaced by `experimental_backgroundImage`; use `@shopify/flash-list` on 0.83; experimental APIs unsupported. Known issue: an RN 0.83 app that pulls in a 0.72-built library crashes at launch. [release notes](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- JS engine: Hermes by default per RN docs mirror. [javascript-environment](https://developer.amazon.com/docs/react-native-vega/0.83/javascript-environment) (RN-for-Vega 0.83)
- Namespace: OS-provided packages are `@amazon-devices/*` (e.g. `@amazon-devices/react-native-kepler`, `@amazon-devices/kepler-compatibility`, `@amazon-devices/kepler-file-system`, `@amazon-devices/kepler-ui-components`). `vega project install` resolves them against `[os.version].target`; ordinary npm packages resolve normally. [package-management](https://developer.amazon.com/docs/vega-api/2.0/package-management) (2.0)
- Standard chain (also the CI chain):
```
vega project install --fix
npm install
vega project doctor
npx react-native build-vega
```
- Guarding APIs newer than `min`: `isPresentOnOS('<npm lib>', '<lib semver>')` from `@amazon-devices/kepler-compatibility`; suppress with `// @os-version-ok` on the line before the import. Native: `TryQueryInterface<T>()` + null check. [target-os-version](https://developer.amazon.com/docs/vega/2.0/target-os-version) (2.0)

Supported libraries (package `@amazon-devices/<name>` ranges; RN 0.83 / RN 0.72) [supported-libraries](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0):
- Community: react-native-fast-image ~4.0.0 / ~3.0.0; react-native-mmkv >=1.0.11-rn-83 / ~1.0.0; react-native-netinfo ~2.0.20 / ~2.0.0; react-native-pager-view ~2.0.3 both; react-native-qrcode-svg ~3.0.0 / ~2.0.0; react-native-screens ~3.0.0 / ~2.0.0; react-navigation ^8.0.0 / ^7.0.0; shopify-react-native-performance-navigation (+bottom-tabs, drawer) not supported on 0.83 / ~3.0.0.
- Expo (0.83 ~3.0.0 / 0.72 ~2.0.0): expo-application, asset, checkbox, constants, crypto, device, file-system, font, gl, image-manipulator, image, linear-gradient, linking, localization, network, splash-screen, sqlite, system-ui; expo-notifications not supported on 0.83.
- System-distributed: lottie-react-native, react-native-gesture-handler (~4.0.0 / ~2.0.0), react-native-async-storage__async-storage, react-native-cookies__cookies, react-native-device-info, react-native-localize, react-native-masked-view__masked-view, react-native-reanimated (~4.0.0 / ~2.0.0), react-native-safe-area-context, react-native-svg (~3.0.0 / ~2.0.0), react-native-vector-icons, react-native-worklets (0.83 only), react-navigation, shopify__flash-list (0.72 only), shopify__react-native-performance(-lists-profiler) (0.72 only).
- Pure-JS libs tested: formik, Jest 29.7.0, Lodash, Reassure, Redux, react-native-axios, react-native-scalable-image, react-native-toast-message, react-native-url-polyfill, react-native-uuid, Socket.io, TVFocusGuideView.
- Players: Shaka Player (Amazon fork), dash.js, hls.js (Amazon patches), Bitmovin, Dolby OptiView, Lura. Third-party services list (Adjust, Braze, Bugsnag, Conviva, Firebase, Sentry, Segment, etc.).
- Community directory filter: https://reactnative.directory/packages?vegaos=true

## 9. CI pipelines

[continuous-integration](https://developer.amazon.com/docs/vega/2.0/continuous-integration) (2.0):
```
export NONINTERACTIVE=true
export SKIP_VVD_INSTALL=true
export VEGA_SDK_VERSION=2.0.10945
curl -fsSL https://sdk-installer.vega.labcollab.net/get_vvm.sh | bash
vega -v
```
| Variable | Default | Meaning |
|---|---|---|
| `NONINTERACTIVE` | FALSE | skip prompts; required for CI |
| `VEGA_SDK_VERSION` | latest | pin SDK version |
| `SKIP_SDK_INSTALL` | FALSE | skip SDK install entirely |
| `SKIP_VVD_INSTALL` | FALSE | skip VVD |

Official Dockerfile base: `FROM --platform=linux/amd64 ubuntu:22.04`; apt `curl tar jq ca-certificates build-essential`; installer; `ENV PATH="/root/vega/bin:${PATH}"`; `nodejs npm`; `NPM_CONFIG_GLOBALCONFIG="/etc/npmrc"`. Build: `docker build . --tag vega-sdk --build-arg VEGA_SDK_VERSION=2.0.10945 --build-arg SKIP_VVD_INSTALL=true`. Note the Dockerfile installs distro `nodejs`, which on 22.04 is older than the 22.14.0 the install page asks for.

Caching: only generic advice ("Use caching strategies for downloaded artifacts", multi-stage builds); no cache paths given. Gate: `vega project doctor` after `install --fix` + package manager, before build. Builds fail without `[os.version]`.

## 10. Amazon Devices Builder Tools (ADBT) MCP server

[adbt get-started](https://developer.amazon.com/docs/adbt/get-started) (unversioned, modified 2026-10-23), [npm README](https://www.npmjs.com/package/@amazon-devices/amazon-devices-buildertools-mcp) (v1.0.21, license UNLICENSED):
- Contents: MCP server (standalone npm package), Agent Skills, steering documents. "Full support for Vega OS and limited support for Fire OS." Not bundled with the SDK.
- Prereqs: "Node.js 18 or later" (use Node 24 if Node 26 hits an SQLite3 bindings error); Vega SDK 0.22+ for Vega workflows.
```
npx -y @amazon-devices/amazon-devices-buildertools-mcp@latest init-context
npx -y @amazon-devices/amazon-devices-buildertools-mcp@latest init-context --agent claude-code-cli --force
npx -y @amazon-devices/amazon-devices-buildertools-mcp@latest check-status [--agent <agent>]
npx -y @amazon-devices/amazon-devices-buildertools-mcp@latest clean-context [--agent <agent> --force]
```
- init-context options: `-a, --agent <type>` (cursor, claude-code-cli, github-copilot, kiro, cline, opencode, codex, gemini-cli, other), `-p, --context-document-path`, `-d, --skip-context-document`, `-m, --skip-mcp-config`, `--install-agent` (installs `adbt-dev`; Claude Code file `~/.claude/agents/adbt-dev.md`), `-c, --clean`, `-f, --force`, `--update`. Writes user-level MCP config and skills dir.
- MCP tools: `analyze_perfetto_traces`, `get_app_hot_functions`, `list_documents`, `read_asset`, `read_document`, `search_documentation`, `symbolicate_acr`. MCP prompts include `diagnose_crash`, `diagnose-slowAppLaunch`, `diagnose_ui_fluidity`, `diagnose_key_input_latency`, `react_native_for_vega_ai_assisted_rn_upgrade`, etc.
- Skills installed: amazon-devices-vega-app-manifest, -app-migration, -app-performance, -best-practices, -build-and-run, -developer-mode-init, -focus-management, -media-player, -navigation, -rn-upgrade, -setup-sdk, -ui-components, caption/audio-description skills, vega-multi-tv-migration, plus Android ones.
- Optional `.adbt-config.json` at repo root: `{"platform": {"default": "vega_os"}}` (`vega_os|fire_os|both`, `paths` map for monorepos).
- Data: "doesn't collect, transmit, or store any of your confidential information"; but "documentation search queries issued by your AI agent are sent to Amazon Services"; "Amazon collects anonymous telemetry data ... feature usage patterns, error rates, and performance metrics. You can disable telemetry collection in the settings." The exact setting/env var is not documented on the page or in the npm README.

SDK telemetry (separate): on by default; collects sdk_version, os, cpu, hashed `machine_id`, encrypted `directed_id` (customer ID), `package_id`, stack traces, etc. Opt out: set `"optIn": false` in `${HOME}/vega/telemetry/config.json`, or VS Code setting "Kepler: Telemetry" -> Off. [telemetry](https://developer.amazon.com/docs/vega/2.0/telemetry) (2.0)

## 11. Vega device specs

From [Fire TV device specifications](https://developer.amazon.com/docs/device-specs/device-specifications-fire-tv-streaming-media-player) (page data, modified 2026-05-14) and [release notes OS builds](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0):

| Device | Model | OS | SoC / CPU | ABI | RAM | GPU | Storage | Max output | OS 2.0 build |
|---|---|---|---|---|---|---|---|---|---|
| Fire TV Stick 4K Select (2025) | AFTCA002 | Vega OS 2.0 | MT8698 MCM, 4x Cortex-A55 up to 1.7 GHz | 32-bit | 1GB LPDDR4 | G310v2 up to 500 MHz, OpenGL ES 3.1 | 8 GB | 4K (HDR10, HDR10+, HLG) | 2602000052920 |
| Fire TV Stick 4K - 3rd Gen (2026) | AFTCR001 | Vega OS 2.0 | MediaTek MT8698E, 4x CA55 up to 1.7 GHz | 32-bit | 1GB x32 LPDDR4-3733 | ARM G310 V2 up to 500 MHz, OpenGL ES 3.1 | 8 GB | 3840x2160 @60 | 2502000007420 |
| Fire TV Stick HD (2026) | AFTCL001 | Vega OS 2.0 | MediaTek MT8698D, 4x CA55 up to 1.7 GHz | 32-bit | 1GB x32 LPDDR4-3200 | ARM G310 V2 up to 500 MHz, OpenGL ES 3.1 | 8 GB | not in page data (HD model) | 2602000052920 |

- 32-bit ABI matches the docs' "For Fire TV devices, use the `armv7` architecture."
- Known issue: HEVC hardware decoder unavailable on Fire TV Stick HD (2026) running OS 2.0; fall back to H.264. [release notes](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Fire TV Stick needs up to 1 A at 5 V; laptop USB may not supply it. [fire-tv-stick](https://developer.amazon.com/docs/vega/2.0/fire-tv-stick) (2.0)

## Unverified / open questions

- Full `vega build` flags and whether `vega build` differs from `npx react-native build-vega` (only `vega build --help` is referenced).
- Full option list of `vega virtual-device start` beyond `--timeout`, `--no-gui`, `--no-gl-accel`, `--display-res`.
- Exact Fire TV Stick HD (2026) max resolution (field absent in page data). "Fire TV Streaming Media Player" is named in run-apps as a Vega example but no Vega spec entry was found for it.
- VVD on Linux arm64 (e.g. Raspberry Pi) is not listed as a host anywhere; treat as unsupported.
- Whether `vega project generate` output layout differs from the sample (no doc page lists the template's file tree).
- How to disable ADBT telemetry ("in the settings" is all the doc says).
- Whether CI Node requirement (22.14.0+) is enforced in non-interactive installs; the official Dockerfile uses Ubuntu 22.04 distro `nodejs`.
- `[tasks.work]` key: overview page uses `component-id`, the section page uses `id`.
- Cache paths for CI (`~/vega/sdk/...`) are inferred from the VDA path, not documented as a cache target.

## Sources

- https://developer.amazon.com/docs/vega/2.0/install-vega-sdk
- https://developer.amazon.com/docs/vega/2.0/sdk-manager
- https://developer.amazon.com/docs/vega/2.0/cli-tools
- https://developer.amazon.com/docs/vega/2.0/cli-overview
- https://developer.amazon.com/docs/vega/2.0/vda-tools
- https://developer.amazon.com/docs/vega/2.0/setup-overview
- https://developer.amazon.com/docs/vega/2.0/setup-extension
- https://developer.amazon.com/docs/vega/2.0/vega-studio-commands
- https://developer.amazon.com/docs/vega/2.0/build-an-app
- https://developer.amazon.com/docs/vega/2.0/hello-world
- https://developer.amazon.com/docs/vega/2.0/run-apps
- https://developer.amazon.com/docs/vega/2.0/fast-refresh
- https://developer.amazon.com/docs/vega/2.0/debugger
- https://developer.amazon.com/docs/vega/2.0/developer-mode
- https://developer.amazon.com/docs/vega/2.0/vvd-limitations
- https://developer.amazon.com/docs/vega/2.0/kvd-issues
- https://developer.amazon.com/docs/vega/2.0/fire-tv-stick
- https://developer.amazon.com/docs/vega/2.0/continuous-integration
- https://developer.amazon.com/docs/vega/2.0/target-os-version
- https://developer.amazon.com/docs/vega/2.0/app-manifest
- https://developer.amazon.com/docs/vega/2.0/manifest-package
- https://developer.amazon.com/docs/vega/2.0/manifest-components
- https://developer.amazon.com/docs/vega/2.0/manifest-needs
- https://developer.amazon.com/docs/vega/2.0/manifest-wants
- https://developer.amazon.com/docs/vega/2.0/manifest-tasks
- https://developer.amazon.com/docs/vega/2.0/manifest-message
- https://developer.amazon.com/docs/vega/2.0/manifest-offers
- https://developer.amazon.com/docs/vega/2.0/manifest-processes
- https://developer.amazon.com/docs/vega/2.0/manifest-extras
- https://developer.amazon.com/docs/vega/2.0/manifest-os-version
- https://developer.amazon.com/docs/vega/2.0/vega-runtime-module
- https://developer.amazon.com/docs/vega/2.0/app-version
- https://developer.amazon.com/docs/vega/2.0/telemetry
- https://developer.amazon.com/docs/vega/2.0/vega-release-notes
- https://developer.amazon.com/docs/vega-api/2.0/supported-libraries
- https://developer.amazon.com/docs/vega-api/2.0/package-management
- https://developer.amazon.com/docs/react-native-vega/0.83/react_overview
- https://developer.amazon.com/docs/react-native-vega/0.83/javascript-environment
- https://developer.amazon.com/docs/adbt/get-started
- https://www.npmjs.com/package/@amazon-devices/amazon-devices-buildertools-mcp
- https://developer.amazon.com/docs/device-specs/device-specifications-fire-tv-streaming-media-player
- https://github.com/AmazonAppDev/vega-scrolling-sample
