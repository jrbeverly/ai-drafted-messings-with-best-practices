# Focus Management

Focus management. Focus indicators. Visible focus. :focus-visible. :focus. Custom focus styles. Focus trapping. Modal focus. Focus restoration. Programmatic focus. tabindex. Dynamic content focus. SPA focus management. Outline. Box-shadow. WCAG focus appearance. Dialog focus. Skip link targets.

## Principle

Manage keyboard focus deliberately so that users always know where they are on the page, can predict where focus will move next, and never lose their place when content changes dynamically. Visible focus indicators must meet WCAG contrast requirements and appear consistently on every interactive element. When JavaScript shows, hides, or modifies content -- opening a modal, expanding an accordion, filtering search results, or navigating between views -- focus must be explicitly moved to the appropriate element and restored when the interaction ends. In Hugo static sites, focus management patterns belong in layout templates and shared JavaScript so that every generated page provides predictable, visible focus behavior without requiring per-page configuration.

## Visible Focus Indicators

### Why Focus Indicators Matter

Focus indicators are the keyboard equivalent of a mouse cursor. Without a visible indicator, keyboard users cannot determine which element will respond to their next keystroke. Removing or hiding focus indicators is one of the most common accessibility failures.

### Default Browser Focus Styles

Browsers provide default focus styles that vary across platforms:

```css
/* Chrome/Edge default: blue outline */
:focus {
  outline: -webkit-focus-ring-color auto 1px;
}

/* Firefox default: dotted outline */
:focus {
  outline: 1px dotted;
}

/* Safari default: blue glow */
:focus {
  outline: auto 5px -webkit-focus-ring-color;
}
```

These defaults are functional but often thin, low-contrast, or visually inconsistent across browsers. Custom focus styles provide a better, uniform experience while meeting WCAG requirements.

### :focus vs :focus-visible

The `:focus` pseudo-class applies whenever an element has focus, whether from keyboard navigation, mouse click, or programmatic focus. The `:focus-visible` pseudo-class applies only when the browser determines that focus should be visually indicated -- typically during keyboard navigation but not after mouse clicks.

```css
/* :focus applies on EVERY focus event (keyboard, mouse, programmatic) */
button:focus {
  outline: 3px solid #0056b3;
}
/* Problem: Shows focus ring after mouse clicks, which many designers find undesirable */

/* :focus-visible applies only when the browser thinks a visual indicator is needed */
button:focus-visible {
  outline: 3px solid #0056b3;
}
/* Result: Focus ring appears on Tab navigation but not after mouse clicks */
```

### Recommended Pattern: Progressive Enhancement

```css
/* Base: ensure focus is always visible (fallback for older browsers) */
:focus {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}

/* Enhancement: hide focus for mouse users in modern browsers */
:focus:not(:focus-visible) {
  outline: none;
}

/* Show focus only for keyboard navigation */
:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}
```

### When to Use :focus Instead of :focus-visible

Some elements should always show focus indicators regardless of input method:

```css
/* Text inputs: always show focus so users know which field is active */
input:focus,
textarea:focus,
select:focus {
  outline: 3px solid #0056b3;
  outline-offset: 0;
  border-color: #0056b3;
}

/* Buttons and links: use :focus-visible to avoid ring after click */
button:focus-visible,
a:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}
```

## Custom Focus Styles That Meet WCAG

### WCAG 2.2 Focus Appearance Requirements

WCAG 2.2 Success Criterion 2.4.11 (Focus Appearance, Level AA) requires:

1. **Minimum area:** Focus indicator encloses the component or has minimum area of the perimeter length times 2 CSS pixels
2. **Contrast ratio:** At least 3:1 between the focus indicator and both the unfocused state and adjacent background
3. **Not obscured:** Focus indicator is not entirely hidden by author-created content

### Outline-Based Focus Indicators

```css
/* Simple high-contrast outline */
:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}

/* Rounded outline matching element border-radius */
:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
  border-radius: 4px;
}

/* Thicker outline for enhanced visibility */
:focus-visible {
  outline: 4px solid #0056b3;
  outline-offset: 3px;
}
```

### Box-Shadow Focus Indicators

Box-shadow follows the element's border-radius and can create more polished focus styles.

