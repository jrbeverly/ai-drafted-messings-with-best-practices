# Apple App Site Association (AASA)

JSON file at `/.well-known/apple-app-site-association` that enables iOS Universal Links and Handoff between your website and iOS app.

## Why It Matters

- Enables Universal Links: tapping a link opens your iOS app directly instead of Safari
- Required for Handoff, Shared Web Credentials, and App Clips
- No file extension -- iOS expects this exact path

## Location

`https://example.com/.well-known/apple-app-site-association` (no `.json` extension)

## Format

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [
          { "/": "/products/*", "comment": "Product pages" },
          { "/": "/articles/*", "comment": "Articles" },
          { "/": "/account/*", "exclude": true, "comment": "Keep in Safari" }
        ]
      }
    ]
  }
}
```

## Find Your App ID

Format: `TEAMID.BundleID`

- **Team ID:** Apple Developer account > Membership > Team ID
- **Bundle ID:** Xcode project > General > Bundle Identifier
- Example: `A1B2C3D4E5.com.example.myapp`

## Legacy Format (iOS 12 and Earlier)

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.example.app",
        "paths": ["/products/*", "/articles/*", "NOT /account/*"]
      }
    ]
  }
}
```

## Hugo Setup

Place at `static/.well-known/apple-app-site-association` (no extension).

## Serving Requirements

- **MIME type:** `application/json` (Apple requires this)
- **Protocol:** HTTPS required (no HTTP fallback)
- **Response:** Direct 200 OK (Apple CDN fetches this; redirects may fail)
- **No authentication:** Must be publicly accessible
- **File size:** < 128 KB

## Verification

- Apple validation tool: https://search.developer.apple.com/appsearch-validation-tool/
- `curl -I https://example.com/.well-known/apple-app-site-association`

## Pitfalls to Avoid

- Adding `.json` extension to the filename (iOS expects no extension)
- Serving with a redirect (Apple's CDN may not follow redirects)
- Wrong MIME type (must be `application/json`)
- Requiring authentication to access the file
- Mismatched Team ID or Bundle ID (links will not open in app)
- File larger than 128 KB

## Related

- [well-known-directory.md](./well-known-directory.md)
- [assetlinks-json.md](./assetlinks-json.md)
