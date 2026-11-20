# Vega OS: media playback, monetization, Fire TV integrations

> Fact sheet. Collected 2026-11-20 from Amazon's Vega docs (latest version: 2.0) and the sources listed at the end; every line names its source page and doc version. Vega changes between SDK versions: re-check a fact against the current docs (or `vega <command> --help`) before relying on it, and prefer the docs when they disagree with this file.


Researched 2026-11-20. Docs version check: `https://developer.amazon.com/docs/vega/latest/<page>` redirects to `/docs/vega/2.0/<page>`. Paths `/docs/vega/0.25/` and `/docs/vega/2.1/` return "404 Page not found". Older trees `0.21`, `0.22`, `0.23`, `0.24` are still published. **Latest = 2.0.** Every fact below comes from the 2.0 tree (`/docs/vega/2.0/...` or `/docs/vega-api/2.0/...`) unless it says otherwise.

Many 2.0 pages show **React Native 0.72** and **React Native 0.83** tabs. The manifest `runtime-module` differs between the two (see the snippets below).

Note on snippets: these pages are rendered from MDX, and placeholders in angle brackets (for example `<your partner id>`) were lost when the HTML was converted. Where a snippet shows `""` or a missing path segment, the docs have a placeholder there.

---

## 1. Media playback (W3C Media API / Vega Media Player)

