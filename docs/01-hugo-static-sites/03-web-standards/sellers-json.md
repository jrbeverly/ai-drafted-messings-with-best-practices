# sellers.json

IAB standard for ad tech platforms to disclose their sellers, enabling supply chain transparency.

## Why It Matters

- Lets ad buyers verify seller legitimacy across the supply chain
- Works with ads.txt: publisher declares sellers, platform confirms via sellers.json
- Combats unauthorized ad inventory reselling

## Do You Need It?

**Most publishers: No.** This file is created by **ad tech platforms** (SSPs, ad exchanges, ad networks), not individual website owners. Check your ad platform's sellers.json to verify your listing.

## Format

```json
{
  "contact_email": "sellers@adplatform.com",
  "version": "1.0",
  "sellers": [
    {
      "seller_id": "pub-1234567890",
      "name": "Example Publisher LLC",
      "domain": "example.com",
      "seller_type": "PUBLISHER",
      "is_confidential": 0,
      "is_passthrough": 0
    }
  ]
}
```

## Seller Types

| Type | Meaning |
|------|---------|
| `PUBLISHER` | Direct content owner |
| `INTERMEDIARY` | Reseller or ad network |
| `BOTH` | Acts as both |

## Verification Flow

1. Buyer sees ad from `example.com`
2. Checks `example.com/ads.txt` -- finds `adplatform.com, pub-123, DIRECT`
3. Checks `adplatform.com/sellers.json` -- finds `pub-123` mapped to `example.com`
4. Seller is verified as legitimate

## Serving Requirements

- **Location:** `/sellers.json` (root of ad platform domain)
- **MIME type:** `application/json; charset=utf-8`
- **Required fields:** `contact_email`, `version`, `sellers` array

## Pitfalls to Avoid

- Wrong MIME type (`text/plain` instead of `application/json`)
- Invalid JSON (trailing commas, missing quotes)
- Missing required root fields (`contact_email`, `version`)
- Lowercase `seller_type` (`publisher` instead of `PUBLISHER`)

## Related

- [ads-txt.md](./ads-txt.md)
- [app-ads-txt.md](./app-ads-txt.md)
