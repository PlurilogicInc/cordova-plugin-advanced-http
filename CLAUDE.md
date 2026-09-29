# CLAUDE.md

Plurilogic fork of [silkimen/cordova-plugin-advanced-http](https://github.com/silkimen/cordova-plugin-advanced-http) (forked at upstream 3.3.1, now 3.3.4). It gives the Cordova app native HTTP on iOS (AFNetworking, vendored as `SM_AF*`), which avoids WKWebView CORS limits and supports SSL pinning and client certificates.

The consuming app targets **Android, iOS, macOS and Electron**. This fork ships native code **for iOS only**. Keep that in mind for every change and every call site.

## Platform support

| App target | Native code | What `cordova.plugin.http` does |
|---|---|---|
| iOS | `src/ios/` | Works |
| macOS | none (no `osx` platform in `plugin.xml`) | Works only if the macOS build is the iOS app running as Mac Catalyst or "Designed for iPad". With `cordova-osx` or any other macOS platform, native calls fail. |
| Android | **removed** (commit `f3ebce9`) | The JS object exists, but every native call fails (`CordovaHttpPlugin` class not found) |
| Electron | none | The JS object exists, but native calls fail or never call back |

**Gotcha:** the `<js-module>` entries in `plugin.xml` are not scoped to a platform, so `cordova.plugin.http` is defined on **every** platform. Checking `if (cordova.plugin.http)` does **not** tell you whether the plugin works. Branch on the platform instead:

```js
const useNativeHttp = window.cordova && cordova.platformId === 'ios';
// Otherwise use fetch/XMLHttpRequest (Android WebView, Electron),
// or Electron's main-process net module when CORS or cert pinning matters.
```

The plugin still declares `cordova-plugin-file` as a dependency, so installing it pulls that plugin into every platform.

## Changes from upstream

- Android and browser platforms removed (`f3ebce9`). The README still documents them, so ignore those parts.
- iOS `get/head/delete/options` and `post/put/patch` now run inside `commandDelegate runInBackground` (`2a71f2a`, changelog 3.3.3). `uploadFiles` and `downloadFile` do **not**; they still start on the calling thread.
- Default timeouts raised from 60 s to **90 s** (`www/global-configs.js`: `timeout`, `connectTimeout`, `readTimeout`).
- Version is kept the same in `package.json` **and** `plugin.xml`. Bump both.

## Using it from the app

The API is upstream's (see `README.md`, ignoring the Android/browser notes). It uses callbacks, not promises, and is exposed as `cordova.plugin.http`:

```js
cordova.plugin.http.setDataSerializer('json');       // 'urlencoded' (default) | 'json' | 'utf8' | 'multipart' | 'raw'
const reqId = cordova.plugin.http.sendRequest(url, {
  method: 'post', data, headers, responseType: 'json', timeout: 90
}, res => { /* res.status, res.data, res.headers, res.url */ },
   err => { /* err.status (HTTP status or ErrorCode), err.error */ });
cordova.plugin.http.abort(reqId);
```

- `sendRequest` and the method shortcuts (`get`, `post`, and so on) return a request id that you can pass to `abort()`.
- Negative `status` values are the plugin's own error codes (`cordova.plugin.http.ErrorCode`): `GENERIC -1`, `SSL_EXCEPTION -2`, `SERVER_NOT_FOUND -3`, `TIMEOUT -4`, `UNSUPPORTED_URL -5`, `NOT_CONNECTED -6`, `POST_PROCESSING_FAILED -7`, `ABORTED -8`.
- Global config (headers, serializer, timeouts, followRedirect) lives in one module-level object shared by all callers. `setRequestTimeout` sets all three timeouts. On iOS, only `readTimeout` is applied to the session (`setTimeout:readTimeout`), and the `*ConnectTimeout` functions only matter on Android.
- The plugin keeps cookies in its **own** tough-cookie jar persisted in `localStorage` (`www/cookie-handler.js`). They are not shared with WKWebView, `fetch`, or the other platforms' HTTP stacks.
- Trust modes: `setServerTrustMode('default' | 'nocheck' | 'pinned' | 'legacy')`. Pinned certificates are `.cer` files in the app's `www/certificates/`. Never ship `nocheck` in production builds.
- `uploadFile` and `downloadFile` take `file://` paths, which `cordova-plugin-file` provides. `downloadFile` succeeds with a `FileEntry`.

## Layout

- `www/`: JS layer, loaded as Cordova js-modules (CommonJS factory functions wired together in `www/advanced-http.js`).
  - `public-interface.js`: the public API and argument order passed to `exec`. The argument order must match `[command.arguments objectAtIndex:n]` in `CordovaHttpPlugin.m`.
  - `helpers.js`: validation, data serialization, response post-processing, and cookie injection.
  - `global-configs.js`: defaults. `umd-tough-cookie.js` and `lodash.js` are vendored bundles; do not edit them by hand.
- `src/ios/CordovaHttpPlugin.m`: native entry points (`get`, `post`, `uploadFiles`, `downloadFile`, `abort`, `setServerTrustMode`, `setClientAuthMode`, and others). A new `SM_AFHTTPSessionManager` is created per request.
- `src/ios/SM_AFNetworking/`: AFNetworking with prefixed symbols to avoid clashes with other plugins (upstream #459). Keep the `SM_` prefix if you update it.
- `plugin.xml`: every new `www/` or `src/ios/` file must be registered here, or it won't be installed.
- `temp/`: local scratch Cordova app, gitignored. `test/`: mocha specs and the e2e app template (still references Android).

## Commands

```bash
npm run test:js      # mocha unit tests for www/ (no device needed)
npm run build:ios    # build the e2e test app (needs Xcode, cordova CLI)
npm run test:ios     # refresh test certs, build, then run e2e (BrowserStack/Sauce creds)
```

`test:js` currently has one known failure on recent Node versions: `injectRawResponseHandler … post-processing fails` asserts on V8's old `Unexpected token N in JSON` message. It is not a regression.

`.github/workflows/ci.yml` and `.travis.yml` still include Android jobs, which will fail because the Android sources are gone.

## When editing

- If you change the JS↔native contract, update `public-interface.js` and `CordovaHttpPlugin.m` together. Also add a mocha spec in `test/js-specs.js` for JS-side behavior.
- iOS thread safety: `reqDict` (the request id → task map used by `abort`) is a plain `NSMutableDictionary` that is now touched from background threads. Guard it (for example with `@synchronized`) before adding more concurrent access.
- Behavior differences from upstream belong in `CHANGELOG.md` under a new version, with the version bumped in `package.json` and `plugin.xml`.
- The app consumes this plugin from the Plurilogic GitHub repo, not npm (`git+https://github.com/Plurilogic/cordova-plugin-advanced-http.git`). After pushing, re-add or update the plugin in the app (`cordova plugin rm/add`) to pick up changes.
