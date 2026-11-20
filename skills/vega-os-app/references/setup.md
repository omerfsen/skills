# Setup: dev machine and Fire TV Stick

Exact sources and more detail: `docs/toolchain.md` (sections 1, 4, 5, 10, 11) and `docs/security-appstore.md` (section 3). Docs version 2.0.

> Vega moves fast: if a command fails, check `vega --help` / `vega <command> --help` and developer.amazon.com/docs/vega before improvising.

## 1. Dev machine (once)

### Supported hosts (SDK 2.0)

| Host | Status |
|------|--------|
| macOS 10.15+ on Apple Silicon | Recommended. Only host where VVD can play DRM (Widevine L3). |
| macOS on Intel | Supported; end of life 2026-12-15. |
| Ubuntu 20.04 / 22.04 / 24.04 x86_64 | Supported; VVD needs KVM. No WebView in VVD (same on Intel Mac). |
| Windows, WSL | Not supported. |
| Linux arm64 (e.g. Raspberry Pi) | Not listed as a host anywhere: treat as unsupported. |

Needs about 20 GB free disk space and **Node.js 22.14.0 or later** (older doc versions said 18.x). Pin the exact Node version in `.nvmrc`.

### macOS steps

1. Xcode Command Line Tools: `xcode-select --install`
2. Homebrew from brew.sh.
3. Node via nvm, pinned (`.nvmrc`), check with `node --version`.
4. Rosetta 2 (Apple Silicon) and utilities:

   ```bash
   [[ $(arch) == "arm64" ]] && softwareupdate --install-rosetta --agree-to-license; brew update && brew install binutils coreutils gawk findutils grep jq lz4 gnu-sed watchman
   ```

5. Vega CLI + SDK (close VS Code first; takes 10–15 minutes). Download and read the script before running it:

   ```bash
   curl -fsSL https://sdk-installer.vega.labcollab.net/get_vvm.sh -o get_vvm.sh
   less get_vvm.sh
   bash get_vvm.sh && source ~/vega/env
   vega --version          # Active SDK Version + Vega CLI Version
   vega sdk ls             # a new SDK is not always the active one
   ```

   The installer adds the CLI to `~/vega/bin`, the latest SDK with the Vega Virtual Device, the Vega Studio extension (if VS Code is installed) and shell config lines. It also offers Amazon Devices Builder Tools (ADBT); see `ecosystem.md` before saying yes.

6. **Turn off SDK telemetry** (on by default; collects a hashed machine ID, an encrypted customer ID, package IDs and stack traces): set `"optIn": false` in `~/vega/telemetry/config.json`, or VS Code setting "Kepler: Telemetry" → Off.
7. VS Code + Vega Studio (`amazon-devices.kepler-studio`); reinstall with `vega setup-studio --install`.
8. GitHub CLI: `brew install gh && gh auth login`
9. Free Amazon Developer account at developer.amazon.com (registered to the publisher entity from the project values), then sign the CLI in:

   ```bash
   vega devmode login
   ```

### Ubuntu notes

Dependencies, the Snap-curl trap and KVM setup are in `docs/toolchain.md` section 1. Remove Snap curl first; the installer stops with `jq is required` if jq is missing.

### Vega Virtual Device (no stick needed)

```bash
vega virtual-device start      # optional: --timeout 120 --no-gui --display-res=1920,1080
vega virtual-device status
vega virtual-device stop
```

- Device ID in commands: `VirtualDevice`. Keyboard: Enter = Select, arrows = D-pad, Esc = Back, F1 = Home, F2 = Menu.
- Enable full input simulation: `vega device shell -d VirtualDevice`, then `vsm developer-mode enable` (VVD reboots).
- Not on VVD: Login with Amazon, Simple Sign-In, DIAL, the Fire TV launcher, Alexa, most DRM. Put optional features in `[wants]`, not `[needs]`, or install fails. Full list: `docs/toolchain.md` section 4.
- Needs at least 2 GB free host memory.

## 2. Fire TV Stick (once)

Vega devices (all OS 2.0, `armv7` builds, 1 GB RAM, 4× Cortex-A55): Fire TV Stick 4K Select (AFTCA002), Fire TV Stick 4K 3rd gen (AFTCR001), Fire TV Stick HD 2026 (AFTCL001; HEVC hardware decoding unavailable, use H.264).

1. Power it from the wall adapter (needs up to 1 A at 5 V; laptop USB may not be enough). Join the same Wi-Fi as the dev machine, sign in, install updates (Settings > My Fire TV > About > Check for Updates).
2. Note the IP address (Settings > My Fire TV > About > Network; `ifconfig` does not work in the dev shell).
3. Unlock Developer options: Settings > My Fire TV > About, press Select 7 times on the device name, press Back.
4. Developer options > Developer Mode > Continue, and note the 6-digit code. (Developer Mode opens the stick to VDA and sideloading; to turn it off later use the same setting, or `vega exec vda shell vsm developer-mode disable`; both reboot the stick.)
5. Enable it from the dev machine (the stick reboots):

   ```bash
   vega devmode enable-device --code <6-digit code>
   ```

6. Connect and verify:

   ```bash
   vega exec vda connect <stick-ip>     # VDA default port 5555
   vega exec vda devices                # the stick should be listed
   vega device shell                    # opens the Developer Mode shell
   ```

7. Optional, in the shell: `cat /etc/os-release`, `uname -a` (the shell is restricted; these may be blocked).

## Good to know

- Developer Mode only allows sideloading packages without the `com.amazon` prefix.
- Streaming apps and the Appstore keep working normally in Developer Mode.
- The stick accepts VDA connections over the network while Developer Mode is on: keep it on a trusted network.
