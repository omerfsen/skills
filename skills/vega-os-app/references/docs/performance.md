# Vega OS performance and stability

> Fact sheet. Collected 2026-11-20 from Amazon's Vega docs (latest version: 2.0) and the sources listed at the end; every line names its source page and doc version. Vega changes between SDK versions: re-check a fact against the current docs (or `vega <command> --help`) before relying on it, and prefer the docs when they disagree with this file.


Researched 2026-11-20.

**Doc version.** The latest Vega docs are **2.0** (`/docs/vega/2.0/...`), for "Vega SDK and OS version 2.0" (GA; release notes last modified 2026-10-28). 2.0 comes after 0.24. Versions 0.25, 0.26, 1.x, 2.1 and 3.0 return 404, and `/docs/vega/latest/...` also resolves. I compared these pages between 2.0 and 0.24 after normalising version paths: faq, best_practices, measure-app-kpis (KPI table and CLI help), js-thread-perf, fluidity-foreground, monitor-cpu-usage, concurrent-rendering, avoid-overdraw, detect-overdraw, webview-app-performance-best-practices, crash-report, logs, loggingctl, acr-summary-report, foreground-background, performance-cli. Their body text is identical, or differs only in links or "last modified" dates. eslint-plugin and monitor-record-app-memory have small 2.0 changes, noted below. All facts below come from the 2.0 page unless marked otherwise. RN-for-Vega API pages are versioned by React Native version (`/docs/react-native-vega/0.83/...`, or `0.72`).

Base URL: `https://developer.amazon.com/docs/vega/2.0/`

---

## 1. KPIs Amazon measures, and their targets

Source: [Measure App KPIs](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0, last modified 2026-10-13; identical to 0.24).

| KPI | Scenario | Guideline |
|---|---|---|
| TTFF (time to first frame) | Cool start | < 1.5 s |
| TTFF | Warm start (background to foreground) | < 0.5 s |
| TTFD (time to fully drawn) | Cool start | < 8.0 s |
| TTFD micro KPIs: JS bundle load time, network calls time | Cool start | N/A (informational) |
| TTFD | Warm start | < 1.5 s |
| Foreground Memory (PSS) | App in foreground | < 400 MiB |
| Background Memory (PSS) | App in background | < 150 MiB |
| Video Fluidity | Video playback | > 99% |
| TTFVF (time to first video frame), plus micro KPIs "time to player start" and "player start to first video frame" (Vega Media Player only) | Video playback | < 2500 ms |
| 3+ / 5+ video consecutive dropped frames | Video playback | N/A (count) |
| Key pressed latency / key released latency (UI thread to JS callback) | Video playback | < 100 ms |
| UI Fluidity (% of frames rendered during scrolling/navigation) | UI interaction | > 99% |
| App Event Response Time - Focus (native UI thread to JS thread scheduling latency for onFocus/onBlur) | UI interaction | < 200 ms. Over 200 ms "indicate[s] JavaScript thread congestion" |
| 3+ / 5+ consecutive dropped frames, 5+ consecutive delayed focus events | UI interaction | N/A (count) |

