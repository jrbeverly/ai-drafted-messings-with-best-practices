# Skip Navigation Links

Skip navigation links. Skip to main content. Skip to search. Skip to footer. Bypass blocks. WCAG 2.4.1. Screen reader navigation. Keyboard accessibility. Visible on focus. Focus management. Hugo baseof.html. Skip link styling. Bypass repetitive content.

## Principle

Provide skip navigation links as the first focusable elements on every page so that keyboard and screen reader users can bypass repetitive content blocks (site header, navigation menus, sidebars) and jump directly to the primary content or other key regions. WCAG 2.1 Success Criterion 2.4.1 (Bypass Blocks, Level A) requires a mechanism to skip repeated navigation on every page. Skip links are the most common and reliable technique. In Hugo static sites, skip links belong in the `baseof.html` layout template so they appear on every generated page automatically without per-page configuration.

## The Skip to Main Content Link

### How It Works

A skip link is an anchor element placed as the very first focusable element in the document body. It links to the `id` of the `<main>` content area. Sighted keyboard users see the link appear when they press Tab for the first time. Screen reader users hear it announced immediately when navigating the page. Clicking or pressing Enter on the link moves focus to the main content, bypassing all header and navigation elements.

```html
<!-- Skip link: first element inside <body> -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<!-- Site header with navigation (skipped by the link) -->
<header>
  <nav aria-label="Primary">
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/blog">Blog</a></li>
      <li><a href="/about">About</a></li>
      <li><a href="/contact">Contact</a></li>
    </ul>
  </nav>
</header>

<!-- Target: main content receives focus when skip link is activated -->
<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
  <p>Content begins here.</p>
</main>
```

### Why tabindex="-1" on the Target

The skip link target (`<main>`) needs `tabindex="-1"` to receive programmatic focus when the link is activated. Without it, some browsers scroll to the target but do not move keyboard focus, which means the next Tab press returns to the top of the page instead of continuing from the main content.

```html
<!-- DO: tabindex="-1" allows the skip link to move focus to main -->
<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
</main>

<!-- DON'T: Missing tabindex -- focus may not move in some browsers -->
<main id="main-content">
  <h1>Page Title</h1>
</main>
```

**Browser behavior without tabindex="-1":**

| Browser | Scroll to target | Focus moves to target |
|---|---|---|
| Chrome | Yes | No (focus stays at skip link) |
| Firefox | Yes | Yes (focus moves correctly) |
| Safari | Yes | No (focus stays at skip link) |

Adding `tabindex="-1"` makes the behavior consistent across all browsers.

## Visible on Focus Pattern

### The CSS Pattern

Skip links are visually hidden by default and become visible only when they receive keyboard focus. This keeps the visual design clean for mouse users while ensuring the link is available for keyboard users.

```css
/* Skip link: hidden until focused */
.skip-link {
  position: absolute;
  top: -100%;
  left: 0.5rem;
  z-index: 10000;
  padding: 0.75rem 1.5rem;
  background-color: #0056b3;
  color: #ffffff;
  font-size: 1rem;
  font-weight: 700;
  text-decoration: none;
  border-radius: 0 0 4px 4px;
  transition: top 0.15s ease-in-out;
}

/* Visible when focused via keyboard */
.skip-link:focus {
  position: fixed;
  top: 0;
}

/* Focus indicator on the skip link itself */
.skip-link:focus-visible {
  outline: 3px solid #ffffff;
  outline-offset: 2px;
}
```

### Alternative CSS Approach Using sr-only Pattern

```css
/* Using the sr-only / clip pattern */
.skip-link {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
  /* Visual styles applied when visible */
  background-color: #0056b3;
  color: #ffffff;
  font-size: 1rem;
  font-weight: 700;
  text-decoration: none;
  z-index: 10000;
}

/* Reveal on focus */
.skip-link:focus {
  position: fixed;
  top: 0.5rem;
  left: 0.5rem;
  width: auto;
  height: auto;
  padding: 0.75rem 1.5rem;
  margin: 0;
  overflow: visible;
  clip: auto;
  white-space: normal;
  border-radius: 4px;
}
```

