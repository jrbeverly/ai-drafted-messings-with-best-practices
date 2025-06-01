# Color Contrast and WCAG Compliance

Color contrast. WCAG 2.1. AA compliance. AAA compliance. Contrast ratio. 4.5:1 ratio. 3:1 ratio. Large text. UI components. Focus indicators. Dark mode. Color blindness. Protanopia. Deuteranopia. Tritanopia. CSS custom properties. Accessible color systems. Theme variables. Hugo color themes.

## Principle

Ensure all text, interactive elements, and meaningful graphical objects meet WCAG 2.1 contrast ratio requirements so that users with low vision, color blindness, or situational impairments (bright sunlight, low-quality displays) can perceive and interact with content. Color must never be the sole means of conveying information -- every color-coded signal needs a secondary indicator such as text, shape, pattern, or position. In Hugo static sites, color systems are defined in CSS custom properties and referenced across layout templates and partials so that contrast compliance is structural rather than per-page, making it possible to maintain accessible light and dark themes from a single set of design tokens.

## WCAG 2.1 Contrast Ratio Requirements

### Understanding Contrast Ratios

Contrast ratio measures the relative luminance difference between foreground and background colors. The ratio ranges from 1:1 (no contrast, identical colors) to 21:1 (maximum contrast, black on white). WCAG defines minimum ratios based on text size and element type.

```
Contrast ratio formula:
(L1 + 0.05) / (L2 + 0.05)

Where L1 = relative luminance of the lighter color
      L2 = relative luminance of the darker color
      Relative luminance ranges from 0 (black) to 1 (white)
```

### AA Level Requirements (Minimum Conformance)

AA is the baseline conformance level required by most accessibility laws and standards.

| Element Type | Minimum Ratio | Examples |
|---|---|---|
| Normal text (under 18pt or under 14pt bold) | 4.5:1 | Body paragraphs, navigation links, form labels, captions |
| Large text (18pt+ or 14pt+ bold) | 3:1 | Page headings, section titles, hero text |
| UI components and graphical objects | 3:1 | Buttons, form borders, icons, focus indicators, chart elements |

**What counts as large text:**
- 18pt (24px) regular weight or larger
- 14pt (18.66px) bold weight or larger
- These thresholds apply to the rendered size, not the CSS font-size value

### AAA Level Requirements (Enhanced Conformance)

AAA provides the highest level of accessibility. It is not required for legal compliance in most jurisdictions but is recommended for content-heavy sites where readability is critical.

| Element Type | Minimum Ratio | Examples |
|---|---|---|
| Normal text (under 18pt or under 14pt bold) | 7:1 | Long-form articles, documentation, legal text |
| Large text (18pt+ or 14pt+ bold) | 4.5:1 | Headings in content-focused layouts |

### Practical Contrast Examples

```css
/* AA compliant combinations for normal text (4.5:1 minimum) */

/* Dark text on light background */
color: #1a1a1a; background: #ffffff;    /* 17.4:1 -- exceeds AAA */
color: #333333; background: #ffffff;    /* 12.6:1 -- exceeds AAA */
color: #595959; background: #ffffff;    /*  7.0:1 -- meets AAA */
color: #767676; background: #ffffff;    /*  4.5:1 -- meets AA exactly */
color: #999999; background: #ffffff;    /*  2.8:1 -- FAILS AA */

/* Light text on dark background */
color: #ffffff; background: #1a1a1a;    /* 17.4:1 -- exceeds AAA */
color: #e0e0e0; background: #1a1a1a;    /* 13.1:1 -- exceeds AAA */
color: #a0a0a0; background: #1a1a1a;    /*  6.3:1 -- meets AA */
color: #767676; background: #1a1a1a;    /*  3.9:1 -- FAILS for normal text */

/* Common color-on-white combinations */
color: #0056b3; background: #ffffff;    /*  7.2:1 -- meets AAA (blue link) */
color: #d32f2f; background: #ffffff;    /*  5.6:1 -- meets AA (red error) */
color: #2e7d32; background: #ffffff;    /*  5.1:1 -- meets AA (green success) */
color: #f57c00; background: #ffffff;    /*  3.0:1 -- FAILS AA for normal text */
```

