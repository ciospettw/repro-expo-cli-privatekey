# Repros: Expo dev-client code-signing error reporting

A single minimal Expo app (`create-expo-app --template blank` + `expo-updates` + `expo-dev-client`) that reproduces two related issues found while debugging a code-signing-enabled dev client:

1. **[cli]** `expo start --help` documents a `--private-key-path` default that isn't implemented, so signing a dev manifest without the flag fails with an unexpected HTTP 500. *(headless, curl only)*
2. **[dev-launcher]** that 500 is reported in the launcher UI as "Failed to connect", hiding the server's real error. *(dev client)*

Related PRs: expo/expo#47795 (docs fix), #47797 (surface the error).

> Note: an earlier third item (mapping `exp://` tunnel hosts to `https`) was investigated and **retracted** — the dev-client tunnel flow already uses `https` via `constructDevClientUrl`, and changing the `exp://`→`http` convention is better handled by #47255. See expo/expo#47798.

## Setup

```bash
npm install
```

Code-signing files are committed on purpose (`certs/certificate.pem` + `certs/private-key.pem`, generated with `expo-updates codesigning:generate`). They are **throwaway test keys**; do not reuse them. `app.json` contains:

```json
"updates": {
  "codeSigningCertificate": "./certs/certificate.pem",
  "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
}
```

---

## Issue 1 — `--private-key-path` help documents a default that doesn't exist (headless)

`expo start --help` says `--private-key-path` defaults to *"private-key.pem in the same directory as the certificate"*. It doesn't — for project code signing the flag is required, and omitting it returns a 500.

```bash
npx expo start --port 8099
# then, in another terminal:
curl -s -D - \
  -H 'expo-platform: ios' -H 'expo-dev-client: true' \
  -H 'expo-expect-signature: sig, keyid="main", alg="rsa-v1_5-sha256"' \
  http://localhost:8099/
```

**Actual:** `HTTP 500` `{"error":"CommandError: Must specify --private-key-path argument to sign development manifest for requested code signing key"}`.
**With the flag it works:**
```bash
npx expo start --port 8099 --private-key-path certs/private-key.pem
# → 200 OK with an expo-signature: keyid="main", … header
```

The behavior (requiring the flag) is intentional and covered by an existing test; the **help text** is what's inaccurate. Fix: expo/expo#47795.

---

## Issue 2 — dev launcher masks the 500 as "Failed to connect" (dev client)

Build a development build and connect it to the server from Issue 1 (started **without** `--private-key-path`).

```bash
npx expo run:ios      # or run:android — produces a development build
# in a separate terminal:
npx expo start        # NO --private-key-path, so signed manifests 500
```

In the dev launcher UI, open the dev server via the **server list** or **"Enter URL manually"**.

**Actual:** the launcher shows a generic connectivity error (on SDK 55 literally `Failed to connect to <url>`), even though the server answered with a 500 and a precise body.
**Expected:** the launcher surfaces the server's message, e.g. *"The development server responded with status code 500: Must specify --private-key-path…"*.

The manifest parser (`EXDevLauncherManifestParser.m` / `DevLauncherManifestParser.kt`) discards the response body on a non-2xx status. Fix: expo/expo#47797. Observed on an iOS 26.5 physical device.

---

## Environment

```
  expo-env-info 2.1.0 environment info:
    System: OS: macOS 26.0.1
    Binaries: Node 22.17.0, npm 10.9.2
    Managers: CocoaPods 1.16.2
    IDEs: Xcode 26.3/17C529
    npmPackages: expo ~57.0.6, expo-updates ~57.0.7, expo-dev-client ~57.0.6, react 19.2.3, react-native 0.86.0
    Expo Workflow: managed
```

`npx expo-doctor@latest`: **20/20 checks passed.**
