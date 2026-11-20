# Release, publish, maintain, retire

Exact sources: `docs/security-appstore.md` (sections 2, 4, 5), `docs/toolchain.md` (section 2). Docs version 2.0, Appstore pages unversioned.

> Vega moves fast: if a command fails, check `vega --help` / `vega <command> --help` and developer.amazon.com/docs/vega before improvising.

## 1. Release

1. Bump the version (semver) in `package.json` and `CHANGELOG.md`. Each update needs a higher `--build-version` **and** a higher `--build-number` (integer; the Appstore blocks downgrades with it).
2. Merge to `main`, then tag: `git tag v1.0.0 && git push --tags`
3. Pre-release hygiene: delete `proxy-config.json` and `assets/raw/ca.pem`, `vega clean`, check `manifest.toml` lists only the privileges you use. Keep `assets/raw/AppstoreAuthenticationKey.pem` if the app uses IAP (needed for Live App Testing / production mode; see `docs/media-iap-integrations.md` section 2).
4. CI builds the release package from the tag:

   ```bash
   npx react-native build-vega --build-type Release --build-version 1.0.0 --build-number 1
   vega exec vpt validate build/armv7-release/<AppName>_armv7.vpkg
   vega exec vpt info build/armv7-release/<AppName>_armv7.vpkg --json
   ```

5. Smoke-test that exact `.vpkg` on the stick, run the KPI Visualizer (`quality-gates.md`) and the Appstore cases below.

Signing: Amazon removes your signature on upload and re-signs with a per-account key. So Login with Amazon, IAP and Live App Testing need the production API key built from the **Appstore certificate hashes** (Developer Console > App files).

## 2. Appstore cases to pass before submitting (test on a physical stick)

- Exit with Back and with Home, relaunch (no audio leak).
- Switch apps: no audio overlap, crash, blank screen or A/V desync.
- HDMI unplug 5 min / input change / TV off: playback pauses, resumes at the same position.
- Device restart: login is kept.
- Screensaver after 5 min idle: paused, no audio.
- Alexa interruption: pause, then resume.
- Bluetooth audio routing and sync.
- General: loading indicator after 15 s, every sign-in option works, no long stretches below 25 fps, visible focus, no touch UI, no navigation traps, readable at 10 ft, data persists across restart and update.

## 3. Publish to the Amazon Appstore

1. Developer Console Dashboard > **Add a New App**. If a Fire OS version exists, add a **Vega release track** to that listing instead (same package name; shared listing, reviews, IAP and entitlements). A separate Vega listing makes customers pay twice.
2. **App Setup**: general information, store listing (short description ≤ 2,000 bytes, plain text, first 200 chars shown on Fire TV), pricing and availability, policies and compliance (content rating, privacy policy URL, **user data privacy questionnaire**: publishing is blocked if incomplete).
3. **Fire TV assets**:

   | Asset | Size | Notes |
   |-------|------|-------|
   | App icon | 1280×720 PNG, no transparency | subject inside the 882×448 safe area |
   | Screenshots (3–10) | 1920×1080 JPG/24-bit PNG | no personal data; capture with `gwsi-tool-screenshooter` |
   | Background image | 1920×1080 JPG/24-bit PNG | subject on the right, inside 1214×830; no logos or titles |

   The in-package manifest icon (512×512 PNG, ≤ 1 MB) is separate.
4. **In-App Items** tab: create IAP items before submitting (SKUs must match the code; unique across your apps).
5. **Release track**: upload `build/armv7-release/<AppName>_armv7.vpkg`; under Amazon Fire TV > Amazon Vega TV tick the devices; add testing instructions with test credentials (no login = rejection); certify export compliance. The manifest must register `com.amazon.category.main`.
6. Optional: **Live App Testing** (up to 5 tracks, production services) with a small group first.
7. **Submit for review.** Statuses: Submitted → In review → Approved → Live rollout. No official SLA; contact Amazon after six days in review. The approval email goes to the account owner.

## 4. Update and maintain

- **New version:** repeat sections 1–3 with higher version and build number. Staged rollout is available; force updates only after 100% rollout and 24 h live.
- **SDK updates:** `vega sdk list-remote`, `vega sdk install <version>`, update `vega-sdk-requirements.json` and CI's `VEGA_SDK_VERSION`, then `vega project install --fix`, `npm install` (commit the lockfile; CI uses `npm ci`), `vega project doctor`, rebuild, test on the stick. Read the release notes' breaking changes and known issues first.
- **New Vega OS versions:** watch the release notes; test on the stick when it auto-updates; raise `[os.version] target` when ready.
- **Monitor:** Appstore reviews, crash reports, IAP reports; KPI Visualizer on each release.
- **Housekeeping:** monthly dependency review (pinned versions, security advisories).

## 5. Retire (if ever)

Unpublish from the Developer Console (whether existing users keep the installed app is not covered in the Vega docs; check before announcing), archive the repo, keep the last release `.vpkg` and the signing and API key details.