### Architecture and package
- The Vega Media Player exposes W3C APIs to React Native: `HTMLMediaElement` (basic playback), MSE (adaptive streaming) and EME (DRM). The native layer is built on GStreamer with hardware decode. [source](https://developer.amazon.com/docs/vega/2.0/media-player) (v2.0)
- The package is `@amazon-devices/react-native-w3cmedia`. Exports used in the docs: `VideoPlayer`, `AudioPlayer`, `KeplerVideoView`, `KeplerVideoSurfaceView`, `KeplerCaptionsView`, `KeplerMediaControlHandler`, `decodingInfo`, plus a `Video`/`Audio` React component. The headless build is imported from `@amazon-devices/react-native-w3cmedia/dist/headless`. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_react-native-w3cmedia) (v2.0)
- **The docs give three different version pins.** The setup guide says `"~2.1.80"` ([source](https://developer.amazon.com/docs/vega/2.0/media-player-setup)). The API README says `"~2.3.2"` ([source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_react-native-w3cmedia)). The headless-playback RN 0.83 tab says `"~4.0.0"` ([source](https://developer.amazon.com/docs/vega/2.0/headless-js-playback)). All are v2.0. The likely reading is that ~2.x goes with RN 0.72 and ~4.x with RN 0.83.
- Unlike a browser `<video>`, you must call `initialize()` before playback. `VideoPlayer` is the controller and is separate from the view that renders it. [source](https://developer.amazon.com/docs/vega/2.0/media-player) (v2.0)
- Video needs a `KeplerVideoSurfaceView`. In `onSurfaceViewCreated`, call `setSurfaceHandle(handle)`. In `onSurfaceViewDestroyed`, call `clearSurfaceHandle(handle)` and then `deinitialize()`. For captions, add `KeplerCaptionsView` and pass its handle to `setCaptionViewHandle()`. [source](https://developer.amazon.com/docs/vega/2.0/media-player-select-playback) (v2.0)
- Babel change, required to avoid "ReferenceError: Property 'React' doesn't exist". [source](https://developer.amazon.com/docs/vega/2.0/media-player-setup) (v2.0)

```js
module.exports = {
    presets: [ ['module:metro-react-native-babel-preset', { useTransformReactJSXExperimental: true }] ],
    plugins: [
      [
        '@babel/plugin-transform-react-jsx',
        {
          runtime: 'automatic',
        },
      ],
    ]
  };
```

### Manifest entries for playback
From the setup guide [source](https://developer.amazon.com/docs/vega/2.0/media-player-setup) (v2.0):

```toml
[wants]
[[wants.service]]
id = "com.amazon.mediametrics.service" # Required for playback analytics and monitoring,
[[wants.privilege]]
id = "com.amazon.devconf.privilege.accessibility" # Required for captions.
[[wants.service]]
id = "com.amazon.mediabuffer.service" # Required for media buffer management.
[[wants.service]]
id = "com.amazon.mediatransform.service" # Required for media decode/encode
[[wants.service]]
id = "com.amazon.audio.stream" # Required for audio streaming.
[[wants.service]]
id = "com.amazon.audio.control" # Required for audio control.
# legacy permissions.
[[wants.service]]
id = "com.amazon.media.server" # Required for core media decode/encode. Superseded by com.amazon.mediatransform.service.
[[wants.service]]
id = "com.amazon.media.playersession.service" # Required for managing media controls. Superseded by KMC.
```

- The "General requirements" page and the API README also list a GIPC service, and the requirements page adds `com.amazon.audio.system` for AudioManager API clients. [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) (v2.0)

```toml
[[wants.service]]
id = "com.amazon.gipc.uuid.*"
[offers]
[[offers.service]]
id = "com.amazon.gipc.uuid.*"

[[wants.service]]
id = "com.amazon.audio.system"
```

- The FAQ's "permissions for different media playback use cases" block also lists `com.amazon.inputd.service` ("Required for remote button events"), `com.amazon.inputmethod.service`, and the DRM entries below plus `id="com.amazon.media.secureplayback"` under `[[needs.privilege]]`. It warns that a missing permission can mean playback never starts or a black screen. [source](https://developer.amazon.com/docs/vega/2.0/media-player-faq) (v2.0; the FAQ is an accordion, and this text was extracted from the page payload)

### Formats, codecs, DRM, captions
- URL mode plays MP4, MP3, MKV, FLV, OGG, FLAC, AVI, MOV, 3GPP, MPEG2TS and WEBM. MSE mode plays HLS (.m3u8, fMP4 or MPEG2TS segments), DASH (.mpd, fMP4 or WEBM) and SmoothStreaming. [source](https://developer.amazon.com/docs/vega/2.0/media-player) (v2.0)
- Video codecs: AV1 (4K60, HDR10/HDR10+/HLG), HEVC (4K60, Main10), H.264 (4K60, High up to L5.2), VP9 (4K60, Profile 2), VP8 (1080p30, software), MPEG-2 (1080p60), MPEG-4 (1080p30). Audio codecs: AAC, HE-AAC v1/v2, xHE-AAC, MP3, MP2, Dolby AC3/eAC3/AC4, Opus, FLAC, Vorbis, PCM, and DTS/DTS-HD (HDMI passthrough only). Apps must check the output device's capabilities before choosing codecs. [source](https://developer.amazon.com/docs/vega/2.0/media-player) (v2.0)
- DRM: Widevine L1 (hardware) for HD/4K and L3 (software) for SD. PlayReady SL2000/3000 (hardware) and SL150 (software). [source](https://developer.amazon.com/docs/vega/2.0/media-player) (v2.0)
- DRM works **only with adaptive streaming (HLS/DASH)**. It plays on a Fire TV Stick, and on the Vega Virtual Device only on Mac M-series machines, at L3 only. Anywhere else you get `Shaka Error DRM.FAILED_TO_CREATE_CDM`. Turn off fast refresh while testing DRM, because the two conflict (known issue). [source](https://developer.amazon.com/docs/vega/2.0/media-player-device) (v2.0)
- The content config used by the Amazon Shaka wrapper: [source](https://developer.amazon.com/docs/vega/2.0/media-player-device) (v2.0)

```js
const content =
{
    secure: "true", // true : Use Secure Video Buffers. false: Use Unsecure Video Buffers.
    uri: "https://storage.googleapis.com/wvmedia/cenc/h264/tears/tears.mpd",
    drm_scheme: "com.widevine.alpha", // com.microsoft.playready, com.widevine.alpha
    drm_license_uri: "https://proxy.uat.widevine.com/proxy?provider=widevine_test",
},
```

```toml
[wants]
[[wants.service]]
id = "com.amazon.drm.key" # Required for DRM video playback
[[wants.service]]
id = "com.amazon.drm.crypto" # Required for DRM video playback
[[needs.privilege]]
id = "com.amazon.privilege.security.file-sharing" # Required for DRM video playback
```

- Audio cannot use the hardware-secure path. Use `robustness: 'HW_SECURE_ALL'` for video and always `'SW_SECURE_CRYPTO'` for audio. If WV L1 playback fails, append `multikey=true` to the URL so each track gets its own key. [source](https://developer.amazon.com/docs/vega/2.0/media-player-faq) (v2.0)
- The first Vega Fire TV supports **one secure video decoder instance**. You must `deinitialize()` the first `videoPlayer` before you initialize a second one for back-to-back secure playback. [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) (v2.0)
- Captions: in-band and out-of-band CEA-608, CEA-708, TTML, WebVTT and SRT, in both URL and MSE mode, with language selection. [source](https://developer.amazon.com/docs/vega/2.0/media-player-faq) (v2.0)
- Media Capabilities: `decodingInfo` is exported from `@amazon-devices/react-native-w3cmedia`. Players that need it install it as `global.navigator.mediaCapabilities.decodingInfo = decodingInfoImpl`. The Vega Shaka build ships this polyfill. The results take the connected TV or AVR into account. [source](https://developer.amazon.com/docs/vega/2.0/media-capabilities) (v2.0)
- Codec selection must follow the audio capability of the HDMI sink, and must handle a mid-playback switch between Bluetooth and HDMI (for example, Dolby is available on HDMI but not on Bluetooth). Use Shaka `manifest.hls.sequenceMode = true` for HLS with MPEG2TS. [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) (v2.0)

### Players
- Use URL mode only for flat content. Adaptive, DRM and live content requires MSE mode plus a JS player. [source](https://developer.amazon.com/docs/vega/2.0/media-player-select-playback) (v2.0)
- Compatible players: Shaka Player (Amazon fork), Dash.js and Hls.js (with Amazon patches), plus Bitmovin, Dolby OptiView and Lura Player by Akta (access on request from each vendor). [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (v2.0). The FAQ also lists THEOPlayer and Brightcove. [source](https://developer.amazon.com/docs/vega/2.0/media-player-faq) (v2.0)
- The Shaka patches are Apache-2.0 tarballs at `https://amzndevresources.com/vega/media-player/shaka-rel-v<ver>-devices_scope.tar.gz`. Listed builds include 4.16.13-r1.2 (newest; moves to `@xmldom/xmldom`), 4.8.5-r1.7, 4.6.18-r2.16 and 4.3.6-r2.5. Workflow: `tar -xzf`, then `shaka-rel/scripts/setup.sh`, then copy `shaka-rel/src/*` to `<app>/src/` and `dist` to `<app>/src/shakaplayer`. Extra app dependencies: `"xmldom": "^0.8.10"`, `"base-64": "1.0.0"`, `"fastestsmallesttextencoderdecoder": "1.0.22"`. For a manual build, apply `git am shaka-rel/shaka-patch/*.patch -3` on upstream and run `build/all.py` (use node 16.20.2 if glibc errors appear). [source](https://developer.amazon.com/docs/vega/2.0/media-player-shaka-player) (v2.0)
- Wrapper usage: `new ShakaPlayer(videoPlayer, {secure, abrEnabled, abrMaxWidth, abrMaxHeight})`, then `.load(content, autoplay)`, then `.unload()`. [source](https://developer.amazon.com/docs/vega/2.0/media-player-shaka-player) (v2.0)
- Porting rules for other MSE players: replace XHR with `fetch`; no `ReadableStream` (read the whole response asynchronously); polyfill `document`, `DOMParser` (xmldom), `MediaSource` and `HTMLMediaElement`; set `navigator.userAgent = "kepler"`; for hls.js, disable workers, progressive mode and subtitles. [source](https://developer.amazon.com/docs/vega/2.0/mse-players) (v2.0)

### Media controls (KMC = Vega Media Controls), Alexa transport control
- If you play through W3C Media, call `player.setMediaControlFocus(componentInstance)` (with `componentInstance` from `useComponentInstance()` in `@amazon-devices/react-native-kepler`). That alone gives remote and Alexa play/pause/seek. To override a command, extend `KeplerMediaControlHandler` and pass it as the second argument. [source](https://developer.amazon.com/docs/vega/2.0/media-player-media-control) (v2.0)
- W3C Media's built-in KMC does **not** include Next/Previous. Apps that don't use W3C Media use the KMC API directly. [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-overview) (v2.0)
- Packages: `"@amazon-devices/kepler-media-controls": "~1.0.25"` and `"@amazon-devices/kepler-media-types": "~1.0.0"`. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-controls) (v2.0). The get-started guide shows `~1.0.0`.
- Provider API: `MediaControlServerComponentAsync.getOrMakeServer()`, then `setHandlerForComponent(handler, componentInstance)`, then `updateMediaSessionStates([...])`, with the handler implementing `IMediaControlHandlerAsync`. Client API: `MediaControlComponentAsync.makeMediaControlEndpointLocator()`. [source](https://developer.amazon.com/docs/vega/2.0/media-controls-get-started) (v2.0)
- Manifest (RN 0.72 tab). The RN 0.83 tab is identical except for `runtime-module = "/com.amazon.kepler.runtime.react_native_kepler_4@IReactNativeKepler_0"`. Core commands (Play, Pause, Stop, TogglePlayPause) need no opt-in. Other commands must be listed in `command_options` or `features`, or they are never delivered. [source](https://developer.amazon.com/docs/vega/2.0/media-controls-get-started) (v2.0)

```toml
[[components.interactive]]
id = "com.amazondeveloper.media.sample.main"
launch-type = "singleton"
categories = ["com.amazon.category.kepler.media"]
runtime-module = "/com.amazon.kepler.keplerscript.runtime.loader_2@IKeplerScript_2_0"

[[extras]]
key = "interface.provider"
component-id = "com.amazondeveloper.media.sample.main"
[extras.value.application]
[[extras.value.application.interface]]
interface_name = "com.amazon.kepler.media.IMediaPlaybackServer"
command_options = ["StartOver", "Previous", "Next", "SkipForward", "SkipBackward"]
# AdvancedSeek | VariableSpeed | AudioTracks | TextTracks
features = ["AdvancedSeek", "VariableSpeed", "AudioTracks", "TextTracks"]
```

### Headless playback / background
- Headless JS playback moves the player into a `[[components.service]]` on its own JS thread. Amazon claims up to 30% better time to first video frame (TTFVF). Manifest requirements: an interactive `.main` and a service `.service` in the same `[[processes.group]]`, plus `[[wants.service]] id=".service"` and `[[offers.service]] id=".service"`. Entry points are registered in `service.js` through `HeadlessEntryPointRegistry` from `@amazon-devices/headless-task-manager` (`<pkg>.service::onStartService` / `::onStopService`). [source](https://developer.amazon.com/docs/vega/2.0/headless-js-playback) (v2.0)
- Headless dependencies, RN 0.72: `headless-task-manager ~1.0.0`, `kepler-player-server ~2.0.4`, `kepler-player-client ~2.0.4`, `keplerscript-turbomodule-api ~1.2.1`, `react-native-kepler ~2.0.0`, `react-native-w3cmedia ~2.1.66`. RN 0.83: `kepler-player-server ~4.0.0`, `kepler-player-client ~4.0.0`, `react-native-kepler ~4.0.0`, `react-native-w3cmedia ~4.0.0`, with `react 19.2.0` and `react-native 0.83.0` (all under `@amazon-devices/`). [source](https://developer.amazon.com/docs/vega/2.0/headless-js-playback) (v2.0)
- Backgrounding: the docs require releasing media resources when the Lifecycle Manager sends the app to the background (`shakaPlayer.detach()` → `videoPlayer.deinitialize()` → `unload()` → `destroy()`), and for VOD, returning to the detail page. [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) (v2.0). The headless UI should call `IPlayerClient::unloadSync` on background. [source](https://developer.amazon.com/docs/vega/2.0/headless-js-playback) (v2.0)

---

## 2. In-app purchasing (IAP)

- The package is `@amazon-devices/keplerscript-appstore-iap-lib`, pinned `"~2.13.0"` in the v2.0 docs. Methods: `getUserData`, `getProductData` (up to 100 SKUs), `purchase`, `getPurchaseUpdates`, `modifySubscription`, `notifyFulfillment`, all on `PurchasingService` and all returning Promises. Hooks: `useIapProductData`, `useIapPurchaseUpdates`, `useIapUserData`. There is also `requestUserProfileAccess` (Quick Subscribe). [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-appstore-iap-lib) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/vega-iap-overview) (v2.0)
- Production manifest: [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-appstore-iap-lib) (v2.0)

```toml
[wants]
[[wants.service]]
id = "com.amazon.iap.core.service"
[[wants.module]]
id = "/com.amazon.iap.core@IIAPCoreUI"
[needs]
[[needs.module]]
id = "/com.amazon.kepler.appstore.iap.purchase.core@IAppstoreIAPPurchaseCoreService"
```

```js
PurchasingService.purchase({ sku: sku }).then((res) => { /* handle */ });
PurchasingService.getProductData({ skus: Constants.SKUS }).then((res) => { /* ... */ });
```

- **App Tester (sandbox).** Add the following to the manifest. You cannot have SDK Tester on the same device. [source](https://developer.amazon.com/docs/vega/2.0/configure-app-tester) (v2.0)

```toml
[wants]
[[wants.service]]
id = "com.amazon.iap.tester.service" # This is required for tester (Sandbox mode)
[[wants.module]]
id = "/com.amazonappstore.iap.tester@IIAPTesterUI"
[needs]
[[needs.module]]
id = "/com.amazon.kepler.appstore.iap.purchase.core@IAppstoreIAPPurchaseCoreService"
```

- App Tester comes as separate `.vpkg` downloads for RN 0.83 and RN 0.72. Install on a device with `vega exec vda push <vpkg> /tmp` and then `vpm install /tmp/<vpkg>`. Install on the VVD with `vega run-app <vpkg> com.amazonappstore.iap.tester.ui -d VirtualDevice`. Launch with `vmsgr send pkg://com.amazonappstore.iap.tester.ui`. [source](https://developer.amazon.com/docs/vega/2.0/configure-app-tester) (v2.0)
- Item data: download the JSON file from the Developer Console, name it `amazon.sdktester.json`, and push it to `/tmp/scratch/com.amazonappstore.iap.tester/`. On the VVD use `vega device copy-to -s <file> --destination /tmp/scratch/com.amazonappstore.iap.tester`. `itemType` is `CONSUMABLE`, `ENTITLED` or `SUBSCRIPTION`; subscriptions use `subscriptionParent`. [source](https://developer.amazon.com/docs/vega/2.0/configure-app-tester) (v2.0)
- Sandbox switch: an `amazon.config.json` containing `"debug.amazon.sandboxmode": "debug"`, pushed to the device. Change the value to `prod` or delete the file to turn sandbox off. [source](https://developer.amazon.com/docs/vega/2.0/use-app-tester) (v2.0). For production mode, remove `/tmp/scratch/<your-app-id>/amazon.config.json`. [source](https://developer.amazon.com/docs/vega/2.0/iap-production-mode) (v2.0)
- **Live App Testing.** LAT is supported. Enable production mode as follows: download `AppstoreAuthenticationKey.pem` from Developer Console → Upload Your App File → View public key; put it in the app's `assets/raw` (the docs misspell this as "assests/raw"); add the `com.amazon.iap.core.service` and `/com.amazon.iap.core@IIAPCoreUI` wants. [source](https://developer.amazon.com/docs/vega/2.0/iap-production-mode) (v2.0)
- **RVS.** The URL format is `<Protocol>//<Server>[/<sandbox>]/version/<Operation_version_number>/verifyReceiptId/developer/<Shared_Secret>/user/<UserId>/receiptId/<ReceiptId>`, and the current operation version is `1.0`. Response codes: 200 OK, 400 invalid, 410 no longer valid (treat as canceled), 429 throttled, 496 bad shared secret, 497 bad user ID, 500 server error. Response fields include `autoRenewing`, `betaProduct` (LAT), `cancelDate`, `cancelReason`, `freeTrialEndDate`, `gracePeriodEndDate`, `productType`, `receiptId`, `renewalDate`, `term`, `termSku` and `testTransaction`. [source](https://developer.amazon.com/docs/vega/2.0/vega-rvs-overview) (v2.0)
- The RVS Cloud Sandbox endpoint is `https://appstore-sdk.amazon.com/sandbox/version/1.0/verifyReceiptId/developer/<secret>/user/<UserId>/receiptId/<ReceiptId>`. Any non-empty secret works there, and it validates App Tester receipts. [source](https://developer.amazon.com/docs/vega/2.0/rvs-cloud) (v2.0)
- The 2.0 overview still says "In the preview release, you can use RVS in sandbox mode." [source](https://developer.amazon.com/docs/vega/2.0/vega-rvs-overview) (v2.0)
- Subscriptions: create them in the Developer Console under In-App Items → Add Single IAP (Consumable, Entitlement or Subscription). Terms can be Weekly, Bi-Weekly, Monthly, Bi-Monthly, Quarterly, Semi-Annually or Annually. Free trials can be No, 7 days, 14 days, 1 month, 2 months or 3 months. Each term has a child SKU. [source](https://developer.amazon.com/docs/vega/2.0/vega-submit-iap) (v2.0)
- Test cases Amazon expects: notifyFulfillment after a purchase, purchases surviving clear data, Home and Back, an upgrade giving two receipts (one cancelled, one active), a downgrade being deferred, and accelerated renewals. [source](https://developer.amazon.com/docs/vega/2.0/vega-iap-testing) (v2.0)
- **Quick Subscribe / one-click account sharing.** The flow is: `requestUserProfileAccess()` gives an authorization code; your server calls `POST https://appstore-sdk.amazon.com/version/1.0/auth/o2/token` (`grant_type=authorization_code`, `code`, `client_id`, `client_secret`) to get an `Atza|...` bearer token; the server then calls `GET https://appstore-sdk.amazon.com/version/1.0/user/profile` to get name, zip code and email. [source](https://developer.amazon.com/docs/vega/2.0/quick-subscribe-overview) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/account-information-sharing) (v2.0)
- **OpenIAP / react-native-iap.** Amazon blog, 2026-08-24: stable releases with Vega support are react-native-iap 16.3.1, expo-iap 5.3.1 and openiap-google-amazon 3.3.1. For React Native for Vega, install `@amazon-devices/keplerscript-appstore-iap-lib` alongside react-native-iap. In Expo, set `modules.amazon.vegaOS: true`. [source](https://developer.amazon.com/apps-and-games/blogs/2026/08/amazon-sponsors-openiap-fire-os-vega-os) (blog, not versioned)
- OpenIAP setup page (third-party): `yarn add @amazon-devices/keplerscript-appstore-iap-lib@~2.13.0 @amazon-devices/react-native-kepler@^2.0.0`, with dev dependency `@amazon-devices/kepler-cli-platform ~0.22.0`. The Expo plugin generates Vega files automatically when the root has a `manifest.toml`. Vega is not an Android flavor. Native Android, Flutter, KMP, MAUI and Godot have no Vega target. [source](https://www.openiap.dev/docs/setup/store/amazon) (third-party)
- Release tracks: with a shared listing, the Fire OS and Vega binaries share one ASIN and share IAP items. [source](https://developer.amazon.com/docs/vega/2.0/app-submission) (v2.0, search summary only; not opened)

---

## 3. Fire TV integrations

### Content Launcher (search, browse, Alexa quick play)
- The package is `"@amazon-devices/kepler-media-content-launcher": "~2.0.22"`. Classes: `ContentLauncherServerComponent`, then `getOrMakeServer()`, then `setHandlerForComponent(handler, componentInstance)`. Handler signature: `handleLaunchContent(contentSearch: IContentSearch, autoPlay: boolean, optionalFields: ILaunchContentOptionalFields)`, returning a response built with `factory.makeLauncherResponseBuilder().contentLauncherStatus(ContentLauncherStatusType.SUCCESS | AUTH_FAILED | URL_NOT_AVAILABLE).build()`. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-content-launcher) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-integration-guide) (v2.0)
- `autoPlay=true` means quick play: start playback immediately. `autoPlay=false` means show search results. In `externalIdList`, the recognized names are `amzn_id` (catalog ID / LaunchId) and `launch_url` (a deep link, for example from Matter Casting). Video entities have `type` 13 (`ContentSearchParamType::VIDEO`). For an episode request, the docs pass the show ID, and the app resolves the episode itself. [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-integration-guide) (v2.0)
- All catalog-integrated apps must integrate Content Launcher and should integrate Account Login. Only Content Launcher-integrated content appears in the home-screen featured rows ("Next up for you" and others), with no further integration needed. Alexa sends quick play ("Alexa, watch X", "...Season N Episode M") through Content Launcher, and transport control through KMC. [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-overview) (v2.0)
- Manifest: [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-content-launcher) (v2.0)

```toml
[[components.interactive]]
id = "com.amazondeveloper.media.sample.main"
runtime-module = "/com.amazon.kepler.keplerscript.runtime.loader_2@IKeplerScript_2_0"
launch-type = "singleton"
categories = ["com.amazon.category.main", "com.amazon.category.kepler.media"]

[[extras]]
key = "interface.provider"
component-id = "com.amazondeveloper.media.sample.main"
[extras.value.application]
[[extras.value.application.interface]]
interface_name = "com.amazon.kepler.media.IContentLauncherServer"
attribute_options = ["partner-id"]
static-values = { partner-id = "" }   # placeholder for your partner id in the docs

[needs]
[[needs.module]]
id = "/com.amazon.kepler.media@IContentLauncher1"
```

### Account Login
- Packages: `"@amazon-devices/kepler-media-account-login": "~1.1.24"` and `"@amazon-devices/headless-task-manager": "~1.2.4"`. The integration guide shows `~1.1.12`. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-account-login) (v2.0)
- Report `StatusType.SIGNED_IN` or `SIGNED_OUT` on every launch, whenever subscription status changes, and when asked. The login status is served by a **service component**, so the system can query it without waking the UI. Implement `handleReadStatus()` in `IAccountLoginHandlerAsync`, and push changes with `updateStatus()` via `AccountLoginServerComponent.makeStatusBuilder()`. Persist the status (the example uses AsyncStorage) and register entry points with `HeadlessEntryPointRegistry.registerHeadlessEntryPoint2('<pkg>.interface.provider::onStartService', ...)`. [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-account-login-integration-guide) (v2.0)
- Manifest additions on top of Content Launcher: [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-account-login-integration-guide) (v2.0)

```toml
[[components.service]]
id = "com.amazondeveloper.media.sample.interface.provider"
runtime-module = "/com.amazon.kepler.headless.runtime.loader_2@IKeplerScript_2_0"
launch-type = "singleton"

[[offers.service]]
id = "com.amazondeveloper.media.sample.interface.provider"
required-privileges = ["com.amazon.multimedia.privilege.session.manage"]

[[extras.value.application.interface]]
interface_name = "com.amazon.kepler.media.IAccountLoginServer"
attribute_options = ["Status"]
override_attribute_component = { Status = "com.amazondeveloper.media.sample.interface.provider" }

[[needs.module]]
# The dot (.) after "media" is intentional in this format.
id = "/com.amazon.kepler.media.@IAccountLogin1"
```

- UX behavior: when the user is signed out, the Fire TV detail page shows "Subscribe", which opens your app straight to the sign-in or sign-up flow. After the user authenticates, the selected content should play. [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-overview) (v2.0)

### Catalog ingestion / Universal Search and Browse
- EMBER (Enhanced Metadata Bridge for Entertainment Resources) is an XML catalog validated against an XSD. You upload it to an Amazon S3 bucket on a regular schedule (cron is suggested) and Amazon runs content matching. It enables Universal Search and Browse, the Buy Box and featured rows. [source](https://developer.amazon.com/docs/catalog/ember-catalog-integration-overview.html) (catalog docs, not versioned; this page does not mention Vega); [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-overview) (v2.0)

### Content Personalization (watch activity, watchlist, "continue watching")
- Packages: `@amazon-devices/kepler-content-personalization ~1.3.3`, `@amazon-devices/kepler-epg-provider ~1.9.124`, `@amazon-devices/headless-task-manager ~1.0.0`. Requirements: catalog ingestion, and a supported device ("check with your Amazon contact"). The API is `ContentPersonalizationServer.reportNewPlaybackEvent(new PlaybackEventBuilder()...build())`, plus `report{New,Removed,Refreshed}CustomerList*`, and a headless data-pull service. [source](https://developer.amazon.com/docs/vega/2.0/get-started-with-vega-content-personalization) (v2.0)

```toml
[[wants.service]]
id = "com.amazon.tv.developer.dataservice"
[[needs.privilege]]
id = "com.amazon.tv.content-personalization.privilege.provide-data"
[[extras.value.application.interface]]
interface_name = "com.amazon.kepler.media.IContentPersonalizationServer"
attribute_options = ["SupportedCustomerLists", "DataRefreshComponentId"]
[extras.value.application.interface.static_values]
DataRefreshComponentId = "com.amazondeveloper.keplervideoapp.content.dataRefresh.provider"
```

### Live TV / linear
- Packages: `@amazon-devices/kepler-channel`, `kepler-epg-provider`, `kepler-epg-sync-scheduler`. You must be **allowlisted** for Live TV. Your content appears in the On Now and Recent rows, the Live tab, and a 14-day guide and search. Required pieces: a provider logo (monochrome, 34px or less high), EPG integration, channel integration, and a "Sync Sources" screen. [source](https://developer.amazon.com/docs/vega/2.0/live-tv-overview) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/get-started-with-linear-tv) (v2.0)

### Deep links (Vega has no Android intents)
- Vega uses the Vega Messaging Library and the React Native `Linking` API. For Fire OS compatibility there is limited support for intent-style URIs (featured content). Declare each URI as a `[[message]]`, map them to a component with `[[offers.message-target]]`, and group them in an `[[offers.module]]`. [source](https://developer.amazon.com/docs/vega/2.0/deep-links-vega-apps) (v2.0)

```toml
[[message]]
uri = "deep-link-sample-app://home"
sender-privileges = ["*"]
receiver-privileges = ["self"]

[offers]
[[offers.message-target]]
uses-component = "com.amazondeveloper.kepler.deeplink.sampleapp.main"
uris = ["deep-link-sample-app://home", "https://deep-link-sample-app.com/search"]

[[offers.module]]
id = "/com.amazondeveloper.kepler.deeplink.sampleapp@IDeeplinks"
includes-messages = ["deep-link-sample-app://home", "https://deep-link-sample-app.com/search"]
```

- To receive a link: `Linking.getInitialURL()` for a cold start, and `Linking.addEventListener('url', ...)` while running. To send one: `Linking.canOpenURL` then `Linking.openURL`. To test: `vda shell vmsgr send "deep-link-sample-app://details?asin=B00MULA"`. [source](https://developer.amazon.com/docs/vega/2.0/deep-links-vega-apps) (v2.0)
- URI schemes: `pkg://<component-id>` addresses a component directly and is the only scheme third parties can define new messages on. `os://settings`, `os://home` and `os://browser` reach the OS. `amzns://apps?p=<pkg>|asin=<asin>|si=<id>` opens an app or the store. http/https URIs match on host and path and are preferred over custom schemes. [source](https://developer.amazon.com/docs/vega/2.0/messaging-addressing-schemes) (v2.0)
- Featured-content deep links are the same encoded links Fire OS uses, received through `Linking`. [source](https://developer.amazon.com/docs/vega/2.0/featured-content-deeplinks) (v2.0)

### "Interactive components"
- These are manifest `[[components.interactive]]` entries (UI), as opposed to `[[components.service]]` (headless) and task components. Content Launcher and KMC run in the interactive component named by `[[extras]] key="interface.provider"`. Account Login and the personalization data refresh run in service components. [source](https://developer.amazon.com/docs/vega/2.0/app-manifest) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/content-launcher-integration-guide) (v2.0)

### Second screen (DIAL)
- Manifest: `[[extras]] key="dial_id"` (your registered DIAL name, the same as Fire OS whisperplay `dialid`), `key="dial_launch_component"`, and an optional `key="dial_origins"` for CORS. The launch URI carries `DIAL_ADDITIONAL_DATA_URL=http://127.0.0.1:8009/apps/<id>/dial_data`. You can POST to `additionalDataUrl` "to share a session token" with the phone app. [source](https://developer.amazon.com/docs/vega/2.0/dial-vega) (v2.0)

---

## 4. Ads
- Amazon Publisher Services (APS) header bidding uses the package `"@amazon-devices/kepler-aps-client": "~1.0.11"`. `ApsClientLibraryAPI.getDeviceParams(ApsIntegrationType.S2S | C2S)` returns `h`, `w`, `dnt`, `ifa`, `ua`, `make`, `language`, `model`, `os` and `osv`. The integration is either C2S (recommended timeout 1000 ms) or S2S (500 ms), following the AMS OTT guides. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-aps-client) (v2.0); [source](https://developer.amazon.com/docs/vega/2.0/amazon-publisher-services) (v2.0)

```toml
[wants]
[[wants.module]]
id = "/com.amazon.kepler.apslib@IApsDeviceParametersMap"
[[wants.service]]
id = "com.amazon.admanager.service.main"   # required for dnt and ifa
```

- SSAI is supported only if the codec stays the same across main→ad and ad→main transitions. For CSAI, the FAQ points to the community KB article "Sample Code for Client Side Ad-Insertion (CSAI)". [source](https://developer.amazon.com/docs/vega/2.0/media-player-faq) (v2.0)
- Third-party ad and measurement services listed as Vega-compatible: Google IMA DAI (through your Google account manager), FreeWheel, Yospace, TrueX, Smartclip, plus analytics and measurement vendors (Conviva, Nielsen, Comscore, Mux, NPAW and others). [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (v2.0)

## 5. TV login patterns
- Amazon's Vega docs document **Quick Subscribe with one-click account information sharing** (above), **Account Login status reporting**, and **DIAL token hand-off**. No Amazon Vega page on device-code or QR-code sign-in was found. `react-native-qrcode-svg` is a Vega-ported library: `~3.0.0` for RN 0.83 and `~2.0.0` for RN 0.72. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) (v2.0)

---

## Unverified / open questions
- The exact `react-native-w3cmedia` version per RN line (`~2.1.80`, `~2.3.2` and `~4.0.0` all appear in v2.0 docs). Confirm against the npm registry or the SDK release notes.
- The APS README's package.json example combines `react-native-kepler ~2.0.0` with `react-native 0.83.0`, which looks like a docs error. The headless page pairs RN 0.83 with `react-native-kepler ~4.0.0`.
- The "Selecting the Playback Mode" audio sample imports `VideoPlayer` but instantiates `AudioPlayer`, which is a docs bug. `AudioPlayer` is a real export according to the README.
- The FAQ permissions block contains `id = "com.amazon.gipc.uuid."` (probably a lost `*` from the page rendering).
- Background audio (music keeping playing behind another app) is not documented as a supported mode. The docs only require releasing media resources on background. The headless service is about thread isolation, not background audio.
- The production RVS host is not stated on the Vega 2.0 RVS page, which still says "preview release ... sandbox mode". Fire OS docs use `https://appstore-sdk.amazon.com/version/1.0/verifyReceiptId/...`, but this was not confirmed for Vega.
- Whether `amazon.config.json` goes to `/tmp/scratch/<app-id>/` for sandbox mode: the production-mode page implies it does, while the use-app-tester page only says "create a directory".
- The App Tester `.vpkg` download URLs were not captured. The page has separate RN 0.83 and RN 0.72 downloads.
- The value format for Content Launcher `partner-id` is a placeholder in the docs and is presumably assigned during catalog onboarding.
- Alexa beyond Content Launcher and KMC (for example, Video Skill API or Alexa for video publishers on Vega): the APS page links `/apps-and-games/alexa-for-video-publishers`, which was not opened.
- The pages for the Hls.js and Dash.js players (`/docs/vega/2.0/media-player-hls-player`, `media-player-dash-player`) were not opened.
- Whether the v2.0 App Submission page confirms shared IAP across release tracks: this comes from a search summary only.

## Sources
- https://developer.amazon.com/docs/vega/2.0/media-player
- https://developer.amazon.com/docs/vega/2.0/media-player-setup
- https://developer.amazon.com/docs/vega/2.0/media-player-select-playback
- https://developer.amazon.com/docs/vega/2.0/media-player-shaka-player
- https://developer.amazon.com/docs/vega/2.0/media-player-device
- https://developer.amazon.com/docs/vega/2.0/media-player-faq
- https://developer.amazon.com/docs/vega/2.0/media-player-requirements
- https://developer.amazon.com/docs/vega/2.0/media-player-media-control
- https://developer.amazon.com/docs/vega/2.0/media-capabilities
- https://developer.amazon.com/docs/vega/2.0/mse-players
- https://developer.amazon.com/docs/vega/2.0/headless-js-playback
- https://developer.amazon.com/docs/vega/2.0/media-controls-get-started
- https://developer.amazon.com/docs/vega/2.0/foreground-background
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_react-native-w3cmedia
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-controls
- https://developer.amazon.com/docs/vega-api/2.0/supported-libraries
- https://developer.amazon.com/docs/vega/2.0/appstore-integrations-overview
- https://developer.amazon.com/docs/vega/2.0/vega-iap-overview
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-appstore-iap-lib
- https://developer.amazon.com/docs/vega/2.0/configure-app-tester
- https://developer.amazon.com/docs/vega/2.0/use-app-tester
- https://developer.amazon.com/docs/vega/2.0/iap-production-mode
- https://developer.amazon.com/docs/vega/2.0/vega-iap-testing
- https://developer.amazon.com/docs/vega/2.0/vega-submit-iap
- https://developer.amazon.com/docs/vega/2.0/vega-rvs-overview
- https://developer.amazon.com/docs/vega/2.0/rvs-cloud
- https://developer.amazon.com/docs/vega/2.0/quick-subscribe-overview
- https://developer.amazon.com/docs/vega/2.0/account-information-sharing
- https://developer.amazon.com/apps-and-games/blogs/2026/08/amazon-sponsors-openiap-fire-os-vega-os
- https://www.openiap.dev/docs/setup/store/amazon
- https://developer.amazon.com/docs/vega/2.0/content-launcher-overview
- https://developer.amazon.com/docs/vega/2.0/content-launcher-integration-guide
- https://developer.amazon.com/docs/vega/2.0/content-launcher-account-login-integration-guide
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-content-launcher
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-media-account-login
- https://developer.amazon.com/docs/catalog/ember-catalog-integration-overview.html
- https://developer.amazon.com/docs/vega/2.0/get-started-with-vega-content-personalization
- https://developer.amazon.com/docs/vega/2.0/live-tv-overview
- https://developer.amazon.com/docs/vega/2.0/get-started-with-linear-tv
- https://developer.amazon.com/docs/vega/2.0/deep-links-vega-apps
- https://developer.amazon.com/docs/vega/2.0/messaging-addressing-schemes
- https://developer.amazon.com/docs/vega/2.0/featured-content-deeplinks
- https://developer.amazon.com/docs/vega/2.0/app-manifest
- https://developer.amazon.com/docs/vega/2.0/dial-vega
- https://developer.amazon.com/docs/vega/2.0/amazon-publisher-services
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-aps-client