## How to Check Contrast

### Browser DevTools

**Chrome DevTools:**
1. Right-click an element and select Inspect
2. In the Styles panel, click the color swatch next to any `color` property
3. The color picker displays the contrast ratio against the background
4. A checkmark indicates AA or AAA compliance
5. Click "Show more" to see the AA and AAA thresholds with suggested compliant colors

**Firefox DevTools:**
1. Open the Accessibility Inspector (F12 then Accessibility tab)
2. Select any element to see its color contrast ratio
3. The Accessibility panel highlights contrast violations
4. Use the Color Picker (Shift+Click on color swatch) for contrast information

### Online Tools

**WebAIM Contrast Checker (https://webaim.org/resources/contrastchecker/):**
- Enter foreground and background hex colors
- Shows contrast ratio and AA/AAA pass/fail for both normal and large text
- Provides lightness slider to find the nearest compliant color

**Accessible Colors (https://accessible-colors.com/):**
- Input your color and font size
- Suggests the closest accessible color that meets AA or AAA

**Colour Contrast Analyser (desktop application):**
- Eyedropper tool picks colors directly from any screen element
- Works outside the browser (design tools, PDFs, native applications)

### Automated Testing

**axe-core (CI/CD integration):**

```bash
# Install axe-core CLI
npm install -g @axe-core/cli

# Test contrast on a Hugo site
axe http://localhost:1313/ --tags color-contrast

# Test all pages from sitemap
npx pa11y-ci --sitemap http://localhost:1313/sitemap.xml --standard WCAG2AA
```

**Lighthouse (Chrome DevTools):**
1. Open DevTools (F12)
2. Navigate to the Lighthouse panel
3. Select Accessibility
4. Run audit -- contrast violations appear under "Background and foreground colors do not have a sufficient contrast ratio"

## Color as Not the Sole Indicator

### The Problem

When color is the only way to convey meaning, users who cannot perceive color differences miss the information entirely. This includes users with color blindness (approximately 8% of men and 0.5% of women) and users viewing content on monochrome displays or in high-glare conditions.

### Common Violations

```html
<!-- DON'T: Color is the only indicator of required fields -->
<form>
  <label style="color: red">Name</label>
  <input type="text">
  <p style="color: red; font-size: 0.8em">* Red fields are required</p>
</form>

<!-- DO: Use text, icons, and color together -->
<form>
  <label for="name">
    Name <span aria-hidden="true">*</span>
    <span class="sr-only">(required)</span>
  </label>
  <input type="text" id="name" required aria-required="true">
</form>
```

```html
<!-- DON'T: Status indicated by color alone -->
<ul>
  <li style="color: green">Build passed</li>
  <li style="color: red">Tests failed</li>
  <li style="color: orange">Deployment pending</li>
</ul>

<!-- DO: Status indicated by text, icon, and color -->
<ul>
  <li>
    <svg aria-hidden="true" class="icon-success"><use href="#icon-check"></use></svg>
    <span class="status-success">Build: Passed</span>
  </li>
  <li>
    <svg aria-hidden="true" class="icon-error"><use href="#icon-x"></use></svg>
    <span class="status-error">Tests: Failed</span>
  </li>
  <li>
    <svg aria-hidden="true" class="icon-warning"><use href="#icon-clock"></use></svg>
    <span class="status-pending">Deployment: Pending</span>
  </li>
</ul>
```

### Links Within Text

Links within paragraphs must be distinguishable from surrounding text by more than color alone. WCAG requires a 3:1 contrast ratio between link color and surrounding text, or an additional non-color indicator (underline, bold, icon).

```css
/* DON'T: Links distinguished only by color */
a {
  color: #0056b3;
  text-decoration: none;
}

/* DO: Links have underline as a secondary indicator */
a {
  color: #0056b3;
  text-decoration: underline;
}

/* DO: Links have underline on hover/focus if removed by default */
a {
  color: #0056b3;
  text-decoration: none;
  border-bottom: 1px solid currentColor;
}

a:hover,
a:focus-visible {
  text-decoration: underline;
}
```

## Focus Indicator Contrast Requirements

### WCAG 2.2 Focus Appearance (Level AA)

Focus indicators must have sufficient contrast to be perceivable by users with low vision. WCAG 2.2 Success Criterion 2.4.11 requires:

- The focus indicator has a contrast ratio of at least **3:1** against adjacent unfocused colors
- The focus indicator has a contrast ratio of at least **3:1** against the focused component itself
- The focus indicator encloses the component or has a minimum area of the component's perimeter times 2 CSS pixels

```css
/* DO: High-contrast focus indicator */
:focus-visible {
  outline: 3px solid #0056b3;     /* Blue outline */
  outline-offset: 2px;            /* Gap between element and outline */
}

/* Contrast check:
   #0056b3 (blue) against #ffffff (white background): 7.2:1 -- passes
   #0056b3 (blue) against #1a1a1a (dark background):  2.6:1 -- FAILS on dark
*/

/* DO: Focus indicator that works on both light and dark backgrounds */
:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
  box-shadow: 0 0 0 5px rgba(255, 255, 255, 0.8),
              0 0 0 8px #0056b3;
}
/* The white inner shadow creates contrast on dark backgrounds */
/* The blue outer ring creates contrast on light backgrounds */

/* DON'T: Low-contrast focus indicator */
:focus-visible {
  outline: 1px solid #cccccc;     /* 1.6:1 against white -- FAILS */
}

/* DON'T: Focus indicator only changes background color */
button:focus-visible {
  background-color: #f0f0f0;     /* Subtle change, may not be visible */
}
```

### Double-Ring Focus Pattern

The double-ring pattern ensures the focus indicator is visible on any background color.

```css
/* Universal focus indicator -- visible on light and dark backgrounds */
:focus-visible {
  outline: 2px solid #ffffff;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px #0056b3;
}

/*
  Inner ring: white (#ffffff)
  Outer ring: blue (#0056b3)
  On light backgrounds: blue ring is visible (7.2:1 against white)
  On dark backgrounds: white ring is visible (17.4:1 against black)
  Both rings are always visible regardless of background
*/
```

## Color Blindness Considerations

### Types of Color Vision Deficiency

| Type | Prevalence | Affected Colors | Impact |
|---|---|---|---|
| Protanopia (no red cones) | ~1% of men | Red and green appear similar | Red warnings invisible against green |
| Deuteranopia (no green cones) | ~1% of men | Red and green appear similar | Traffic-light color coding fails |
| Tritanopia (no blue cones) | ~0.01% of population | Blue and yellow appear similar | Blue links on yellow backgrounds |
| Achromatopsia (no color vision) | ~0.003% of population | All colors appear as grayscale | Only luminance contrast matters |

### Designing for Color Blindness

```css
/* DON'T: Red/green for success/error (indistinguishable for protanopia/deuteranopia) */
.success { color: #00cc00; }    /* Green */
.error { color: #cc0000; }      /* Red */
/* These appear nearly identical to ~8% of men */

/* DO: Use distinct hues plus secondary indicators */
.success {
  color: #2e7d32;              /* Dark green -- meets AA on white */
  border-left: 4px solid #2e7d32;
}

.success::before {
  content: "\2713 ";           /* Checkmark character */
}

.error {
  color: #d32f2f;              /* Dark red -- meets AA on white */
  border-left: 4px solid #d32f2f;
}

.error::before {
  content: "\2717 ";           /* Cross mark character */
}

/* DO: Charts use patterns in addition to colors */
.chart-bar-a {
  background-color: #0056b3;
  background-image: repeating-linear-gradient(
    45deg, transparent, transparent 5px, rgba(255,255,255,0.3) 5px, rgba(255,255,255,0.3) 10px
  );
}

.chart-bar-b {
  background-color: #d32f2f;
  background-image: repeating-linear-gradient(
    -45deg, transparent, transparent 5px, rgba(255,255,255,0.3) 5px, rgba(255,255,255,0.3) 10px
  );
}

.chart-bar-c {
  background-color: #2e7d32;
  /* Solid fill -- no pattern -- distinct from striped bars */
}
```

### Safe Color Palettes

Colors that remain distinguishable across most types of color vision deficiency:

```css
/* Blue + Orange palette (safe for protanopia and deuteranopia) */
--color-primary: #0056b3;      /* Blue */
--color-accent: #e65100;       /* Deep orange */
/* These are distinct even without red/green perception */

/* Blue + Yellow palette (safe for all common types) */
--color-primary: #1565c0;      /* Blue */
--color-accent: #f9a825;       /* Amber/yellow */
/* Avoid for tritanopia -- add shape/pattern as secondary indicator */

/* High-contrast neutral palette */
--color-text: #1a1a1a;         /* Near-black text */
--color-muted: #595959;        /* Gray text (7:1 on white) */
--color-border: #767676;       /* Gray borders (4.5:1 on white) */
--color-surface: #ffffff;      /* White background */
```

## CSS Custom Properties for Accessible Color Systems

### Defining Color Tokens

```css
/* Light theme color tokens */
:root {
  /* Text colors */
  --color-text-primary: #1a1a1a;         /* 17.4:1 on white -- body text */
  --color-text-secondary: #595959;       /*  7.0:1 on white -- captions, metadata */
  --color-text-disabled: #767676;        /*  4.5:1 on white -- disabled state */
  --color-text-inverse: #ffffff;         /* For use on dark backgrounds */

  /* Background colors */
  --color-bg-primary: #ffffff;
  --color-bg-secondary: #f5f5f5;
  --color-bg-tertiary: #e8e8e8;

  /* Interactive colors */
  --color-link: #0056b3;                 /*  7.2:1 on white */
  --color-link-visited: #6a1b9a;         /*  8.3:1 on white */
  --color-link-hover: #003d80;           /* 10.3:1 on white */
  --color-focus-ring: #0056b3;           /*  7.2:1 on white */

  /* Semantic colors */
  --color-success: #2e7d32;              /*  5.1:1 on white -- AA for normal text */
  --color-warning: #e65100;              /*  5.2:1 on white -- AA for normal text */
  --color-error: #c62828;               /*  6.7:1 on white -- AA for normal text */
  --color-info: #0277bd;                /*  5.5:1 on white -- AA for normal text */

  /* Border colors */
  --color-border: #767676;               /*  4.5:1 on white -- meets UI component requirement */
  --color-border-input: #595959;         /*  7.0:1 on white -- form inputs need clear borders */
}
```

### Dark Theme Color Tokens

```css
/* Dark theme overrides */
@media (prefers-color-scheme: dark) {
  :root {
    /* Text colors */
    --color-text-primary: #e8e8e8;       /* 14.7:1 on #121212 */
    --color-text-secondary: #a0a0a0;     /*  6.8:1 on #121212 */
    --color-text-disabled: #767676;      /*  3.7:1 on #121212 -- meets 3:1 for UI */
    --color-text-inverse: #1a1a1a;

    /* Background colors */
    --color-bg-primary: #121212;
    --color-bg-secondary: #1e1e1e;
    --color-bg-tertiary: #2c2c2c;

    /* Interactive colors -- adjusted for dark backgrounds */
    --color-link: #64b5f6;               /*  8.1:1 on #121212 */
    --color-link-visited: #ce93d8;       /*  6.4:1 on #121212 */
    --color-link-hover: #90caf9;         /* 10.2:1 on #121212 */
    --color-focus-ring: #64b5f6;         /*  8.1:1 on #121212 */

    /* Semantic colors -- adjusted for dark backgrounds */
    --color-success: #66bb6a;            /*  6.6:1 on #121212 */
    --color-warning: #ffa726;            /*  7.9:1 on #121212 */
    --color-error: #ef5350;              /*  4.6:1 on #121212 */
    --color-info: #4fc3f7;               /*  8.9:1 on #121212 */

    /* Border colors */
    --color-border: #767676;             /*  3.7:1 on #121212 */
    --color-border-input: #a0a0a0;       /*  6.8:1 on #121212 */
  }
}
```

### Applying Color Tokens

```css
/* Base typography */
body {
  color: var(--color-text-primary);
  background-color: var(--color-bg-primary);
}

/* Links */
a {
  color: var(--color-link);
  text-decoration: underline;
}

a:visited {
  color: var(--color-link-visited);
}

a:hover {
  color: var(--color-link-hover);
}

/* Focus indicators */
:focus-visible {
  outline: 3px solid var(--color-focus-ring);
  outline-offset: 2px;
}

/* Form inputs */
input,
textarea,
select {
  color: var(--color-text-primary);
  background-color: var(--color-bg-primary);
  border: 2px solid var(--color-border-input);
}

/* Error messages */
.field-error {
  color: var(--color-error);
  border-left: 3px solid var(--color-error);
  padding-left: 0.5rem;
}

/* Status badges */
.badge-success {
  color: var(--color-text-inverse);
  background-color: var(--color-success);
}

.badge-error {
  color: var(--color-text-inverse);
  background-color: var(--color-error);
}
```

## Dark Mode Considerations

### Contrast Ratios Change with Theme

Colors that pass contrast checks in light mode may fail in dark mode, and vice versa. Every color pairing must be tested in both themes independently.

```css
/* This blue passes AA on white but fails on dark gray */
/* Light mode: #0056b3 on #ffffff = 7.2:1 -- passes */
/* Dark mode:  #0056b3 on #121212 = 2.6:1 -- FAILS */

/* Solution: use different shades per theme */
:root {
  --color-link: #0056b3;           /* For light backgrounds */
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-link: #64b5f6;         /* Lighter blue for dark backgrounds */
  }
}
```

### Dark Mode Anti-Patterns

```css
/* DON'T: Pure black background with pure white text */
body {
  color: #ffffff;
  background: #000000;
}
/* 21:1 contrast is technically compliant but causes eye strain
   and halation (glowing text effect) for users with astigmatism */

/* DO: Slightly off-white text on dark gray */
body {
  color: #e8e8e8;
  background: #121212;
}
/* 14.7:1 contrast -- exceeds AAA while reducing eye strain */

/* DON'T: Inverting all colors blindly */
@media (prefers-color-scheme: dark) {
  img { filter: invert(1); }         /* Destroys photographs */
  * { color: white !important; }     /* Overrides all color tokens */
}

/* DO: Use semantic tokens that switch per theme */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text-primary: #e8e8e8;
    --color-bg-primary: #121212;
  }
}
```

### Testing Dark Mode Contrast

```bash
# Chrome DevTools: emulate prefers-color-scheme: dark
# 1. Open DevTools (F12)
# 2. Cmd+Shift+P (Mac) or Ctrl+Shift+P (Windows)
# 3. Type "rendering"
# 4. Select "Show Rendering"
# 5. Under "Emulate CSS media feature prefers-color-scheme"
#    select "dark"
# 6. Rerun Lighthouse accessibility audit in dark mode

# Automated: test both themes in CI/CD
npx pa11y http://localhost:1313/ --standard WCAG2AA
# Then toggle theme and test again
```

## Hugo Templates for Theme Color Variables

### Head Partial with Theme Meta Tags

**layouts/partials/head.html:**

```go-html-template
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

{{/* Theme color for browser chrome (address bar on mobile) */}}
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#121212" media="(prefers-color-scheme: dark)">

{{/* Preload the stylesheet to prevent flash of unstyled content */}}
<link rel="stylesheet" href="{{ "css/main.css" | relURL }}">

<title>{{ if .IsHome }}{{ site.Title }}{{ else }}{{ .Title }} | {{ site.Title }}{{ end }}</title>
```

### Hugo Configuration for Color Parameters

**hugo.toml:**

```toml
[params]
  [params.colors]
    # Light theme
    textPrimary = "#1a1a1a"
    bgPrimary = "#ffffff"
    linkColor = "#0056b3"
    focusRing = "#0056b3"

    # Dark theme
    darkTextPrimary = "#e8e8e8"
    darkBgPrimary = "#121212"
    darkLinkColor = "#64b5f6"
    darkFocusRing = "#64b5f6"
```

### Hugo Partial for Inline Critical CSS with Color Variables

**layouts/partials/critical-css.html:**

```go-html-template
<style>
  :root {
    --color-text-primary: {{ site.Params.colors.textPrimary | default "#1a1a1a" }};
    --color-bg-primary: {{ site.Params.colors.bgPrimary | default "#ffffff" }};
    --color-link: {{ site.Params.colors.linkColor | default "#0056b3" }};
    --color-focus-ring: {{ site.Params.colors.focusRing | default "#0056b3" }};
  }

  @media (prefers-color-scheme: dark) {
    :root {
      --color-text-primary: {{ site.Params.colors.darkTextPrimary | default "#e8e8e8" }};
      --color-bg-primary: {{ site.Params.colors.darkBgPrimary | default "#121212" }};
      --color-link: {{ site.Params.colors.darkLinkColor | default "#64b5f6" }};
      --color-focus-ring: {{ site.Params.colors.darkFocusRing | default "#64b5f6" }};
    }
  }

  body {
    color: var(--color-text-primary);
    background-color: var(--color-bg-primary);
  }

  a { color: var(--color-link); }
  :focus-visible {
    outline: 3px solid var(--color-focus-ring);
    outline-offset: 2px;
  }
</style>
```

### Theme Toggle with Accessible Color Switching

**layouts/partials/theme-toggle.html:**

```go-html-template
<button
  id="theme-toggle"
  aria-label="Switch to dark theme"
  aria-live="polite"
  class="theme-toggle">
  <svg aria-hidden="true" focusable="false" class="icon-sun">
    <use href="#icon-sun"></use>
  </svg>
  <svg aria-hidden="true" focusable="false" class="icon-moon" hidden>
    <use href="#icon-moon"></use>
  </svg>
</button>
```

```js
document.addEventListener('DOMContentLoaded', () => {
  const toggle = document.getElementById('theme-toggle');
  if (!toggle) return;

  const iconSun = toggle.querySelector('.icon-sun');
  const iconMoon = toggle.querySelector('.icon-moon');

  // Determine initial theme from localStorage or system preference
  const stored = localStorage.getItem('theme');
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  const isDark = stored === 'dark' || (!stored && prefersDark);

  applyTheme(isDark);

  toggle.addEventListener('click', () => {
    const currentlyDark = document.documentElement.getAttribute('data-theme') === 'dark';
    applyTheme(!currentlyDark);
    localStorage.setItem('theme', !currentlyDark ? 'dark' : 'light');
  });

  function applyTheme(dark) {
    document.documentElement.setAttribute('data-theme', dark ? 'dark' : 'light');
    toggle.setAttribute('aria-label', dark ? 'Switch to light theme' : 'Switch to dark theme');

    if (iconSun && iconMoon) {
      iconSun.hidden = dark;
      iconMoon.hidden = !dark;
    }
  }
});
```

```css
/* Theme toggle styling with data-theme attribute */
[data-theme="dark"] {
  --color-text-primary: #e8e8e8;
  --color-bg-primary: #121212;
  --color-link: #64b5f6;
  --color-focus-ring: #64b5f6;
}

[data-theme="light"] {
  --color-text-primary: #1a1a1a;
  --color-bg-primary: #ffffff;
  --color-link: #0056b3;
  --color-focus-ring: #0056b3;
}
```

## Contrast Validation in Development

### Manual Validation Workflow

```
1. Design phase
   [ ] All color pairings checked with WebAIM Contrast Checker
   [ ] Light and dark theme variants tested independently
   [ ] Interactive state colors (hover, focus, active, disabled) checked
   [ ] Color is never the sole indicator of state or meaning

2. Development phase
   [ ] CSS custom properties define all colors (no hardcoded hex in components)
   [ ] Focus indicators tested against all background colors they appear on
   [ ] Form validation errors use icon + text + color (not color alone)
   [ ] Links within paragraphs have underline or 3:1 contrast vs surrounding text

3. Testing phase
   [ ] Lighthouse accessibility audit passes (score 90+)
   [ ] axe-core reports zero contrast violations
   [ ] Browser color contrast picker confirms ratios on key elements
   [ ] Dark mode tested separately from light mode
   [ ] Grayscale rendering checked (Chrome DevTools > Rendering > Emulate vision deficiencies > Achromatopsia)
```

### Simulating Color Vision Deficiency

**Chrome DevTools:**
1. Open DevTools (F12)
2. Open Command Palette (Cmd+Shift+P or Ctrl+Shift+P)
3. Type "rendering" and select "Show Rendering"
4. Scroll to "Emulate vision deficiencies"
5. Select: Protanopia, Deuteranopia, Tritanopia, or Achromatopsia
6. Verify all information remains perceivable without color

**Firefox:**
1. Open DevTools (F12)
2. Open the Accessibility tab
3. Use the "Simulate" dropdown to select color vision deficiency types
4. Verify content remains distinguishable

## Best Practices

**DO:**
- Meet 4.5:1 contrast ratio for normal text and 3:1 for large text (WCAG AA minimum)
- Meet 3:1 contrast ratio for UI components, graphical objects, and focus indicators
- Define all colors as CSS custom properties so themes are centrally managed
- Test contrast ratios in both light and dark themes independently
- Use underline or border on links within body text as a secondary indicator beyond color
- Provide text, icon, or pattern alongside color to convey status and meaning
- Use the double-ring focus pattern (light inner ring, dark outer ring) for universal visibility
- Simulate color vision deficiency in DevTools during development
- Maintain separate color token values for light and dark themes
- Use slightly off-white text on dark backgrounds (#e8e8e8 on #121212) to reduce halation

**DON'T:**
- Rely on color alone to convey information (required fields, errors, status, links in text)
- Use pure black (#000000) backgrounds with pure white (#ffffff) text in dark themes
- Assume light mode colors will pass contrast checks on dark backgrounds
- Remove focus outlines without providing a visible replacement with 3:1 contrast
- Use thin (1px) focus indicators that are difficult to perceive
- Hardcode hex colors in components instead of referencing design tokens
- Use orange or yellow text on white backgrounds for normal-sized text (rarely meets 4.5:1)
- Skip dark mode contrast testing because light mode passed
- Use `filter: invert(1)` as a dark mode strategy
- Use red and green as the only distinction between success and error states

## Guidelines

### Essential

- All normal text meets 4.5:1 contrast ratio against its background (WCAG AA)
- All large text meets 3:1 contrast ratio against its background (WCAG AA)
- All UI components (borders, icons, focus indicators) meet 3:1 contrast ratio
- Color is never the sole means of conveying information
- CSS custom properties define the color system with light and dark theme variants
- Focus indicators have at least 3:1 contrast against adjacent colors
- Links within body text have an underline or other non-color visual distinction

### Recommended

- Color tokens stored in a centralized CSS file or Hugo configuration parameters
- Dark mode uses `prefers-color-scheme` media query with separately tested token values
- `<meta name="theme-color">` set for both light and dark schemes
- Interactive elements tested in all states (default, hover, focus, active, disabled)
- Color vision deficiency simulation run during development using DevTools
- Error messages use icon plus text plus color (triple redundancy)
- Charts and data visualizations use patterns or labels in addition to color coding

### Advanced

- Automated contrast checking in CI/CD pipeline with axe-core or pa11y
- Design tokens generated from a system that validates contrast ratios at build time
- AAA compliance (7:1 normal text, 4.5:1 large text) for content-heavy pages
- Theme toggle component with accessible labeling and localStorage persistence
- Color contrast regression tests that compare screenshots across theme changes
- User preference for high contrast mode detected and applied with custom tokens

## Benefits

Universal Readability. Meeting WCAG contrast ratios ensures text is legible for users with low vision, color blindness, aging eyes, and situational impairments like bright sunlight.

Theme Consistency. CSS custom properties with separate light and dark token values guarantee contrast compliance across both themes from a single color system definition.

Reduced Color Dependence. Using text, icons, and patterns alongside color ensures information reaches users regardless of their color perception ability.

Maintainable Design System. Centralized color tokens in Hugo configuration and CSS custom properties make it possible to adjust the entire site's color scheme from one location.

Legal Compliance. Meeting WCAG 2.1 AA contrast requirements satisfies accessibility legislation in the EU (EN 301 549), US (Section 508), and other jurisdictions.

Visible Focus Indicators. The double-ring focus pattern ensures keyboard users can see which element has focus on any background color.

## Related

- [focus-management.md](./focus-management.md) - Focus indicator styles and programmatic focus patterns
- [semantic-html.md](./semantic-html.md) - Structural elements that convey meaning beyond visual styling