```css
/* Box-shadow focus ring */
:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px #0056b3;
}

/* Box-shadow with offset (gap between element and ring) */
:focus-visible {
  outline: none;
  box-shadow: 0 0 0 2px #ffffff,
              0 0 0 5px #0056b3;
}
/* Inner white ring creates gap, outer blue ring creates indicator */

/* Box-shadow with glow for enhanced visibility */
:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px #0056b3,
              0 0 8px rgba(0, 86, 179, 0.4);
}
```

### Double-Ring Pattern for Universal Backgrounds

The double-ring pattern ensures focus indicators are visible on both light and dark backgrounds.

```css
/* Visible on any background color */
:focus-visible {
  outline: 2px solid #ffffff;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px #0056b3;
}

/* On light backgrounds: blue outer ring contrasts with background */
/* On dark backgrounds: white inner ring contrasts with background */
/* Both rings are always visible regardless of surrounding colors */
```

### Element-Specific Focus Styles

```css
/* Links: subtle underline enhancement */
a:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
  text-decoration-thickness: 3px;
}

/* Buttons: ring around the button */
button:focus-visible,
[role="button"]:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}

/* Cards with focusable heading links */
.card:focus-within {
  box-shadow: 0 0 0 3px #0056b3;
}

/* Form inputs: border color change plus outline */
input:focus-visible,
textarea:focus-visible,
select:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 0;
  border-color: #0056b3;
}

/* Checkboxes and radios */
input[type="checkbox"]:focus-visible,
input[type="radio"]:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: 2px;
}

/* Navigation items: background highlight plus ring */
nav a:focus-visible {
  outline: 3px solid #0056b3;
  outline-offset: -3px;
  background-color: rgba(0, 86, 179, 0.1);
}

/* Skip link: becomes visible on focus */
.skip-link:focus-visible {
  position: fixed;
  top: 0.5rem;
  left: 0.5rem;
  z-index: 10000;
  padding: 0.75rem 1.5rem;
  background: #0056b3;
  color: #ffffff;
  outline: 3px solid #ffffff;
  outline-offset: 2px;
  text-decoration: none;
  font-weight: bold;
}
```

## Focus Management in Dynamic Content

### When Programmatic Focus Is Required

Focus must be actively managed whenever the visible content changes in a way that alters the user's context:

| Scenario | Focus Target | Why |
|---|---|---|
| Modal dialog opens | The dialog container or first focusable element | User needs to interact with modal content |
| Modal dialog closes | The element that triggered the modal | User returns to their previous context |
| Accordion section expands | The expanded content or first element in it | User needs to read the new content |
| Inline edit mode activates | The edit input field | User is ready to type |
| Page content replaces (SPA) | The main heading or main content area | User needs to know the page changed |
| Error summary appears | The error summary container | User needs to address form errors |
| Toast notification appears | Do NOT move focus (use aria-live instead) | Moving focus would disrupt the user's task |
| Search results update | Do NOT move focus (use aria-live instead) | User is still typing in the search field |

### The focus() Method

```js
// Basic programmatic focus
const element = document.getElementById('target');
element.focus();

// Focus with scroll prevention (useful when element is already visible)
element.focus({ preventScroll: true });

// Focus a non-focusable element: add tabindex="-1" first
const heading = document.querySelector('h1');
heading.setAttribute('tabindex', '-1');
heading.focus();
// Remove tabindex after focus leaves to keep element out of tab order
heading.addEventListener('blur', () => {
  heading.removeAttribute('tabindex');
}, { once: true });
```

### tabindex for Programmatic Focus

```html
<!-- tabindex="-1" makes an element focusable via JavaScript only -->
<!-- It does NOT add the element to the Tab order -->

<!-- Main content target for skip link and page transitions -->
<main id="main-content" tabindex="-1">
  <h1>Page Title</h1>
</main>

<!-- Error summary focused after form validation -->
<div id="error-summary" tabindex="-1" role="alert">
  <h2>Please correct the following errors:</h2>
  <ul>
    <li><a href="#email">Email is required</a></li>
    <li><a href="#password">Password must be at least 12 characters</a></li>
  </ul>
</div>

<!-- Dialog container receives initial focus when opened -->
<div id="confirm-dialog" role="dialog" aria-modal="true"
     aria-labelledby="dialog-title" tabindex="-1" hidden>
  <h2 id="dialog-title">Confirm Action</h2>
  <p>Are you sure you want to proceed?</p>
  <button id="dialog-confirm">Confirm</button>
  <button id="dialog-cancel">Cancel</button>
</div>
```

