# assetlinks.json (Android App Links)

JSON file at `/.well-known/assetlinks.json` that verifies your Android app is authorized to handle links to your domain.

## Why It Matters

- Enables Android App Links (deep links that open directly in your app, no disambiguation dialog)
- Required for Digital Asset Links verification between website and Android app
- Also used for Android Instant Apps and password credential sharing

## Location

`https://example.com/.well-known/assetlinks.json`

## Format

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.app",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90"
      ]
    }
  }
]
```

## Get Your SHA-256 Fingerprint

```bash
# From debug keystore
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android

# From release keystore
keytool -list -v -keystore my-release-key.keystore
```

Or from Google Play Console: Setup > App signing > SHA-256 certificate fingerprint.

## Hugo Setup

Place at `static/.well-known/assetlinks.json`.

## Serving Requirements

- **MIME type:** `application/json`
- **Protocol:** HTTPS required
- **Response:** 200 OK (not a redirect)
- **Access:** No authentication, publicly accessible

## Verification

- Google Digital Asset Links tool: https://developers.google.com/digital-asset-links/tools/generator
- `adb shell am start -a android.intent.action.VIEW -d "https://example.com/path"`

## Pitfalls to Avoid

- Wrong or missing SHA-256 fingerprint (use signing key, not upload key if using Play App Signing)
- Serving with redirect instead of direct 200 response
- Wrong MIME type (must be `application/json`)
- Requiring authentication to access the file
- Missing HTTPS (Android requires it)

## Related

- [well-known-directory.md](./well-known-directory.md)
- [apple-app-site-association.md](./apple-app-site-association.md)