- The OS computes TTFF with no app markers. TTFD needs an app-side "fully drawn" marker (see section 2). [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- "Before you publish your app to the App Store, measure the performance of the release variant." [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- Report health colours: green meets the guideline, yellow is within 10% of it, red exceeds it by more than 10%. The VS Code window shows the P90 over 3 iterations. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- A KPI value of `-1` means data is unavailable. For TTFD it usually means the app never called the Report Fully Drawn API. Heavy trace loss gives N/A KPIs. `--ignore-trace-loss` shows them anyway, but the values "appear better than actual performance". [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- Vega apps "require a UI Fluidity KPI score of 99% or higher" for interactions such as scrolling. [source](https://developer.amazon.com/docs/vega/2.0/investigate-ui-fluidity) (2.0); also [Identify UI Rendering Issues](https://developer.amazon.com/docs/vega/2.0/ui-rendering) (2.0)
- What the visualizer does per KPI:
  - Cool start: launch, wait 10 s, close.
  - Warm start: launch, launch another app, relaunch, wait 15 s.
  - Background memory: launch, background it, wait 15 s.
  - Default UI fluidity test: 2 sets of horizontal scrolls (5 left, 5 right), 2 sets of vertical scrolls (5 down, 5 up), 900 ms between actions.
  - Foreground memory and video fluidity need a custom test scenario.

  [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- Report files are `aggregated-kpi-report-[timestamp].json` and `[test-name]-kpi-report-[timestamp].json`, in the project's `generated` directory. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)

### KPI Visualizer CLI (exact commands)

Source: [Measure App KPIs](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0).

```bash
npm install @amazon-devices/kepler-performance-api   # prerequisite

# 1. Check host + device readiness (Appium, driver, CPU/mem headroom)
vega exec perf doctor --app-name=com.example.myapp.main
vega exec perf doctor --app-name=com.example.myapp.main --device-serial-number=G000XX1234567890

# 2. Run (default KPIs = cool-start TTFF + TTFD, 3 iterations)
vega exec perf kpi-visualizer --app-name=com.example.myapp.main
vega exec perf kpi-visualizer --app-name=com.example.myapp.main --kpi warm-start-latency
vega exec perf kpi-visualizer --app-name=com.example.myapp.main --kpi ui-fluidity --test-scenario ui_fluidity_test.py
vega exec perf kpi-visualizer --app-name=com.example.myapp.main --certification   # 30 iterations, P90
vega exec perf kpi-visualizer --help
```

- `--kpi` values: `cool-start-latency`, `ui-fluidity`, `warm-start-latency`, `foreground-memory`, `background-memory`, `video-fluidity`. `video-fluidity` needs `--test-scenario`.
- Other flags from `--help`:
  - `--iterations N`
  - `--record-cpu-profiling`
  - `--sourcemap-file-path`
  - `--grpc-port`
  - `--certification`
  - `--expected-video-fps`
  - `--test-scenario <python>`
  - `--monitor-processes` (e.g. `webview.renderer_service`)
  - `--ignore-trace-loss`
  - `--version`

  [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- `--app-name` takes the ID of the default interactive component from `manifest.toml` (e.g. `com.amazondeveloper.keplervideoapp.main`). [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- In VS Code, run the command palette entry **Vega: Launch App KPI Visualizer**, or open Vega Studio panel > App KPI Visualizer. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- With the CPU Profiler on, a fluidity run produces 3 artifacts: a fluidity graph, a CPU profile (flamegraph) and a Perfetto trace. Trace files are named `iter_<N>_vs_trace.protobuf`. Open them in the Perfetto UI and look at the `Toolkit/Render thread` track. [source](https://developer.amazon.com/docs/vega/2.0/ui-rendering) (2.0)

### Test scenarios (Appium, Python)

Source: [Measure Fluidity and Foreground Memory](https://developer.amazon.com/docs/vega/2.0/fluidity-foreground) (2.0).

- Requires Appium Server, the Appium Vega Driver and Appium Inspector.
- Generate the template with the VS Code command **Vega: Generate test scenario template**. It produces a Python `TestRunner` class with two members:
  - `prep`: setup, e.g. navigating to and starting a video.
  - `run`: the measured interactions, e.g. key presses or `sleep` during playback.
- D-pad input goes through Appium `executeScript` with `jsonrpc: injectInputKeyEvent`. Example argument for D-pad Down:

```json
[[{"inputKeyEvent": "108", "holdDuration": 0}]]
```

### Other measurement tools (exact commands)

Source: [Vega Studio Performance CLI](https://developer.amazon.com/docs/vega/2.0/performance-cli) (2.0).

```bash
vega exec perf --help
vega exec perf record --app-name com.<COMPANY>.<APP>.main [--recording-timeout 60] \
     [--trace-categories app,graphics,ui,video] [--enable-cpu-sched] [--output-dir DIR] [--device-serial-number SN]
#   default trace categories: app, graphics, ui, lifecycle, input, video, webview; default timeout 120 s
vega exec perf activity-monitor --app-name com.<COMPANY>.<APP>.main [--record-cpu-profiling --sourcemap-file-path /path/to/sourcemap]
vega exec perf memory-monitor --app-name com.<COMPANY>.<APP>.main [--keplerscript-app --relative-gc-count]
#   in memory-monitor: press s to start/stop recording, q to quit
vega exec perf kpi-visualizer --app-name com.<COMPANY>.<APP>.main [--overridden-video-fps 30]
```

- Activity Monitor (VS Code: **Vega: Start Activity Monitor**): live CPU and memory graphs, recording, and a thread-state view (Running, Runnable, Runnable (Preempted), Sleeping, and so on). It works on Fire TV Stick only. Record in **Release** build mode. [source](https://developer.amazon.com/docs/vega/2.0/monitor-cpu-usage), [js-thread-perf](https://developer.amazon.com/docs/vega/2.0/js-thread-perf) (2.0)
- Memory Monitor (VS Code: **Vega: Start Memory Monitor**) shows PSS, USS and JS heap. The JS heap view has allocated bytes, external bytes, GC event count, GC CPU time and %, and max GC pause. The recording timeout defaults to 120 s. "Increasing USS over time may signal a memory leak." [source](https://developer.amazon.com/docs/vega/2.0/monitor-record-app-memory) (2.0)
- Flashlight (open source) gives CPU and memory, with FPS "planned". It runs on macOS and Linux only, with `PLATFORM=keplerdevice node packages/commands/measure/dist/server/bin.js measure` (use `keplersimulator` for the VVD). [source](https://developer.amazon.com/docs/vega/2.0/flashlight) (2.0)
- Custom trace markers: the React Native `Systrace` API is supported (`Systrace.beginEvent`, `Systrace.counterEvent`). Amazon warns that it adds overhead and can produce false positives. [source](https://developer.amazon.com/docs/vega/2.0/ui-rendering) (2.0)
- Overdraw: add the launch argument `SHOW_OVERDRAWN=true` in Vega Studio launch options, or send it via vmsgr. Colours: blue = 1x, green = 2x, pink = 3x, red = 4x or more. The target is no pink or red.

  ```bash
  vda shell vmsgr send "pkg://com.amazon.keplersampleapp.main?SHOW_OVERDRAWN=true"
  ```

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices), [detect-overdraw](https://developer.amazon.com/docs/vega/2.0/detect-overdraw) (2.0)
- Other tools on the overview page: Chrome DevTools, React Native DevTools, Charles Proxy, Network Proxy, and ADBT (Amazon Devices Builder Tools for AI), which does TTFF/TTFD diagnosis, re-render detection and best-practice validation. [source](https://developer.amazon.com/docs/vega/2.0/improve-performance-overview) (2.0)

---

## 2. Readiness API: the fully drawn marker (required for TTFD)

Source: [Measure App KPIs](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0).

- Use `useReportFullyDrawn()` from `@amazon-devices/kepler-performance-api`. Call it once after the cool-start main screen and its initial data are rendered, and again on every warm start (background to active).
- Amazon's sample:

```tsx
import { useReportFullyDrawn } from '@amazon-devices/kepler-performance-api';
import { useKeplerAppStateManager } from '@amazon-devices/react-native-kepler';

const reportFullyDrawnCallback = useReportFullyDrawn();
const KeplerAppStateManager = useKeplerAppStateManager();
const [appState, setAppState] = useState(KeplerAppStateManager.getCurrentState());

useEffect(() => { reportFullyDrawnCallback(); }, [reportFullyDrawnCallback]); // cool start
const handleAppStateChange = useCallback((stateChange: any) => {
  if (appState.match(/^(inactive|background)$/) && stateChange === 'active') {
    reportFullyDrawnCallback();                                          // warm start
  }
  if (stateChange.match(/^(inactive|background|active|unknown)$/)) setAppState(stateChange);
}, [appState, reportFullyDrawnCallback]);
useEffect(() => {
  const sub = KeplerAppStateManager.addAppStateListener('change', handleAppStateChange);
  return () => sub.remove();
}, [handleAppStateChange]);
```

- `addAppStateListener` is deprecated in favour of `addEventListener`; see section 5.
- If extra async work must finish first, put its completion state in the effect's dependency array. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- A second form appears in Amazon's own sample on the js-thread-perf page: `global.performance.reportFullyDrawn()`, with `// @ts-ignore`. [source](https://developer.amazon.com/docs/vega/2.0/js-thread-perf) (2.0)
- WebView apps call `reportFullyDrawnCallback()` in the `WebView` `onLoad` handler, only for the real home URL, and hide the splash screen there too. [source](https://developer.amazon.com/docs/vega/2.0/webview-app-performance-best-practices) (2.0)
- The ESLint rule `@amazon-devices/kepler/detect-report-fully-drawn` (warn) checks that `useReportFullyDrawn()` is used correctly. [source](https://developer.amazon.com/docs/vega/2.0/eslint-plugin) (2.0)

---

## 3. Startup (TTFF/TTFD) best practices

- **Native splash screen.** Use `SplashScreenManager`. It renders from raw images in the vpkg, so the JS thread stays free. It auto-dismisses after the first frame. To control dismissal, call `usePreventHideSplashScreen()` in the root component and later `useHideSplashScreenCallback()()`. Both come from `@amazon-devices/react-native-kepler`. [best_practices](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0); [SplashScreenManager](https://developer.amazon.com/docs/react-native-vega/0.83/splashscreenmanager) (RN 0.83)
  - Assets: `assets/raw/SplashScreenImages.zip`, containing `_loop/loop00000.png...` and `desc.txt`. The first line of `desc.txt` is `W H FPS`, then parts in the form `c 0 0 _loop`.
  - Supported resolutions: 1920x1080 (TV profile) or 960x480 (multimodal). 4K is not supported.
  - Build the zip with `zip -r SplashScreenImages.zip _loop desc.txt`. Do not use macOS Finder "Compress".
  - The older `preventAutoHideAsync()` and `hideAsync()` are replaced by the hooks.

  [source](https://developer.amazon.com/docs/react-native-vega/0.83/splashscreenmanager) (RN 0.83)
- **2.0 known issue: PNG splash animations play at about 8 fps.** Workaround: ASTC frames in `assets/raw/compressedSplashScreenImages.zip` give 30 fps. However, an ASTC splash can render black after the first launch until reboot. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Common causes of slow launch: large JS bundle and high network latency at startup. [source](https://developer.amazon.com/docs/vega/2.0/faq) (2.0)
- **Bundle size.** Run `npx react-native-bundle-visualizer`. Import per function: `import debounce from 'lodash/debounce'` instead of `import { debounce } from 'lodash'`. In Amazon's example this cut 480.77 KB. [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)
- **Code splitting.** Use `React.lazy` with `<Suspense>` to shrink the initial bundle and render sooner. [source](https://developer.amazon.com/docs/vega/2.0/best_practices), [concurrent-rendering](https://developer.amazon.com/docs/vega/2.0/concurrent-rendering) (2.0)
- **RN 0.83 / Hermes.** RN 0.83 on Vega brings React 19 with the React Compiler, Hermes v1 ("improved runtime performance and faster startup"), Fabric/Concurrent Renderer on by default, and Metro startup about 15x faster. [SDK 0.24 release notes](https://developer.amazon.com/docs/vega/0.24/vega-release-notes) (0.24); [2.0 release notes](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)

---

## 4. Rendering, lists, images, animation, focus (JS thread vs UI thread)

Source: [App Performance Best Practices](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0, last modified 2026-09-28) unless noted.

- **Re-renders.** The page recommends:
  - `memo`, with a custom `arePropsEqual` where needed
  - `useCallback` and `useMemo` for expensive values, with full dependency arrays
  - fewer `useState` calls
  - `useRef` for mutable values that don't render
  - `useTransition` for non-urgent updates
  - `StrictMode` in development
  - `eslint-plugin-react-compiler`
  - why-did-you-render, and the React DevTools option "Record why each component rendered"

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)
- **On RN 0.83 the React Compiler handles memoization at build time.** The 2.0 release notes say you "no longer need to add `React.memo` or `useCallback` by hand". The best-practices page has not been updated to match. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **Concurrent rendering.**
  - Use `useTransition` (`startTransition`) for state you own and `useDeferredValue` for props or external values.
  - Pair both with memoized children, otherwise the benefit is lost.
  - Validate with the React Native DevTools Profiler and `vega exec perf kpi-visualizer --kpi ui-fluidity --test-scenario ...`, targeting > 99%.

  [source](https://developer.amazon.com/docs/vega/2.0/concurrent-rendering) (2.0)
- **Lists.**
  - `ScrollView` is fine only for small data sets.
  - For `FlatList`, set `getItemLayout`, `windowSize` and `initialNumToRender`, and memoize items. Amazon's example uses `windowSize={Platform.isTV ? 5 : 13}`.
  - Never nest a vertical FlatList inside a vertical ScrollView. Use `ListHeaderComponent` / `ListFooterComponent` instead.

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices), [js-thread-perf](https://developer.amazon.com/docs/vega/2.0/js-thread-perf) (2.0)
- **FlashList.** "Vega fully supports Shopify's FlashList", and Amazon recommends it over FlatList.
  - Set `estimatedItemSize`.
  - Avoid `useState` inside recycled cells, because state carries over between items.
  - Remove `key` props from items.
  - Use `getItemType` for heterogeneous cells.
  - FlatList-only props (`windowSize`, `getItemLayout`, `initialNumToRender`, `maxToRenderPerBatch`, `updateCellsBatchingPeriod`) do nothing in FlashList.

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)
  - **Package name depends on the RN version.** On RN 0.72 use `@amazon-devices/shopify__flash-list` (~2.0.0, upstream 1.6.3). On RN 0.83 use the public `@shopify/flash-list`; the Amazon package is not available and fails at runtime. [2.0 release notes](https://developer.amazon.com/docs/vega/2.0/vega-release-notes), [supported libraries](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)
- **Images.** The Vega `Image` component already caches natively, comparable to react-native-fast-image or expo-image. Ship several sizes and use thumbnail or cropped assets in lists, which saves decode CPU and memory. [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)
  - Known issue in 0.24: `expo-image` TurboModule features can crash on autolinking. [source](https://developer.amazon.com/docs/vega/0.24/vega-release-notes) (0.24)
- **Animations.**
  - Use `Animated` with `useNativeDriver: true` so the JS thread doesn't compute each frame.
  - Schedule work with `InteractionManager.runAfterInteractions()`.
  - The ESLint rule `@amazon-devices/kepler/animated` is set to **error** and fails the build if the native driver isn't used.

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices), [eslint-plugin](https://developer.amazon.com/docs/vega/2.0/eslint-plugin) (2.0)
  - Reanimated is supported: RN 0.83 uses `@amazon-devices/react-native-reanimated` ~4.0.0 (upstream 4.1.3) plus `react-native-worklets` ~1.0.0; RN 0.72 uses ~2.0.0 (upstream 3.5.4). [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)
  - The docs give no Vega-specific Reanimated performance guidance.
- **Focus (D-pad).**
  - Keep `onFocus` and `onBlur` trivial: at most one React render per focus change, using conditional styles.
  - Use native animations for scale or opacity on focus, and debounce them so a fast scroll doesn't trigger every item's animation.

  [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)
  - The KPI for this is App Event Response Time - Focus, < 200 ms. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- **Key press latency causes.** A busy JS thread (heavy computation, re-renders, inefficient handlers) or a starved JS thread (other processes using the CPU). [source](https://developer.amazon.com/docs/vega/2.0/faq) (2.0)
- **Slow scroll causes.** Inefficient lists, JS-thread computation, complex item layouts, large images, and re-renders during scroll. [source](https://developer.amazon.com/docs/vega/2.0/faq) (2.0)
- **Overdraw.** Remove unnecessary backgrounds, flatten the hierarchy and reduce transparency. [source](https://developer.amazon.com/docs/vega/2.0/detect-overdraw) (2.0)
- **Frame size mismatch.** A frame larger than the display loads extra pixels to the GPU. Check that frame width and height match the device display in the Perfetto `Tools/Render` track. [source](https://developer.amazon.com/docs/vega/2.0/ui-rendering) (2.0)
- **`setNativeProps()` crashes on RN 0.83** (Fabric) with `TypeError: undefined is not a function`. Use state-driven props instead. [source](https://developer.amazon.com/docs/vega/0.24/vega-release-notes) (0.24)
- **Heavy JS work in media apps.** In 2.0, dash.js parsing large multi-period DASH manifests can block the JS thread long enough to trigger an ANR after about 25–60 minutes. Workaround: run the player in a headless JS service, and don't `JSON.stringify` or log the manifest. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **WebView apps.**
  - Bundle and defer scripts, and use HTTP/2 or HTTP/3, a CDN and `Cache-Control`.
  - Keep input handlers light, and consider Web Workers.
  - Animate with GPU CSS (`transform`, `will-change`).
  - Use hardware-decoded codecs.
  - Don't mix heavy CSS animation with video.
  - Render only visible elements.

  [source](https://developer.amazon.com/docs/vega/2.0/webview-app-performance-best-practices) (2.0)

### Vega ESLint plugin

Source: [Install and Configure the Vega ESLint Plugin](https://developer.amazon.com/docs/vega/2.0/eslint-plugin) (2.0, last modified 2026-10-21).

```bash
npm install --save-dev @amazon-devices/eslint-plugin-kepler jsonc-eslint-parser
```

```js
// .eslintrc: extends "plugin:@amazon-devices/eslint-plugin-kepler/kepler"   (all rules)
// subsets: .../performance, .../system-distributed-libs, .../accessibility  (accessibility added in 2.0)
// plugins: ["@amazon-devices/eslint-plugin-kepler"]   (0.24 page said "@amazon-devices/kepler")
// lint script formatter:
//   eslint src test package.json --ext .ts,.tsx --format node_modules/@amazon-devices/eslint-plugin-kepler/dist/formatters/default.js
```

Performance rules:

- `kepler/flat-list` (warn)
- `kepler/check-subscription` (warn): checks that `useAddVegaAppStateListenerCallback` and `useAddUserInputListenerCallback` subscriptions are removed
- `kepler/animated` (**error**)
- `kepler/detect-report-fully-drawn` (warn)
- `kepler/detect-splash-screen` (warn)

---

## 5. App lifecycle: foreground/background, suspend, kill, state

- The Vega **Lifecycle Manager (LCM)** moves apps between foreground and background, driven by the Fire TV home launcher. [source](https://developer.amazon.com/docs/vega/2.0/foreground-background) (2.0)
- On Vega, app state applies **per interactive component**, not per process. "Foregrounded (or active) means an interactive component can have visible UI." [source](https://developer.amazon.com/docs/react-native-vega/0.83/keplerappstate) (RN 0.83)
- **Do not use `AppState` from `'react-native'`.** It is `@deprecated` on RN 0.83 for Vega interactive apps. Use VegaAppState. [source](https://developer.amazon.com/docs/react-native-vega/0.83/appstate) (RN 0.83)
  - The 2.0 foreground-background page still shows the plain RN `AppState` sample. That page is stale on this point.
- **VegaAppState API** ([page titled "VegaAppState", URL `.../keplerappstate`](https://developer.amazon.com/docs/react-native-vega/0.83/keplerappstate), RN 0.83, last modified 2026-05-13):
  - `useKeplerAppStateManager()` returns `IKeplerAppStateManager`, imported from `@amazon-devices/react-native-kepler`.
  - Its methods:
    - `getCurrentState()`
    - `addEventListener(event, cb)`: returns a subscription; call `.remove()` to unsubscribe. Requires `@amazon-devices/react-native-kepler` >= 2.1.0.
    - `addEventListenerWithReplay(event, cb)`: replays events that fired before it subscribed. Only the first listener of each event gets the replay.
    - `getComponentInstance()`
  - Deprecated:
    - `addAppStateListener`
    - `useGetCurrentKeplerAppStateCallback`
    - `useAddKeplerAppStateListenerCallback`
    - `useAddKeplerAppStateListenerWithReplayCallback`
    - `useComponentInstance`
  - States: `'active' | 'background' | 'inactive' | 'unknown'`. `inactive` "currently never happens for typical React Native apps".
  - Events: `'change' | 'memoryWarning' | 'blur' | 'focus' | 'reconfigure' | 'displayChange'`. `reconfigure` carries `reconfigureReason: 'homePressed'`; `displayChange` carries `displayConnected` or `displayDisconnected`.

```tsx
const mgr = useKeplerAppStateManager();
useEffect(() => {
  const s1 = mgr.addEventListener('change', (next) => { /* 'active' | 'background' ... */ });
  const s2 = mgr.addEventListener('memoryWarning', () => { /* drop caches */ });
  return () => { s1.remove(); s2.remove(); };
}, []);
```

- **Going to background, Amazon's checklist:** stop playback to release decoders, update lifecycle state, free memory, and enable warm start. [source](https://developer.amazon.com/docs/vega/2.0/foreground-background) (2.0)
- **Do the background cleanup synchronously.**
  - "The KeplerScript runtime suspends the JavaScript thread after the background transition, so any work after an `await` may never execute."
  - If native media resources (decoders, video surfaces) are not released, the OS kills the app with `CRASH_REASON: ReclaimMediaResource`.
  - Fix: "release native media resources synchronously in your AppState background handler before any async operations."

  [source](https://developer.amazon.com/docs/vega/2.0/acr-summary-report) (2.0)
- **Ways the OS ends an app** (Resource Manager; no minidump or stack trace):
  - `LowMemoryKiller`
  - `LowCPUKiller`
  - `LRUDepthMaintenance`: too many background apps
  - `ReclaimMediaResource`

  LCM can also end an app with `AppNotResponding`, which has a stack trace. [source](https://developer.amazon.com/docs/vega/2.0/acr-summary-report) (2.0)
- **Save and restore state.** I found no Vega page that prescribes a persistence API for app state. Supported storage includes `@amazon-devices/react-native-async-storage__async-storage` (RN 0.83 ~2.1.0). [supported libraries](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0). See the open questions.
- **Simulating lifecycle transitions:**

```bash
vda shell vmsgr send os://home                 # send app to background (same as Home key)
vda shell vlcm list                            # list running apps / visibility
vda shell vmsgr send orpheus://<appID>         # bring to foreground (warm start) or launch another app
vega exec vda shell vmsgr send pkg://com.example.myapp.main   # launch by package (loggingctl page form)
```

  [foreground-background](https://developer.amazon.com/docs/vega/2.0/foreground-background), [loggingctl](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)
- **Headless tasks (2.0).** A native Task Manager periodic task has a minimum interval of 60 s and runs for at most 5 min. There is no JS API yet; copy the sample's `PeriodicTasks` Turbo Module. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
  - The triage page gives Task Manager tasks 10 minutes before an ANR kill. [source](https://developer.amazon.com/docs/vega/2.0/triage-guidelines) (2.0)
- **Clean up listeners.** Remove Turbo Module listeners (VegaAppState, DeviceInfo, Keyboard) in the `useEffect` cleanup to avoid "dangling memory". [source](https://developer.amazon.com/docs/vega/2.0/best_practices) (2.0)

---

## 6. Memory limits and low-memory handling

- **Hard limits:** foreground apps 420 MB, background apps 150 MB. Going over triggers a Low Memory Kill, which crashes the app. High memory also costs JS time in GC. [source](https://developer.amazon.com/docs/vega/2.0/faq) (2.0; same in 0.22)
- **KPI guidelines:** foreground < 400 MiB, background < 150 MiB (PSS). [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)
- **JS heap.** Exceeding the Hermes heap limit gives `HermesGC OOM`, followed by a `SIGSEGV` crash in `keplerscript-runtime-loader*.so`. Watch the JS heap graph as it approaches "200-MB". Amazon's example log has `heapSize = 205520896`:

```
... keplerscript-ru[7087]: ... HermesGC OOM: [] reason = Max heap size was exceeded , numCollections = 177, heapSize = 205520896, allocated = 201732128 ...
```

  [source](https://developer.amazon.com/docs/vega/2.0/javascript-crash-memory) (2.0)
- **Leak workflow.**
  1. Memory Monitor recording.
  2. Chrome DevTools > Memory > Heap Snapshot (download it).
  3. Run `npm install -g memlab`, then `memlab find-leaks --baseline <s1> --target <s2> --final <s3> --trace-all-objects`.
  4. Read the retainer traces.

  [source](https://developer.amazon.com/docs/vega/2.0/javascript-crash-memory) (2.0)
- **`memoryWarning`** is a VegaAppState event: "Received when the app receives a memory warning." [source](https://developer.amazon.com/docs/react-native-vega/0.83/keplerappstate) (RN 0.83)
- **Per-device limits.** The docs give one set of limits, with no breakdown per device. 2.0 devices are Fire TV Stick 4K (3rd Gen), Fire TV Stick HD and Fire TV Stick 4K Select. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **Crash SDKs can't see LMK kills.** The kernel sends `SIGKILL`, so libraries such as Sentry can't record the crash. `tryGetLastTerminationData()` currently reports only ANR terminations. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0, known issue)
  - That API is native: `IAppLifecycleControl2.tryGetLastTerminationData()` returns `ITerminationInfo?` in `com.amazon.kepler.app_lifecycle`. [source](https://developer.amazon.com/docs/vega-native/2.0/native-api-migration-guide) (2.0)

---

## 7. ANR watchdogs

Source: [Triage Guidelines](https://developer.amazon.com/docs/vega/2.0/triage-guidelines) (2.0). The page says defaults "can be overridden by profile, platform, and product configurations".

| Cause | Default timeout |
|---|---|
| Not returning from an LCM transition callback (e.g. onForeground); apps must return "immediately" | 8 s |
| Interactive app not creating a surface while in the foreground | 12 s |
| Not responding to input in time | set by GWSI |
| App main thread blocked or deadlocked (Thread Monitor; enabled per device) | 45 s |
| **JS thread blocked or deadlocked** | "by VS (8 seconds)" |
| Task Manager task | 10 min |

- The ACR fields to read are `LCM_ANR_REASON` (`Lifecycle Callback`, `Display Input Event`, `Thread Monitor`, `Task Execution Limits Exceeded`), plus `LCM_APP_TRANSITION` and `LCM_ANR_THREAD_NAME`. [source](https://developer.amazon.com/docs/vega/2.0/triage-guidelines) (2.0)
- For ANRs on the JS thread, the ACR shows `CRASH_LANG: Unknown`. [source](https://developer.amazon.com/docs/vega/2.0/acr-summary-report) (2.0)
- **2.0 known issue:** calling `MediaSource.removeSourceBuffer()` while `appendBuffer()` is still running blocks the JS thread and causes an ANR. Check `sourceBuffer.updating === false` or wait for `updateend` first. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)

---

## 8. Logs and crash reports: getting them off the device

### loggingctl (inside the device shell)

Source: [Use Loggingctl to Manage System Logs](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0).

```bash
vega exec vda shell                       # enter device shell, then:
loggingctl log -v com.example.myapp -f                 # follow logs for your package (recommended filter)
loggingctl log -v com.example.myapp -p err -f          # errors only (console.error)
loggingctl log -v com.example.myapp -p warning -S today
loggingctl log -v com.example.myapp -S 16:00 -U 16:05 -o short_precise
# console.log -> info, console.warn -> warning, console.error -> err
```

- `loggingctl` only sees apps installed **before** its session started. If your app was installed after, restart loggingctl, reinstall with `vega exec vda shell vpm install <vpkg>`, and launch with `vmsgr send`. [source](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)
- `loggingctl log` shows the current boot only. Older logs: use `--boot` on the VVD, or `loggingctl vault` on a Fire TV Stick. [source](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)
- **Log rate limit.**
  - The default budget is 300 lines/s per process, tracked per tag and level. Lines over budget are **silently dropped**.
  - The suppression window grows (1, 2, 4, 16 ... up to 256 s). Look for `lc_ProcessConfigManager ... suppressed` / `... config_monitor ... suppressed` warnings.
  - Default level is `info` on release builds and `debug` on debug builds.

```bash
loggingctl config --set-rate com.example.myapp 1000
loggingctl config --set-rate com.example.myapp 1000 -t com.example.myapp.trace=2000
loggingctl config --set-level com.example.myapp warning
loggingctl config --reset com.example.myapp        # valid rates 1..60000
```

  [source](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)
- **Crash reports (ACR).** Files live under `/var/lib/data/ace/dropbox/vault/SYSTEM_TOMBSTONE/acr/`. The file name encodes timestamp, UID/GID, PID, signal (6 = SIGABRT, 11 = SIGSEGV, 3 = SIGQUIT, 4 = SIGILL) and package.

```bash
loggingctl vault --ls SYSTEM_TOMBSTONE/acr
loggingctl vault --json-dump SYSTEM_TOMBSTONE/acr
loggingctl vault --last-index SYSTEM_TOMBSTONE
loggingctl vault --size SYSTEM_TOMBSTONE <index>
loggingctl vault --cat SYSTEM_TOMBSTONE <index> > crash_report.acr
```

  [source](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)

### Vega Studio (VS Code)

- Live logs: the **Show Logs** status-bar button streams to the "App Logs" channel. `console.log` output goes to the Output > **React Native** channel. [source](https://developer.amazon.com/docs/vega/2.0/logs) (2.0)
- Copy logs: right-click the device in **Device Files** > **Copy logs**, or run **Vega Device Management: copy logs from device**. [source](https://developer.amazon.com/docs/vega/2.0/logs) (2.0)
- ACRs:
  - Get them from Device Files > Crash Reports, or with **Vega Device Management: Copy ACR from device** (by package ID or PID).
  - Symbolicate with **Vega Device Management: Symbolicate ACR**. Automatic mode works if you built with `react-native build-vega`; manual mode needs `index.bundle` and `*.bundle.map`, or a debug rootfs for native code.
  - Each symbolication generates an ACR summary markdown.

  [crash-report](https://developer.amazon.com/docs/vega/2.0/crash-report), [acr-summary-report](https://developer.amazon.com/docs/vega/2.0/acr-summary-report) (2.0)
- **Your crash or Amazon's?** In JS traces, frames containing `@amazon-devices` are Amazon code. In native traces, frames with no workspace path, or `??`, are OS code. Send Amazon support the **original ACR**, not the summary. [source](https://developer.amazon.com/docs/vega/2.0/detect-crash) (2.0)
- `vega device list` checks device connectivity. [source](https://developer.amazon.com/docs/vega/2.0/acr-issues) (2.0)
- On-device Developer Tools menu, with no host connection needed: hold D-pad Center for 1 s, add Down, and hold both for 3–4 s. Or run `vega device launch-app --appName com.amazon.ssm.main`. [source](https://developer.amazon.com/docs/vega/2.0/system-developer-tools) (2.0)
- Third-party crash reporting: Sentry is listed as supported, via a community KB article. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)

---

## 9. Stability: network, errors, compatibility

- **Network status** (Vega port of NetInfo):
  - Package: `@amazon-devices/keplerscript-netmgr-lib` (RN 0.83 ~2.0.20, RN 0.72 ~2.0.0).
  - Manifest: `[[wants.service]] id = "com.amazon.network.service"` and `[[wants.privilege]] id = "com.amazon.network.privilege.net-info"`. This privilege is granted by the user at runtime; check it with `checkPrivilege` / `requestPrivilege`.
  - Methods: `fetch()`, `refresh()`, `addEventListener()`, `useNetInfo()`.

  [source](https://developer.amazon.com/docs/vega-api/2.0/react-native-net-info) (2.0)
- **No video decoder means no error event (2.0).** If the pipeline can't create a video decoder, no `error` event reaches JS and playback "appears to load indefinitely". Check codecs first with `KeplerMediaDescriptor.queryMediaCapabilities()`. Example: HEVC is unavailable on Fire TV Stick HD (2026), so fall back to H.264. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **RN 0.83 crash at launch** (`SIGABRT`/`SIGSEGV`, log line `Failed to get system bundle path for: index.amzn__react-native-kepler-2.hermes.bundle`): a dependency pulls in the RN 0.72 runtime, most often `@amazon-devices/kepler-ui-components` 2.x. Upgrade it to >= 3.0.0. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **Compatibility gate (2.0).**
  - `vega project doctor` runs compatibility lint: `no-unguarded-os-api`, `no-removed-api`, `warn-deprecated-api`.
  - It exits non-zero on failure, so it works as a CI gate.
  - Guard newer APIs with `isPresentOnOS('<lib>', '<libVersion>')` from `@amazon-devices/kepler-compatibility`.
  - Set `[os.version]` min/target in `manifest.toml`.

  [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- **JS exceptions** are written to the journald logs. Correlate them with the repro timestamp. [source](https://developer.amazon.com/docs/vega/2.0/triage-guidelines) (2.0)
- **Error boundaries.** No Vega doc page covers React error boundaries. See the open questions.

---

## 10. Appstore performance checklist

- The Vega submission page has no performance checklist. Its requirements are a VPKG, the `com.amazon.category.main` category, and increasing build numbers. [source](https://developer.amazon.com/docs/vega/2.0/app-submission) (2.0, last modified 2026-11-16)
- The shared Appstore [Test Criteria](https://developer.amazon.com/docs/app-testing/test-criteria.html) page (last modified 2026-11-16) is served under the Vega doc nav. Its Fire TV group 2a items:
  - **2.5 Overall Performance:** no drop below 25 fps for an extended period; 55–60 fps average recommended; no force close, crash or hard lock.
  - **2.6:** average eMMC writes < 50 MB/hr in the foreground and < 2 MB/hr in the background. Don't buffer audio or video to eMMC.
  - **2.7 Stability:** no FPS loss, force close, crash, hard lock or error messages, including under conflicting inputs.
  - **Group 2c (KPIs)** points to Fire OS performance scripts.

  The page does not say whether these apply to Vega in the same way.
- The KPI Visualizer's `--certification` mode (30 iterations, P90) is the closest thing to a Vega pre-submission performance gate in the docs. [source](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis) (2.0)

---

## Unverified / open questions

- **Are the KPI guidelines Appstore pass/fail gates?** The Vega docs call them "guidelines", and none of the pages I read states they are enforced at certification.
- **Memory numbers disagree.**
  - FAQ: hard limit 420 MB foreground and 150 MB background, in "MB".
  - KPI table: < 400 MiB foreground guideline.
  - fluidity-foreground says foreground memory is measured as RSS; measure-app-kpis says PSS.
  - No per-device limits are published.
- **JS heap ceiling.** "200-MB threshold" is the doc's wording. The exact Hermes max heap per device is not stated.
- **JS-thread ANR timeout.** "by VS (8 seconds)" is from the triage page. "VS" is not defined, and the page says timeouts can be overridden per device or product.
- **CLI flag inconsistencies in Amazon's docs:**
  - `--overridden-video-fps` (performance-cli) vs `--expected-video-fps` (kpi-visualizer `--help`).
  - `--iteration` (step text) vs `--iterations` (help).
  - `--ignore-trace-loss` (flag) vs `--ignore-trace-loss <true/false>`.
  - Verify with `vega exec perf kpi-visualizer --help` on the installed SDK.
- **VS Code command name.** investigate-ui-fluidity still says "Kepler: Launch App KPI Visualizer"; other pages say "Vega: ...".
- **Lifecycle API names are mixed** (Kepler* vs Vega*). The RN 0.83 page is titled VegaAppState, but the code uses `useKeplerAppStateManager` and `IKeplerAppStateManager`. Its deprecation notes point to "VegaAppStateManager", which I found on no separate page. The web-app page uses `useAddVegaAppStateListenerCallback` / `useGetCurrentVegaAppStateCallback`. The ESLint rule name also references `useAddVegaAppStateListenerCallback`. I haven't confirmed which exports exist in the current `@amazon-devices/react-native-kepler`.
- **State persistence on kill, and what the user sees** (cold restart vs restored) after LMK or LRU kills: not documented on any page I read.
- **Error boundaries / global JS error handler** (`ErrorUtils.setGlobalHandler`) on Vega: not covered by Vega docs. Standard React behaviour is assumed but not confirmed.
- **Hermes bytecode / bundle precompilation** and inline-requires guidance specific to Vega: not found. Only `react-native-bundle-visualizer` and `lazy` are recommended.
- **Reanimated** performance guidance on Vega (UI-thread worklets vs Animated native driver): only the version support is documented.
- **The community series "Vega App Performance Series, Part 1"** (community.amazondeveloper.com, needs sign-in) was seen only as a search snippet. I did not use it as a source.
- **The FAQ's "Flashlight" FPS support** is "planned", with no date.

## Sources

Vega docs 2.0, the latest version (base `https://developer.amazon.com/docs/vega/2.0/`):
- [Vega SDK Release Notes 2.0](https://developer.amazon.com/docs/vega/2.0/vega-release-notes)
- [Improve App Performance](https://developer.amazon.com/docs/vega/2.0/improve-performance-overview)
- [App Performance Best Practices](https://developer.amazon.com/docs/vega/2.0/best_practices)
- [App Performance FAQ](https://developer.amazon.com/docs/vega/2.0/faq)
- [Measure App KPIs](https://developer.amazon.com/docs/vega/2.0/measure-app-kpis)
- [Measure Fluidity and Foreground Memory](https://developer.amazon.com/docs/vega/2.0/fluidity-foreground)
- [Vega Studio Performance CLI](https://developer.amazon.com/docs/vega/2.0/performance-cli)
- [Monitor CPU Usage](https://developer.amazon.com/docs/vega/2.0/monitor-cpu-usage)
- [Monitor and Record Memory](https://developer.amazon.com/docs/vega/2.0/monitor-record-app-memory)
- [Investigate JavaScript Memory](https://developer.amazon.com/docs/vega/2.0/javascript-crash-memory)
- [Investigate JavaScript Thread Performance](https://developer.amazon.com/docs/vega/2.0/js-thread-perf)
- [Identify UI Rendering Issues](https://developer.amazon.com/docs/vega/2.0/ui-rendering)
- [Investigate UI Fluidity Issues](https://developer.amazon.com/docs/vega/2.0/investigate-ui-fluidity)
- [Concurrent Rendering](https://developer.amazon.com/docs/vega/2.0/concurrent-rendering)
- [Detect Overdraw](https://developer.amazon.com/docs/vega/2.0/detect-overdraw)
- [Vega ESLint Plugin](https://developer.amazon.com/docs/vega/2.0/eslint-plugin)
- [Web App Performance Best Practices](https://developer.amazon.com/docs/vega/2.0/webview-app-performance-best-practices)
- [Flashlight](https://developer.amazon.com/docs/vega/2.0/flashlight)
- [Transition App State between Background and Foreground](https://developer.amazon.com/docs/vega/2.0/foreground-background)
- [Symbolicate App Crash Reports](https://developer.amazon.com/docs/vega/2.0/crash-report)
- [Debug Crashes with ACR Summary](https://developer.amazon.com/docs/vega/2.0/acr-summary-report)
- [Detect Where the App Crash Originates](https://developer.amazon.com/docs/vega/2.0/detect-crash)
- [Fix Crash Analysis Issues](https://developer.amazon.com/docs/vega/2.0/acr-issues)
- [Manage Files, Crash Reports, and Logs](https://developer.amazon.com/docs/vega/2.0/logs)
- [Use Loggingctl](https://developer.amazon.com/docs/vega/2.0/loggingctl)
- [Triage Guidelines](https://developer.amazon.com/docs/vega/2.0/triage-guidelines)
- [System Developer Tools](https://developer.amazon.com/docs/vega/2.0/system-developer-tools)
- [App Submission for Vega](https://developer.amazon.com/docs/vega/2.0/app-submission)

Other API docs:
- [Supported Libraries (vega-api 2.0)](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries)
- [react-native-netinfo (vega-api 2.0)](https://developer.amazon.com/docs/vega-api/2.0/react-native-net-info)
- [Native API Migration Guide (vega-native 2.0)](https://developer.amazon.com/docs/vega-native/2.0/native-api-migration-guide)
- [VegaAppState (RN 0.83)](https://developer.amazon.com/docs/react-native-vega/0.83/keplerappstate)
- [AppState (RN 0.83, deprecated)](https://developer.amazon.com/docs/react-native-vega/0.83/appstate)
- [SplashScreenManager (RN 0.83)](https://developer.amazon.com/docs/react-native-vega/0.83/splashscreenmanager)

Older versions and other pages:
- [Vega SDK Release Notes 0.24](https://developer.amazon.com/docs/vega/0.24/vega-release-notes) (RN 0.83 intro, known issues)
- [Appstore Test Criteria](https://developer.amazon.com/docs/app-testing/test-criteria.html)
- [developer.amazon.com/llms.txt](https://developer.amazon.com/llms.txt) (index; lists 0.21 links only)
