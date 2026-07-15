# Repro: `expo start` ignores the documented default for `--private-key-path`

Minimal reproduction for: **`@expo/cli` does not implement the documented default for `--private-key-path`, so signed development manifest requests fail with HTTP 500 even when `private-key.pem` sits next to the configured certificate.**

`expo start --help` says:

```
--private-key-path <path>  Path to private key for code signing.
                           Default: "private-key.pem" in the same directory as the
                           certificate specified by the expo-updates configuration in app.json.
```

This project has exactly that layout (`certs/certificate.pem` + `certs/private-key.pem`, generated with `expo-updates codesigning:generate`), yet the default is never applied.

## Setup

```bash
npm install
```

The code signing files are committed on purpose so the repro is self-contained. They are **throwaway test keys** generated only for this repository — do not reuse them.

`app.json` already contains:

```json
"updates": {
  "codeSigningCertificate": "./certs/certificate.pem",
  "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
}
```

## Steps to reproduce

1. Start the dev server **without** `--private-key-path`:

   ```bash
   npx expo start --port 8099
   ```

2. In another terminal, request the manifest the way a code-signing-enabled client does (with an `expo-expect-signature` header):

   ```bash
   curl -s -D - \
     -H 'expo-platform: ios' \
     -H 'expo-dev-client: true' \
     -H 'expo-expect-signature: sig, keyid="main", alg="rsa-v1_5-sha256"' \
     http://localhost:8099/
   ```

### Actual (the bug)

```
HTTP/1.1 500 Internal Server Error
{"error":"CommandError: Must specify --private-key-path argument to sign development manifest for requested code signing key"}
```

…even though `certs/private-key.pem` (the documented default) exists.

### Expected

`200 OK` with an `expo-signature` header, because the CLI should default to `private-key.pem` next to the configured certificate.

## Proof the key/cert are valid (only the default is missing)

Passing the flag explicitly works:

```bash
npx expo start --port 8099 --private-key-path certs/private-key.pem
# → the same curl now returns 200 with:
# expo-signature: keyid="main", sig="…", alg="rsa-v1_5-sha256"
```

So the certificate and key are correct and matching — the only problem is that the documented default path is not resolved.

## Proposed fix

https://github.com/expo/expo/pull/47795 — resolve `privateKeyPath ?? path.resolve(path.dirname(codeSigningCertificatePath), 'private-key.pem')` before reading the key.
