# Dev loop: build, install, run, debug, logs

Exact sources: `docs/toolchain.md` (sections 3, 4, 5), `docs/performance.md` (section 8), `docs/testing-ux-ecosystem.md` (sections 1.4, 1.5, 2). Docs version 2.0.

> Vega moves fast: if a command fails, check `vega --help` / `vega <command> --help` and developer.amazon.com/docs/vega before improvising.

Run from `apps/vega`.

## Build

```bash
npx react-native build-vega                       # or Vega Studio > Build; see `vega build --help`
npx react-native build-vega --build-type Release --build-version 1.2.3 --build-number 101
vega clean                                        # remove build artefacts
```

Output: `build/<arch>-<debug|release>/<AppName>_<arch>.vpkg`

| Target | Arch |
|--------|------|
| Fire TV Stick (all Vega models) | `armv7` |
| VVD on Apple Silicon | `aarch64` |
| VVD on Intel Mac / Ubuntu | `x86_64` |

The CLI builds the native part and the package; JavaScript is bundled by Metro during `build-vega`.

## Install and run

```bash
vega device list                                   # find the device serial; VVD is "VirtualDevice"

# install + launch in one go (app ID = interactive component ID, e.g. <package-id>.main)
vega run-app <path-to.vpkg> <app-id> -d <device>

# or step by step
vega device install-app -p <path-to.vpkg> --device <device>
vega device launch-app --device <device> --appName <app-id>
vega device is-app-running --device <device> --appName <app-id>
vega device terminate-app --device <device> --appName <app-id>
vega device uninstall-app --device <device> --appName <app-id>
vega device installed-apps --device <device>
```

## Fast Refresh (debug builds)

Vega Studio does this automatically in Debug mode. From the CLI:

```bash
npx react-native build-vega --build-type Debug   # or: npm run build:debug
npm start
vega device start-port-forwarding --device <device> -p 8081 --forward false
vega run-app build/<arch>-debug/<AppName>_<arch>.vpkg <app-id> -d <device>
```

## Debugging

- On RN 0.83, app logs no longer appear in the Metro terminal: use `loggingctl` (below) or DevTools.
- RN 0.83: React Native DevTools (port forwarding on 8081, Metro running, press `j` in the Metro terminal). Network tab works; no request blocking or throttling.
- RN 0.72: Chrome DevTools (Flipper also only on 0.72).
- Vega Studio debugger: `launch.json` type `kepler` (JS/TS) and `keplerNative` (C++ / Turbo Modules).
- Network inspection: Vega Studio Network Proxy. It writes `proxy-config.json` and `assets/raw/ca.pem` into the app: **delete both before any release build**.

## Logs, screenshots, crashes

```bash
vega exec vda shell                                # then, inside the device shell:
loggingctl log -v <package-id> -f                  # follow app logs (drops lines above ~300/s, silently)

vega exec vda shell gwsi-tool-screenshooter /tmp/test.png
vega exec vda pull /tmp/test.png

vega device copy-logs --device <device> --artifact system/var_log --dir <existing-dir>
```

Crash reports (ACR) and symbolication: `docs/performance.md` section 8 (Vega Studio "Copy ACR from device" + "Symbolicate JS ACR"). Low-memory kills are SIGKILL with no stack trace.

## Tips

- VVD for fast UI iteration; the real stick for performance, video, DRM and remote-control (D-pad focus) testing.
- If install or launch fails, check `vega exec vda devices`, reconnect (`vega exec vda connect <stick-ip>`) or reboot the stick; then see the Fire TV Stick and VDA sections of the troubleshooting index (`ecosystem.md`).
- After switching SDK versions: `rm -rf ./build ./node_modules`, reinstall, `npm start -- --reset-cache`, rebuild.