## Focus Trapping in Modals

### Why Focus Trapping Is Necessary

When a modal dialog is open, the content behind the overlay is visually obscured. If keyboard focus can escape the modal, the user interacts with invisible content, losing awareness of where they are and what they are doing. Focus must cycle within the modal until it is closed.

### Complete Focus Trap Implementation

```js
/**
 * Creates a focus trap within a container element.
 * Tab and Shift+Tab cycle through focusable children.
 * Does not allow focus to leave the container.
 */
function createFocusTrap(container) {
  const focusableSelector = [
    'a[href]',
    'button:not([disabled])',
    'input:not([disabled])',
    'select:not([disabled])',
    'textarea:not([disabled])',
    '[tabindex]:not([tabindex="-1"])',
  ].join(', ');

  function getFocusableElements() {
    return Array.from(container.querySelectorAll(focusableSelector))
      .filter(el => !el.hasAttribute('hidden') && el.offsetParent !== null);
  }

  function handleKeydown(event) {
    if (event.key !== 'Tab') return;

    const focusable = getFocusableElements();
    if (focusable.length === 0) return;

    const firstFocusable = focusable[0];
    const lastFocusable = focusable[focusable.length - 1];

    if (event.shiftKey) {
      if (document.activeElement === firstFocusable) {
        event.preventDefault();
        lastFocusable.focus();
      }
    } else {
      if (document.activeElement === lastFocusable) {
        event.preventDefault();
        firstFocusable.focus();
      }
    }
  }

  return {
    activate() {
      container.addEventListener('keydown', handleKeydown);
    },
    deactivate() {
      container.removeEventListener('keydown', handleKeydown);
    }
  };
}
```

### Using the Focus Trap with a Dialog

```js
let previouslyFocused = null;
let trap = null;

function openDialog(dialogId) {
  const dialog = document.getElementById(dialogId);
  const overlay = document.querySelector('.dialog-overlay');

  // Store the element that had focus before the dialog opened
  previouslyFocused = document.activeElement;

  // Show the dialog and overlay
  dialog.hidden = false;
  overlay.hidden = false;

  // Prevent background content from being scrollable
  document.body.style.overflow = 'hidden';

  // Mark background content as inert (prevents assistive technology access)
  document.getElementById('main-content').setAttribute('inert', '');
  document.querySelector('header').setAttribute('inert', '');
  document.querySelector('footer').setAttribute('inert', '');

  // Create and activate focus trap
  trap = createFocusTrap(dialog);
  trap.activate();

  // Move focus into the dialog
  dialog.focus();

  // Close on Escape
  dialog.addEventListener('keydown', handleEscape);
}

function closeDialog(dialogId) {
  const dialog = document.getElementById(dialogId);
  const overlay = document.querySelector('.dialog-overlay');

  // Deactivate focus trap
  if (trap) {
    trap.deactivate();
    trap = null;
  }

  // Remove Escape handler
  dialog.removeEventListener('keydown', handleEscape);

  // Hide dialog and overlay
  dialog.hidden = true;
  overlay.hidden = true;

  // Restore background scrolling and interactivity
  document.body.style.overflow = '';
  document.getElementById('main-content').removeAttribute('inert');
  document.querySelector('header').removeAttribute('inert');
  document.querySelector('footer').removeAttribute('inert');

  // Restore focus to the element that opened the dialog
  if (previouslyFocused && previouslyFocused.focus) {
    previouslyFocused.focus();
  }
  previouslyFocused = null;
}

function handleEscape(event) {
  if (event.key === 'Escape') {
    closeDialog(event.currentTarget.id);
  }
}
```

### The inert Attribute

The `inert` attribute makes an element and all its descendants non-interactive and invisible to assistive technology. It is the correct way to prevent focus from reaching background content when a modal is open.

```html
<!-- When dialog is open, mark background as inert -->
<header inert>...</header>
<main id="main-content" inert>...</main>
<footer inert>...</footer>

<!-- Dialog is NOT inert -- user interacts with it -->
<div role="dialog" aria-modal="true" id="my-dialog">
  <h2>Dialog Title</h2>
  <button>Action</button>
  <button>Cancel</button>
</div>
<div class="dialog-overlay"></div>
```

### Native HTML dialog Element

The `<dialog>` element provides built-in focus management when opened with `.showModal()`.