### Contrast Requirements for Skip Links

The skip link must meet contrast requirements when visible:

```css
/* Text contrast: white (#ffffff) on blue (#0056b3) = 7.2:1 -- passes AAA */
/* Focus indicator: white outline on blue background = 7.2:1 -- passes */

/* DON'T: Low-contrast skip link */
.skip-link {
  background-color: #f0f0f0;
  color: #999999;        /* 2.8:1 -- FAILS AA */
}

/* DO: High-contrast skip link */
.skip-link {
  background-color: #0056b3;
  color: #ffffff;        /* 7.2:1 -- passes AAA */
}
```

## Multiple Skip Links

### When Multiple Skip Links Are Needed

Sites with complex layouts benefit from multiple skip links that let users jump to different sections. Common targets include main content, search, navigation, and footer.

```html
<!-- Multiple skip links in a group -->
<div class="skip-links">
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <a href="#search" class="skip-link">Skip to search</a>
  <a href="#site-footer" class="skip-link">Skip to footer</a>
</div>
```

### Styling Multiple Skip Links

```css
/* Container for multiple skip links */
.skip-links {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 10000;
  display: flex;
  gap: 0.25rem;
  padding: 0.25rem;
}

/* Individual skip link */
.skip-links .skip-link {
  position: static;
  padding: 0.5rem 1rem;
  background-color: #0056b3;
  color: #ffffff;
  font-size: 0.875rem;
  font-weight: 700;
  text-decoration: none;
  border-radius: 4px;
  white-space: nowrap;
}

/* Show the entire container when any skip link receives focus */
.skip-links:focus-within {
  position: fixed;
  top: 0;
}

/* Individual link focus styling */
.skip-links .skip-link:focus-visible {
  outline: 3px solid #ffffff;
  outline-offset: 2px;
}
```

### Target Elements for Multiple Skip Links

```html
<body>

  <!-- Skip links (first focusable elements) -->
  <div class="skip-links">
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <a href="#search" class="skip-link">Skip to search</a>
    <a href="#site-footer" class="skip-link">Skip to footer</a>
  </div>

  <header>
    <nav aria-label="Primary">...</nav>

    <!-- Search target -->
    <search id="search" aria-label="Site search" tabindex="-1">
      <label for="search-input" class="sr-only">Search</label>
      <input type="search" id="search-input" name="q" placeholder="Search...">
      <button type="submit" aria-label="Submit search">
        <svg aria-hidden="true" focusable="false"><use href="#icon-search"></use></svg>
      </button>
    </search>
  </header>

  <!-- Main content target -->
  <main id="main-content" tabindex="-1">
    <h1>Page Title</h1>
    <p>Content begins here.</p>
  </main>

  <!-- Footer target -->
  <footer id="site-footer" tabindex="-1">
    <nav aria-label="Footer">...</nav>
    <p>&copy; 2026 Site Name</p>
  </footer>

</body>
```

## Hugo baseof.html Implementation

### Single Skip Link (Most Common)

**layouts/_default/baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ site.Language.Lang }}">
<head>
  {{ partial "head.html" . }}
</head>
<body>

  <a href="#main-content" class="skip-link">Skip to main content</a>

  <header>
    {{ partial "site-header.html" . }}
  </header>

  {{ partial "breadcrumb.html" . }}

  <main id="main-content" tabindex="-1">
    {{ block "main" . }}{{ end }}
  </main>

  {{ block "aside" . }}{{ end }}

  <footer id="site-footer">
    {{ partial "site-footer.html" . }}
  </footer>

