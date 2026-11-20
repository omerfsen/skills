# Quality gates: fast, stable, secure

What "good" means for a Vega app, with Amazon's numbers. Check each gate per feature and before every release. Sources and detail: `docs/performance.md`, `docs/security-appstore.md`, `docs/testing-ux-ecosystem.md`. Docs version 2.0.

> Numbers and APIs come from Amazon's 2.0 docs; re-check them against the current docs before treating a gate as final.

## Fast

Measure on a **release build on a physical stick** with the App KPI Visualizer:

```bash
npm install --save-exact @amazon-devices/kepler-performance-api@<version>
vega exec perf doctor --app-name=<app-id>
vega exec perf kpi-visualizer --app-name=<app-id>          # default: cool start TTFF + TTFD
# other KPIs: --kpi warm-start-latency | ui-fluidity | video-fluidity | foreground-memory | background-memory
```

Amazon calls these "guidelines", not Appstore gates; treat them as this skill's release gates.

| KPI | Guideline |
|-----|--------|
| Time to first frame (cool / warm) | < 1.5 s / < 0.5 s |
| Time to fully drawn (cool / warm) | < 8 s / < 1.5 s |
| UI fluidity (scroll/navigation frames rendered) | > 99% |
| Focus event response | < 200 ms |
| Key press latency | < 100 ms |
| Time to first video frame | < 2500 ms |
| Video fluidity | > 99% |
| Foreground / background memory (PSS) | < 400 MiB / < 150 MiB |

Practices that get you there (detail in `docs/performance.md` sections 3–4):

- Call `useReportFullyDrawn()` (`@amazon-devices/kepler-performance-api`) after the first screen and its data render, and again on every warm start; without it TTFD reads `-1`.
- Lists: `@shopify/flash-list` on RN 0.83 (`@amazon-devices/shopify__flash-list` on 0.72). No `ScrollView` + `map` for long rows.
- Animations: `useNativeDriver: true`. Keep focus handlers to one render; avoid overdraw; trim imports.
- Use the native splash screen manager; load the minimum for the first screen.
- Add `@amazon-devices/eslint-plugin-kepler` and fail CI on its errors.

## Stable

- **Memory:** hard limits 420 MB foreground and 150 MB background; over that the low-memory killer sends SIGKILL (no stack trace). The Hermes heap fails near 200 MB. Watch the `memoryWarning` event and drop caches.
- **Lifecycle:** use VegaAppState, not React Native `AppState`. On going to background, release media resources synchronously: the JS thread is suspended, so work after an `await` may never run. Remove listeners in `useEffect` cleanup.
- **Watchdogs:** JS thread blocked ~8 s, a lifecycle callback 8 s, surface creation 12 s, main thread 45 s → the app is killed. Keep heavy work off the JS thread (and, as general practice, split long work into chunks).
- **Persistence:** sign-in and user data must survive restart and update (Appstore test). Show a loading indicator if loading takes more than 15 s.
- **Network** (general practice): handle offline and slow networks (Vega NetInfo port: `@amazon-devices/keplerscript-netmgr-lib` per its API page, which needs `[[wants.service]] id = "com.amazon.network.service"` and the run-time privilege `com.amazon.network.privilege.net-info`; the supported-libraries page lists it as react-native-netinfo, so check the current name); time out and retry with backoff; never block the first frame on the network.
- **Errors** (general React Native practice; the Vega docs give no specific guidance): wrap screens in error boundaries and report errors to a crash service from the supported list (Sentry, Bugsnag, Firebase…), remembering low-memory kills won't show up there.
- **Compatibility:** `vega project doctor` must pass; guard APIs newer than `[os.version] min` with `isPresentOnOS(...)`; don't pull a library built for RN 0.72 into an RN 0.83 app (crash at launch).

## Secure

- **Least privilege:** declare only the privileges and services you use (`[[needs.privilege]]` / `[[wants.privilege]]`); request run-time privileges with `@amazon-devices/security-manager-lib` and handle denial (revoking kills the app).
- **Storage:** AsyncStorage is unencrypted: no tokens there. Use `@amazon-devices/react-native-mmkv` with an `encryptionKey`, or encrypt with `@amazon-devices/keplercrypto` (AES-GCM; not available on the Virtual Device, so test it on the stick). There is no keychain-style library on Vega; keep long-lived secrets on the backend.
- **Network:** HTTPS only. WebView (RN 0.83) blocks HTTP unless a `[network-traffic-policy.cleartext]` section allows it; leave it out. Never relax WebView defaults (`mixedContentMode`, `allowFileAccess`, `onSslError`).
- **Release hygiene:** delete Network Proxy files (`proxy-config.json`, `assets/raw/ca.pem`) and run `vega clean` before a release build. Never ship debug builds.
- **Identity:** Login with Amazon needs separate debug and production API keys (production uses the Appstore certificate hashes). Verify IAP receipts on your backend (Receipt Verification Service), never only on the device.
- **Supply chain:** exact versions, committed lockfile, `npm ci` in CI, `npm audit` reviewed, Actions pinned to SHAs, secrets only in CI.
- **Privacy:** collect the minimum; publish a privacy policy; complete the Developer Console privacy questionnaire (required to publish); follow COPPA rules if children may use the app.
- **Telemetry:** turn off SDK and tool telemetry on dev machines and CI.

## TV UX

- D-pad only: every screen reachable and escapable with arrows, Select and Back; no touch assumptions; no navigation traps.
- Visible focus on every focusable element (only `TouchableOpacity` and `TouchableHighlight` change on focus by default; `Button`, `Pressable` and `View focusable` need your own `onFocus`/`onBlur` styling).
- Focus APIs from `@amazon-devices/react-native-kepler` (`TVFocusGuideView`, `FocusManager`); `useTVEventHandler` only observes keys; Back via `BackHandler`; Home cannot be intercepted.
- Keep UI inside the outer 5% safe zone (from Amazon's 2020 Fire OS design guide; no Vega-specific guide exists); design for 1920×1080 (960×540 dp at scale 2); readable at 10 feet.
- Accessibility: VoiceView labels and roles (on RN 0.83 prefer `aria-*` and `role`; `accessibilityLabel`/`accessibilityRole` are slated for removal); run the Vega Studio A11y Inspector.
- Playback: pause on HDMI unplug, input change, screensaver and Alexa interruption, and resume from the same position (Appstore tests).