```html
<dialog id="native-dialog" aria-labelledby="native-dialog-title">
  <h2 id="native-dialog-title">Confirm Deletion</h2>
  <p>This action cannot be undone.</p>
  <form method="dialog">
    <button value="cancel">Cancel</button>
    <button value="confirm">Delete</button>
  </form>
</dialog>
```

```js
const dialog = document.getElementById('native-dialog');

// showModal() automatically:
// 1. Makes background content inert
// 2. Traps focus inside the dialog
// 3. Centers the dialog with a backdrop
// 4. Closes on Escape
dialog.showModal();

// Focus returns to the triggering element after close
dialog.addEventListener('close', () => {
  // dialog.returnValue contains the button value
  if (dialog.returnValue === 'confirm') {
    performDeletion();
  }
});
```

## Restoring Focus After Dialog Close

### The Focus Restoration Pattern

When a dialog, popover, dropdown, or any overlay closes, focus must return to the element that triggered it. Without restoration, focus moves to the beginning of the document, forcing the user to navigate back to their previous position.

```js
// Generic focus restoration utility
function withFocusRestore(openFn, closeFn) {
  let trigger = null;

  return {
    open(triggerElement, ...args) {
      trigger = triggerElement || document.activeElement;
      openFn(...args);
    },
    close(...args) {
      closeFn(...args);
      if (trigger && trigger.focus) {
        trigger.focus();
      }
      trigger = null;
    }
  };
}

// Usage with a dropdown menu
const dropdown = withFocusRestore(
  (menuId) => {
    const menu = document.getElementById(menuId);
    menu.hidden = false;
    menu.querySelector('a, button').focus();
  },
  (menuId) => {
    const menu = document.getElementById(menuId);
    menu.hidden = true;
  }
);

// Open: stores trigger, shows menu, focuses first item
dropdown.open(document.getElementById('menu-trigger'), 'dropdown-menu');

// Close: hides menu, restores focus to trigger
dropdown.close('dropdown-menu');
```

### Focus Restoration When the Trigger Disappears

If the triggering element is removed from the DOM when the dialog closes (e.g., a "delete" action that removes the item), focus must move to a sensible alternative.

```js
function closeDialogAndRestoreFocus(dialogId, trigger) {
  const dialog = document.getElementById(dialogId);
  dialog.hidden = true;

  // Check if the trigger still exists in the DOM
  if (trigger && document.body.contains(trigger)) {
    trigger.focus();
  } else {
    // Trigger was removed -- focus a nearby logical element
    // Option 1: Focus the list that contained the deleted item
    const list = document.querySelector('.item-list');
    if (list) {
      list.setAttribute('tabindex', '-1');
      list.focus();
      list.addEventListener('blur', () => {
        list.removeAttribute('tabindex');
      }, { once: true });
    } else {
      // Option 2: Focus main content
      const main = document.getElementById('main-content');
      main.setAttribute('tabindex', '-1');
      main.focus();
    }
  }
}
```

## Focus Management for SPA and Dynamic Content

### Client-Side Page Transitions

When Hugo sites use client-side navigation libraries (Turbo, htmx, Barba.js), the page content replaces without a full browser reload. The browser does not reset focus after a client-side transition, so focus may land on a now-removed element or stay at the previous location.

```js
// Turbo: manage focus after page load
document.addEventListener('turbo:load', () => {
  const main = document.getElementById('main-content');
  if (main) {
    main.setAttribute('tabindex', '-1');
    main.focus({ preventScroll: false });
    main.addEventListener('blur', () => {
      main.removeAttribute('tabindex');
    }, { once: true });
  }
});

// htmx: manage focus after content swap
document.addEventListener('htmx:afterSwap', (event) => {
  const target = event.detail.target;
  const heading = target.querySelector('h1, h2');
  if (heading) {
    heading.setAttribute('tabindex', '-1');
    heading.focus();
    heading.addEventListener('blur', () => {
      heading.removeAttribute('tabindex');
    }, { once: true });
  }
});
```

### Focus After Content Filtering

When search or filter operations replace a list of results, announce the change via aria-live and keep focus in the search field. Do not move focus to the results.

```html
<label for="filter-input">Filter articles</label>
<input type="search" id="filter-input" aria-describedby="filter-status">

<div id="filter-status" role="status" aria-live="polite" aria-atomic="true">
  <!-- Updated by JavaScript: "8 articles match your filter" -->
</div>

<ul id="article-list" aria-label="Filtered articles">
  <!-- Results populated by JavaScript -->
</ul>
```