</body>
</html>
```

### Multiple Skip Links with Hugo Conditional Logic

**layouts/_default/baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ site.Language.Lang }}">
<head>
  {{ partial "head.html" . }}
</head>
<body>

  <div class="skip-links">
    <a href="#main-content" class="skip-link">Skip to main content</a>
    {{ if site.Params.search.enabled }}
      <a href="#search" class="skip-link">Skip to search</a>
    {{ end }}
    <a href="#site-footer" class="skip-link">Skip to footer</a>
  </div>

  <header>
    {{ partial "site-header.html" . }}
    {{ if site.Params.search.enabled }}
      {{ partial "search.html" . }}
    {{ end }}
  </header>

  {{ partial "breadcrumb.html" . }}

  <main id="main-content" tabindex="-1">
    {{ block "main" . }}{{ end }}
  </main>

  {{ block "aside" . }}{{ end }}

  <footer id="site-footer" tabindex="-1">
    {{ partial "site-footer.html" . }}
  </footer>

</body>
</html>
```

### Search Partial with Skip Link Target

**layouts/partials/search.html:**

```go-html-template
<search id="search" aria-label="Site search" tabindex="-1">
  <form action="/search/" method="get">
    <label for="search-input" class="sr-only">Search articles</label>
    <input
      type="search"
      id="search-input"
      name="q"
      placeholder="Search..."
      autocomplete="off">
    <button type="submit" aria-label="Submit search">
      <svg aria-hidden="true" focusable="false">
        <use href="#icon-search"></use>
      </svg>
    </button>
  </form>
</search>
```

### Documentation Site with Section Skip Links

For documentation sites with sidebar navigation, a skip link to bypass the sidebar is especially valuable.

**layouts/docs/baseof.html:**

```go-html-template
<!DOCTYPE html>
<html lang="{{ site.Language.Lang }}">
<head>
  {{ partial "head.html" . }}
</head>
<body>

  <div class="skip-links">
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <a href="#docs-nav" class="skip-link">Skip to documentation navigation</a>
    <a href="#site-footer" class="skip-link">Skip to footer</a>
  </div>

  <header>
    {{ partial "site-header.html" . }}
  </header>

  <div class="docs-layout">
    <aside id="docs-nav" aria-label="Documentation navigation" tabindex="-1">
      {{ partial "docs-sidebar.html" . }}
    </aside>

    <main id="main-content" tabindex="-1">
      {{ block "main" . }}{{ end }}
    </main>

    {{ if .TableOfContents }}
      <nav aria-label="Table of contents" class="toc-sidebar">
        <h2 id="toc-heading">On this page</h2>
        <div aria-labelledby="toc-heading">
          {{ .TableOfContents }}
        </div>
      </nav>
    {{ end }}
  </div>

  <footer id="site-footer" tabindex="-1">
    {{ partial "site-footer.html" . }}
  </footer>

</body>
</html>
```

## Skip Link Behavior with JavaScript

### Ensuring Focus Moves to the Target

Some browsers scroll to the target element when a skip link is clicked but do not move keyboard focus. JavaScript can ensure consistent behavior.

```js
document.addEventListener('DOMContentLoaded', () => {
  const skipLinks = document.querySelectorAll('.skip-link');

  skipLinks.forEach(link => {
    link.addEventListener('click', (event) => {
      const targetId = link.getAttribute('href').substring(1);
      const target = document.getElementById(targetId);

      if (target) {
        // Ensure target is focusable
        if (!target.hasAttribute('tabindex')) {
          target.setAttribute('tabindex', '-1');
          target.addEventListener('blur', () => {
            target.removeAttribute('tabindex');
          }, { once: true });
        }

        // Move focus to the target
        target.focus();

        // Prevent the default scroll-without-focus behavior
        event.preventDefault();
      }
    });
  });
});
```

### Skip Link and Client-Side Navigation

When using Turbo, htmx, or other client-side navigation libraries, skip links need to work after page transitions.

