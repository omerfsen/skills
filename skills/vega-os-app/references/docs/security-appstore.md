# Vega OS: security, privacy and Amazon Appstore submission

> Fact sheet. Collected 2026-11-20 from Amazon's Vega docs (latest version: 2.0) and the sources listed at the end; every line names its source page and doc version. Vega changes between SDK versions: re-check a fact against the current docs (or `vega <command> --help`) before relying on it, and prefer the docs when they disagree with this file.


Researched 2026-11-20. Latest Vega docs version: **2.0**. The version selector on `/docs/vega/2.0/app-submission` shows "Version 2.0 (LATEST)", and the page was last modified 2026-11-16. `/docs/vega-api/latest/...` redirects to `/docs/vega-api/2.0/...`. The release notes say "Vega SDK 2.0, Vega OS 2.0, and the Vega Developer Tools are now generally available, and they're no longer in open beta". Generic Appstore pages (`/docs/app-submission/...`, `/docs/policy-center/...`, `/docs/app-testing/...`) are not versioned. They are tagged "unversioned" below and were read the same day.

Tags: **[Vega 2.0]** = /docs/vega/2.0 or /docs/vega-api/2.0. **[Appstore, unversioned]** = generic Appstore docs that apply to both Fire OS and Vega. **[General]** = not Vega-specific.

---

## 1. Security model

### 1.1 Sandbox and file system
- Apps run in a sandbox with a limited view of the file system. App-visible paths: [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-file-system) [Vega 2.0]
  - `/pkg`, `/pkg/assets` and `/pkg/lib` are read-only (package contents).
  - `/data` is writable. It survives reboots and package upgrades, is "not shared between apps or services", and is deleted on uninstall.
  - `/tmp` is writable and private to each app. It is cleared on reboot.