```js
const filterInput = document.getElementById('filter-input');
const statusRegion = document.getElementById('filter-status');

filterInput.addEventListener('input', () => {
  const results = filterArticles(filterInput.value);
  renderResults(results);

  // Announce count via live region -- do NOT move focus
  statusRegion.textContent = `${results.length} articles match your filter`;
});
```

### Focus After Accordion Expansion

When an accordion panel expands, focus can remain on the trigger button (user expectation for accordions) or move to the panel content if the panel is long and scrolling is required.

```js
function toggleAccordion(trigger) {
  const panelId = trigger.getAttribute('aria-controls');
  const panel = document.getElementById(panelId);
  const isExpanded = trigger.getAttribute('aria-expanded') === 'true';

  trigger.setAttribute('aria-expanded', String(!isExpanded));
  panel.hidden = isExpanded;

  // Focus stays on the trigger -- standard accordion behavior
  // Screen reader announces "expanded" or "collapsed" state
}
```

## Hugo Templates with Focus Management

### Base Layout with Focus Targets

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

  {{/* tabindex="-1" allows skip link and SPA transitions to focus main */}}
  <main id="main-content" tabindex="-1">
    {{ block "main" . }}{{ end }}
  </main>

  <footer>
    {{ partial "site-footer.html" . }}
  </footer>

  {{ partial "dialog.html" . }}

</body>
</html>
```

### Dialog Partial with Focus Management

**layouts/partials/dialog.html:**

```go-html-template
{{/*
  Reusable dialog partial.
  JavaScript handles focus trapping and restoration.
  Include once in baseof.html -- content is populated dynamically.
*/}}
<div id="site-dialog"
     role="dialog"
     aria-modal="true"
     aria-labelledby="dialog-title"
     tabindex="-1"
     hidden>
  <div class="dialog-content">
    <h2 id="dialog-title">Dialog</h2>
    <div id="dialog-body">
      {{/* Content injected by JavaScript */}}
    </div>
    <div class="dialog-actions">
      <button id="dialog-action" class="btn-primary">Confirm</button>
      <button id="dialog-close" class="btn-secondary">Cancel</button>
    </div>
  </div>
</div>
<div class="dialog-overlay" id="dialog-overlay" hidden></div>
```

### Mobile Navigation with Focus Management

**layouts/partials/mobile-nav.html:**

```go-html-template
<div class="mobile-nav">
  <button
    id="mobile-menu-toggle"
    aria-expanded="false"
    aria-controls="mobile-menu"
    aria-label="Open navigation menu">
    <svg aria-hidden="true" focusable="false" class="icon-menu">
      <use href="#icon-hamburger"></use>
    </svg>
    <svg aria-hidden="true" focusable="false" class="icon-close" hidden>
      <use href="#icon-close"></use>
    </svg>
  </button>

  <nav id="mobile-menu" aria-label="Mobile navigation" hidden>
    <ul>
      {{ range site.Menus.main }}
        <li>
          <a href="{{ .URL }}"
            {{ if eq $.RelPermalink .URL }}aria-current="page"{{ end }}>
            {{ .Name }}
          </a>
        </li>
      {{ end }}
    </ul>
  </nav>
</div>
```

```js
document.addEventListener('DOMContentLoaded', () => {
  const toggle = document.getElementById('mobile-menu-toggle');
  const menu = document.getElementById('mobile-menu');
  if (!toggle || !menu) return;

  const iconMenu = toggle.querySelector('.icon-menu');
  const iconClose = toggle.querySelector('.icon-close');

  toggle.addEventListener('click', () => {
    const isExpanded = toggle.getAttribute('aria-expanded') === 'true';
    setMenuState(!isExpanded);
  });

  menu.addEventListener('keydown', (event) => {
    if (event.key === 'Escape') {
      setMenuState(false);
      toggle.focus();
    }
  });

  function setMenuState(open) {
    toggle.setAttribute('aria-expanded', String(open));
    toggle.setAttribute('aria-label', open ? 'Close navigation menu' : 'Open navigation menu');
    menu.hidden = !open;

    if (iconMenu && iconClose) {
      iconMenu.hidden = open;
      iconClose.hidden = !open;
    }

    if (open) {
      // Focus the first link in the menu
      const firstLink = menu.querySelector('a');
      if (firstLink) firstLink.focus();
    }
  }
});
```

## CSS for Focus Indicators

### Complete Focus Stylesheet

```css
/*
 * Focus indicator styles for Hugo sites.
 * Include in the global stylesheet.
 * Covers all interactive elements with WCAG-compliant indicators.
 */

