# Web Monetization

Web Monetization API. Payment pointers. Content monetization. Interledger Protocol. Hugo implementation.

## Principle

Use the Web Monetization API to enable micropayments for content. Implement payment pointers for passive revenue generation. Provide alternative monetization beyond advertising.

## What is Web Monetization?

**Web Monetization:** Proposed W3C standard for streaming micropayments to websites

**Purpose:**
- Monetize content without ads
- Micropayments from visitors
- Privacy-respecting revenue model
- Alternative to paywalls

**How it works:**
1. Website includes payment pointer meta tag
2. Visitor has Web Monetization provider (e.g., Coil)
3. Provider streams micropayments while visitor browses
4. Publisher receives accumulated payments

**Specification:** https://webmonetization.org/

## Payment Pointer

### Meta Tag Syntax

```html
<meta name="monetization" content="$wallet.example.com/your-payment-pointer">
```

### Payment Pointer Formats

**Interledger Payment Pointer:**

```html
<!-- ILP payment pointer -->
<meta name="monetization" content="$ilp.uphold.com/your-id">

<!-- Coil payment pointer -->
<meta name="monetization" content="$coil.xrptipbot.com/your-id">

<!-- GateHub -->
<meta name="monetization" content="$ilp.gatehub.net/your-id">
```

### Getting a Payment Pointer

**Providers:**
1. **Uphold** - Create account, get ILP pointer
2. **GateHub** - Digital wallet with ILP support
3. **Coil** - Web Monetization provider (deprecated as of 2023, alternatives emerging)

## Hugo Implementation

### Basic Template

**layouts/partials/head/web-monetization.html:**

```go-html-template
{{/* Web Monetization */}}
{{ with .Site.Params.monetization }}
  <meta name="monetization" content="{{ . }}">
{{ end }}

{{/* Override per page */}}
{{ with .Params.monetization }}
  {{/* Page-specific payment pointer (e.g., for guest authors) */}}
  <meta name="monetization" content="{{ . }}">
{{ end }}
```

### Site Configuration

**config.toml:**

```toml
[params]
  monetization = "$ilp.uphold.com/your-payment-pointer"
```

### Per-Author Payment Pointers

**For guest authors who should receive payment:**

```yaml
---
title: "Guest Post: Hugo Performance"
author: Jane Doe
monetization: "$ilp.uphold.com/janedoe-pointer"
---
```

**Template:**

```go-html-template
{{ $monetization := .Site.Params.monetization }}

{{ with .Params.monetization }}
  {{ $monetization = . }}
{{ else }}
  {{/* Look up author's payment pointer */}}
  {{ with .Params.author }}
    {{ $authorPage := $.Site.GetPage (printf "/authors/%s" (. | urlize)) }}
    {{ with $authorPage }}
      {{ with .Params.monetization }}
        {{ $monetization = . }}
      {{ end }}
    {{ end }}
  {{ end }}
{{ end }}

{{ with $monetization }}
  <meta name="monetization" content="{{ . }}">
{{ end }}
```

### Revenue Splitting

**Split revenue between site and author:**

```go-html-template
{{/* Probabilistic revenue splitting */}}
{{ $sitePointer := .Site.Params.monetization }}
{{ $authorPointer := .Params.monetization }}

{{ if and $sitePointer $authorPointer }}
  {{/* 70% author, 30% site - using random selection */}}
  {{ $rand := (mod (now.UnixNano) 100) }}
  {{ if lt $rand 70 }}
    <meta name="monetization" content="{{ $authorPointer }}">
  {{ else }}
    <meta name="monetization" content="{{ $sitePointer }}">
  {{ end }}
{{ else }}
  {{ with $authorPointer }}
    <meta name="monetization" content="{{ . }}">
  {{ else }}
    {{ with $sitePointer }}
      <meta name="monetization" content="{{ . }}">
    {{ end }}
  {{ end }}
{{ end }}
```

**Note:** True revenue splitting requires JavaScript on the client side. The Hugo template approach above is a build-time approximation.

## JavaScript API

### Detecting Web Monetization

```html
<script>
if (document.monetization) {
  // Web Monetization is supported
  document.monetization.addEventListener('monetizationstart', () => {
    console.log('Payment started');
    // Unlock premium content, remove ads, etc.
  });

  document.monetization.addEventListener('monetizationprogress', (event) => {
    console.log('Payment received:', event.detail.amount, event.detail.assetCode);
  });
} else {
  // Web Monetization not available
  console.log('Web Monetization not supported');
}
</script>
```

### Exclusive Content for Monetized Visitors

```html
<script>
document.addEventListener('DOMContentLoaded', () => {
  const exclusiveContent = document.querySelector('.exclusive-content');

  if (document.monetization) {
    document.monetization.addEventListener('monetizationstart', () => {
      // Show exclusive content
      exclusiveContent.style.display = 'block';
    });
  }
});
</script>

<!-- Default: hidden -->
<div class="exclusive-content" style="display:none">
  <p>Thank you for supporting this site! Here's exclusive content...</p>
</div>
```

### Remove Ads for Paying Visitors

```html
<script>
if (document.monetization) {
  document.monetization.addEventListener('monetizationstart', () => {
    // Hide advertisements
    document.querySelectorAll('.ad-container').forEach(ad => {
      ad.style.display = 'none';
    });

    // Show thank you message
    document.querySelector('.monetization-thanks').style.display = 'block';
  });
}
</script>
```

## Best Practices

**✅ DO:**
- Include payment pointer on all pages
- Support per-author pointers for guest content
- Provide exclusive content as incentive
- Test with Web Monetization browser extension
- Communicate monetization to visitors
- Use as supplement, not primary revenue

**❌ DON'T:**
- Gate essential content behind monetization
- Use monetization as sole revenue model
- Forget to verify payment pointer works
- Include multiple monetization tags (only one per page)
- Rely on deprecated providers

## Guidelines

### Essential

**Minimum implementation:**
- Payment pointer meta tag
- Valid payment pointer from provider
- Single meta tag per page

### Recommended

**For better monetization:**
- Per-author payment pointers
- Exclusive content for supporters
- Thank-you messaging
- Revenue splitting
- Analytics tracking

### Advanced

**For maximum revenue:**
- JavaScript API integration
- Progressive content unlocking
- Ad removal for supporters
- Revenue sharing with authors
- Multiple provider support

## Benefits

Ad-Free Revenue. Monetize without advertising.

Privacy Respecting. No tracking or data collection.

Passive Income. Automatic micropayments from visitors.

User Experience. No intrusive ads or paywalls.

Standards Based. W3C proposed standard.

## Related

- [meta-tags-comprehensive.md](./meta-tags-comprehensive.md) - All HTML meta tags
- [rel-attributes.md](./rel-attributes.md) - Link relationship attributes