- Files in `assets/raw/` are read-only at runtime under `/pkg/assets/raw/<filename>`. To change one, copy it to `/data` first. [source](https://developer.amazon.com/docs/vega/2.0/vpt) [Vega 2.0]
- VPT packs only `manifest.toml`, `lib/`, `bin/`, `assets/`, `bundle/`, `config/`, `shared/lib/` and `shared/assets/`. It drops `.debug` files and `*.bundle.map` source maps from `bundle/`. `assets/` is **not** filtered, so any secret or certificate placed there ships in the package. [source](https://developer.amazon.com/docs/vega/2.0/vpt) [Vega 2.0]

### 1.2 Privileges (permissions) in manifest.toml
- Vega uses privileges to gate security- and privacy-sensitive features such as the camera and microphone. There are two kinds: **install-time** (recorded at install, granted automatically when requested) and **run-time** (granted by the user, who can deny them). [source](https://developer.amazon.com/docs/vega/2.0/request-runtime-privileges) [Vega 2.0]
- Privileges are declared in `manifest.toml` as `[[needs.privilege]]` (required) or `[[wants.privilege]]` (optional, the app must degrade gracefully without it). [source](https://developer.amazon.com/docs/vega/2.0/request-runtime-privileges) [Vega 2.0]
- `[needs]` is also used by the package manager and the Appstore to decide **which devices the app can be installed on**, so over-declaring needs reduces reach. [source](https://developer.amazon.com/docs/vega/2.0/manifest-needs) [Vega 2.0]
- `[[wants.service]]` declares optional system services. If an app uses a service it did not declare, it gets an error at start. Example log from Amazon's docs: [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib) [Vega 2.0]
  ```
  E securitymgr:[SecurityManager.cpp:124] Privilege 'com.amazon.wifi.privilege.read-wifi-credential' not granted
  ```
- Manifest snippets as documented: [source](https://developer.amazon.com/docs/vega/2.0/app-manifest) [Vega 2.0]
  ```toml
  schema-version = 1

  [needs]
  [[needs.module]]
  id = "/com.amazon.camera.module@ICamera"
  [[needs.privilege]]
  id = "com.amazon.media.secureplayback"

  [wants]
  [[wants.service]]
  id = "com.amazon.inputmethod.service"
  [[wants.privilege]]
  id = "com.amazon.camera.privilege.access"

  [[message]]
  uri = "samplepkg://play"
  sender-privileges = ["com.amazon.security.example1"]
  receiver-privileges = ["com.amazon.security.example2"]
  ```
- Privilege IDs found in the 2.0 docs. There is no consolidated public list (see open questions).
  - `com.amazon.audio.privilege.microphone.access` (run-time) [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib) [Vega 2.0]
  - `com.amazon.camera.privilege.access` [source](https://developer.amazon.com/docs/vega/2.0/app-manifest) [Vega 2.0]
  - `com.amazon.media.secureplayback` [source](https://developer.amazon.com/docs/vega/2.0/app-manifest) [Vega 2.0]
  - `com.amazon.network.privilege.net-info`: required by `@amazon-devices/react-native-netinfo` for full data. The user grants it at run time. If it is denied, `ssid` returns `"<unknown ssid>"` and `bssid` returns `"02:00:00:00:00:00"`. The library also needs `[[wants.service]] id = "com.amazon.network.service"`. [source](https://developer.amazon.com/docs/vega-api/2.0/react-native-net-info) [Vega 2.0]
  - `com.amazon.devconf.privilege.accessibility`: "Required for captions". [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) [Vega 2.0]
  - `com.amazon.privilege.package.query`: requires a platform signature, so third-party apps cannot get it. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib) [Vega 2.0]
  - `com.amazon.graphics.privilege.display.manager` "isn't available to third-party apps". [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) [Vega 2.0]
- Media apps must declare these services: `com.amazon.mediametrics.service`, `com.amazon.media.server`, `com.amazon.gipc.uuid.*` (in both wants and offers), `com.amazon.media.playersession.service`, `com.amazon.mediabuffer.service` and `com.amazon.mediatransform.service`. Audio apps also need `com.amazon.audio.stream`, `com.amazon.audio.control` and `com.amazon.audio.system`. [source](https://developer.amazon.com/docs/vega/2.0/media-player-requirements) [Vega 2.0]

### 1.3 Requesting run-time privileges (Security Manager API)
- Package: `"@amazon-devices/security-manager-lib": "~1.0.1767254405"`. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib) [Vega 2.0]
- `SecurityManager.getPrivilegeState(privilege)` returns `PrivilegeState.ALLOW` or `DENY`. `DENY` also covers "never asked". `requestPrivilege(privilege)` starts the system Privilege Request Handler. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib) [Vega 2.0]
- Rules from the docs: [source](https://developer.amazon.com/docs/vega/2.0/request-runtime-privileges) [Vega 2.0]
  - Use least privilege, and request at the moment the privileged API is used.
  - Show your own explanation dialog before the system prompt.
  - Degrade gracefully when denied. You may deep-link to the Settings pane.
  - "The app shouldn't continuously prompt the user". Re-request only after a new user action. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib)
  - If the user revokes a privilege, **the system terminates the app**.
  - Users manage consent in the privacy dashboard in Settings. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib)
- Documented example (shortened):
  ```ts
  SecurityManager.getPrivilegeState('com.amazon.audio.privilege.microphone.access').then(state => {
    if (state == PrivilegeState.ALLOW) navigation.navigate('RecordScreen');
    else SecurityManager.requestPrivilege('com.amazon.audio.privilege.microphone.access')
      .then(s => s == PrivilegeState.ALLOW ? navigation.navigate('RecordScreen')
                                           : navigation.navigate('MicrophoneAccessDeniedScreen'));
  });
  ```

### 1.4 Network security
- **WebView cleartext policy:** "Starting with WebView version 4.0.2 (RN 0.83), WebView prevents apps from loading plain-text (HTTP) URLs". If the manifest has no `network-traffic-policy.cleartext` section, all HTTP is blocked and only HTTPS loads. WebView 3.5.11 (RN 0.72) and lower are not affected. If both lists are set, `blocked-domains` wins. [source](https://developer.amazon.com/docs/vega/2.0/develop-your-app-with-webview) [Vega 2.0]
  ```toml
  # allowlist: block all HTTP except these
  [network-traffic-policy.cleartext]
  allowed-domains = ["debug.myapp.com", "*.test.internal.com", "localhost"]

  # blocklist: allow HTTP except these
  [network-traffic-policy.cleartext]
  allowed-domains = ["*"]
  blocked-domains = ["sensitive-api.com", "*.bank.com", "payment.example.com"]
  # Omit the section entirely = block all HTTP (default secure behaviour)
  ```
- WebView TLS errors: `onSslError` is called for recoverable certificate errors (untrusted issuer, expired, self-signed). The default is to cancel/reject. The host calls `proceed()` or `cancel()`. Non-recoverable handshake failures go to `onError` with `ERROR_FAILED_SSL_HANDSHAKE (-11)`. Client-certificate authentication supports PKCS#12 (`handleWithPkcs12`). The default is to cancel without a client certificate. [source](https://developer.amazon.com/docs/vega-api/2.0/webview-component-reference) [Vega 2.0]
- Other WebView security defaults: [source](https://developer.amazon.com/docs/vega-api/2.0/webview-component-reference) [Vega 2.0]
  - `mixedContentMode` defaults to `never` (MIXED_CONTENT_NEVER_ALLOW). Amazon says `always` is the "least secure" and does not recommend it.
  - `allowFileAccess` defaults to `false`.
  - `javaScriptEnabled` defaults to `true`.
  - `domStorageEnabled` defaults to `false`.
  - `thirdPartyCookiesEnabled` defaults to `false`.
  - There is a separate WebView Cookie Manager API.
- Policy: personal data must be handled securely, "e.g., transmitting it over HTTPS". [source](https://developer.amazon.com/docs/policy-center/privacy-security) [Appstore, unversioned]
- **Debug proxy cleanup (a release blocker for security):** Vega Studio's Network Proxy writes `proxy-config.json` and a CA certificate `ca.pem` into the app's `assets/raw`. Amazon says: "Before you submit your app for production: Delete the proxy-config.json file and any certificate file (for example, assets/raw/ca.pem) from the app", then run `vega clean` and rebuild. [source](https://developer.amazon.com/docs/vega/2.0/network-proxy) [Vega 2.0]
- No Vega page found documents a cleartext policy for React Native `fetch`/XHR, TLS minimum versions or certificate pinning. See open questions.

### 1.5 Cryptography and secure storage
- **Vega Crypto API**, `"@amazon-devices/keplercrypto": "~2.1"`. It covers key generation and import (RSA, ECC, symmetric), ECDSA, RSASSA-PKCS1/PSS, RSA-OAEP, AES-256 in CBC, CTR and **GCM**, HMAC, digests (SHA-256 and others), HKDF, PBKDF2, ECDH, key wrapping, `getRandom()` and a `WebCrypto` class. Keys have `purposes` (`KeyPurpose.SIGN/VERIFY/ENCRYPT/DECRYPT`) and an `exportable` flag. It is not supported in the simulator. [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplercrypto) [Vega 2.0]
  ```ts
  import { Crypto, SymmetricAlgorithm, KeyPurpose } from '@amazon-devices/keplercrypto';
  const crypto = new Crypto();
  const builder = crypto.makeGcmCipherContextBuilder(
    crypto.getSymmetricAlgorithmByName(SymmetricAlgorithm.AES256));
  ```
- `@amazon-devices/react-native-async-storage__async-storage` is documented as "asynchronous, **unencrypted**, persistent" key-value storage. Do not put tokens there in plaintext. [source](https://developer.amazon.com/docs/vega-api/2.0/react-native-async-storage) [Vega 2.0]
- `@amazon-devices/react-native-mmkv` (RN 0.83: `>=1.0.11-rn-83`; RN 0.72: `~1.0.0`) supports an `encryptionKey` config and `recrypt(newKey)`. [source](https://developer.amazon.com/docs/vega-api/2.0/react-native-mmkv) [Vega 2.0]
- `expo-crypto` is supported (`~3.0.0` on RN 0.83). [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) [Vega 2.0]
- The supported-libraries list contains **no** keychain/keystore/secure-store library (no react-native-keychain, expo-secure-store or encrypted-storage equivalent). [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) [Vega 2.0]

### 1.6 Identity and tokens
- Login with Amazon (`@amazon-devices/keplerscript-identity-lwa-lib` `~0.0.58`) needs an API key created from the package ID plus the MD5 and SHA256 signing hashes. **Debug and production need different API keys.** Debug hashes come from `vpm info <package_id>` (Signers info) on the device. Production uses the **Appstore certificate hashes** from the Developer Console. The production key is also required for IAP and Live App Testing. It goes in the manifest: [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-identity-lwa-lib) [Vega 2.0]
  ```toml
  [wants]
  [[wants.service]]
  id = "com.amazon.identity.lwa.authorizationmanager.service"

  [[extras]]
  key = "api_key"
  value = "xyz"
  ```
- The LWA API key is a JWT. Decoding it gives the `clientID` used by backend services. (It is an identifier, not a secret, but it is stored in the manifest in clear.) [source](https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-identity-lwa-lib) [Vega 2.0]
- Apps that use Login with Amazon or Amazon Device Messaging must associate a Security Profile during submission. [source](https://developer.amazon.com/docs/app-submission/submitting-apps-to-amazon-appstore.html) [Appstore, unversioned]

---

## 2. Packaging, signing and package ID

- **Package ID rules** (`[package] id`): unique across the Amazon Appstore, and it **must not be changed** or updates break. Allowed characters are `A-Z a-z 0-9 . _`. Use reverse DNS with at least 2 segments, each starting with a letter. Component IDs must use the package ID as a prefix (VPT validate error example). [source](https://developer.amazon.com/docs/vega/2.0/manifest-package) [Vega 2.0], [source](https://developer.amazon.com/docs/vega/2.0/vpt)
  ```toml
  [package]
  id = "com.foo.bar"           # valid; "de.fooz" valid
  title = "Foo App"            # or "@text/<id>" with assets/text/puff.json
  icon = "@image/foo_icon.png" # 512x512 PNG, max 1MB, under assets/image
  # invalid: "a_single_word", "1numeral.foo"
  ```
- For a shared Fire OS + Vega listing, the Vega package name must equal the Fire OS package name. [source](https://developer.amazon.com/docs/vega/2.0/app-submission) [Vega 2.0]
- A VPKG contains a digest (checksums of every file) and a detached signature over the digest. VPT can `pull`, `push`, `sign` (`--resign`), `verify`, `checksum` and `generate-csr`: [source](https://developer.amazon.com/docs/vega/2.0/vpt) [Vega 2.0]
  ```sh
  vega exec vpt pull --json-signature app.vpkg digest.json.sig
  vega exec vpt sign --resign digest.json.sig developer-private.key developer-cert.pem digest.json.sig
  vega exec vpt push --json-signature digest.json.sig app.vpkg
  vega exec vpt validate app.vpkg      # manifest + ABI checks "for AppStore submission requirements"
  vega exec vpt info app.vpkg --json   # id, version, build_number
  ```
- **Appstore re-signing:** "Amazon removes the signature you used to sign your app and re-signs it with an Amazon signature that is unique to you, does not change, and is the same for all apps in your account." The SHA-1, MD5 and SHA-256 hashes appear under App files > Appstore certificate hashes. [source](https://developer.amazon.com/docs/app-submission/understanding-submission) [Appstore, unversioned], [source](https://developer.amazon.com/docs/app-submission/add-release-details.html) [Appstore, unversioned]
- Build output: release VPKGs are at `build/<arch>-release/<app>_<arch>.vpkg`. The file to upload for submission is `build/armv7-release/<app>_armv7.vpkg` (`aarch64`/`x86_64` are for the virtual device). Debug or Release is chosen in Vega Studio or with `-b Debug|Release`. [source](https://developer.amazon.com/docs/vega/2.0/build-an-app) [Vega 2.0], [source](https://developer.amazon.com/docs/vega/2.0/cli-tools)
- Versioning: the version is a free-form string (`--build-version`). The build number is an increasing integer from 1 to 2^63-1 (`--build-number`). On update **both** must be greater than the previous submission. The Appstore uses the build number to block downgrades. Fire OS versionCode and Vega build number are checked independently. [source](https://developer.amazon.com/docs/vega/2.0/app-version) [Vega 2.0], [source](https://developer.amazon.com/docs/vega/2.0/app-submission)
  ```sh
  npx react-native build-vega --build-type Release --build-version 1.2.3 --build-number 101
  ```

---

## 3. Developer Mode: security implications

- Developer Mode allows sideloading and debugging, gives the developer shell and VDA access over network or USB, and allows package management and file-system access. Enabling it requires **authentication with an Amazon Developer Account** and reboots the device. [source](https://developer.amazon.com/docs/vega/2.0/developer-mode) [Vega 2.0]
- Flow: Settings > My Fire TV > About > press select 7 times > Developer options > Developer Mode > Continue. The TV shows a 6-digit code. Then run `vega devmode login` (OAuth in browser) and `vega devmode enable-device --code <code>`. The device reboots. [source](https://developer.amazon.com/docs/vega/2.0/developer-mode) [Vega 2.0]. The CLI flow was announced 2026-08-06 [source](https://community.amazondeveloper.com/t/authenticate-and-enable-developer-mode-from-the-cli/28885)
- "You can only sideload non-Amazon packages (packages without the `com.amazon` prefix)." [source](https://developer.amazon.com/docs/vega/2.0/developer-mode) [Vega 2.0]
- States are enable, authenticating and disable. "Enable" gives vsh shell, vpm package management and component debugging. Disabling also reboots. A factory reset deregisters the device and disables Developer Mode. [source](https://developer.amazon.com/docs/vega/2.0/developer-mode) [Vega 2.0]
- The default dev shell can only reach `/tmp`. Component shells (`vega exec vda shell -t component-id <id>`) see that app's dirs (`/scratch`). [source](https://developer.amazon.com/docs/vega/2.0/developer-mode) [Vega 2.0]
- The Developer Mode shell cannot read `/proc/net` and cannot turn on VoiceView through `vdcm` ("No permission for operation"). [source](https://developer.amazon.com/docs/vega/2.0/vega-release-notes) [Vega 2.0]
- Vega Studio debug sessions write `/data/shared_preferences.json` on the device. [source](https://developer.amazon.com/docs/vega/2.0/build-an-app) [Vega 2.0]
- Amazon (2026-08-06): "We're also working to make app sideloading more secure by introducing certificate-based app signing from the CLI." [source](https://community.amazondeveloper.com/t/authenticate-and-enable-developer-mode-from-the-cli/28885)

---

## 4. Privacy

- **Privacy policy:** If the app or any third-party SDK collects personal data, it must: [source](https://developer.amazon.com/docs/policy-center/privacy-security) [Appstore, unversioned]
  - collect only the data it needs;
  - show a legally adequate privacy policy **in the app and on the detail page**;
  - get consent;
  - use a prominent in-flow disclosure plus affirmative consent for unexpected collection;
  - handle data securely (HTTPS).

  Personal data is defined broadly. It includes authentication info, microphone/camera data, location, and sensitive device or usage data.
- Permissions must be requested only when needed and in context. [source](https://developer.amazon.com/docs/policy-center/privacy-security) [Appstore, unversioned]
- Third-party SDKs: within 30 days of Amazon's request you must disclose integrated external programs. If Amazon asks you to remove one, you must comply within 6 months. [source](https://developer.amazon.com/docs/policy-center/privacy-security) [Appstore, unversioned]
- **User data privacy section / privacy labels:** If you answer Yes to "Does your app collect user data or transfer user data to third-parties?", you must give a privacy policy URL and complete the questionnaire. The answers drive privacy labels on Fire TV, tablets and Amazon.com. [source](https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details) [Appstore, unversioned]
  - Since **Q2 2025**, new apps and updates cannot be published with an incomplete questionnaire.
  - Third-party SDK collection must be included.
  - Anonymous, non-user-associated data need not be disclosed.
  - Data categories: User Info, Financial, Device or other IDs (incl. Advertising ID), Health & Fitness, Location, Web Browsing, Photos/Videos, Audio, App info & performance (crash logs, diagnostics), App activity, Messages, Files, Calendar, Contacts.

  [source](https://developer.amazon.com/docs/app-submission/appstore-privacy-labels) [Appstore, unversioned]
- **Children / COPPA:** "Children" means under 13, or under 16 in the EU, Australia and Japan. Child-directed apps: [source](https://developer.amazon.com/docs/policy-center/privacy-children) [Appstore, unversioned]
  - must have age-appropriate content and comply with COPPA;
  - **cannot use Amazon Advertising/Associates** to deliver ads to children;
  - may only use "child-suitable" SDKs to collect personal information.

  Mixed-audience apps count as child-directed unless you confirm children are not using the app. They can use non-child-suitable SDKs only behind measures such as a neutral age screen.
- Target audience is declared under Pricing and Availability. The tiers are 0-12 (or 13-15 in some marketplaces), mixed, and 16+. [source](https://developer.amazon.com/docs/app-submission/add-pricing-and-availability-details) [Appstore, unversioned]
- Content rating questionnaire "Additional Information" flags: account creation/personal info collected, advertisements, gambling, location detection, UGC/user-to-user communication. [source](https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details) [Appstore, unversioned]
- US state age laws (Texas SB 2420, Utah SB 142, Louisiana HB 570): you must report "significant changes" in the update flow. The effective date must be within 90 days and at least 24 h out. The GetUserAgeData API is documented for **Fire tablets** through an Android ContentProvider. [source](https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details) [Appstore, unversioned], [source](https://developer.amazon.com/docs/app-submission/user-age-verification) [Appstore, unversioned]
- The Vega third-party services list includes CMP options (Didomi, OneTrust) and analytics/ads SDKs. Each must be disclosed in the privacy questionnaire. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) [Vega 2.0]

---

## 5. Appstore submission for Vega

### 5.1 Prerequisites
- You need: [source](https://developer.amazon.com/docs/vega/2.0/app-submission) [Vega 2.0], [source](https://developer.amazon.com/docs/app-submission/submitting-apps-to-amazon-appstore.html) [Appstore, unversioned]
  - a VPKG binary;
  - the manifest **must** register `com.amazon.category.main` on an interactive component, or the app can't launch from Home and "will fail the Amazon Appstore submission process";
  - a free developer account;
  - image assets and descriptions;
  - **a physical device for testing**.
- Run `vega exec vpt validate` first. It checks the manifest (e.g. a missing main category or a bad component ID prefix) and native ABI. [source](https://developer.amazon.com/docs/vega/2.0/vpt) [Vega 2.0]
- Vega VPKG submissions have been supported since 2025-09-30. [source](https://developer.amazon.com/docs/app-submission/release-notes.html) [Appstore, unversioned]

### 5.2 Steps in the Developer Console
1. Dashboard > **Add a New App**, then save the New App Submission page. [source](https://developer.amazon.com/docs/app-submission/submitting-apps-to-amazon-appstore.html)
2. **App Setup** (each section needs a green check before you can submit): [source](https://developer.amazon.com/docs/app-submission/submitting-apps-to-amazon-appstore.html)
   - **General Information:** console-only App title, optional SKU (max 150 chars, `a-zA-Z0-9_.-`), category, language support, support email, phone and website. [source](https://developer.amazon.com/docs/app-submission/add-general-information)
   - **Store Listing:**
     - Display title.
     - Short description: max 2,000 bytes. Fire TV shows the first 200 chars. Plain text only; HTML "will fail testing".
     - Long description: max 4,000 chars, not shown on Fire TV.
     - 3-5 feature bullets and optional keywords.

     [source](https://developer.amazon.com/docs/app-submission/add-store-listing-information)
   - **Pricing and Availability:**
     - Free or Paid. The minimum is USD 0.99 (EUR 0.69, GBP 0.59, JPY 85, INR 10.12, etc.) and the maximum is USD 999.99.
     - Countries default to all.
     - Target audience.
     - Paid apps and IAP require payment, bank and tax info.

     [source](https://developer.amazon.com/docs/app-submission/add-pricing-and-availability-details)
   - **Policies and Compliance:** content rating questionnaire, user data privacy and privacy policy URL, and IP documentation if requested (a signed letter or license; screenshots not accepted). [source](https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details)
3. **Release Tracks** > Create a release, choosing Fire OS, Vega OS or both. [source](https://developer.amazon.com/docs/app-submission/add-release-details.html)
   - Upload the VPKG (alias "VPKG1").
   - Under Supported devices, open the **Amazon Fire TV** tab > **Amazon Vega TV** section and tick the devices.
   - Optionally set a release date and time. Publishing starts then and takes a few hours.
   - Add **Testing instructions**. Include test credentials if login is required: "If Amazon cannot successfully launch your app during testing, the app will be rejected."
   - Certify **Export compliance**.
4. **Submit for review.** Statuses: [source](https://developer.amazon.com/docs/app-submission/add-release-details.html)
   - In progress, Submitted (cancellable), In review, Approved, Live rollout (staged %), Live on Appstore.
   - Pending action: an email asks for fixes.
   - Rejected: a new version is required.
   - Suppressed.
   - Approval email goes to the **account owner** address only.
- IAP items go on the In-App Items tab before submission (Consumable, Entitlement, Subscription; SKU max 150 chars, unique across all your apps). [source](https://developer.amazon.com/docs/vega/2.0/vega-submit-iap) [Vega 2.0]

### 5.3 Required Fire TV assets (Fire TV tab, "used for both Fire OS and Vega OS")
Source: [add-store-listing-information](https://developer.amazon.com/docs/app-submission/add-store-listing-information) [Appstore, unversioned]

| Asset | Required | Size | Format | Notes |
|---|---|---|---|---|
| App Icon | Yes | 1280 x 720 px | PNG, no transparency | Keep the subject and title within the 882 x 448 px safe area (badges overlay it) |
| Screenshots | Yes, 3-10 | 1920 x 1080 px landscape | JPG or 24-bit PNG, no transparency | No personal data (use dummy data). On Vega, capture with `gwsi-tool-screenshooter` |
| Background Image | Yes | 1920 x 1080 px landscape | JPG or 24-bit PNG, no transparency | Subject on the right, inside the 1214 x 830 px safe area; no logos or titles |
| Feature Rotator | Ad product | n/a | n/a | Exclusive to entertainment advertisers |

- Fire Tablet tab, for the tablet listing and the Amazon website: small icon 114 x 114 and large icon 512 x 512 PNG (transparent), 3-10 screenshots, optional promo image 1024 x 500, and up to 5 videos. [source](https://developer.amazon.com/docs/app-submission/add-store-listing-information) [Appstore, unversioned]
- The Live App Testing field list names "Small icon" and "Fire TV assets" as separate store fields. [source](https://developer.amazon.com/docs/vega/2.0/live-app-testing) [Vega 2.0]
- Test criteria 3.15 expects "a large icon (512 × 512 PNG with transparency)" and 3-10 screenshots. [source](https://developer.amazon.com/docs/app-testing/test-criteria) [Appstore, unversioned]
- The in-package icon (manifest `[package] icon`) is a separate thing: 512 x 512 PNG, max 1 MB, shown in Settings > Applications. [source](https://developer.amazon.com/docs/vega/2.0/manifest-package) [Vega 2.0]
- Size limits: uploads can be up to 2.5 GB, and you get a warning if over 50 MB or +10% from the last upload. [source](https://developer.amazon.com/docs/app-submission/understanding-submission). On Fire TV, 2 GB or less is recommended and installed size must be at most 4 GB. [source](https://developer.amazon.com/docs/app-testing/test-criteria)

### 5.4 What Amazon tests (common rejection causes)
- Approval checks that the app "Works as outlined in your product description; Does not impair the functionality of the device; **Does not put customer data at risk**; Complies with" the PML agreement and Content Policy. [source](https://developer.amazon.com/docs/app-submission/add-release-details.html) [Appstore, unversioned]
- Vega-specific tests that the Appstore runs (from the test-before-submission page): [source](https://developer.amazon.com/docs/vega/2.0/test-before-submission) [Vega 2.0]
  - exit with Back and relaunch, for VOD and live;
  - exit with Home and relaunch (no audio leak on the launcher);
  - switch between apps (no audio overlap, crash, blank screen or A/V desync);
  - HDMI hot-plug: unplug for 5 min, then playback paused and resumes from the same position;
  - change HDMI input or turn the TV off: paused and resumes;
  - relaunch after device restart: **login details must be retained**;
  - screensaver/ambient after 5 min idle: paused and no audio (audio apps continue);
  - Alexa interruption: pause, then resume after the card, for VOD, ads and live;
  - Bluetooth audio routing and sync.
  - "you must test your app on a Fire TV Stick before submitting". Vega Virtual Device is not enough.
- General Fire TV criteria (written for Fire OS but referenced from the Vega page): [source](https://developer.amazon.com/docs/app-testing/test-criteria) [Appstore, unversioned]
  - show a loading indicator if loading takes more than 15 s;
  - every sign-in option must work;
  - no FPS below 25 for extended periods;
  - eMMC writes under 50 MB/hr in the foreground and 2 MB/hr in the background;
  - use 90% of the screen (80% minimum, centered);
  - readable from about 10 ft;
  - minimal text entry;
  - visible focus highlight;
  - no touch UI;
  - no navigation traps;
  - pause on HDMI disconnect;
  - user data must persist across restart and update (including sign-in credentials);
  - remote buttons behave correctly;
  - metadata images correct.
- Remove debug artefacts before release: `proxy-config.json` and `assets/raw/ca.pem` from Network Proxy. [source](https://developer.amazon.com/docs/vega/2.0/network-proxy) [Vega 2.0]

### 5.5 Review times
- "If after waiting for six days your app is still in review, you can optionally use the Contact Us form". [source](https://developer.amazon.com/docs/app-submission/add-release-details.html) [Appstore, unversioned]
- Metadata-only updates review faster than binary updates. [source](https://developer.amazon.com/docs/app-submission/update-published-app) [Appstore, unversioned]
- No official SLA was found.

### 5.6 Live App Testing (LAT) on Vega
- LAT distributes the app to pre-defined testers against **production** Amazon services (IAP, ADM and others). You can run up to 5 parallel test tracks ("Test 1"–"Test 5"). Use separate tracks to test the Fire OS and Vega binaries. A passing version can be promoted to live. [source](https://developer.amazon.com/docs/vega/2.0/live-app-testing-understanding) [Vega 2.0], [source](https://developer.amazon.com/docs/vega/2.0/live-app-testing)
- Required LAT fields: app file, supported devices, app title, category, app icon, language, content rating, export compliance. Defaults are availability = all countries, pricing = Free. LAT does **not** collect user data privacy, target audience or Fire TV assets. Complete those after promotion. [source](https://developer.amazon.com/docs/vega/2.0/live-app-testing) [Vega 2.0]
- LAT and IAP builds need the production LWA API key built from the Appstore hashes (see 1.6).
- The App Submission API for Testing (REST) can register an app for Appstore DevTest and add testers. [source](https://developer.amazon.com/docs/vega/2.0/app-submission-api-testing) [Vega 2.0]

### 5.7 Release tracks with Fire OS
- Recommended: **one listing with release tracks**. It gives one ASIN and shared metadata, pricing, IAP items, reviews and entitlements, with binaries managed separately. Requirements: the same package name in both binaries, and Vega IAP libraries if the app uses IAP. [source](https://developer.amazon.com/docs/vega/2.0/app-submission) [Vega 2.0]
- Not recommended: a separate Vega listing. It gives a separate listing, reviews and pricing, and entitlements are **not interchangeable**, so customers pay twice. [source](https://developer.amazon.com/docs/vega/2.0/app-submission) [Vega 2.0]
- Release tracks in the submission flow date from 2026-11-16. [source](https://developer.amazon.com/docs/app-submission/release-notes.html)
- To add Fire OS to a Vega-only live app: "Add upcoming version" on the single track. [source](https://developer.amazon.com/docs/app-submission/update-published-app)
- Two binaries can't target the same device. [source](https://developer.amazon.com/docs/app-submission/add-release-details.html)

### 5.8 Availability and updates
- Countries default to all. Devices not sold in the selected countries are marked unsupported. [source](https://developer.amazon.com/docs/app-submission/add-pricing-and-availability-details) [Appstore, unversioned]
- Updates need a new build number. Staged rollout lets you pick an initial percentage. [source](https://developer.amazon.com/docs/app-submission/update-published-app) [Appstore, unversioned]
- Force-update guidance: [source](https://developer.amazon.com/docs/app-submission/update-published-app) [Appstore, unversioned]
  - wait at least 24 h after the update goes live;
  - enforce only after a 100% rollout;
  - scope it per release track.
- Content ratings change only with a new binary. [source](https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details)

---

## 6. Supply-chain hygiene (general guidance, not Vega-specific) [General]
- Use `npm ci` in CI/release builds. It requires `package-lock.json`, "will exit with an error, instead of updating the package lock" on mismatch, never writes `package.json` or lockfiles, and deletes `node_modules` first. `--ignore-scripts` stops install scripts from running. [source](https://docs.npmjs.com/cli/v11/commands/npm-ci)
- `npm audit` sends the dependency tree to the registry and reports known vulnerabilities. `--audit-level=<level>` sets the failure threshold without filtering output. `--omit=dev` leaves dev dependencies out. `npm audit signatures` verifies registry signatures **and provenance attestations**. [source](https://docs.npmjs.com/cli/v11/commands/npm-audit)
- Vega-specific angles:
  - Prefer Amazon's `@amazon-devices/*` ports at the versions in the supported-libraries table, with RN 0.83 vs 0.72 tabs. [source](https://developer.amazon.com/docs/vega-api/2.0/supported-libraries) [Vega 2.0]
  - Some SDKs are distributed outside npm, e.g. an AppsFlyer download link on Google Drive and access by email for Bitmovin or Conviva. Pin and checksum these yourself (inference).
  - Source maps (`*.bundle.map`) are excluded from the VPKG automatically. [source](https://developer.amazon.com/docs/vega/2.0/vpt) [Vega 2.0]
  - Amazon may ask you to disclose third-party SDKs within 30 days. [source](https://developer.amazon.com/docs/policy-center/privacy-security)

---

## 7. Accessibility
- Vega supports **VoiceView** (screen reader). Toggle it by holding **Back + Menu for 2 s**. It has Standard and Enhanced navigation modes. [source](https://developer.amazon.com/docs/vega/2.0/assistive-technologies-vega) [Vega 2.0]
- Use the RN accessibility props `accessibilityLabel`, `accessibilityRole`, `accessibilityState` and `accessibilityHint`, and use `AccessibilityInfo` to detect the screen reader. A Vega Studio Accessibility Inspector and a WebView accessibility guide exist. [source](https://developer.amazon.com/docs/vega/2.0/accessibility-vega) [Vega 2.0]
- Captions use the W3C text-track APIs in `@amazon-devices/react-native-w3cmedia` 2.1.14 or later. They need `[[wants.privilege]] id = "com.amazon.devconf.privilege.accessibility"` and the `VTTCue` polyfill. [source](https://developer.amazon.com/docs/vega/2.0/implement-closed-captions-vega) [Vega 2.0]
- No Vega or Appstore page found makes VoiceView support or captions an explicit **submission requirement**. See open questions.

---

## Unverified / open questions
- **Certificate pinning, TLS minimum version, and cleartext policy for RN `fetch`/XHR** (outside WebView): no Vega 2.0 page found. The cleartext manifest policy is documented only for WebView 4.0.2+. A Vega Studio "Enable SSL Proxy" option and a `ca.pem` in `assets/raw` suggest app-level CA trust config exists for debugging, but the mechanism is not documented.
- **Secure key storage:** it is unclear whether Vega Crypto keys can be persisted or are hardware-backed (TEE). There is no keychain/keystore-style API in the public list. Storing an MMKV `encryptionKey` securely has no documented solution.
- **Full list of privilege IDs:** there is no consolidated public list. The IDs above are collected from individual pages.
- The search-snippet claim "don't mention 'Vega' in any metadata field" could not be confirmed on the 0.21, 0.22 or 2.0 submission pages fetched. Treat it as unverified.
- Whether a Vega-only Fire TV listing still requires the tablet-tab 114 px/512 px icons. LAT lists "Small icon" separately, and test criteria 3.15 says a 512 x 512 large icon is required. It is ambiguous for Vega-only listings.
- The certificate-based CLI signing for sideloading (announced 2026-08-06) has no shipped doc page yet. Who signs debug VPKGs by default (the SDK's developer key?) is not explicitly documented. `vpm info` "Signers info" shows the debug signers.
- Whether the GetUserAgeData API or age signals exist on Vega/Fire TV. The docs cover Fire tablets and an Android ContentProvider only.
- Whether the Appstore SDK PEM file / Amazon DRM applies to VPKGs. The DRM field is "on the Fire OS track", and the PEM instructions cover Android and Unity only.
- Accessibility as a hard review gate. No explicit requirement was found, only guidance.
- The Live App Testing FAQ content is client-rendered and was not retrieved.
- Review-time SLA: only the "six days then contact us" guidance exists.

## Sources
- https://developer.amazon.com/docs/vega/2.0/app-submission
- https://developer.amazon.com/docs/vega/0.22/app-submission
- https://developer.amazon.com/docs/vega/0.21/app-submission.html
- https://developer.amazon.com/docs/vega/2.0/app-security-overview
- https://developer.amazon.com/docs/vega/2.0/request-runtime-privileges
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_security-manager-lib
- https://developer.amazon.com/docs/vega/2.0/app-manifest
- https://developer.amazon.com/docs/vega/2.0/manifest-package
- https://developer.amazon.com/docs/vega/2.0/manifest-needs
- https://developer.amazon.com/docs/vega/2.0/manifest-wants
- https://developer.amazon.com/docs/vega/2.0/develop-your-app-with-webview
- https://developer.amazon.com/docs/vega-api/2.0/webview-component-reference
- https://developer.amazon.com/docs/vega/2.0/network-proxy
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplercrypto
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_kepler-file-system
- https://developer.amazon.com/docs/vega-api/2.0/react-native-async-storage
- https://developer.amazon.com/docs/vega-api/2.0/react-native-mmkv
- https://developer.amazon.com/docs/vega-api/2.0/react-native-net-info
- https://developer.amazon.com/docs/vega-api/2.0/supported-libraries
- https://developer.amazon.com/docs/vega-api/2.0/vega-apis-overview
- https://developer.amazon.com/docs/vega-api/2.0/README.amazon-devices_keplerscript-identity-lwa-lib
- https://developer.amazon.com/docs/vega/2.0/vpt
- https://developer.amazon.com/docs/vega/2.0/app-version
- https://developer.amazon.com/docs/vega/2.0/build-an-app
- https://developer.amazon.com/docs/vega/2.0/cli-tools
- https://developer.amazon.com/docs/vega/2.0/developer-mode
- https://developer.amazon.com/docs/vega/2.0/vega-release-notes
- https://developer.amazon.com/docs/vega/2.0/test-before-submission
- https://developer.amazon.com/docs/vega/2.0/media-player-requirements
- https://developer.amazon.com/docs/vega/2.0/live-app-testing
- https://developer.amazon.com/docs/vega/2.0/live-app-testing-understanding
- https://developer.amazon.com/docs/vega/2.0/vega-submit-iap
- https://developer.amazon.com/docs/vega/2.0/app-submission-api-testing
- https://developer.amazon.com/docs/vega/2.0/accessibility-vega
- https://developer.amazon.com/docs/vega/2.0/assistive-technologies-vega
- https://developer.amazon.com/docs/vega/2.0/implement-closed-captions-vega
- https://developer.amazon.com/docs/app-submission/submitting-apps-to-amazon-appstore.html
- https://developer.amazon.com/docs/app-submission/add-general-information
- https://developer.amazon.com/docs/app-submission/add-store-listing-information
- https://developer.amazon.com/docs/app-submission/add-pricing-and-availability-details
- https://developer.amazon.com/docs/app-submission/add-policy-and-compliance-details
- https://developer.amazon.com/docs/app-submission/add-release-details.html
- https://developer.amazon.com/docs/app-submission/understanding-submission
- https://developer.amazon.com/docs/app-submission/update-published-app
- https://developer.amazon.com/docs/app-submission/release-notes.html
- https://developer.amazon.com/docs/app-submission/appstore-privacy-labels
- https://developer.amazon.com/docs/app-submission/user-age-verification
- https://developer.amazon.com/docs/policy-center/privacy-security
- https://developer.amazon.com/docs/policy-center/privacy-children
- https://developer.amazon.com/docs/app-testing/test-criteria
- https://community.amazondeveloper.com/t/authenticate-and-enable-developer-mode-from-the-cli/28885
- https://docs.npmjs.com/cli/v11/commands/npm-ci
- https://docs.npmjs.com/cli/v11/commands/npm-audit