/* === Base Focus Indicator === */

/* Fallback for browsers without :focus-visible support */
:focus {
  outline: 3px solid var(--color-focus-ring, #0056b3);
  outline-offset: 2px;
}

/* Remove outline for mouse users in supporting browsers */
:focus:not(:focus-visible) {
  outline: none;
}

/* Keyboard-only focus indicator */
:focus-visible {
  outline: 3px solid var(--color-focus-ring, #0056b3);
  outline-offset: 2px;
}

/* === Element-Specific Overrides === */

/* Text inputs: always show focus (not just :focus-visible) */
input:focus,
textarea:focus,
select:focus {
  outline: 3px solid var(--color-focus-ring, #0056b3);
  outline-offset: 0;
  border-color: var(--color-focus-ring, #0056b3);
}

/* Buttons */
button:focus-visible,
[role="button"]:focus-visible {
  outline: 3px solid var(--color-focus-ring, #0056b3);
  outline-offset: 2px;
}

/* Links */
a:focus-visible {
  outline: 3px solid var(--color-focus-ring, #0056b3);
  outline-offset: 2px;
  border-radius: 2px;
}

/* Navigation links: inset outline */
nav a:focus-visible {
  outline-offset: -3px;
}

/* Skip link: visible on focus */
.skip-link {
  position: absolute;
  top: -100%;
  left: 0.5rem;
  z-index: 10000;
  padding: 0.75rem 1.5rem;
  background: var(--color-focus-ring, #0056b3);
  color: #ffffff;
  text-decoration: none;
  font-weight: bold;
  border-radius: 0 0 4px 4px;
}

.skip-link:focus {
  position: fixed;
  top: 0;
}

/* === Dark Mode Adjustments === */

@media (prefers-color-scheme: dark) {
  :focus-visible {
    outline-color: var(--color-focus-ring, #64b5f6);
  }

  /* Double ring for visibility on dark backgrounds */
  button:focus-visible,
  a:focus-visible,
  [role="button"]:focus-visible {
    outline: 2px solid #ffffff;
    outline-offset: 2px;
    box-shadow: 0 0 0 4px var(--color-focus-ring, #64b5f6);
  }
}

/* === Dialog Focus === */

[role="dialog"]:focus {
  outline: none;
  /* Dialog container does not need a visible ring */
  /* Focus is managed by the content inside */
}

/* === Disabled State === */

[disabled],
[aria-disabled="true"] {
  cursor: not-allowed;
  opacity: 0.5;
}

[disabled]:focus,
[aria-disabled="true"]:focus {
  outline: none;
  /* Disabled elements should not show focus indicators */
}
```

## Testing Focus Management

### Manual Testing Checklist

```
1. Focus Visibility
   [ ] Every focusable element has a visible focus indicator
   [ ] Focus indicator has at least 3:1 contrast against adjacent colors
   [ ] Focus indicator is visible on both light and dark backgrounds
   [ ] :focus-visible hides ring on mouse click for buttons and links
   [ ] Text inputs always show focus ring (even on mouse click)

2. Tab Order
   [ ] Tab moves through all interactive elements in logical visual order
   [ ] Shift+Tab moves backward through the same elements
   [ ] No elements receive focus unexpectedly (hidden elements, decorative content)
   [ ] Skip link is first focusable element on the page

3. Modal Dialogs
   [ ] Focus moves into the modal when it opens
   [ ] Tab cycles through elements inside the modal only
   [ ] Shift+Tab wraps from first to last element inside the modal
   [ ] Escape closes the modal
   [ ] Focus returns to the triggering element after close
   [ ] Background content is inert (not focusable or operable)

4. Dynamic Content
   [ ] Focus moves to new content when relevant (expanded accordion, loaded page)
   [ ] Focus stays in the search field when results update
   [ ] aria-live regions announce changes without moving focus
   [ ] Client-side navigation moves focus to main content area

5. Focus Restoration
   [ ] Closing a dropdown returns focus to the trigger button
   [ ] Closing a modal returns focus to the trigger element
   [ ] Dismissing a popover returns focus to the trigger
   [ ] If trigger is removed, focus moves to nearest logical element

6. Edge Cases
   [ ] Focus is not trapped in non-modal components
   [ ] SVG icons inside buttons do not create extra tab stops
   [ ] Programmatically focused elements have tabindex="-1"
   [ ] tabindex="-1" is removed after blur to avoid persistent tab stops
```

### Automated Testing

```bash
# axe-core: check for focus-related violations
npx axe http://localhost:1313/ --tags "focus" "keyboard"

# pa11y: WCAG 2.2 focus appearance checks
npx pa11y http://localhost:1313/ --standard WCAG2AA
```

## Best Practices

**DO:**
- Provide visible focus indicators on every interactive element with at least 3:1 contrast
- Use `:focus-visible` for buttons and links to distinguish keyboard from mouse focus
- Use `:focus` (not just `:focus-visible`) on text inputs so the active field is always clear
- Use the double-ring pattern (light inner, dark outer) for focus visible on any background
- Move focus into modal dialogs when they open and trap Tab within them
- Return focus to the triggering element when closing any overlay (modal, dropdown, popover)
- Use `tabindex="-1"` on non-interactive elements that need programmatic focus
- Remove `tabindex="-1"` after blur to avoid adding elements permanently to the tab order
- Use the `inert` attribute on background content when a modal is open
- Move focus to `<main>` after client-side page transitions
- Announce dynamic content changes via `aria-live` instead of moving focus when the user is mid-task

**DON'T:**
- Remove focus outlines globally with `*:focus { outline: none }` without providing replacements
- Use thin (1px) or low-contrast focus indicators that fail the 3:1 requirement
- Forget to restore focus when closing modals, dropdowns, or popovers
- Move focus to search results while the user is still typing
- Trap focus in non-modal components (only modals should trap focus)
- Use positive `tabindex` values (1, 2, 3) which create unpredictable tab order
- Leave `tabindex="-1"` on elements permanently after programmatic focus
- Create focus indicators that only change background color (hard to perceive)
- Assume the `<dialog>` element handles all edge cases without testing
- Skip focus management testing because automated tools do not catch focus flow issues

## Guidelines

### Essential

- Every focusable element has a visible focus indicator meeting 3:1 contrast (WCAG 2.4.11)
- `:focus-visible` used for buttons and links; `:focus` used for form inputs
- Focus never removed with `outline: none` without a visible replacement
- Modal dialogs trap focus and restore it on close
- Skip link target (`<main>`) has `tabindex="-1"` for programmatic focus
- No positive `tabindex` values in the codebase

### Recommended

- Double-ring focus pattern used for universal visibility across backgrounds
- `inert` attribute applied to background content when modal is open
- Mobile menu moves focus to first link on open and restores on Escape
- Client-side navigation libraries move focus to `<main>` after page transition
- Focus indicator CSS uses CSS custom properties for theme consistency
- Accordion triggers keep focus (do not move it to expanded content)
- Dynamic content changes announced via `aria-live` without moving focus

### Advanced

- Native `<dialog>` element used with `.showModal()` for built-in focus management
- `createFocusTrap()` utility shared across all modal and overlay components
- Automated focus flow testing in CI/CD that verifies tab order and focus restoration
- Focus management for web components using Shadow DOM and delegatesFocus
- Programmatic focus cleanup: `tabindex="-1"` removed on blur via one-time event listener
- Focus indicator styles tested in Chrome, Firefox, and Safari for rendering consistency

## Benefits

Clear Focus Location. Visible focus indicators ensure keyboard users always know which element will respond to their next action.

Predictable Navigation. Deliberate focus management after content changes prevents the user from losing their place on the page.

Safe Modal Interaction. Focus trapping in dialogs prevents keyboard users from accidentally interacting with obscured background content.

Smooth Transitions. Moving focus to main content after client-side navigation gives keyboard users an experience equivalent to a full page reload.

Reduced Cognitive Load. Focus restoration after closing overlays returns users to their previous context without requiring re-navigation.

Theme-Resilient Indicators. The double-ring pattern and CSS custom property integration ensure focus indicators remain visible across light and dark themes.

## Related

- [keyboard-navigation.md](./keyboard-navigation.md) - Tab order, key events, and roving tabindex patterns
- [color-contrast-wcag.md](./color-contrast-wcag.md) - Contrast requirements for focus indicator colors
- [skip-navigation-links.md](./skip-navigation-links.md) - Skip links that depend on focus targets in the page
