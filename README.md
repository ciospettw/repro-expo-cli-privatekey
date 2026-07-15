# Repros: Expo dev-client / CLI code-signing & error-reporting bugs

A single minimal Expo app (`create-expo-app --template blank` + `expo-updates` + `expo-dev-client`) that reproduces three related bugs found together while debugging a code-signing-enabled dev client:

1. **[cli]** `expo start` ignores the documented default for `--private-key-path` → signed dev manifests fail with HTTP 500. *(headless, curl only — no device needed)*
2. **[dev-launcher]** that 500 is reported to the user as "Failed to connect", hiding the server's real error. *(dev client)*
3. **[dev-launcher]** `exp://` tunnel URLs (`*.exp.direct`) are rewritten to cleartext `http://` and blocked by iOS ATS. *(dev client + tunnel)*

Related PRs: expo/expo#47795, #47797, #47798.

## Setup

```bash
npm install
```

Code-signing files are committed on purpose (`certs/certificate.pem` + `certs/private-key.pem`, generated with `expo-updates codesigning:generate`, sitting in the same directory — exactly the default `--private-key-path` location). They are **throwaway test keys**; do not reuse them. `app.json` contains:

```json
"updates": {
  "codeSigningCertificate": "./certs/certificate.pem",
  "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
}
```

---

## Bug 1 — `--private-key-path` default not implemented (headless)

`expo start --help` says `--private-key-path` defaults to *"private-key.pem in the same directory as the certificate"*. It doesn't.

```bash
npx expo start --port 8099
# then, in another terminal:
curl -s -D - \
  -H 'expo-platform: ios' -H 'expo-dev-client: true' \
  -H 'expo-expect-signature: sig, keyid="main", alg="rsa-v1_5-sha256"' \
  http://localhost:8099/
```

**Actual:** `HTTP 500` `{"error":"CommandError: Must specify --private-key-path argument to sign development manifest for requested code signing key"}` — even though `certs/private-key.pem` exists.
**Expected:** `200 OK` with an `expo-signature` header.

Proof the key/cert are valid (only the default is missing):
```bash
npx expo start --port 8099 --private-key-path certs/private-key.pem
# the same curl now returns 200 with: expo-signature: keyid="main", sig="…", alg="rsa-v1_5-sha256"
```

---

## Bug 2 — dev launcher masks the 500 as "Failed to connect" (dev client)

Build a development build and connect it to the server from Bug 1 (started **without** `--private-key-path`).

```bash
npx expo run:ios      # or run:android — produces a development build
# in a separate terminal:
npx expo start        # NO --private-key-path, so signed manifests 500
```

In the dev launcher UI, open the dev server via the **server list** or **"Enter URL manually"** (not a deep link).

**Actual:** the launcher shows a generic connectivity error — on SDK 55 literally `Failed to connect to <url>` — even though the server answered with a 500 and a precise body.
**Expected:** the launcher surfaces the server's message, e.g. *"The development server responded with status code 500: Must specify --private-key-path…"*.

Contrast: opening the **dev-client deep link** for the same URL surfaces the real error, because that code path (`EXDevLauncherController.onDeepLink`) includes `error.localizedDescription`, while the UI path (`DevLauncherViewModel.openApp`) discards it (`onError: { _ in "Failed to connect to \(url)" }`). Underneath both, the manifest parser (`EXDevLauncherManifestParser.m` / `DevLauncherManifestParser.kt`) discards the response body on a non-2xx status. Fix: expo/expo#47797.

Observed on an iOS 26.5 physical device with a code-signing-enabled development build.

---

## Bug 3 — `exp://` tunnel URLs downgraded to cleartext http (dev client)

```bash
npx expo start --dev-client --tunnel --private-key-path certs/private-key.pem
# note the tunnel host, e.g. xxxxx.exp.direct
```

On the device, open `exp://xxxxx.exp.direct` (manual entry or deep link).

**Actual (iOS):** fails to connect. `DevLauncherUrl` rewrites `exp://` → `http://` unconditionally; `*.exp.direct` is TLS-only, so the cleartext request is blocked by App Transport Security.
**Expected:** `exp://` tunnel hosts map to `https` (which is what Expo CLI already uses for tunnel manifest URLs in `UrlCreator.constructDevClientUrl`). Opening the same host as `https://xxxxx.exp.direct` works. Fix: expo/expo#47798.

---

## Environment

```
  expo-env-info 2.1.0 environment info:
    System:  OS: macOS 26.0.1
    Binaries: Node 22.17.0, npm 10.9.2
    Managers: CocoaPods 1.16.2
    IDEs: Xcode 26.3/17C529
    npmPackages: expo ~57.0.6, expo-updates ~57.0.7, expo-dev-client ~57.0.6, react 19.2.3, react-native 0.86.0
    Expo Workflow: managed
```

`npx expo-doctor@latest`: **20/20 checks passed.**