```js
// Turbo: ensure skip links work after each navigation
document.addEventListener('turbo:load', () => {
  // Re-verify all skip link targets have tabindex="-1"
  const targets = ['main-content', 'search', 'site-footer'];
  targets.forEach(id => {
    const target = document.getElementById(id);
    if (target && !target.hasAttribute('tabindex')) {
      target.setAttribute('tabindex', '-1');
    }
  });
});
```

## Testing Skip Navigation

### Manual Testing

```
1. Basic Skip Link Functionality
   [ ] Press Tab on page load -- skip link is the first element to receive focus
   [ ] Skip link becomes visible when it receives focus
   [ ] Skip link text is descriptive ("Skip to main content")
   [ ] Skip link has sufficient contrast when visible (4.5:1 for text)
   [ ] Pressing Enter on skip link moves focus to main content area
   [ ] Next Tab press after skip link activation focuses the first interactive
       element inside main content (not the navigation)
   [ ] Skip link appears on every page of the site

2. Multiple Skip Links
   [ ] All skip links appear when tabbing through them
   [ ] Each skip link targets a valid, existing element on the page
   [ ] Each target element has tabindex="-1"
   [ ] Skip links disappear when they lose focus (only visible on focus)
   [ ] Tab order of skip links matches their visual order

3. Screen Reader Testing
   [ ] VoiceOver: skip link is announced as the first element on the page
   [ ] NVDA: skip link is announced in browse mode
   [ ] After activating skip link, screen reader reading position
       starts from the main content area
   [ ] Screen reader announces the target heading (h1) after skip link activation

4. Browser Testing
   [ ] Chrome: focus moves to target after skip link activation
   [ ] Firefox: focus moves to target after skip link activation
   [ ] Safari: focus moves to target after skip link activation
       (requires tabindex="-1" on target)

5. Visual Testing
   [ ] Skip link does not cause layout shift when it appears
   [ ] Skip link does not overlap important content when visible
   [ ] Skip link has clear focus indicator
   [ ] Skip link disappears cleanly when focus moves away
```

### Automated Testing

```bash
# axe-core: checks for bypass blocks (WCAG 2.4.1)
npx axe http://localhost:1313/ --tags "bypass"

# pa11y: checks skip link presence and targets
npx pa11y http://localhost:1313/ --standard WCAG2AA

# Lighthouse: skip link is part of the Navigation audit
# Open DevTools > Lighthouse > Accessibility > Run audit
```

### Common Issues

**Issue 1: Skip link target has no tabindex**

```html
<!-- DON'T: Target without tabindex -- focus may not move in Chrome/Safari -->
<main id="main-content">
  <h1>Page Title</h1>
</main>

<!-- DO: Target with tabindex="-1" -->
<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
</main>
```

**Issue 2: Skip link hidden with display: none**

```css
/* DON'T: display: none removes the link from the tab order entirely */
.skip-link {
  display: none;
}
.skip-link:focus {
  display: block;     /* Too late -- element was not in tab order */
}

/* DO: Use position or clip to hide visually while keeping in tab order */
.skip-link {
  position: absolute;
  top: -100%;
}
.skip-link:focus {
  position: fixed;
  top: 0;
}
```

**Issue 3: Skip link points to nonexistent ID**

```html
<!-- DON'T: Target ID does not exist on the page -->
<a href="#content" class="skip-link">Skip to main content</a>
<!-- There is no element with id="content" -->

<!-- DO: Verify the target ID matches an actual element -->
<a href="#main-content" class="skip-link">Skip to main content</a>
<main id="main-content" tabindex="-1">...</main>
```

**Issue 4: Skip link is not the first focusable element**

```html
<!-- DON'T: Cookie banner or other element receives focus first -->
<body>
  <div class="cookie-banner">
    <button>Accept cookies</button>     <!-- First tab stop -->
  </div>
  <a href="#main-content" class="skip-link">Skip to main content</a>

<!-- DO: Skip link is absolutely first -->
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <div class="cookie-banner">
    <button>Accept cookies</button>
  </div>
```

## Best Practices

