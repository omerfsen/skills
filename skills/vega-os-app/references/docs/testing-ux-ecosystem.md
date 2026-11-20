# Vega OS: testing, TV UX and developer ecosystem

> Fact sheet. Collected 2026-11-20 from Amazon's Vega docs (latest version: 2.0) and the sources listed at the end; every line names its source page and doc version. Vega changes between SDK versions: re-check a fact against the current docs (or `vega <command> --help`) before relying on it, and prefer the docs when they disagree with this file.


Researched 2026-11-20. All doc pages were fetched directly (curl of the rendered page), not taken from search snippets.

## Doc versions: read this first

- **Latest Vega docs are `/docs/vega/2.0/`**. Vega SDK 2.0, Vega OS 2.0 and the Vega Developer Tools are GA ("no longer in open beta"), and OS 2.0 "has rolled out to all Vega-powered Fire TV devices". [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Version history in URL paths: 0.21, 0.22, 0.23, 0.24, then 2.0. `0.25`–`0.28`, `1.0`, `1.2` and `2.1` all return 404 (checked 2026-11-20). The SDK version jumped from 0.24 to 2.0; there is no SDK 1.x doc set. [source](https://developer.amazon.com/docs/vega/2.0/vega-get-started) (2.0)
- Three doc trees, each versioned on its own:
  - `/docs/vega/<sdk>/…`: guides (2.0 latest)
  - `/docs/vega-api/<sdk>/…`: library/API reference (2.0 latest)
  - `/docs/react-native-vega/<rn>/…`: React Native for Vega API pages, keyed by **React Native** version (`0.72` or `0.83`, not by SDK version)
- I diffed every 0.24 page used below against its 2.0 copy. Almost all are identical apart from the version in links. Where 2.0 differs, the 2.0 text is used and the change is called out.
- Search engines and many cross-links still point to 0.21–0.24 and to `react-native-vega/0.72`. A skill should link to `/docs/vega/2.0/` and `/docs/react-native-vega/0.83/`.
- SDK 2.0 pin used in Amazon's CI example: `VEGA_SDK_VERSION=2.0.10945`. [source](https://developer.amazon.com/docs/vega/2.0/continuous-integration) (2.0)

---

## 1. Testing

### 1.1 Test tool landscape

- Amazon lists four test tools: IAP App Tester, Appstore DevTest (IAP), **Appium** (UI automation via the "Appium Vega Driver") and Live App Testing. No Vega-specific unit-test framework appears in this list. [source](https://developer.amazon.com/docs/vega/2.0/vega-test-tools-overview) (2.0)
- "You can use Vega Virtual Device for initial development and testing, but you must test your app on a Fire TV Stick before submitting to the Amazon Appstore." [source](https://developer.amazon.com/docs/vega/2.0/vega-test-tools-overview) (2.0)
- Jest is on Amazon's list of tested pure-JS libraries at version **29.7.0**. Reassure 0.10.1 is listed as well. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)

### 1.2 Unit and component tests (Jest), RN 0.83

Source for this subsection: [Step 6: Testing Updates](https://developer.amazon.com/docs/vega/2.0/rn-migration-testing) (2.0; last modified 2026-11-04, same as 0.24).

- The Vega app template uses the stock `react-native` Jest preset. No Vega-specific preset is mentioned. `@amazon-devices/react-native-kepler` must be in `transformIgnorePatterns`, because some 4.0.x versions (e.g. 4.0.1) ship ESM in `jest/setup.js`. Without it Jest fails with `SyntaxError: Cannot use import statement outside a module`.

```js
// jest.config.js (as given by Amazon)
module.exports = {
  preset: 'react-native',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: {
    '^@amazon-devices/react-native-kepler$': '<rootDir>/test/__mocks__/react-native-kepler.js',
    '^~/(.*)$': '<rootDir>/src/$1',
  },
  transformIgnorePatterns: [
    'node_modules/(?!(react-native|@react-native|@amazon-devices/react-native-kepler|@amazon-devices)/)',
  ],
};
```

- Mock `@amazon-devices/react-native-kepler`. Otherwise `TVFocusGuideView` loads the native module and tests fail with `Invariant Violation: __fbBatchedBridgeConfig is not set`:

```js
// test/__mocks__/react-native-kepler.js
const React = require('react');
const {View} = require('react-native');
module.exports = {
  TVFocusGuideView: React.forwardRef((props, ref) =>
    React.createElement(View, {...props, ref}),
  ),
};
```

- TV mocks in `jest.setup.js`: Vega TV devices "render a 1920x1080 canvas at scale 2, so the window is 960x540 in layout units (dp)", and `PixelRatio.get()` returns 2. The page also gives a full `NativeAnimatedModule` mock with 23 methods, needed for any test that triggers navigation animations.

```js
jest.mock('react-native/Libraries/Utilities/Dimensions', () => {
  const window = { width: 960, height: 540, scale: 2, fontScale: 1 };
  const impl = {
    get: jest.fn(() => window),
    set: jest.fn(),
    addEventListener: jest.fn(() => ({ remove: jest.fn() })),
  };
  return { __esModule: true, default: impl, ...impl };
});
```

- RN 0.83 moved many internals to `export default`. Mocks must return `{ __esModule: true, default: impl, ...impl }`, or tests fail with `Cannot read properties of undefined (reading 'create')`. Affected modules: StyleSheet, flattenStyle, NativeEventEmitter, Platform, Dimensions, PixelRatio, useWindowDimensions, BatchedBridge/NativeModules.
- React 19 requires `act()`. With `react-test-renderer` use `await ReactTestRenderer.act(() => { ReactTestRenderer.create(<App />); })`. With `@testing-library/react-native`, do **not** wrap `render()` in `act()`, because that causes "Can't access .root on unmounted test renderer".
- `react-navigation__stack` timers leak after teardown ("You are trying to import a file after the Jest environment has been torn down"). Fix with `jest.useFakeTimers()` in `beforeEach` and `jest.runOnlyPendingTimers(); jest.useRealTimers();` in `afterEach`.
- Regenerate snapshots after migrating with `npm test -- -u`.
- SDK 2.0 breaking change: RN 0.83 Vega packages (core, Gesture Handler, Lottie, React Navigation, Reanimated, SVG, Screens) expose only their public JS interface, so deep imports no longer resolve. "If any of your Jest tests depend on a library's internal implementation, mock those modules in your test code instead." [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)

### 1.3 UI test automation: Appium Vega Driver

- Install (Node 16/18/20/22 supported by the driver; SDK 2.0 itself requires Node **22.14.0+**). [source](https://developer.amazon.com/docs/vega/2.0/appium-install) (2.0)

```bash
npm install -g appium@2.2.2
appium driver install --source=npm @amazon-devices/appium-kepler-driver@3.30.0
appium driver list
# vda must be on PATH for Appium:
export PATH="$(dirname "$(vega exec which vda)"):$PATH"
```

- Setup: start `appium` (default port 4723; check with `curl http://localhost:4723/status`), then turn on the on-device automation toolkit. [source](https://developer.amazon.com/docs/vega/2.0/appium-setup) (2.0)

```bash
vega exec vda shell "touch /tmp/automation-toolkit.enable"
```

- Required capabilities. The platform name is still **`Kepler`** on 2.0. [source](https://developer.amazon.com/docs/vega/2.0/appium-sessions) (2.0)

```json
{
  "platformName": "Kepler",
  "appium:automationName": "automation-toolkit/JSON-RPC",
  "kepler:device": "vda://default"
}
```
  Use `vda://<DSN>` when more than one device is attached (`vega exec vda devices` lists DSNs). Only the `vda` bridge is supported. Optional `appium:appURL` (e.g. `com.amazon.keplervideoapp.main`) picks the app to launch.
- Locators: only **UiSelector** and **XPath** are supported. Example: `driver.find_element("UiSelector", '{"args": {"text": "Element Text"}}')` or `'{"args": {"test_id": "my_button"}}'`. The Inspector page recommends `test_id`, `role` or `description` over the dynamic `id`. [source](https://developer.amazon.com/docs/vega/2.0/appium-locator-strategies) (2.0), [source](https://developer.amazon.com/docs/vega/2.0/appium-inspector) (2.0)
- Supported commands: find_element(s), click, send_keys, get_attribute (`UiObject:focused`, `UiObject:test-id`, etc.), text, press_keycode, tap, is_enabled, is_displayed, implicitly_wait, get_window_rect, page_source, get_screenshot, install_app (`.vpkg`), activate_app, terminate_app, remove_app, execute_script, push_file, pull_file. [source](https://developer.amazon.com/docs/vega/2.0/appium-commands) (2.0)
- D-pad from Appium uses Linux input event codes through JSON-RPC: KEY_LEFT 105, KEY_RIGHT 106, KEY_UP 103, KEY_DOWN 108, KEY_PENTER (select) 96, KEY_BACK 158, KEY_HOMEPAGE 170. [source](https://developer.amazon.com/docs/vega/2.0/appium-commands) (2.0)

```python
driver.execute_script("jsonrpc: injectInputKeyEvent", [{"inputKeyEvent": "108", "holdDuration": 1000}])  # down
driver.execute_script("shell", "echo hello world")   # run a shell command on device
```

- Known limitation: `click()` can land at the wrong coordinates when the device feeds a capture card instead of a real TV. The workaround is D-pad (`setFocus` + `pressKeyCode` with `DPAD_CENTER`). [source](https://developer.amazon.com/docs/vega/2.0/appium-commands) (2.0)
- Appium Inspector works (desktop app, same capabilities JSON). VVD's "Dump Render Tree" context-menu item gives the same XML as Inspector's App Source. [source](https://developer.amazon.com/docs/vega/2.0/appium-inspector) (2.0), [source](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- Resolved in SDK/OS 2.0: on some OS 1.2 builds the automation toolkit service didn't start (port 8383 listened but nothing answered), so `findElement`/`takeScreenshot`/`getPageSource` failed with `socket hang up`/`ECONNRESET`. It now starts in Developer Mode. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Troubleshooting for flaky locators and waits: [source](https://developer.amazon.com/docs/vega/2.0/test-automation-issues) (2.0)
- Doc errata: in 0.24 the Python examples on appium-write-tests and appium-sessions had broken indentation, and the `find_elements` example used an unterminated string. Both are fixed in the 2.0 copies. Use 2.0.

### 1.4 Remote-control simulation from the CLI: `inputd-cli`

Source: [Simulate Device Input with inputd-cli](https://developer.amazon.com/docs/vega/2.0/inputdcli) (2.0, last modified 2026-11-04).

- Built into the device. Input automation needs a **Dev mode shell**; the `app_user` context can only read information.
- On VVD, enable developer mode first. Without it only `get_camera_cover_open_close`, `get_mic_state` and `get_screen_size` are available. The VVD reboots after this:

```bash
vega device shell -d VirtualDevice
vsm developer-mode enable
```
- 2.0 changed the access command. 0.24 said `vega exec inputd-cli --help`; 2.0 says:

```bash
vega exec vda shell inputd-cli --help
vega exec inputd-cli button_press KEY_BACK          # form shown on the same 2.0 page
vega exec inputd-cli send_text 'hello\ world\!'
```
- Usage: `button_press <key> ([short|long|superLong|repeat]|--holdDuration <ms>)`, `touch`, `swipe`, `gesture_swipe_{up,down,left,right}`, `mouse_move`, `mouse_click`, `send_text`, `get_screen_size`, `set_mic_state`. Key names come from Linux `input-event-codes.h` (KEY_UP/DOWN/LEFT/RIGHT/ENTER/BACK/MENU…).
- Console mode for scripted sequences:

```bash
vega exec "{ echo -e "button_hold KEY_LEFT"; sleep 0.5; for i in {1..30}; do echo -e "button_press KEY_DOWN"; sleep 0.5; done; echo -e "button_release KEY_LEFT"; echo exit; } | inputd-cli start"
```

### 1.5 Screenshots, render-tree dumps, logs

- Screenshot (PNG) from the CLI. [source](https://developer.amazon.com/docs/vega/2.0/vda-tools) (2.0)

```bash
vega exec vda shell gwsi-tool-screenshooter /tmp/test.png
vega exec vda pull /tmp/test.png
```
- Appium `get_screenshot_as_base64()` also works. Do not capture DRM content. [source](https://developer.amazon.com/docs/vega/2.0/appium-commands) (2.0)
- VVD context menu has "Dump Render Tree" (XML) and "Dump Rendered Text" (TXT). VVD flags: `vega virtual-device start --no-gui`, `--no-gl-accel` (Ubuntu x86 only), `--display-res=1920,1080`. [source](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- **Screen recording:** none of the Vega pages I read document a CLI screen-recording tool (see open questions).
- Logs on the device use `loggingctl`, reached through `vega exec vda shell`. [source](https://developer.amazon.com/docs/vega/2.0/loggingctl) (2.0)

```bash
vega exec vda shell loggingctl log -o short-precise -f
loggingctl log -v "com.amazondeveloper.yourapp" -f          # filter by package (recommended)
loggingctl log -v "com.amazondeveloper.yourapp" -p err -f   # console.error only
loggingctl vault --ls SYSTEM_TOMBSTONE/acr                  # list crash reports (ACR)
loggingctl vault --cat SYSTEM_TOMBSTONE <index> > crash_report.acr
```
  - `console.log` maps to info, `console.warn` to warning, `console.error` to err.
  - LoggingCtl only sees apps installed **before** the session started. After installing, restart it.
  - Default budget is 300 lines/s per process, per tag+level. Excess lines are **silently dropped**; look for `suppress` warnings. Raise the limit with `loggingctl config --set-rate <pkg> 1000` and reset with `--reset`.
  - 2.0 addition: take the `--cat` index from the `NN#` filename prefix. `--last-index` doesn't always match an entry.
- In Vega Studio: the "Show Logs" status-bar button streams to the "App Logs" output channel, and `console.log` appears in the "React Native" output channel. [source](https://developer.amazon.com/docs/vega/2.0/logs) (2.0)
- RN 0.83 change: "logs no longer appear in the Metro terminal". Use React Native DevTools Console instead. [source](https://developer.amazon.com/docs/vega/2.0/react-devtools) (2.0)

### 1.6 VVD vs device

- VVD is fine for development, but the Fire TV Stick is required before submission. VVD has a virtual remote and mouse and keyboard input. [source](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)
- `(script) Launch - Perf` (release mode with performance monitoring) is "not available on Vega Virtual Device". [source](https://developer.amazon.com/docs/vega/2.0/debugger) (2.0)
- Pre-submission manual test matrix (Appstore runs these): exit with Back and relaunch, exit with Home and relaunch (no audio leak), app switching, HDMI hot-plug, HDMI input/TV power-off, relaunch after reboot (login retained), screensaver (playback stays paused, Back returns to app), Alexa interruption during playback, Bluetooth audio. [source](https://developer.amazon.com/docs/vega/2.0/test-before-submission) (2.0)

### 1.7 CI

Source: [Integrating the Vega SDK into CI Pipelines](https://developer.amazon.com/docs/vega/2.0/continuous-integration) (2.0).

- Supported hosts: Ubuntu 20.04/22.04/24.04/26.04 x86_64, macOS arm64 (recommended) or x86_64.

```bash
export NONINTERACTIVE=true
export SKIP_VVD_INSTALL=true
export VEGA_SDK_VERSION=2.0.10945
curl -fsSL https://sdk-installer.vega.labcollab.net/get_vvm.sh | bash
# installs to /root/vega/bin in the Dockerfile example (ENV PATH="/root/vega/bin:${PATH}")
```
- Vega accessibility/performance lint for CI: `npm install --save-dev @amazon-devices/eslint-plugin-kepler jsonc-eslint-parser`, then extend `plugin:@amazon-devices/eslint-plugin-kepler/kepler`. Sub-configs: `performance`, `system-distributed-libs`, `accessibility`. [source](https://developer.amazon.com/docs/vega/2.0/eslint-plugin) (2.0)
- `vega project doctor` (new in 2.0) runs compatibility linters (`no-unguarded-os-api`, `no-removed-api`, `warn-deprecated-api`). It "exits with a non-zero status when a check fails, so you can use it as a gate in your CI pipeline". [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)

---

## 2. Debugging

- Overview of the options: crash reports (ACR) with symbolication, AI-assisted ACR via Amazon Devices Builder Tools (ADBT), Vega Studio line-by-line debugging, the in-app Dev Menu, logs and LoggingCtl. [source](https://developer.amazon.com/docs/vega/2.0/debug-apps-overview) (2.0)
- **Vega Studio debugger** (VS Code extension, GDB underneath). `launch.json` `type` is `"kepler"` for JS/TS or `"keplerNative"` for C++ Turbo Modules. `deviceId` takes `"${command:kepler.state.getSelectedDeviceId}"`, `"KeplerVirtualDevice"`, or a serial. `disableAppTimeout: true` (the default) keeps ANR timeouts from killing paused apps. Enable dev mode on VVD with `vega exec vda shell vsm developer-mode enable`. [source](https://developer.amazon.com/docs/vega/2.0/debugger) (2.0)

```json
{ "name": "(script) Launch", "type": "kepler", "request": "launch", "keplerMode": "debug",
  "cwd": "${workspaceFolder}", "deviceId": "${command:kepler.state.getSelectedDeviceId}", "disableAppTimeout": true }
```
- **React Native DevTools** is the recommended debugger and profiler for **RN 0.83**. It has Components, Profiler, Network, Performance, Memory, Console and Sources tabs. [source](https://developer.amazon.com/docs/vega/2.0/react-devtools) (2.0)

```bash
vega device start-port-forwarding --port 8081 --forward false   # add -d VirtualDevice or -d <DSN>
npm start                                                        # Metro; press j to open DevTools, d for Dev Menu, r reload
vega run-app build/armv7-debug/keplersample_armv7.vpkg com.amazon.keplersampleapp.main
```
  - Network tab works on Vega: Fetch/XHR, HLS `.ts`/DASH `.m4s`, HTTPS without cert setup, HAR export, override headers/response. Block request, replay, throttling, map-remote and WebSocket frames are **not** functional, so use Charles Proxy for those. It is interactive only, with no CLI/headless use.
  - Known limitations: the on-device element highlight doesn't work (repeated `TypeError: Cannot read property 'style' of null`), and reloading from DevTools can crash the app.
  - In cloud dev environments, add `LocalForward 8081 127.0.0.1:8081` to the SSH config.
- **Chrome DevTools** is for RN 0.72 apps. Start it from Vega Studio ("Vega: Launch Dev Tools"); the bundle shows under `localhost:8081 > index.bundle` with `&platform=kepler&dev=true&minify=false`. [source](https://developer.amazon.com/docs/vega/2.0/chrome-devtools) (2.0)
- **Flipper:** the Dev Menu's "Open Debugger" opens React Native DevTools on RN 0.83, or "the Flipper debugger (React Native 0.72)". Amazon notes that React Native 0.74 dropped Flipper support. [source](https://developer.amazon.com/docs/vega/2.0/app-developer-menu) (2.0), [source](https://developer.amazon.com/docs/vega/2.0/chrome-devtools) (2.0)
- **Dev Menu** (press `d` in Metro): Reload, Open Debugger, Show Element Inspector, Disable Fast Refresh, Configure Bundler, Show Perf Monitor. [source](https://developer.amazon.com/docs/vega/2.0/app-developer-menu) (2.0)
- **Network inspection:**
  - Vega Studio Network Proxy (built on `mitm-http-proxy`): start it from the command palette with "Start Network Proxy", then run `vda reverse tcp:8888 tcp:8888`. It captures HTTP/HTTPS/WebSocket/XHR/Media Player traffic and supports breakpoints. **Before production, delete `proxy-config.json` and `assets/raw/ca.pem`, then run `vega clean`.** [source](https://developer.amazon.com/docs/vega/2.0/network-proxy) (2.0)
  - Charles Proxy is also documented. [source](https://developer.amazon.com/docs/vega/2.0/charles-proxy) (0.24 page; 2.0 copy exists)
- **System Developer Tools menu** on the device (System X-Ray overlay, **Safe Zone** overscan overlay, Network Proxy, Network Advisor). Open it with the remote (hold Center, add Down for 3–4 s, release, press Menu) or with `vega device launch-app --appName com.amazon.ssm.main`. [source](https://developer.amazon.com/docs/vega/2.0/system-developer-tools) (2.0)

---

## 3. TV UX on Vega

### 3.1 Focus management

Main source: [Focus Management](https://developer.amazon.com/docs/vega/2.0/focus-management) (2.0, identical to 0.24).

- Three paradigms, in precedence order (later overrides earlier):
  1. Cartesian (implicit) focus. It uses weighted distance, ignores z-order, and edge overlap affects the result.
  2. Focus props/methods.
  3. Specialized components (`TVFocusGuideView`).
- A View is focusable when `focusable` or `accessible` is true. Touchables, `Button` and `Pressable` are focusable by default. `ScrollView`/`FlatList` use index-based focus and scroll only when the next item is off-screen.
- **No platform focus indicator:** only TouchableOpacity/TouchableHighlight change by default (opacity 0.2, via `activeOpacity`). Style focus yourself with `onFocus`/`onBlur`, and prefer physical changes (border, size) over color/opacity so focus stays visible in High Contrast mode.
- Directional overrides: `nextFocusUp/Down/Left/Right`, set to a `findNodeHandle()` value. Pointing at self blocks that direction; `undefined` restores Cartesian behavior.
- **No built-in focus restoration**, including in `@amazon-devices/react-navigation__stack` and `__native-stack`. Store the last-focused ref per screen and call `FocusManager.setFocus()` inside `useFocusEffect()`. `hasTVPreferredFocus` works on initial mount only and is unreliable across screens.
- Hidden or masked containers must set `pointerEvents: 'none'` and `accessibilityElementsHidden: true`. `display: 'none'` also disables focus finding. React Navigation handles this between screens.
- Synchronous focus events: the `enableSynchronousFocusEventsVega` prop is Vega-only. It fixes focus resetting to the first child of a `TVFocusGuideView` during rapid key presses. Rules: don't call `focus()`/`blur()`/`requestTVFocus()` inside handlers, don't change the prop after the first render, and don't mix values under one parent.
- Namespace: import from `react-native` for portability. Vega-only APIs (`FocusManager`, `TVFocusGuideView`, `useTVEventHandler`) come from `@amazon-devices/react-native-kepler`. `TVFocusGuideView` "is a port from `react-native-tvos`". Vega ports TV components into the react-native-kepler namespace instead of using the react-native-tvos fork.

**FocusManager** (`import { FocusManager } from "@amazon-devices/react-native-kepler"`), based on You.i TV's module. [source](https://developer.amazon.com/docs/react-native-vega/0.83/focus-manager) (RN 0.83 page)

```ts
getFocused: () => number | undefined;
setNextFocus: (fromTag, toTag, direction: 'up'|'down'|'left'|'right'|'all') => void;
clearNextFocus: (fromTag, direction = 'all') => void;
setFocusRoot: (tag, isFocusRoot: boolean) => void;   // trap focus inside a subtree
focus: (tag) => void;
blur: (tag) => void;
```
- Known issues: calls from `useEffect` can fail on first mount (add a timeout), and passing refs causes TS2345, so pass `findNodeHandle(ref.current)`.
- Note: the focus-management guide recommends `FocusManager.setFocus()`, but the FocusManager API page documents `focus()` and no `setFocus()`. See open questions.

**TVFocusGuideView** (`@amazon-devices/react-native-kepler`). Props: `destinations` (array of refs), `autoFocus` (focus the first child on the first visit and remember the last child on later visits; `destinations` wins when both are set), `trapFocusUp/Down/Left/Right`, `enabled`. Method: `setDestinations([...])`. Known issues: `setDestinations` fails if the component isn't mounted yet, and strict TS needs `{/* @ts-expect-error */}` above it. [source](https://developer.amazon.com/docs/vega-api/2.0/tvfocusguideview) (2.0)

### 3.2 Remote keys: TVEventHandler / useTVEventHandler / BackHandler

- `import { TVEventHandler, useTVEventHandler } from '@amazon-devices/react-native-kepler'`. A listener can **not** override default behavior; D-pad still moves focus. Events look like `{eventType:'left', eventKeyAction:0}`, where 0 is pressed and 1 is released. While a key is held, repeated `0` events arrive, followed by one `1`; use this for long-press detection. [source](https://developer.amazon.com/docs/react-native-vega/0.83/tvEventHandler) (RN 0.83)
- Fire TV remote key names: `up, down, left, right, select, back, menu, playpause, skip_backward, skip_forward, page_up, page_down, page_left, page_right, info, more`. Universal or TV remotes add `num_0..9`, `key_pad_*`, `enter`, `space`, `backspace`, `escape`, `home`, `delete` and more. [source](https://developer.amazon.com/docs/react-native-vega/0.83/tvEventHandler) (RN 0.83)
- `HWEvent` type: `eventType`, `eventKeyAction`, and optional `deviceIdentifier` (needs privilege `com.amazon.input.device.id.access`). [source](https://developer.amazon.com/docs/react-native-vega/0.83/HWEvent) (RN 0.83)
- `UserInputManager` / `useAddUserInputListenerCallback` is **deprecated**. Registering a listener replaces all existing ones, including platform behavior. Use TVEventHandler instead. [source](https://developer.amazon.com/docs/react-native-vega/0.83/userinputmanager) (RN 0.83)
- Back: `BackHandler.addEventListener('hardwareBackPress', fn)` ("Android and Vega-only"). Return `true` to consume the event. If nothing returns `true`, the default back action exits. `BackHandler.exitApp()` ends only the current interactive component when several exist. [source](https://developer.amazon.com/docs/react-native-vega/0.83/backhandler) (RN 0.83)
- Home: on Fire TV, Home "is a system event and cannot be intercepted". The microphone button is also system-only. Volume, power and app buttons can't be mapped in third-party apps. [source](https://developer.amazon.com/docs/fire-tv/remote-input) (Fire TV docs, unversioned, Fire OS-oriented)
- Upcoming TextInput change: single-line RN 0.83 `TextInput` without `submitBehavior` will use `blurAndSubmit`. Back will fire `onSubmitEditing` with `submitAction: 'previous'`, so filter on that. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)

### 3.3 10-foot design, safe zone, resolution

- Fire TV Design and UX Guidelines: 10-ft UI, low information density, important content first, design for 1080p (1920x1080 px = 960x540 dp), and **keep UI out of the outer 5% of each edge** (safe zone = inner 90%). Body text at least 14sp (~28 px at 1080p), less saturated and cooler colors, every actionable element reachable by D-pad, clear focus state. [source](https://developer.amazon.com/docs/fire-tv/design-and-user-experience-guidelines) (Fire TV docs, last modified 2020-10-29, written for Fire OS; no Vega-specific design guide found)
- Vega matches this: the Jest guidance says devices render 1920x1080 at scale 2 (960x540 dp). [source](https://developer.amazon.com/docs/vega/2.0/rn-migration-testing) (2.0)
- On-device overscan check: System Developer Tools > Safe Zone. [source](https://developer.amazon.com/docs/vega/2.0/system-developer-tools) (2.0)
- 4K-capable devices can be attached to HD TVs. Detect display capability with W3C `decodingInfo()` (polyfill) or the Vega Graphics API. [source](https://developer.amazon.com/docs/vega/2.0/detect-screen-resolution-vega) (2.0)
- Fire TV UI patterns (home rows, 1D lists that wrap at the end, detail view, search) are in the same guideline page.

### 3.4 Navigation library

- React Navigation is "the only 3P community package available on Vega platform for navigation between multiple screens". `wix/react-native-navigation` is **not** supported. [source](https://developer.amazon.com/docs/react-native-vega/0.83/navigation) (RN 0.83)
- Version mapping: on RN 0.83, `@amazon-devices/react-navigation__*` **~8.0.0** is ported from upstream **7.13.3** (core). On RN 0.72 it is `^7.0.0`, ported from upstream 6.x. It is a system-deployed (autolinked) library. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0), [source](https://developer.amazon.com/docs/vega-api/2.0/react-navigation-8-0-0) (2.0)

```json
"@amazon-devices/react-navigation__native": "~8.0.0",
"@amazon-devices/react-navigation__stack": "~8.0.0",
"@amazon-devices/react-navigation__native-stack": "~8.0.0",
"@amazon-devices/react-native-screens": "~3.0.0"
```
  Available sub-packages on RN 0.83: bottom-tabs, core, devtools, drawer, elements, material-top-tabs, native, native-stack, react-native-tab-view, routers, stack. material-bottom-tabs is RN 0.72 only. Imports look like `import {NavigationContainer} from '@amazon-devices/react-navigation__native'`.
- On RN 0.83, `@react-navigation/*` ports do **not** support the module-resolver preset, so import `@amazon-devices/react-navigation__*` directly. [source](https://developer.amazon.com/docs/vega/2.0/rn-migration-configuration) (2.0)
- `flipper-plugin-react-navigation` is not supported. [source](https://developer.amazon.com/docs/vega-api/2.0/react-navigation-8-0-0) (2.0)

### 3.5 Keyboard and text input

- The on-screen keyboard must be declared in `manifest.toml`. [source](https://developer.amazon.com/docs/vega/2.0/enable-keyboard) (0.24 page; identical 2.0 copy)

```toml
[wants]
[[wants.service]]
id = "com.amazon.inputmethod.service"
```
- The `Keyboard` module supports `addListener` (keyboardWillShow/DidShow/WillHide/DidHide/…), `dismiss()`, `isVisible()` and `metrics()`. [source](https://developer.amazon.com/docs/react-native-vega/0.83/keyboard) (RN 0.83)
- Troubleshooting page: [On-Screen Keyboard Issues](https://developer.amazon.com/docs/vega/2.0/keyboard-issues) (2.0, linked from troubleshoot overview; not read in detail)
- Inject text in tests with `inputd-cli send_text` or Appium `send_keys` (focus the field first and add short delays between key sends).
- Voice on-screen navigation (UIController): props `aria-label`, `ariaPosInSet`, `keplerAltLabels`, `alexaEntityType`, `alexaExternalID`. Requires `categories = ["com.amazon.category.kva"]` in the manifest components section. After sideloading, a reboot is needed to enable voice features. [source](https://developer.amazon.com/docs/vega/2.0/using-on-screen-nav) (2.0)

---

## 4. Accessibility

- Assistive tech on Vega Fire TV: **VoiceView** screen reader, **Screen Magnifier**, **Text Banner**. [source](https://developer.amazon.com/docs/react-native-vega/0.83/accessibility) (RN 0.83)
- Turning on VoiceView:
  - On the device: hold **Back + Menu** for 2 s. [source](https://developer.amazon.com/docs/vega/2.0/assistive-technologies-vega) (2.0)
  - The RN page says 3 s and gives an `inputd-cli` script: `{ echo -e button_hold KEY_BACK; echo -e button_hold KEY_MENU; sleep 3; echo -e button_release KEY_BACK; echo -e button_release KEY_MENU; echo exit; } | inputd-cli start` (run inside `vega exec vda shell`). [source](https://developer.amazon.com/docs/react-native-vega/0.83/accessibility) (RN 0.83)
  - Known issue in 2.0: `vdcm set "com.amazon.devconf/system/accessibility/VoiceViewEnabled" "ENABLED"` fails in the Developer Mode shell, so use device settings. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0)
- Screen Magnifier: hold FASTFORWARD + BACK. Text Banner: hold FASTFORWARD + REWIND. Both use the same `inputd-cli` pattern. [source](https://developer.amazon.com/docs/react-native-vega/0.83/accessibility) (RN 0.83)
- Props: use ARIA-style props (`aria-label`, `role`, `aria-checked/disabled/expanded/selected/hidden/live/labelledby`, `aria-value*`). `accessibilityLabel`, `accessibilityRole`, `accessibilityState` and the like are slated for removal. iOS-only a11y props are not supported on Vega.
- Vega-specific props: `accessibilityOrientationText` (describes screen layout for blind users), `accessibilityDescribedBy` (list of `nativeID`s spoken with the focused item, because screen-reader focus follows input focus), and `transportControls`. Roles `heading` and `summary` make VoiceView announce new windows. `announceForAccessibility` exists but "will be removed".
- On Vega, `accessible={true}` on View/Image becomes native `focusable={true}`. Amazon says this "might be de-linked … in a future release", so don't rely on it for focusability. [source](https://developer.amazon.com/docs/react-native-vega/0.83/accessibility) (RN 0.83)
- **A11y Inspector** in Vega Studio 0.24.2+ is on by default and cannot be turned off. It flags issues as you edit. Commands: "A11y Inspector: Run Lint", "A11y Inspector: Fix All". Rules (same as the ESLint plugin): `missing-accessible-label`, `missing-accessibility-state`, `invalid-role-value`, `dangling-described-by-ref`, `dangling-transport-control-ref` (Blocker); `deprecated-prop-rename` (Critical); `missing-accessibility-role`, `missing-accessibility-hint` (Major). Suppress with `{/* a11y-ignore-next-line rule-name */}` or `eslint-disable-next-line @amazon-devices/kepler/<rule>`. [source](https://developer.amazon.com/docs/vega/2.0/vega-studio-accessibility-inspector) (2.0)

---

## 5. Localization and i18n

- Hermes on RN for Vega supports a **subset** of ECMA-402 Intl: Collator, DurationFormat, DateTimeFormat, RelativeTimeFormat, NumberFormat, Locale (methods listed per API). [source](https://developer.amazon.com/docs/react-native-vega/0.83/internationalization-and-localization) (RN 0.83)
- Vega `I18nManager` extensions: `getAppLocale()`, `setAppLocale(bcp47)`, `getSystemLocale()`, `resetLocale()`, `getTimezone()`/`setTimezone()`, and `addListener(I18nManager.SettingEventName.Locale|Timezone, cb)`. RTL is handled through `I18nManager.isRTL`. [source](https://developer.amazon.com/docs/react-native-vega/0.83/internationalization-and-localization) (RN 0.83), [source](https://developer.amazon.com/docs/react-native-vega/0.83/i18nmanager) (RN 0.83)
- String resources use **PUFF-J**, a JSON format with the `.puff.json` extension. Resource IDs match `^(\w[\w-:]*)$`. Values are ICU MessageFormat strings, or structured `plural` (`param` + `pluralItems` with zero/one/two/few/many/`=N`/`other`, where `other` is required) and `select` types. Optional `note` (max 3,000 chars) and `translate:false`. Files are converted at build time. [source](https://developer.amazon.com/docs/vega/2.0/internationalization-puff-j) (0.24 page; 2.0 copy exists)
- Libraries: `react-native-localize` (system-distributed, `~2.1.x` on RN 0.83) and `expo-localization` `~3.0.0`. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)
- VVD "Dump Rendered Text" is suggested for checking localized strings. [source](https://developer.amazon.com/docs/vega/2.0/run-apps) (2.0)

---

## 6. Multi-platform: sharing code with Fire OS, Android TV and tvOS

- On Vega, `Platform.OS === 'kepler'`. `Platform.select({ kepler: … })` works, and the file extension is **`.kepler.tsx/.ts/.jsx/.js`** (not `.vega.*`, which breaks tree shaking). [source](https://developer.amazon.com/docs/vega/2.0/platform-specific-code) (2.0)
- TS error `TS2367` on `Platform.OS === 'kepler'` means the community `@types/react-native` is resolved instead of the SDK types. Remove it (`npm ls @types/react-native`).
- **Module resolver preset.** The real package is **`@amazon-devices/kepler-module-resolver-preset`** (npm, 0.1.17, modified 2026-11-19). The docs call it the Vega Module Resolver Preset (VMRP). It rewrites `react-native-x` imports to `@amazon-devices/react-native-x` at bundle time because "Vega implements split bundling by default" and npm aliases or overrides don't work. [source](https://developer.amazon.com/docs/vega-api/2.0/vega-module-resolver-preset) (2.0)

```bash
npm install --save-dev @amazon-devices/kepler-module-resolver-preset babel-plugin-module-resolver
```
```js
// babel.config.js (doc example; RN 0.72 preset name)
presets: ['module:metro-react-native-babel-preset', 'module:@amazon-devices/kepler-module-resolver-preset']
```
  - 2.0 change: "The Vega app template for React Native 0.83 doesn't include VMRP". On RN 0.83 the Gesture Handler, Reanimated, Worklets, Screens, SVG and `@react-navigation/*` ports **don't support** the preset, so import `@amazon-devices/*` directly. Only a few ports (e.g. device-info, async-storage) still alias. [source](https://developer.amazon.com/docs/vega/2.0/rn-migration-configuration) (2.0)
  - The package name `@vega-tv/react-native-module-resolver-preset` named in some community material **does not exist** on npm (404 on 2026-11-20).
- **react-native-tvos relationship:** Vega is an out-of-tree platform like react-native-macos. It doesn't use react-native-tvos, but ports the TV components (`TVFocusGuideView`) into `@amazon-devices/react-native-kepler`. [source](https://developer.amazon.com/docs/vega/2.0/focus-management) (2.0)
- **Multi-TV sample:** [AmazonAppDev/react-native-multi-tv-app-sample](https://github.com/AmazonAppDev/react-native-multi-tv-app-sample) is a Yarn-workspaces monorepo with `apps/expo-multi-tv` (Expo + react-native-tvos for Android TV, Apple TV, Fire OS and Web), `apps/vega` (Vega SDK) and `packages/shared-ui` (`@multi-tv/shared-ui`). It uses `react-tv-space-navigation` for spatial navigation and `.kepler.ts` platform files. Branch `main` targets Vega OS/SDK 2.0 with RN 0.83; `release-rn-83` targets OS 1.2. (README, read via GitHub API 2026-11-20)
- Vega Studio supports npm and Yarn workspaces (Yarn v2 preferred; v1 works except `nohoist`) under Settings > Vega > Features: Monorepo. [source](https://developer.amazon.com/docs/vega/2.0/monorepo-support) (2.0)
- Libraries: RN Directory has a Vega filter at https://reactnative.directory/packages?vegaos=true. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (2.0)

---

## 7. Ecosystem

### 7.1 Official sample apps (all under github.com/AmazonAppDev, verified via `gh api`, all active in Nov 2026)

Listed on [Sample Apps](https://developer.amazon.com/docs/vega/2.0/sample-app-home) (2.0; the list is rendered client-side and was read from the page's config).

| Repo | What it shows |
|---|---|
| https://github.com/AmazonAppDev/vega-video-sample | W3C media player, search, focus, IAP, content launcher, account login, Live TV |
| https://github.com/AmazonAppDev/vega-tv-interfaces-sample | Recommended TV interface patterns: focus, navigation, i18n, scrolling (linked from Focus Management) |
| https://github.com/AmazonAppDev/vega-sports-app | Sports app, theming, content catalog, VUIC |
| https://github.com/AmazonAppDev/vega-audio-sample | Music player |
| https://github.com/AmazonAppDev/vega-scrolling-sample | Home-grid layout and scrolling optimization |
| https://github.com/AmazonAppDev/vega-seekbar-sample | TV seek bar, focus, VUIC |
| https://github.com/AmazonAppDev/vega-epg-sample | EPG (TV guide) components |
| https://github.com/AmazonAppDev/react-native-multi-tv-app-sample | Cross-platform TV monorepo (see section 6) |
| https://github.com/AmazonAppDev/vega-native-component-sample | C++ native UI components inside RN, including D-pad focus and VoiceView (new in 2.0) |
| https://github.com/AmazonAppDev/vega-periodic-task-sample | C++ Turbo Module around Task Manager for periodic headless JS tasks (new in 2.0) |

- The last two repos appear only in the 2.0 sample list, not in 0.24. Both repos exist but have no description set (checked 2026-11-20).

### 7.2 Forum, release notes, known issues, troubleshooting

- Forum: https://community.amazondeveloper.com/c/vega/6 (HTTP 200; linked from the accessibility and Appium Inspector pages). [source](https://developer.amazon.com/docs/vega/2.0/accessibility-vega) (2.0)
- Release notes: https://developer.amazon.com/docs/vega/2.0/vega-release-notes. **Known issues are a section of the release notes** (I found no separate known-issues page). 2.0 known issues relevant here:
  - RN 0.83 apps crash at launch (`SIGABRT`/`SIGSEGV`, `Failed to get system bundle path for: index.amzn__react-native-kepler-2.hermes.bundle`) when a dependency pulls in the RN 0.72 runtime. The usual cause is `@amazon-devices/kepler-ui-components` 2.x; upgrade to 3.0.0 or later.
  - VoiceView can't be enabled with `vdcm` from the Developer Mode shell.
  - `ifconfig` doesn't work in the Developer Mode shell; get the IP from Settings > Network.
  - TypeScript 4 projects fail with `@types/node` 26.
- OS 2.0 build numbers: Fire TV Stick 4K (3rd Gen) 2502000007420; Fire TV Stick HD and 4K Select 2602000052920.
- Troubleshooting index (runtime, build, SDK install, CLI, Fast Refresh, VVD, Fire TV Stick, VDA, keyboard, ACR, monorepo, network proxy): https://developer.amazon.com/docs/vega/2.0/troubleshoot-overview (2.0)
- Amazon's own AI tooling: **Amazon Devices Builder Tools (ADBT) for AI** is "the recommended way to use AI coding agents for Vega development". It is an MCP server; Argent's notes name it `amazon-devices-buildertools-mcp`, with tools such as `symbolicate_acr`, `analyze_perfetto_traces` and `search_documentation`. ADBT 1.0.19+ finds experimental APIs that 2.0 removed. [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) (2.0); get started at https://developer.amazon.com/docs/adbt/get-started

### 7.3 Community agent skills

- **amazonappdev/devices-agent-skills** (MIT; https://github.com/AmazonAppDev/devices-agent-skills, last push 2026-11-17). Contains two skills:
  - `vega-multi-tv-migration`: migrate a Vega app to a multi-platform RN monorepo (Android TV, Apple TV, Vega), working through analysis, implementation, platform support and configuration phases.
  - `rn-tv-ui-best-practices`: focus, layout, typography and D-pad for tvOS, Android TV, Fire TV and Vega. Its `references/vega-specifics.md` says it reflects "Vega SDK 0.22 / React Native for Vega 0.72"; it ships a static helper `references/scripts/audit.js`. [source](https://github.com/AmazonAppDev/devices-agent-skills/tree/main/skills/rn-tv-ui-best-practices)
  - `vega-multi-tv-migration` uses `packages/shared`, `packages/vega` and `packages/expotv` in its quick reference. [source](https://github.com/AmazonAppDev/devices-agent-skills/blob/main/skills/vega-multi-tv-migration/SKILL.md)
  - The repo had no releases on 2026-11-20; `HEAD` was commit `f31e0e6357d5987e003b69b263dd5bdc6748d6ab` (2026-11-17), checked with `gh api`.
  - Install with `gh skill install AmazonAppDev/devices-agent-skills --all` or `npx skills add AmazonAppDev/devices-agent-skills`. The README points proprietary SDK/API workflows to ADBT.
- **software-mansion/argent** (https://github.com/software-mansion/argent). An MCP server plus skills that let an agent control iOS simulators, Android emulators and devices, TVs (Apple TV, Android TV, Fire TV), and Electron/Mac apps: tap, D-pad, e2e record and replay, visual regression, profiling, logs and network. Install: `npx @swmansion/argent@latest init`.
  - Its Vega reference (`packages/skills/skills/argent-tv-interact/references/vega.md`) covers VVD boot/stop, `vsm developer-mode enable` when input is ignored, and the Fast Refresh flow (`vega device install-app -p`, `npm start`, `vega device start-port-forwarding --port 8081 --forward false`, `vega device launch-app -a <appId>`, then check `http://localhost:8081/json/list` for a `Hermes React Native` target). Its debugger tools work on VVD, but component-tree, inspect-element and profiler tools are unsupported because "Vega's React Native forks RN 0.72 and serves the legacy Hermes inspector". That statement may be outdated for RN 0.83 apps.
  - **Telemetry is opt-out** (`argent telemetry disable`). Proprietary binaries are bundled.
- **callstackincubator/agent-skills**, skill `react-native-tv-best-practices` (MIT; https://github.com/callstackincubator/agent-skills/tree/main/skills/react-native-tv-best-practices). A review skill for focus/D-pad, 10-foot layout, playback/DRM, low-memory performance and TV a11y across react-native-tvos, Expo TV, Vega/Kepler and web TV. It starts by detecting the stack, so it doesn't demand react-native-tvos, a tvOS Podfile or a leanback manifest on Vega apps. References are grouped as `focus-*`, `nav-*`, `design-*`, `perf-*`, `video-*`, `a11y-*`, `setup-*`, `test-*`/`release-*`.

---

## Unverified / open questions

- **"Navigable"** as a Vega focus API: no Vega doc page defines it. The word appears only in prose ("touchable or navigable (using DPAD) elements"). Treat it as not an API.
- **`@vega-tv/react-native-module-resolver-preset`** does not exist on npm. The documented package is `@amazon-devices/kepler-module-resolver-preset`.
- **`FocusManager.setFocus()`**: the focus-management guide uses it, but the FocusManager API page (RN 0.83) lists only `getFocused/setNextFocus/clearNextFocus/setFocusRoot/focus/blur`. Check the shipped typings before relying on `setFocus`.
- **Screen recording from the CLI**: not documented in the pages read. Only screenshots (`gwsi-tool-screenshooter`, Appium) are. Unknown whether a recorder exists on device.
- **inputd-cli invocation**: the 2.0 page shows both `vega exec vda shell inputd-cli …` and `vega exec inputd-cli …`. Likewise, both `vega device shell -d VirtualDevice` and (in 0.24) `vega exec device shell -d VirtualDevice` appear. Confirm on a real SDK 2.0 install.
- **VoiceView hold time**: 2 s per the assistive-technologies page, 3 s per the RN accessibility page.
- **Appium `platformName`**: the docs say `Kepler` on 2.0. A search-engine summary claimed "Vega" for 0.23, but the page itself wasn't confirmed. Use `Kepler`.
- **Appium Vega Driver version** pinned in docs: `@amazon-devices/appium-kepler-driver@3.30.0` with Appium 2.2.2. Whether newer drivers or Appium 3 work is not documented.
- **Vega-specific 10-foot design guide**: none found. The Fire TV design guideline page (2020, Fire OS-oriented) is the closest official source. Whether Appstore enforces the 5% safe zone for Vega apps is not stated.
- **React Native DevTools on VVD vs device**: the docs don't separate them. Argent claims legacy-inspector limits that conflict with Amazon's RN 0.83 DevTools page.
- **Does `testID` map to Appium's `test_id`?** Inspector advises `test_id`, and RN `testID` presumably surfaces as `test_id`, but no page states the mapping explicitly.

## Sources

Vega guides (2.0 = latest; 0.24 copies checked and identical unless noted):
- https://developer.amazon.com/docs/vega/2.0/vega-release-notes
- https://developer.amazon.com/docs/vega/2.0/vega-get-started
- https://developer.amazon.com/docs/vega/2.0/vega-test-tools-overview
- https://developer.amazon.com/docs/vega/2.0/rn-migration-testing
- https://developer.amazon.com/docs/vega/2.0/rn-migration-configuration
- https://developer.amazon.com/docs/vega/2.0/appium-install
- https://developer.amazon.com/docs/vega/2.0/appium-setup
- https://developer.amazon.com/docs/vega/2.0/appium-sessions
- https://developer.amazon.com/docs/vega/2.0/appium-write-tests
- https://developer.amazon.com/docs/vega/2.0/appium-commands
- https://developer.amazon.com/docs/vega/2.0/appium-locator-strategies
- https://developer.amazon.com/docs/vega/2.0/appium-inspector
- https://developer.amazon.com/docs/vega/2.0/test-automation-issues
- https://developer.amazon.com/docs/vega/2.0/test-before-submission
- https://developer.amazon.com/docs/vega/2.0/continuous-integration
- https://developer.amazon.com/docs/vega/2.0/eslint-plugin
- https://developer.amazon.com/docs/vega/2.0/inputdcli
- https://developer.amazon.com/docs/vega/2.0/vda-tools
- https://developer.amazon.com/docs/vega/2.0/run-apps
- https://developer.amazon.com/docs/vega/2.0/loggingctl
- https://developer.amazon.com/docs/vega/2.0/logs
- https://developer.amazon.com/docs/vega/2.0/debug-apps-overview
- https://developer.amazon.com/docs/vega/2.0/debugger
- https://developer.amazon.com/docs/vega/2.0/react-devtools
- https://developer.amazon.com/docs/vega/2.0/chrome-devtools
- https://developer.amazon.com/docs/vega/2.0/app-developer-menu
- https://developer.amazon.com/docs/vega/2.0/network-proxy
- https://developer.amazon.com/docs/vega/2.0/system-developer-tools
- https://developer.amazon.com/docs/vega/2.0/focus-management
- https://developer.amazon.com/docs/vega/2.0/using-on-screen-nav
- https://developer.amazon.com/docs/vega/2.0/enable-keyboard
- https://developer.amazon.com/docs/vega/2.0/detect-screen-resolution-vega
- https://developer.amazon.com/docs/vega/2.0/accessibility-vega
- https://developer.amazon.com/docs/vega/2.0/assistive-technologies-vega
- https://developer.amazon.com/docs/vega/2.0/vega-studio-accessibility-inspector
- https://developer.amazon.com/docs/vega/2.0/internationalization-puff-j
- https://developer.amazon.com/docs/vega/2.0/platform-specific-code
- https://developer.amazon.com/docs/vega/2.0/monorepo-support
- https://developer.amazon.com/docs/vega/2.0/sample-app-home
- https://developer.amazon.com/docs/vega/2.0/troubleshoot-overview
- https://developer.amazon.com/docs/vega/0.24/charles-proxy

Vega API reference (2.0):
- https://developer.amazon.com/docs/vega-api/2.0/supported-libraries
- https://developer.amazon.com/docs/vega-api/2.0/tvfocusguideview
- https://developer.amazon.com/docs/vega-api/2.0/react-navigation-8-0-0
- https://developer.amazon.com/docs/vega-api/2.0/vega-module-resolver-preset

React Native for Vega (RN 0.83):
- https://developer.amazon.com/docs/react-native-vega/0.83/focus-manager
- https://developer.amazon.com/docs/react-native-vega/0.83/tvEventHandler
- https://developer.amazon.com/docs/react-native-vega/0.83/HWEvent
- https://developer.amazon.com/docs/react-native-vega/0.83/userinputmanager
- https://developer.amazon.com/docs/react-native-vega/0.83/backhandler
- https://developer.amazon.com/docs/react-native-vega/0.83/navigation
- https://developer.amazon.com/docs/react-native-vega/0.83/keyboard
- https://developer.amazon.com/docs/react-native-vega/0.83/accessibility
- https://developer.amazon.com/docs/react-native-vega/0.83/internationalization-and-localization
- https://developer.amazon.com/docs/react-native-vega/0.83/i18nmanager

Fire TV (unversioned, Fire OS-oriented):
- https://developer.amazon.com/docs/fire-tv/design-and-user-experience-guidelines
- https://developer.amazon.com/docs/fire-tv/remote-input

Community and GitHub (read via `gh api`, 2026-11-20):
- https://community.amazondeveloper.com/c/vega/6
- https://github.com/AmazonAppDev/devices-agent-skills
- https://github.com/software-mansion/argent (README; packages/skills/skills/argent-tv-interact/references/vega.md)
- https://github.com/callstackincubator/agent-skills/tree/main/skills/react-native-tv-best-practices
- https://github.com/AmazonAppDev/react-native-multi-tv-app-sample
- https://github.com/AmazonAppDev/vega-video-sample, vega-tv-interfaces-sample, vega-sports-app, vega-audio-sample, vega-scrolling-sample, vega-seekbar-sample, vega-epg-sample
- npm registry: `@amazon-devices/kepler-module-resolver-preset` (exists, 0.1.17); `@vega-tv/react-native-module-resolver-preset` (404)
