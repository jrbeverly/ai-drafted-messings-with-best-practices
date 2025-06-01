# app-ads.txt

IAB extension of ads.txt for mobile app in-app advertising authorization.

## Why It Matters

- Combats ad fraud in mobile app inventory
- Required for AdMob, Unity Ads, and other in-app ad networks
- Hosted on developer website, discovered via app store listing

## Do You Need It?

**Yes** if your iOS/Android app shows programmatic ads. **No** if website-only or app has no ads.

## Format

Identical to ads.txt:

```text
google.com, pub-1234567890123456, DIRECT, f08c47fec0942fa0
unity3d.com, 1234567, DIRECT, 96cabb5fbdde37a7
contact=app-ads@example.com
```

## How Discovery Works

1. App store listing includes developer website URL
2. Crawler fetches `https://developer-website.com/app-ads.txt`
3. Entries are verified against ad network records

## Setup Steps

1. Create `app-ads.txt` with your ad network entries
2. Host at root of developer website (`/app-ads.txt`)
3. Ensure app store listing points to that website domain
4. Verify file returns 200 OK with `text/plain` MIME type

## Hugo Setup

Place at `static/app-ads.txt`. If sellers are identical to web ads, you can use the same content as `ads.txt`.

## ads.txt vs app-ads.txt

| | ads.txt | app-ads.txt |
|---|---------|-------------|
| For | Websites | Mobile apps |
| Location | `/ads.txt` | `/app-ads.txt` |
| Discovery | Domain itself | App store -> developer website |
| Format | Identical | Identical |

## Pitfalls to Avoid

- Developer website in app store does not match where file is hosted
- Using app bundle ID (`com.example.app`) instead of ad network domain
- File not at domain root
- Wrong MIME type

## Validation

- IAB validator: https://adstxt.guru/
- Google: https://apps.txt.admanager.google.com/

## Related

- [ads-txt.md](./ads-txt.md)
- [sellers-json.md](./sellers-json.md)