**DO:**
- Place the skip link as the very first focusable element inside `<body>`
- Target the `<main>` element with `tabindex="-1"` for consistent cross-browser focus
- Use the "visible on focus" pattern so the link appears only for keyboard users
- Use high-contrast colors for the skip link (at least 4.5:1 for the text)
- Add a clear focus indicator on the skip link itself
- Include the skip link in Hugo's `baseof.html` so every page inherits it
- Use descriptive text ("Skip to main content" not just "Skip")
- Provide multiple skip links on complex layouts (skip to search, skip to footer)
- Test in Chrome, Firefox, and Safari to verify focus moves to the target
- Use `position: fixed` on focus so the link appears at a consistent viewport position

**DON'T:**
- Hide the skip link with `display: none` or `visibility: hidden` (removes it from tab order)
- Place any focusable element before the skip link in the DOM
- Target a nonexistent ID (verify the target element exists on every page)
- Omit `tabindex="-1"` on the skip link target (some browsers will not move focus)
- Use `tabindex="0"` on the target (this adds `<main>` to the Tab order permanently)
- Make the skip link permanently visible (visual clutter for mouse users)
- Use vague link text like "Skip" or "Click here"
- Rely on landmarks alone as skip link replacements (not all users navigate by landmarks)
- Forget to include skip links on pages with different layouts (documentation, landing pages)
- Use JavaScript-only skip links without the underlying anchor and target ID

## Guidelines

### Essential

- Skip link is the first focusable element on every page (WCAG 2.4.1 Level A)
- Skip link targets `<main id="main-content" tabindex="-1">`
- Skip link uses the "visible on focus" CSS pattern (hidden by default, visible on Tab)
- Skip link has at least 4.5:1 contrast ratio when visible
- Skip link is implemented in Hugo's `baseof.html` template for site-wide coverage
- Skip link target ID exists on every page and layout variant

### Recommended

- Multiple skip links on complex layouts (search, navigation sidebar, footer)
- JavaScript fallback ensures focus moves to target in all browsers
- Skip link appears at a fixed viewport position when focused
- Skip link text is descriptive and identifies the target region
- `:focus-within` on skip link container reveals all skip links when any one is focused
- Documentation sites include a skip link to the sidebar navigation

### Advanced

- Automated CI/CD check verifies skip link presence and valid target on every page
- Skip link functionality tested after client-side navigation (Turbo, htmx)
- Skip link behavior verified with VoiceOver, NVDA, and JAWS
- Custom Hugo shortcode for section-specific skip links within long-form content
- Skip link animation (smooth slide-in) uses `prefers-reduced-motion` to disable for users who prefer reduced motion
- `tabindex="-1"` removed from target on blur to avoid permanent tab stop

## Benefits

Faster Keyboard Navigation. Skip links let keyboard users bypass dozens of navigation links with a single Tab and Enter, reaching the main content in two keystrokes instead of twenty or more.

WCAG Level A Compliance. Skip links satisfy Success Criterion 2.4.1 (Bypass Blocks), a Level A requirement that is mandatory for all conformance levels.

Consistent Cross-Page Experience. Implementing skip links in Hugo's `baseof.html` ensures every generated page provides the same bypass mechanism without per-page effort.

Screen Reader Efficiency. Screen reader users hear the skip link announced first, giving them an immediate option to bypass the site header and jump to content.

Reduced Repetitive Navigation. On content-heavy sites with extensive navigation menus, skip links prevent keyboard users from repeatedly traversing the same menu on every page visit.

Accessible Complex Layouts. Multiple skip links (to content, search, sidebar, footer) provide keyboard users with the same quick-access shortcuts that mouse users get from clicking directly on visible page regions.

## Related

- [keyboard-navigation.md](./keyboard-navigation.md) - Tab order and keyboard interaction patterns that skip links complement
- [semantic-html.md](./semantic-html.md) - Landmark elements that screen readers use alongside skip links for navigation
- [focus-management.md](./focus-management.md) - Programmatic focus patterns used by skip link targets
